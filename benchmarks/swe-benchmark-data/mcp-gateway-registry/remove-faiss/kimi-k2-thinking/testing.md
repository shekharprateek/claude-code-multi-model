# Testing Plan: Remove FAISS from the Codebase

*Created: 2026-06-15*
*Related LLD: `./lld.md`*
*Related Issue: `./github-issue.md`*

## Overview

### Scope of Testing
This testing plan covers the complete removal of FAISS from the MCP Gateway Registry and migration to lexical search functionality. Testing must verify:
- No FAISS dependencies remain
- Search API continues to work with lexical matching
- Backward compatibility for API contracts
- Documentation accuracy
- Deployment configuration correctness

### Prerequisites
- [ ] Development environment running at `http://localhost:8000`
- [ ] New build: `docker build -t mcp-gateway-registry:no-faiss .`
- [ ] Original test fixtures loaded
- [ ] Reference data: 500+ mock servers/agents

### Shared Variables
```bash
export REGISTRY_URL="http://localhost:8000"
export API_KEY="test-key-12345"  # From .env
export MCP_HOME="~/.mcp"
```

## 1. Functional Tests

### 1.1 Build and Dependency Tests

#### Test 1.1.1: FAISS Dependency Removed
**Purpose:** Verify faiss-cpu is no longer a dependency

**Command:**
```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo
grep -i "faiss" pyproject.toml
```

**Expected Result:**
- Exit code 1 (no matches)
- No output

**Verification:**
```bash
if ! grep -i "faiss" pyproject.toml; then
    echo "✅ PASS: FAISS dependency removed"
else
    echo "❌ FAIL: FAISS still in dependencies"
    exit 1
fi
```

#### Test 1.1.2: FAISS Import Removed
**Purpose:** Verify no Python imports of faiss

**Command:**
```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo
find . -type f -name "*.py" -exec grep -l "import faiss\|from faiss" {} \;
```

**Expected Result:**
- No Python files should import faiss
- Exit code 1, no output

**Command:**
```bash
! grep -r "import faiss\|from faiss" --include="*.py" registry/
```

#### Test 1.1.3: Service Build Success
**Purpose:** Verify image builds after FAISS removal

**Command:**
```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo
docker build -t mcp-gateway-test:no-faiss . 2>&1 | tee build.log
```

**Expected Result:**
- Build completes successfully
- No errors related to missing faiss modules
- Image size reduced (should see ~50MB reduction)

**Verify:**
```bash
if docker images mcp-gateway-test:no-faiss -q; then
    echo "✅ PASS: Build successful"
    SIZE=$(docker images mcp-gateway-test:no-faiss --format "{{.Size}}")
    echo "Image size: $SIZE"
else
    echo "❌ FAIL: Build failed"
    exit 1
fi
```

### 1.2 API Functional Tests

#### Test 1.2.1: Search Servers Endpoint Works
**Purpose:** Verify /servers/search endpoint returns results

**Setup:**
```bash
# Prerequisites: Registry running with test data
curl -X POST "${REGISTRY_URL}/servers/servers/connect" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"url": "http://localhost:8000/test-server"}'
```

**Command:**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=database" \
  -H "Authorization: Bearer ${API_KEY}" | jq .
```

**Expected Output:**
```json
{
  "results": [
    {
      "service_path": "test-server",
      "server_info": {
        "server_name": "Database Server",
        "description": "PostgreSQL database MCP server",
        "tags": ["database", "postgres", "sql"]
      },
      "score": 0.95  // Changed from vector similarity
    }
  ],
  "count": 1,
  "search_mode": "lexical"
}
```

**Assertions:**
- ✅ HTTP 200 response
- ✅ `search_mode` == "lexical" (critical)
- ✅ `results` array exists
- ✅ Each result has `service_path`, `server_info`, `score`
- ✅ `score` is float between 0.0-1.0
- ✅ Non-empty results for common queries

#### Test 1.2.2: Search Agents Endpoint Works
**Purpose:** Verify /agents/search endpoint returns results

**Command:**
```bash
curl -G "${REGISTRY_URL}/agents/search" \
  --data-urlencode "q=data analyzer" \
  -H "Authorization: Bearer ${API_KEY}" | jq .
```

**Expected Output:**
```json
{
  "results": [
    {
      "service_path": "data-analyzer-agent",
      "agent_info": {
        "name": "Data Analyzer",
        "description": "Analyzes data patterns",
        "tags": ["data", "analysis", "ml"]
      },
      "score": 0.85,
      "type": "a2a_agent"
    }
  ],
  "count": 1,
  "search_mode": "lexical"
}
```

#### Test 1.2.3: Keyword Matching Quality
**Purpose:** Lexical search matches keywords correctly

**Test Cases:**

**Case 1: Exact name match**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=PostgreSQL Database" | jq '.results[0].server_info.server_name'
# Expect: "PostgreSQL Database"
```

**Case 2: Partial match in description**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=postgres" | jq '.results[].score'
# Expect: All scores > 0.5
```

**Case 3: Tag search**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=database" | jq '.results[].server_info.tags'
# Expect: "database" tag present in results
```

**Case 4: Tool name search**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=query" | jq '.results[].server_info.tool_list[].name'
# Expect: tool names with "query" in them
```

#### Test 1.2.4: Empty Query Handling
**Purpose:** Edge case: empty query

**Command:**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=" \
  -H "Authorization: Bearer ${API_KEY}" -w "%{http_code}\n"
```

**Expected:** HTTP 200 with empty results array

#### Test 1.2.5: No Results Response
**Purpose:** Correct handling when no matches

**Command:**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=xyzzy-no-match-12345" \
  -H "Authorization: Bearer ${API_KEY}" | jq .
```

**Expected:**
```json
{
  "results": [],
  "count": 0,
  "search_mode": "lexical"
}
```

### 1.3 CLI Integration Tests

#### Test 1.3.1: CLI Search Command Works
**Purpose:** Verify CLI performs lexical search

**Command:**
```bash
# Build CLI
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo && make build-cli

# Run search
~/.local/bin/mcp-search "database connection" --limit 10 2>&1
```

**Expected Output:**
```
Searching for "database connection" ...
Found 3 results:

1. PostgreSQL Server (score: 0.95)
   Description: PostgreSQL database MCP server
   Tags: database, postgres, sql

2. MySQL Connector (score: 0.82)
   Description: MySQL database connection MCP server
   Tags: database, mysql, sql

3. Database Manager (score: 0.75)
   Description: Generic database management utilities
   Tags: database, tools
```

**Assertions:**
- CLI exits with code 0
- Results displayed with scores
- Shows name, description, and tags

#### Test 1.3.2: CLI Shows Search Mode
**Purpose:** CLI indicates search type

**Command:**
```bash
~/.local/bin/mcp-search "test" --verbose 2>&1 | grep -i "lexical\|keyword"
```

**Expected:** Output indicates "lexical" or "keyword" search mode

### 1.4 Configuration Validations

#### Test 1.4.1: No Embeddings Env Vars Required
**Purpose:** Registry starts without embeddings configuration

**Steps:**
```bash
# Clear any embeddings env vars
unset EMBEDDINGS_PROVIDER
unset EMBEDDINGS_MODEL_NAME
unset EMBEDDINGS_API_KEY

# Start registry
docker run -p 8000:8000 \
  -e MCP_API_KEY=test-key \
  mcp-gateway-test:no-faiss
```

**Expected:**
- Container starts successfully
- No errors about missing embeddings configuration
- Health check passes

**Command:**
```bash
curl -s "${REGISTRY_URL}/health" | jq '.status'
# Expected: "healthy"
```

#### Test 1.4.2: Lexical Search Config Recognized
**Purpose:** New lexical configuration used

**Steps:**
```bash
# Start with lexical configuration
docker run -p 8000:8000 \
  -e MCP_API_KEY=test-key \
  -e LEXICAL_SEARCH_FIELDS="[\"name\",\"description\",\"tools\"]" \
  -e LEXICAL_FUZZY_MATCH=true \
  mcp-gateway-test:no-faiss
```

**Verification:**
```bash
# Search should work with new config
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=postgres" | jq '.search_mode'
# Expected: "lexical"
```

## 2. Backwards Compatibility Tests

### 2.1 API Contract Compatibility

#### Test 2.1.1: Response Schema Preserved
**Purpose:** Existing clients don't break

**Requirements:** Response fields unchanged

**Failure Cases to Test:**

```bash
# Test 1: Missing search_mode field
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=test" | jq 'has("search_mode")'
# Expected: true

# Test 2: Missing results array
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=test" | jq 'has("results")'
# Expected: true

# Test 3: Missing count field
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=test" | jq 'has("count")'
# Expected: true
```

**Client Compatibility Test:**
```bash
# Simulate old client expecting similarity scores
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=database" | jq '.results[0].relevance'
# Expected: Valid float (same as before, different algorithm)
```

#### Test 2.1.2: No New Required Parameters
**Purpose:** Old clients can call API without sending new params

**Command:**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  -H "Authorization: Bearer ${API_KEY}" \
  --data-urlencode "q=database" | jq '.results | length'
```

**Expected:** Works without any new required query parameters

### 2.2 Endpoint URL Compatibility

#### Test 2.2.1: Same Search Endpoints Work
**Purpose:** Verify URL paths unchanged

```bash
# These endpoints must work (no URL changes)
endpoints=(
  "${REGISTRY_URL}/servers/search"
  "${REGISTRY_URL}/agents/search"
)

for endpoint in "${endpoints[@]}"; do
  http_code=$(curl -s -o /dev/null -w "%{http_code}" "${endpoint}?q=test")
  if [ "$http_code" -eq 200 ]; then
    echo "✅ PASS: $endpoint"
  else
    echo "❌ FAIL: $endpoint (HTTP $http_code)"
  fi
done
```

### 2.3 Behavior Compatibility

#### Test 2.3.1: Search Returns Results (No Functional Regression)
**Purpose:** Basic search still works for users

```bash
# Register a test server
curl -X POST "${REGISTRY_URL}/servers/servers/connect" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://example.com/test",
    "name": "Database Server",
    "description": "PostgreSQL database MCP server",
    "tags": ["database", "sql"]
  }'

# Should find it
response=$(curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=database" | jq '.count')

if [ "$response" -gt 0 ]; then
  echo "✅ PASS: Search returns results"
else
  echo "❌ FAIL: Search returns no results"
fi
```

**Acceptance:** Pass if count > 0

#### Test 2.3.2: CLI Command Compatibility
**Purpose:** CLI interface unchanged

```bash
# Old command should still work (no flag changes expected)
~/.local/bin/mcp-search "test" --limit=5 --output=json 2>&1 | jq '.results | length'

# Expected: Valid number (no crashes)
```

### 2.4 Response Field Compatibility

#### Test 2.4.1: Relevance Score Field Maintained
**Purpose:** Score field exists (though algorithm changed)

```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=database" | jq '.results[0] | has("score")'

# Expected: true
```

**Backward Compatibility Note:** Old clients expecting `relevance` field may break if field name changed. Verify field name consistency.

## 3. UX Tests

### 3.1 Error Message Clarity

#### Test 3.1.1: No Embeddings Error Messages
**Purpose:** No confusing FAISS-related errors to users

**Steps:**
1. Start registry without any embeddings config
2. Check logs: `docker logs <container_id>`

**Negative Test:**
```bash
# Check that old error messages don't appear
docker logs <container_id> 2>&1 | grep -i "faiss\|embedding\|model" | grep -i "error\|warn"

# Expected: No output (no FAISS errors/errors)
```

#### Test 3.1.2: Help Text Updated
**Purpose:** CLI shows up-to-date help

**Command:**
```bash
~/.local/bin/mcp-search --help | grep -i "semantic\|faiss\|vector"
# Should NOT find "semantic", "FAISS", or "vector"

~/.local/bin/mcp-search --help | grep -i "keyword\|lexical"
# Should find "keyword" or "lexical"
```

**Expected:** Help text reflects keyword-based search

### 3.2 Search Mode Transparency

#### Test 3.2.1: API Indicates Search Mode
**Purpose:** Users can identify search type from API response

**Command:**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=test" | jq '.search_mode'
```

**Expected:** `"lexical"`

#### Test 3.2.2: CLI Indicates Search Mode (Verbose)
**Purpose:** Verbose mode shows search details

**Command:**
```bash
~/.local/bin/mcp-search "test" --verbose 2>&1 | grep -i "mode\|keyword\|lexical"
```

**Expected:** Output mentions keyword-based search

### 3.3 Score Display Usability

#### Test 3.3.1: Scores Are Intuitive
**Purpose:** Users can understand relevance scores

**Test:**
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=postgres" | jq '.results[] | select(.score > 0.9) | .server_name'

# Should show exact matches first
```

**Acceptance:**
- Exact matches (e.g., name = query) should have scores > 0.9
- Partial matches 0.5-0.7
- Low relevance < 0.3

#### Test 3.3.2: CLI Score Display
**Purpose:** CLI shows interpretable scores

```bash
~/.local/bin/mcp-search "database" --limit 3

# Expected: Shows scores like "0.95" not "0.9532478923"
# Scores should be intuitive (higher = better match)
```

## 4. Deployment Surface Tests

### 4.1 Docker Configuration Tests

#### Test 4.1.1: No FAISS Volumes in Docker Compose
**Purpose:** Docker compose doesn't mount FAISS directories

**Command:**
```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo
grep -i "faiss\|.faiss\|sentence-transformers" docker-compose.yml
```

**Expected:** No matches

#### Test 4.1.2: Container Starts Without FAISS
**Purpose:** No startup errors related to FAISS

```bash
docker-compose up -d registry
sleep 10  # Wait for startup

# Check health endpoint
curl -f "${REGISTRY_URL}/health" || echo "Health check failed"

# Check for FAISS errors in logs
docker-compose logs registry | grep -i "faiss\|embedding" | grep -i "error\|fail"
```

**Expected:** Container healthy, no FAISS errors in logs

#### Test 4.1.3: Image Size Reduction
**Purpose:** Verify image size decreased

```bash
# Build both variants
docker build -t mcp-gateway:before -f- . <<EOF
FROM mcp-gateway-registry:1.24.3
EOF

docker build -t mcp-gateway:after .

# Compare sizes
SIZE_BEFORE=$(docker images mcp-gateway-registry:1.24.3 --format "{{.Size}}")
SIZE_AFTER=$(docker images mcp-gateway:after --format "{{.Size}}")

echo "Before: $SIZE_BEFORE"
echo "After:  $SIZE_AFTER"

# Expected: After size smaller
```

**Acceptance:** 50-100MB reduction expected

### 4.2 Terraform Configuration Tests

#### Test 4.2.1: No Embeddings Variables in Terraform
**Purpose:** Terraform configs don't reference embeddings

```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/terraform/aws-ecs
grep -r "embeddings\|faiss\|EMBEDDING" --include="*.tf" --include="*.tfvars" .
```

**Expected:** No matches (all embeddings variables removed)

#### Test 4.2.2: Terraform Plan Success
**Purpose:** Infrastructure as code remains valid

```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/terraform/aws-ecs
terraform init -backend=false
terraform plan -var='mcp_gateway_version=no-faiss-test'
```

**Expected:** Terraform plan succeeds (validation passes)

⚠️ **Note:** Actual deployment test would require real AWS credentials

### 4.3 Helm / Kubernetes Configuration Tests

#### Test 4.3.1: No FAISS Volumes in Helm Charts
**Purpose:** Kubernetes configs don't mount FAISS directories

```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/charts
grep -r "faiss\|embeddings" --include="*.yaml" --include="*.yml" .
```

**Expected:** No matches

#### Test 4.3.2: Helm Template Validates
**Purpose:** Helm charts remain syntactically correct

```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/charts/mcp-gateway
helm template test . --set mcpGateway.apiKey=test-key --dry-run
```

**Expected:** Helm template renders without errors

### 4.4 Environment Variables Tests

#### Test 4.4.1: No Embeddings Env Vars Required
**Purpose:** Application runs without embeddings environment variables

```bash
docker run --rm -p 8000:8000 \
  -e MCP_API_KEY=test-key \
  -e DATABASE_URL=sqlite:///test.db \
  mcp-gateway-test:no-faiss &

# Wait for startup
sleep 5

# Verify health
curl -s "http://localhost:8000/health" | jq '.status'
```

**Expected:** Healthy, no startup errors

#### Test 4.4.2: Docker Compose Runs Without .env
**Purpose:** No reliance on embeddings in default config

```bash
# Backup .env
mv .env .env.backup

# Try to start
docker-compose up -d registry
sleep 10

# Test
curl -f "http://localhost:8000/health" || echo "FAILED"

# Restore
docker-compose down
mv .env.backup .env
```

**Expected:** Works without .env (no embeddings variables needed)

### 4.5 Rollback Verification

#### Test 4.5.1: Downgrade Path Exists
**Purpose:** Can roll back to previous version with FAISS

```bash
# Build previous version (tag 1.24.3)
docker pull mcp-gateway-registry:1.24.3

# Start previous version
docker run --rm -d -p 8001:8000 \
  -e MCP_API_KEY=test-key \
  mcp-gateway-registry:1.24.3 &

# Verify FAISS functionality exists
sleep 10
curl -s "http://localhost:8001/health" | jq -r '.features.faiss_enabled'

# Expected: true (old version has FAISS)
```

**Expected path exists:** Documented in deployment guide

## 5. End-to-End API Tests

### 5.1 User Journey: Server Registration and Search

**Scenario:** User registers multiple servers and searches across them

```bash
#!/bin/bash
# Setup: Clean state
rm -rf ~/.mcp/*.db

# Start registry
docker-compose up -d registry
sleep 10

BASE_URL="http://localhost:8000"

# Register servers
echo "Registering test servers..."

for i in {1..5}; do
  curl -X POST "${BASE_URL}/servers/connect" \
    -H "Authorization: Bearer test-key" \
    -H "Content-Type: application/json" \
    -d @- <<EOF
    {"url": "http://example.com/server$i", "name": "Database Server $i", "description": "PostgreSQL server", "tags": ["database", "postgres", "sql"]}
EOF
done

# Wait for indexing
echo "Waiting for indexing..."
sleep 2

# Search for "database"
echo "Searching for 'database'..."
RESULT=$(curl -s -G "${BASE_URL}/servers/search" \
  --data-urlencode "q=database" | jq '.count')

if [ "$RESULT" -eq 5 ]; then
  echo "✅ PASS: Found all 5 servers"
else
  echo "❌ FAIL: Expected 5, got $RESULT"
  exit 1
fi

# Search for "postgres"
RESULT=$(curl -s -G "${BASE_URL}/servers/search" \
  --data-urlencode "q=postgres" | jq '.count')

if [ "$RESULT" -eq 5 ]; then
  echo "✅ PASS: Tag-based search works"
else
  echo "❌ FAIL: Tag search failed"
  exit 1
fi

# Search for "mysql" (should not match database servers)
RESULT=$(curl -s -G "${BASE_URL}/servers/search" \
  --data-urlencode "q=mysql" | jq '.count')

if [ "$RESULT" -eq 0 ]; then
  echo "✅ PASS: Correctly excludes non-matching servers"
else
  echo "❌ FAIL: Found unexpected matches"
  exit 1
fi
```

### 5.2 Chain Operations: Search → Inspect → Delete → Verify

**Scenario:** Full CRUD with search verification

```bash
#!/bin/bash
# Register test entity
curl -X POST "${REGISTRY_URL}/servers/connect" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{"url": "http://test.com/test", "name": "Test Server"}'

# Wait for indexing
sleep 2

# Search and capture service_path
SERVICE_PATH=$(curl -s -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=test" | jq -r '.results[0].service_path')

echo "Found service: $SERVICE_PATH"

# Get details
DETAILS=$(curl -s "${REGISTRY_URL}/servers/${SERVICE_PATH}" \
  -H "Authorization: Bearer ${API_KEY}" | jq '.name')

if [ "$DETAILS" = "\"Test Server\"" ]; then
  echo "✅ Details verified"
fi

# Delete entity
curl -X DELETE "${REGISTRY_URL}/servers/${SERVICE_PATH}" \
  -H "Authorization: Bearer ${API_KEY}"

# Verify not in search results
sleep 1
RESULT=$(curl -s -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=test" | jq '.count')

if [ "$RESULT" -eq 0 ]; then
  echo "✅ Deleted entity not in search"
else
  echo "❌ Entity still searchable"
  exit 1
fi
```

### 5.3 Performance Test

**Scenario:** Search response time

```bash
#!/bin/bash
echo "Performance test: 100 sequential searches"

START=$(date +%s)

for i in {1..100}; do
  curl -s -G "${REGISTRY_URL}/servers/search" \
    --data-urlencode "q=database" > /dev/null
done

END=$(date +%s)
DURATION=$((END - START))
AVG=$(echo "scale=2; $DURATION / 100" | bc)

echo "Total time: ${DURATION}s"
echo "Average per search: ${AVG}s"

# Acceptance: Average < 0.1s
if (( $(echo "$AVG < 0.1" | bc -l) )); then
  echo "✅ PASS: Performance acceptable"
else
  echo "❌ FAIL: Too slow: ${AVG}s"
  exit 1
fi
```

### 5.4 Error Scenarios

#### Test 5.4.1: Malformed Query
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=%00%AA%FF" | jq .

# Expected: HTTP 200, returns empty results (not 500 error)
```

#### Test 5.4.2: Special Characters
```bash
curl -G "${REGISTRY_URL}/servers/search" \
  --data-urlencode "q=database>sql" | jq '.results | length'

# Expected: Non-zero (handles special chars gracefully)
```

## 6. Test Execution Checklist

### Pre-test Setup
- [ ] Clean test environment (`docker-compose down -v`)
- [ ] Build latest images (`make build`)
- [ ] Run database migrations
- [ ] Seed with test fixtures

### Test Execution

- [ ] ✅ Section 1.1 (Dependencies) passes
- [ ] ✅ Section 1.2 (API) passes
- [ ] ✅ Section 1.3 (CLI) passes
- [ ] ✅ Section 1.4 (Config) passes
- [ ] ✅ Section 2.1 (API Compatibility) passes
- [ ] ✅ Section 2.2 (URL Compatibility) passes
- [ ] ✅ Section 2.3 (Behavior Compatibility) passes
- [ ] ✅ Section 3.1 (Error Messages) passes
- [ ] ✅ Section 3.2 (Search Mode) passes
- [ ] ✅ Section 3.3 (Score Display) passes
- [ ] ✅ Section 4.1 (Docker Config) passes
- [ ] ✅ Section 4.2 (Terraform Config) passes (syntax check)
- [ ] ✅ Section 4.3 (Helm Config) passes (syntax check)
- [ ] ✅ Section 4.4 (Env Vars) passes
- [ ] ✅ Section 4.5 (Rollback) - path verified
- [ ] ✅ Section 5.1 (E2E Journey) passes
- [ ] ✅ Section 5.2 (Chain Operations) passes
- [ ] ✅ Section 5.3 (Performance) passes (<0.1s per search)
- [ ] ✅ Section 5.4 (Error Scenarios) passes (no crashes)

### Test Coverage Metrics
- [ ] Unit test coverage: >85% of modified code
- [ ] Integration test coverage: All API endpoints
- [ ] CLI test coverage: All search commands
- [ ] Regression test suite passes
- [ ] No FAISS imports remain in codebase
- [ ] No FAISS-related Docker mounts remain

### Post-test Actions
- [ ] Collect test logs
- [ ] Document any deviations
- [ ] Update test plan if needed
- [ ] Sign-off if all pass

---

## References

### Related Tests in Existing Suite
- `tests/unit/search/test_faiss_service.py` ← To be replaced
- `tests/integration/test_search_integration.py` ← To be updated
- `tests/unit/api/test_search_routes.py` ← To be updated

### Test Data
- Sample fixtures in `tests/fixtures/servers.json`
- Mock agent cards in `tests/fixtures/agents.json`

### Known Limitations
- Performance not validated at 10k+ items scale (covered in review as limitation)
- No multi-language support in testing (ASCII only)
- No concurrent load testing (single user tested)
