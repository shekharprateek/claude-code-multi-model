# Testing Plan: Remove FAISS from the codebase

*Created: 2026-07-15*
*Related LLD: `./lld.md`*
*Related Issue: `./github-issue.md`*

## Overview

### Scope of Testing
This plan verifies that FAISS is fully removed from the codebase, that DocumentDB hybrid search replaces all FAISS search behavior, and that no backwards-compatible search regressions are introduced.

### Prerequisites
- [ ] Repository checked out at tag `1.24.4`.
- [ ] `uv` is installed and the virtual environment is up to date.
- [ ] MongoDB/DocumentDB is running for integration tests (single-node MongoDB CE on `localhost:27017` is sufficient).
- [ ] Test environment sets `DOCUMENTDB_HOST=localhost` and `STORAGE_BACKEND=mongodb-ce`.

### Shared Variables

```bash
export REPO_ROOT="$(git rev-parse --show-toplevel)"
export REPO_DIR="$REPO_ROOT/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo"
export REGISTRY_URL="http://localhost"
```

## 1. Functional Tests

### 1.1 Dependency removal

```bash
cd "$REPO_DIR"

# Confirm faiss-cpu is absent from pyproject.toml
grep -i "faiss-cpu" pyproject.toml && echo "FAIL: faiss-cpu still in pyproject.toml" || echo "PASS: faiss-cpu removed from pyproject.toml"

# Confirm faiss-cpu is absent from uv.lock
grep -i '"faiss-cpu"' uv.lock && echo "FAIL: faiss-cpu still in uv.lock" || echo "PASS: faiss-cpu removed from uv.lock"

# Confirm no Python import of faiss
grep -R "import faiss" registry/ cli/ metrics-service/ && echo "FAIL: import faiss found" || echo "PASS: no import faiss"

# Confirm no faiss string literals in source (excluding tests/docs/scripts, which are checked separately)
grep -R "faiss" registry/ --include="*.py" && echo "FAIL: faiss string found in registry source" || echo "PASS: no faiss string in registry source"
```

### 1.2 Factory returns DocumentDB search repository

```bash
cd "$REPO_DIR"
uv run python - <<'PY'
import asyncio
from registry.core.config import settings
settings.storage_backend = "file"  # even file backend must route search to DocumentDB
from registry.repositories.factory import get_search_repository, reset_repositories
reset_repositories()
repo = get_search_repository()
assert repo.__class__.__name__ == "DocumentDBSearchRepository", f"unexpected: {repo.__class__.__name__}"
print("PASS: get_search_repository() returns DocumentDBSearchRepository for file backend")
PY
```

### 1.3 Deleted files

```bash
cd "$REPO_DIR"
for f in \
  "registry/search/service.py" \
  "registry/repositories/file/search_repository.py" \
  "tests/fixtures/mocks/mock_faiss.py" \
  "tests/unit/search/test_faiss_service.py"; do
  if [ -f "$f" ]; then echo "FAIL: $f still exists"; else echo "PASS: $f deleted"; fi
done
```

### 1.4 Semantic search via DocumentDB

Start the registry with `STORAGE_BACKEND=mongodb-ce` and run:

```bash
export ACCESS_TOKEN=$(jq -r '.access_token' .oauth-tokens/ingress.json)

curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "database server", "max_results": 5}' | jq .
```

Expected:
- HTTP 200.
- Response contains `servers`, `tools`, `agents` arrays.
- Results include relevant servers/tools based on DocumentDB hybrid search.

### 1.5 Tag-only search via DocumentDB

```bash
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tags": ["database"], "max_results": 5}' | jq .
```

Expected:
- HTTP 200.
- Results contain entities with the `database` tag.

### 1.6 Server lifecycle updates search index

```bash
# Register a server
curl -s -X POST "$REGISTRY_URL/api/v1/servers" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"server_name": "faiss-removal-test", "description": "test server for faiss removal", "tags": ["test"], "url": "http://example.com/sse"}' | jq .

# Search for it
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "faiss removal test", "max_results": 5}' | jq '.servers[] | select(.server_name == "faiss-removal-test")'

# Delete it
curl -s -X DELETE "$REGISTRY_URL/api/v1/servers/faiss-removal-test" \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Confirm it no longer appears in search
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "faiss removal test", "max_results": 5}' | jq '.servers[] | select(.server_name == "faiss-removal-test")' && echo "FAIL: server still searchable" || echo "PASS: server removed from search"
```

## 2. Backwards Compatibility Tests

### 2.1 Search request shape

```bash
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "database", "entity_types": ["mcp_server"], "max_results": 10}' | jq '.servers | length'
```

Expected: same request shape as before returns results without requiring new fields.

### 2.2 Search response schema

```bash
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "database", "max_results": 5}' | jq 'keys'
```

Expected: response keys remain `servers`, `tools`, `agents` (and any other pre-existing keys).

### 2.3 Telemetry heartbeat backend

If authenticated to the telemetry endpoint:

```bash
curl -s "$REGISTRY_URL/api/v1/telemetry/heartbeat" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | jq '.search_backend'
```

Expected: `"documentdb"`.

### 2.4 Configuration properties removed

```bash
cd "$REPO_DIR"
uv run python - <<'PY'
from registry.core.config import settings
assert not hasattr(settings, "faiss_index_path"), "faiss_index_path should be removed"
assert not hasattr(settings, "faiss_metadata_path"), "faiss_metadata_path should be removed"
print("PASS: FAISS config properties removed")
PY
```

## 3. UX Tests

**Not Applicable** - This change does not add or modify any UI surface. Search endpoints and CLI behavior remain unchanged.

## 4. Deployment Surface Tests

### 4.1 Docker wiring

```bash
cd "$REPO_DIR"
for f in docker/Dockerfile.registry docker/Dockerfile.registry-cpu; do
  if [ -f "$f" ]; then
    grep -i "faiss" "$f" && echo "FAIL: FAISS reference in $f" || echo "PASS: no FAISS in $f"
  fi
done
```

### 4.2 Docker Compose wiring

```bash
cd "$REPO_DIR"
for f in docker-compose.yml docker-compose.podman.yml docker-compose.prebuilt.yml; do
  grep -i "faiss" "$f" && echo "FAIL: FAISS reference in $f" || echo "PASS: no FAISS in $f"
done
```

### 4.3 Build script wiring

```bash
cd "$REPO_DIR"
grep -i "faiss" build_and_run.sh && echo "FAIL: FAISS reference in build_and_run.sh" || echo "PASS: no FAISS in build_and_run.sh"
grep -i "faiss" build-config.yaml && echo "FAIL: FAISS reference in build-config.yaml" || echo "PASS: no FAISS in build-config.yaml"
```

### 4.4 Terraform wiring

```bash
cd "$REPO_DIR/terraform/aws-ecs"
grep -i "faiss" scripts/service_mgmt.sh && echo "FAIL: FAISS reference in service_mgmt.sh" || echo "PASS: no FAISS in service_mgmt.sh"
grep -i "faiss" OPERATIONS.md && echo "FAIL: FAISS reference in OPERATIONS.md" || echo "PASS: no FAISS in OPERATIONS.md"
cd "$REPO_DIR/terraform/telemetry-collector"
grep -i "faiss" lambda/collector/schemas.py && echo "FAIL: FAISS reference in schemas.py" || echo "PASS: no FAISS in schemas.py"
```

### 4.5 Build and smoke test

```bash
cd "$REPO_DIR"
# This is a build-only check; do not start services if DocumentDB is unavailable.
# If you have Docker available:
# docker build -f docker/Dockerfile.registry -t mcp-registry-no-faiss .
# Verify the image builds without faiss-cpu installation errors.
```

## 5. End-to-End API Tests

### 5.1 Full server registration and discovery workflow

```bash
export ACCESS_TOKEN=$(jq -r '.access_token' .oauth-tokens/ingress.json)

# Register a server with tools
curl -s -X POST "$REGISTRY_URL/api/v1/servers" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "server_name": "e2e-search-server",
    "description": "End-to-end semantic search test server",
    "tags": ["e2e", "search"],
    "url": "http://example.com/sse",
    "tool_list": [
      {"name": "query_database", "description": "Query a relational database"}
    ]
  }' | jq .

# Discover by natural language
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "relational database query", "max_results": 5}' | jq '.servers[] | select(.server_name == "e2e-search-server")'

# Discover by tool name
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "query_database", "entity_types": ["tool"], "max_results": 5}' | jq '.tools[] | select(.tool_name == "query_database")'

# Clean up
curl -s -X DELETE "$REGISTRY_URL/api/v1/servers/e2e-search-server" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Expected: server and tool are discoverable via DocumentDB hybrid search and are removed after deletion.

### 5.2 Agent registration and discovery workflow

If A2A agent routes are enabled:

```bash
# Register an agent
curl -s -X POST "$REGISTRY_URL/api/v1/agents" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "e2e-search-agent",
    "description": "End-to-end agent search test",
    "tags": ["e2e", "agent"],
    "url": "http://example.com/agent"
  }' | jq .

# Search for agent
curl -s -X POST "$REGISTRY_URL/api/v1/semantic" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "agent search test", "entity_types": ["a2a_agent"], "max_results": 5}' | jq '.agents[] | select(.name == "e2e-search-agent")'

# Clean up
curl -s -X DELETE "$REGISTRY_URL/api/v1/agents/e2e-search-agent" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

## 6. Test Execution Checklist

- [ ] Section 1.1 (dependency removal) passes
- [ ] Section 1.2 (factory returns DocumentDB repo) passes
- [ ] Section 1.3 (deleted files) passes
- [ ] Section 1.4 (semantic search) passes
- [ ] Section 1.5 (tag-only search) passes
- [ ] Section 1.6 (server lifecycle) passes
- [ ] Section 2.1 (search request shape) passes
- [ ] Section 2.2 (search response schema) passes
- [ ] Section 2.3 (telemetry heartbeat) passes
- [ ] Section 2.4 (config properties removed) passes
- [ ] Section 4.1-4.4 (deployment surface) passes
- [ ] Section 5.1 (E2E server workflow) passes
- [ ] Section 5.2 (E2E agent workflow) passes or is skipped if A2A is disabled
- [ ] Unit tests updated under `tests/unit/`
- [ ] Integration tests updated under `tests/integration/`
- [ ] `uv run pytest tests/` passes with no regressions
