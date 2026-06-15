# Low-Level Design: Remove FAISS from the Codebase

*Created: 2026-06-15*
*Author: Claude*
*Status: Draft*

## Table of Contents
1. [Overview](#overview)
2. [Codebase Analysis](#codebase-analysis)
3. [Architecture](#architecture)
4. [Data Models](#data-models)
5. [API / CLI Design](#api--cli-design)
6. [Configuration Parameters](#configuration-parameters)
7. [New Dependencies](#new-dependencies)
8. [Implementation Details](#implementation-details)
9. [Observability](#observability)
10. [File Changes](#file-changes)
11. [Testing Strategy](#testing-strategy)
12. [Alternatives Considered](#alternatives-considered)
13. [Rollout Plan](#rollout-plan)

## Overview

### Problem Statement
The MCP Gateway Registry currently depends on FAISS (Facebook AI Similarity Search) for vector-based semantic search. While powerful, this adds significant complexity:
- External dependency on `faiss-cpu>=1.7.4`
- Memory overhead for storing embedding vectors
- Operational complexity with embeddings models
- Increased startup time for loading models and indices
- Complex search logic requiring index management

### Goals
- **P0**: Completely remove FAISS dependency from the codebase
- **P0**: Migrate search functionality to lexical (keyword-based) only
- **P0**: Maintain functional search capabilities for servers, agents, and tools
- **P1**: Simplify deployment and reduce resource requirements
- **P1**: Maintain backward compatibility for search API (URLs remain the same, behavior changes)
- **P2**: Update all documentation to reflect lexical-only search

### Non-Goals
- Not replacing FAISS with another vector database (pgvector, Weaviate, etc.)
- Not maintaining hybrid search (semantic + lexical)
- Not preserving similarity scores in responses
- Not supporting configurable hybrid search modes

## Codebase Analysis

### Key Files Reviewed

| File/Directory | Purpose | Relevance to This Change |
|----------------|---------|--------------------------|
| `registry/search/service.py` | Core FAISS service implementation | **DELETE** - This entire service will be removed |
| `registry/api/search_routes.py` | Search API endpoints | **MODIFY** - Refactor to use lexical search |
| `registry/repositories/file/search_repository.py` | Search repository interface | **MODIFY** - Implement lexical search logic |
| `registry/core/config.py` | Configuration settings | **MODIFY** - Remove embeddings/FAISS settings |
| `registry/api/server_routes.py` | Server API (search integration) | **MODIFY** - Remove FAISS calls |
| `registry/api/agent_routes.py` | Agent API (search integration) | **MODIFY** - Remove FAISS calls |
| `cli/agent_mgmt.py` | CLI search command | **MODIFY** - Update help text |
| `docs/embeddings.md` | Embeddings documentation | **DELETE** - Replace with lexical search docs |
| `docker-compose*.yml` | Docker configurations | **MODIFY** - Remove FAISS volumes |
| `pyproject.toml` | Dependencies | **MODIFY** - Remove faiss-cpu dependency |
| `tests/unit/search/test_faiss_service.py` | FAISS unit tests | **DELETE** - Remove all FAISS tests |
| `tests/fixtures/mocks/mock_faiss.py` | Mock FAISS for testing | **DELETE** - No longer needed |
| `tests/integration/test_search_integration.py` | Search integration tests | **MODIFY** - Update for lexical search |

### Existing Patterns Identified

#### 1. Search Repository Pattern
**Location**: `registry/repositories/interfaces.py`
```python
class SearchRepositoryBase(ABC):
    @abstractmethod
    async def search_servers(self, ...) -> list[SearchResult]: ...
```

**How to Follow**: Maintain the same interface but implement with lexical search logic.

#### 2. Configuration Pattern
**Location**: `registry/core/config.py`
```python
embeddings_provider: str = "sentence-transformers"
embeddings_model_name: str = "all-MiniLM-L6-v2"
embeddings_model_dimensions: int = 384
```

**How to Follow**: Remove all embeddings/FAISS configuration, keep only basic search settings.

#### 3. Service Layer Pattern
**Location**: `registry/services/`
Services encapsulate business logic and are called by API routes.

**How to Follow**: Create a new `LexicalSearchService` that replaces `FaissService` functionality.

### Integration Points

| Component | Integration Type | Details |
|-----------|------------------|---------|
| **Docker Compose** | Depends on | FAISS index mounted as volume, removed |
| **API Routes** | Uses | Search endpoints call FAISS service, need refactoring |
| **CLI** | Uses | Search commands reference semantic search, need updates |
| **Config** | Configures | Embeddings settings configure FAISS behavior |
| **Tests** | Tests | Extensive test coverage of FAISS functionality |
| **Docs** | Documents | Multiple docs reference FAISS functionality |

### Constraints and Limitations

1. **Search API Must Remain Functional**: Client applications depend on search endpoints
2. **CLI Search Must Work**: Users have scripts/scripts that use `mcp-search`
3. **No Breaking URL Changes**: Existing `/servers/search`, `/agents/search` endpoints must continue to work
4. **Performance**: Lexical search must be fast enough for interactive use
5. **Filtered Search**: Tool-level filters must continue to work |

## Architecture

### Current Architecture (With FAISS)

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   API Routes    │─────▶│  FaissService   │◀────▶│  FAISS Index   │
│ (search_routes) │      │ (search/service) │      │  (.faiss file) │
└─────────────────┘      └──────────────────┘      └─────────────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │ Embedding Model │ (Local or API)
                        │  (sentence-     │
                        │   transformers/ │
                        │     OpenAI/     │
                        │    Bedrock)     │
                        └──────────────────┘
```

### New Architecture (Lexical Only)

```
┌─────────────────┐      ┌──────────────────────┐
│   API Routes    │─────▶│ Lexical Search Logic │
│ (search_routes) │      │ (in repository)      │
└─────────────────┘      └──────────────────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │ In-Memory Cache  │
                        │ of Indexed Items │
                        └──────────────────┘
```

## Data Models

### API Response Models (No Change Required)

```python
class SearchResult(BaseModel):
    """Search result model - structure remains the same."""
    service_path: str
    server_info: dict  # Entity information
    # Note: relevance_score will change calculation method
```

### Configuration Models (Modified)

```python
# REMOVED Configuration Fields:
# - embeddings_provider
# - embeddings_model_name
# - embeddings_model_dimensions
# - embeddings_api_key
# - faiss_index_path
# - faiss_metadata_path

# REMOVED Configuration Class:
# class EmbeddingConfigGenerator(BaseModel):

# KEPT Configuration (modified):
class LexicalSearchSettings(BaseModel):
    """Lexical search configuration."""
    case_sensitive: bool = False
    search_fields: list[str] = ["name", "description", "tags", "tools"]
    fuzzy_matching: bool = True
    max_results: int = 50
```

## API / CLI Design

### API Endpoints (Modified Implementation, Same Interface)

#### Search Servers
**Endpoint:** `GET /servers/search`
**Query Params:** `q` (search query), `limit` (optional)
**Response:**
```json
{
  "results": [
    {
      "service_path": "string",
      "server_info": { ... },
      "type": "mcp_server",
      "relevance": 0.95
    }
  ],
  "count": 1,
  "search_mode": "lexical"  // Changed from "hybrid"
}
```

**Implementation Details:**
```python
# Old implementation:
results = await faiss_service.search_servers(query)

# New implementation:
results = await search_repository.search_servers(
    query=query,
    entity_type="mcp_server",
    limit=limit
)
```

#### Search Agents
**Endpoint:** `GET /agents/search`
**Same pattern as server search**

### CLI Commands (Modified Help Text)

```bash
# Command unchanged, help text updated:
mcp-search "database query"
# Old help: "Semantic search using FAISS vector index"
# New help: "Keyword-based search across all registered entities"
```

## Configuration Parameters

### Removed Configuration

| Parameter | Type | Description |
|-----------|------|-------------|
| `EMBEDDINGS_PROVIDER` | `string` | Removed - no embeddings needed |
| `EMBEDDINGS_MODEL_NAME` | `string` | Removed - no model |
| `EMBEDDINGS_MODEL_DIMENSIONS` | `int` | Removed - no vectors |
| `EMBEDDINGS_API_KEY` | `string` | Removed - no API calls |
| `EMBEDDINGS_API_BASE` | `string` | Removed - no API endpoint |
| `EMBEDDINGS_AWS_REGION` | `string` | Removed - no AWS service |
| `VECTOR_SEARCH_EF_SEARCH` | `int` | Removed - no vector search |

### New/Modified Configuration

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `lexical_search_fields` | `list[str]` | `["name", "description", "tags", "tools"]` | No | Fields to include in lexical search |
| `lexical_case_sensitive` | `bool` | `false` | No | Enable case-sensitive search |
| `lexical_fuzzy_match` | `bool` | `true` | No | Enable fuzzy string matching |

### Deployment Surface Checklist

Files that must be updated to remove embeddings/FAISS parameters:

- ✅ `.env.example` - Remove all embeddings env vars
- ✅ `docker-compose.yml` - Remove FAISS volume mounts
- ✅ `docker-compose.podman.yml` - Remove FAISS references
- ✅ `docker-compose.prebuilt.yml` - Remove FAISS service comments
- ✅ `build-config.yaml` - Remove FAISS image descriptions
- ✅ `terraform/aws-ecs/terraform.tfvars.example` - Remove embeddings variables
- ✅ `terraform/aws-ecs/variables.tf` - Remove embeddings variables
- ✅ `terraform/aws-ecs/main.tf` - Remove FAISS volume mounts
- ✅ `terraform/aws-ecs/task-definitions/registry.tftpl` - Remove embeddings env vars
- ✅ `charts/mcp-gateway/values.yaml` - Remove embeddings settings
- ✅ `k8s/registry/secret.yaml` - Remove embeddings API key references
- ✅ `k8s/registry/deployment.yaml` - Remove FAISS init containers

## New Dependencies

**None** - We are removing dependencies, not adding new ones.

### Removed Dependencies

| Package | Purpose | Removal Steps |
|---------|---------|---------------|
| `faiss-cpu>=1.7.4` | Vector similarity search | `uv remove faiss-cpu` |
| `sentence-transformers` | Local embeddings (optional) | Can remove if only used for embeddings |
| `litellm` | API-based embeddings (optional) | Can keep for other LLM features |

**Verification:**
```bash
uv remove faiss-cpu
uv lock
```

## Implementation Details

### Step-by-Step Plan

#### Step 1: Remove Dependencies
**Files:** `pyproject.toml`, `uv.lock`

```bash
# Run from repo root
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo

# Remove FAISS dependency
sed -i '/"faiss-cpu/d' pyproject.toml

# Optional: Review if sentence-transformers is used elsewhere
grep -r "sentence-transformers" --include="*.py" .

# Update lock file
uv lock
```

**Estimated Lines:** ~5 lines in pyproject.toml

#### Step 2: Delete FAISS Service
**File:** `registry/search/service.py`**Action:** DELETE FILE

```bash
rm /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/registry/search/service.py
```

**Rationale:** The entire service is FAISS-specific. No code to preserve.

#### Step 3: Refactor Search Repository
**File:** `registry/repositories/file/search_repository.py`
**Lines:** ~400 lines to refactor

**Current State:**
```python
# Uses FaissService for search
from registry.search.service import FaissService

class SearchRepository:
    def __init__(self, faiss_service: FaissService):
        self.faiss = faiss_service
```

**New Implementation:**
```python
class SearchRepository:
    def __init__(self, data_store: dict = None):
        self.indexed_items: dict[str, dict] = data_store or {}
        self.search_fields = ["name", "description", "tags", "tools"]

    async def search_servers(self, query: str, limit: int = 50) -> list[SearchResult]:
        """Lexical search across servers."""
        results = []
        for key, item in self.indexed_items.items():
            if item.get("entity_type") != "mcp_server":
                continue
            score = self._calculate_lexical_score(query, item)
            if score > 0:
                results.append(SearchResult(score=score, **item))
        
        return sorted(results, key=lambda r: r.score, reverse=True)[:limit]

    def _calculate_lexical_score(self, query: str, item: dict) -> float:
        """Calculate relevance score based on keyword matches."""
        query_lower = query.lower()
        text_fields = []
        
        # Extract searchable text
        info = item.get("full_server_info", {})
        text_fields.append(info.get("server_name", ""))
        text_fields.append(info.get("description", ""))
        text_fields.extend(info.get("tags", []))
        
        for tool in info.get("tool_list", []):
            text_fields.append(tool.get("name", ""))
            text_fields.append(tool.get("description", ""))
        
        # Join into searchable text
        searchable_text = " ".join(map(str, text_fields)).lower()
        
        # Simple scoring: exact match > word match > partial match
        if query_lower == searchable_text:
            return 1.0
        elif query_lower in searchable_text:
            # Score based on frequency and position
            occurrences = searchable_text.count(query_lower)
            return min(0.9, 0.5 + (occurrences * 0.1))
        else:
            # Check word-by-word
            query_words = query_lower.split()
            matches = sum(1 for word in query_words if word in searchable_text)
            return min(0.49, matches / len(query_words) * 0.49) if query_words else 0
```

#### Step 4: Update Configurations
**File:** `registry/core/config.py`
**Lines:** ~100 lines related to embeddings/FAISS to remove

**Lines to Remove (500-750):**
```python
# REMOVE:
embeddings_provider: str = "sentence-transformers"
embeddings_model_name: str = "all-MiniLM-L6-v2"
embeddings_model_dimensions: int = 384
embeddings_api_key: str | None = None
embeddings_secret_key: str | None = None
embeddings_api_base: str | None = None
embeddings_aws_region: str | None = "us-east-1"
vector_search_ef_search: int = 100
embeddings_model_dir: Path = ...
embeddings_model_cache_dir: Path = ...
faiss_index_path: Path = ...
faiss_metadata_path: Path = ...
```

**Optional: Add Lexical Search Settings**
```python
# ADD at ~151 (after EmbeddingConfigGenerator is removed):
lexical_search_fields: list[str] = [
    "server_name", "description", "tags", "tool_names", "tool_descriptions"
]
lexical_case_sensitive: bool = False
lexical_fuzzy_match: bool = True
```

#### Step 5: Update API Routes
**File:** `registry/api/search_routes.py`
**Lines:** ~100 lines to refactor

**Key Changes:**
```python
# REMOVE:
from ..search.service import FaissService

# MODIFY: SearchRepository injection
@router.get("/servers/search", ...)
async def search_servers(
    query: str,
    search_repo: SearchRepositoryBase = Depends(get_search_repository),
    limit: int = 50,
):
    # REMOVE:
    # if not await get_faiss_service().is_initialized():
    #     raise HTTPException(...)
    
    # CHANGE:
    # results = await search_repo.search_servers(query, limit=limit)
    # TO:
    results = await search_repo.search_servers(query, limit=limit)
    
    return {
        "results": results,
        "count": len(results),
        "search_mode": "lexical"  # Changed from "hybrid"
    }
```

#### Step 6: Update Search Repository Factory
**File:** `registry/repositories/factory.py`
**Lines:** ~50 lines

```python
# CHANGE factory function:
def get_search_repository() -> SearchRepositoryBase:
    # REMOVE:
    # from registry.search.service import get_faiss_service
    # faiss_service = get_faiss_service()
    # return FileSearchRepository(faiss_service)
    
    # TO:
    from registry.repositories.file.search_repository import FileSearchRepository
    return FileSearchRepository()
```

#### Step 7: Update Docker Compose
**Files:** `docker-compose.yml`, `docker-compose.podman.yml`, `docker-compose.prebuilt.yml`
**Lines:** ~20 lines per file

**Remove:**
```yaml
# FROM docker-compose.yml ~71:
# Registry service (includes nginx, SSL, FAISS, models)
# Remove or simplify comment
```

**Remove volumes:**
```yaml
# Around 130-150:
# volumes:
#   - ${MCP_FAISS_DATA:-~/.mcp-faiss}:/home/mcp/.mcp:rw
#   - ${MCP_EMBEDDINGS_MODEL_DIR:-~/.cache/sentence_transformers}:/home/mcp/.cache:rw  # temp
```

#### Step 8: Delete Test Files
**Files to DELETE:**
```bash
rm /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/tests/unit/search/test_faiss_service.py
rm /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/tests/fixtures/mocks/mock_faiss.py
```

**Files to MODIFY:**
- `tests/integration/test_search_integration.py` - Remove FAISS-specific tests
- `tests/unit/api/test_search_routes.py` - Update to test lexical search
- `tests/conftest.py` - Remove FAISS fixtures

#### Step 9: Update Documentation
**Main File:** `docs/embeddings.md`**Action:** DELETE and recreate as `docs/search.md`

**New File:** `docs/search.md`
```markdown
# Search Guide

## Overview
MCP Gateway Registry provides keyword-based search across registered MCP servers, tools, and A2A agents.

## How It Works
The search uses lexical matching across multiple fields:
- Server/Agent names
- Descriptions
- Tags
- Tool names and descriptions

## Usage

### Search Servers
```bash
curl "http://localhost:8000/servers/search?q=database"
```

### Search Agents
```bash
curl "http://localhost:8000/agents/search?q=data+analysis"
```

### Score Calculation
Results are ranked by:
1. Exact keyword matches (highest score)
2. Multiple keyword matches
3. Substring matches
4. Word frequency in document

## Configuration
No configuration needed - search is enabled by default.
```

**Files to Update:**
- `README.md` - Remove embeddings setup sections
- `docs/configuration.md` - Remove embeddings configuration
- `docs/api-reference.md` - Update search endpoint docs
- `docs/faq/*.md` - Update any FAISS references
- `docs/design/*.md` - Remove FAISS architecture discussions

#### Step 10: Update Build Configuration
**File:** `build-config.yaml`
**Lines:** ~20 lines at end

**Remove:**
```yaml
# Around line 25:
# Main MCP Gateway Registry with nginx reverse proxy, FAISS, models

# Around 30-35:
# description: "MCP Gateway Registry with nginx, FAISS, models"
# description: "MCP Gateway Registry with nginx, FAISS, models (prebuilt)"
```

**Simplify descriptions:**
```yaml
# TO:
description: "MCP Gateway Registry with nginx reverse proxy"
```

#### Step 11: Clean Up Registry Main
**File:** `registry/main.py`
**Lines:** ~50 lines to check

**Remove FAISS initialization code:**
```python
# Around imports - remove:
# from registry.search.service import FaissService

# Around app startup - remove:
# @app.on_event("startup")
# async def setup_faiss():
#     faiss_service = FaissService()
#     await faiss_service.initialize()
```

#### Step 12: Update CLI Help Text
**File:** `cli/agent_mgmt.py`
**Line:** ~34

**Change:**
```python
# FROM:
# "The 'search' command performs natural language semantic search using FAISS vector index"
# TO:
"The 'search' command performs keyword-based search across registered MCP servers and agents"
```

#### Step 13: Remove Terraform References
**Files:** `terraform/aws-ecs/*.tf`

Remove variables:
```hcl
# FROM variables.tf - REMOVE:
variable "embeddings_provider" { ... }
variable "embeddings_model_name" { ... }
variable "embeddings_api_key_secret" { ... }
# etc.
```

Remove from task definition template:
```hcl
# FROM task-definitions/registry.tftpl - REMOVE environments:
# - name: EMBEDDINGS_PROVIDER
#   value: "${embeddings_provider}"
# - name: EMBEDDINGS_MODEL_NAME
#   value: "${embeddings_model_name}"
# etc.
```

### Error Handling

#### New Error Cases
```python
# No new errors introduced
# Existing error cases remain the same:
# - Empty search query (400)
# - Server/agent not found (404)

# REMOVED error cases:
# - FAISS not initialized (500) - no longer applies
# - Embedding model unavailable (503) - no longer applies
```

### Logging

#### Modifications
Reduce logging verbosity (no more model loading logs):
```python
# OLD:
logger.info(f"Loading embedding model with provider: {settings.embeddings_provider}")
logger.info(f"Embedding model loaded successfully. Provider: {settings.embeddings_provider}")
logger.info(f"Loading FAISS index from {settings.faiss_index_path}")

# NEW:
logger.debug(f"Performing lexical search for query: '{query}'")
logger.debug(f"Found {len(results)} results for query: '{query}'")
```

## File Changes

### New Files

| File Path | Description | Lines |
|-----------|-------------|-------|
| `docs/search.md` | New search documentation | ~200 |
| `registry/search/lexical.py` | (Optional) Lexical search utilities | ~150 |

### Modified Files

| File Path | Lines | Change Description |
|-----------|-------|--------------------|
| `pyproject.toml` | 1 | Remove faiss-cpu dependency |
| `registry/core/config.py` | 100 | Remove embeddings/FAISS settings |
| `registry/api/search_routes.py` | 100 | Refactor to use lexical search |
| `registry/repositories/file/search_repository.py` | 400 | Implement lexical search logic |
| `registry/repositories/factory.py` | 50 | Update search repository factory |
| `docker-compose.yml` | 20 | Remove FAISS volumes |
| `docker-compose.podman.yml` | 20 | Remove FAISS references |
| `docs/embeddings.md` | ~428 | Replace with `docs/search.md` |
| `README.md` | ~50 | Remove embeddings setup sections |
| Multiple docs files | ~200 | Update/remove FAISS references |
| `tests/integration/test_search_integration.py` | ~150 | Rewrite for lexical search |
| `tests/unit/api/test_search_routes.py` | ~100 | Update expected responses |
| `build-config.yaml` | 10 | Update descriptions |

### Deleted Files

| File Path | Description |
|-----------|-------------|
| `registry/search/service.py` | ~500 lines - Entire FAISS service |
| `tests/unit/search/test_faiss_service.py` | ~300 lines - FAISS unit tests |
| `tests/fixtures/mocks/mock_faiss.py` | ~100 lines - Mock FAISS |

### Estimated Lines of Code

| Category | Lines | Note |
|----------|-------|------|
| New code | ~350 | Lexical search implementation |
| Deleted code | ~900 | FAISS service + tests |
| Modified code | ~1,150 | Config, APIs, repos, docs, tests |
| **Net change** | **~600** | **Overall reduction in LOC** |

## Testing Strategy

Detailed testing plan is in `./testing.md`. Summary:

- **Unit Tests**: New tests for lexical search scoring logic
- **Integration Tests**: End-to-end search scenarios
- **API Tests**: Verify API response format and search results
- **Regression Tests**: Ensure no FAISS code paths remain
- **Documentation Tests**: Verify docs are accurate