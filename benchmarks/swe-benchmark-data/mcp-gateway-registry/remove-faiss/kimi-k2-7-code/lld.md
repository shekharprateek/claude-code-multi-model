# Low-Level Design: Remove FAISS from the codebase

*Created: 2026-07-15*
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
10. [Scaling Considerations](#scaling-considerations)
11. [File Changes](#file-changes)
12. [Testing Strategy](#testing-strategy)
13. [Alternatives Considered](#alternatives-considered)
14. [Rollout Plan](#rollout-plan)
15. [Open Questions](#open-questions)
16. [References](#references)

## Overview

### Problem Statement
The registry currently supports two semantic search backends:

1. **FAISS** (file-backed, in-memory) via `registry/search/service.py` and `registry/repositories/file/search_repository.py`.
2. **DocumentDB hybrid search** (native text + vector) via `registry/repositories/documentdb/search_repository.py`.

DocumentDB now covers all FAISS use cases with native HNSW vector search and lexical matching. FAISS remains as dead weight: a `faiss-cpu` dependency, native C++ wheels, duplicate indexing logic, and extra deployment surface. This design removes FAISS entirely and makes DocumentDB the single search backend.

### Goals
- Eliminate the `faiss-cpu` dependency and native-library requirement.
- Delete the `FaissService` and `FaissSearchRepository` implementations.
- Route all search indexing and querying through `DocumentDBSearchRepository`.
- Update build, deployment, telemetry, tests, and docs to remove FAISS references.
- Preserve end-user search behavior (servers, agents, tools, tags, mixed semantic search).

### Non-Goals
- Rewriting DocumentDB hybrid search (reuse existing implementation).
- Removing file-backed storage for servers/agents (only search is consolidated).
- Changing embedding providers or model configuration.
- Migrating legacy FAISS index files into DocumentDB (assumed already done or unnecessary).

## Codebase Analysis

### Key Files Reviewed

| File/Directory | Purpose | Relevance to This Change |
|----------------|---------|--------------------------|
| `registry/search/service.py` | `FaissService`: embedding model loading, FAISS index create/load/save, add/remove/search | Delete |
| `registry/repositories/file/search_repository.py` | `FaissSearchRepository`: search repository adapter around `FaissService` | Delete |
| `registry/repositories/documentdb/search_repository.py` | `DocumentDBSearchRepository`: hybrid text+vector search | Keep as sole backend |
| `registry/repositories/factory.py` | Repository factory selecting FAISS vs DocumentDB by `STORAGE_BACKEND` | Update selection logic |
| `registry/repositories/interfaces.py` | `SearchRepositoryBase` abstract class | No API change needed |
| `registry/core/config.py` | Settings including `faiss_index_path` and `faiss_metadata_path` | Remove settings |
| `registry/core/schemas.py` | `FaissMetadata` Pydantic model | Remove model |
| `registry/main.py` | Startup re-index logic guarded by backend type | Simplify; DocumentDB path is persistent |
| `registry/api/search_routes.py` | Search endpoints; error message references FAISS | Update messages |
| `registry/api/server_routes.py` | Imports and calls `faiss_service` for indexing | Replace with `get_search_repository()` |
| `registry/api/agent_routes.py` | Imports and calls `faiss_service` for indexing | Replace with `get_search_repository()` |
| `registry/services/agent_batch_item_processor.py` | Imports and calls `faiss_service` | Replace with repository |
| `registry/core/telemetry.py` | Heartbeat reports `search_backend` (`faiss` or `documentdb`) | Always report `documentdb` |
| `registry/metrics/client.py` | `emit_discovery_metric` accepts `faiss_search_time_ms` | Rename metric field |
| `metrics-service/` | SQLite schema and API docs reference `faiss_search_time_ms` | Rename field |
| `pyproject.toml` | Declares `faiss-cpu>=1.7.4` | Remove dependency |
| `uv.lock` | Locked FAISS wheels | Regenerate after dependency removal |
| `docker/Dockerfile.registry*` | Build steps/comments referencing FAISS | Clean up |
| `docker-compose*.yml` | Comments referencing FAISS | Clean up |
| `build_and_run.sh` | FAISS index file checks and verification | Remove checks |
| `terraform/aws-ecs/scripts/service_mgmt.sh` | `verify_faiss_metadata()` helper | Remove helper and call sites |
| `terraform/telemetry-collector/lambda/collector/schemas.py` | `search_backend` enum allows `faiss` | Remove `faiss` option |
| `tests/fixtures/mocks/mock_faiss.py` | Mock FAISS module | Delete |
| `tests/unit/search/test_faiss_service.py` | FAISS service unit tests | Delete |
| `tests/integration/test_search_integration.py` | Patches `faiss_service.search_mixed` | Update to patch repository |
| `tests/unit/api/test_server_routes.py`, `test_agent_routes.py`, etc. | Patch `registry.search.service.faiss_service` | Update patch targets |

### Existing Patterns Identified

1. **Repository abstraction**: All persistence and search flows through `get_*_repository()` singletons in `registry/repositories/factory.py`. Future implementers should continue this pattern and replace direct `faiss_service` imports with `get_search_repository()`.
2. **Lazy imports in routes**: `server_routes.py` and `agent_routes.py` already import `faiss_service` inside functions to avoid circular imports. The same pattern should be used when importing `get_search_repository()` if circular import risks exist, or the repository can be obtained once per module.
3. **Settings are Pydantic**: `registry/core/config.py` uses `pydantic_settings`. Removing properties is safe; callers must be updated.
4. **Startup re-index**: `registry/main.py` rebuilds the in-memory FAISS index on boot for file backend. DocumentDB is persistent, so this block can be removed entirely.
5. **Telemetry enum**: The telemetry collector accepts `search_backend` values `faiss` or `documentdb`. After removal, only `documentdb` is valid.
6. **Test mocking convention**: Tests patch `registry.search.service.faiss_service` directly. They should be updated to patch `registry.repositories.factory.get_search_repository` or the returned repository object.

### Integration Points

| Component | Integration Type | Details |
|-----------|------------------|---------|
| DocumentDB search repository | Becomes sole backend | `get_search_repository()` returns `DocumentDBSearchRepository` unconditionally |
| Server routes | Uses search repo | `add_or_update_service`, `remove_service` via repository |
| Agent routes | Uses search repo | `add_or_update_agent`, `remove_agent` via repository |
| Agent batch processor | Uses search repo | Index/remove A2A agent cards |
| Telemetry heartbeat | Reports backend | Always `documentdb` |
| Metrics client | Discovery metric | Field renamed from `faiss_search_time_ms` |

### Constraints and Limitations Discovered

- **File backend loses native vector search**: The file-backed server/agent repositories remain, but their search capability was provided by FAISS. After removal, file-backend deployments must either use DocumentDB for search or accept that semantic search is unavailable. This design chooses DocumentDB as the required search backend.
- **Lazy `faiss_service` imports**: Several route modules import `faiss_service` inside functions. These must be carefully replaced to avoid accidentally importing the deleted module.
- **Tests rely on global `faiss` mock**: `tests/conftest.py` injects `MockFaissModule` into `sys.modules["faiss"]`. Removing FAISS means this mock and its consumers can be deleted, but any test that incidentally relied on `faiss` being present for unrelated import side effects must be checked.
- **Metrics schema change**: Renaming `faiss_search_time_ms` affects downstream dashboards and the telemetry collector. A coordinated rename or deprecation period may be needed.

## Architecture

### System Context Diagram

```
+-------------+     +----------------------+     +------------------+
| API Routes  |---->| SearchRepositoryBase |---->| DocumentDBSearch |
| (servers,   |     | (factory singleton)  |     | Repository       |
| agents,     |     |                      |     | (hybrid text+    |
| search)     |     |  DocumentDB only     |     |  vector)         |
+-------------+     +----------------------+     +------------------+
                              |
                              v
                       +-------------+
                       | Embeddings  |
                       | Client      |
                       +-------------+
```

### Sequence Diagram: Server Create/Update

```
Client -> ServerRoutes: POST /servers
ServerRoutes -> ServerService: create/update
ServerRoutes -> SearchRepository: index_server(path, server_info, is_enabled)
SearchRepository -> DocumentDB: upsert search document
SearchRepository --> ServerRoutes: done
ServerRoutes --> Client: response
```

### Sequence Diagram: Semantic Search

```
Client -> SearchRoutes: POST /semantic
SearchRoutes -> SearchRepository: search(query, entity_types, ...)
SearchRepository -> DocumentDB: vector + keyword + RRF
DocumentDB --> SearchRepository: ranked docs
SearchRepository --> SearchRoutes: grouped results
SearchRoutes --> Client: SemanticSearchResponse
```

## Data Models

### Models to Remove

```python
# registry/core/schemas.py
class FaissMetadata(BaseModel):
    """FAISS metadata model."""

    id: int
    text_for_embedding: str
    full_server_info: ServerInfo
```

### Settings to Remove

```python
# registry/core/config.py
@property
def faiss_index_path(self) -> Path:
    return self.servers_dir / "service_index.faiss"

@property
def faiss_metadata_path(self) -> Path:
    return self.servers_dir / "service_index_metadata.json"
```

### Model Changes

- `registry/core/config.py`: Remove the two properties above. No other schema changes.
- `registry/core/schemas.py`: Remove `FaissMetadata`.
- `terraform/telemetry-collector/lambda/collector/schemas.py`: Update the `search_backend` regex from `^(faiss|documentdb)$` to `^documentdb$` (or remove validation if the value is now constant).
- `metrics-service/app/storage/database.py`, `migrations.py`, `docs/database-schema.md`, `docs/api-reference.md`: Rename `faiss_search_time_ms` to `vector_search_time_ms`.
- `registry/metrics/client.py`: Rename parameter and metadata key.

## API / CLI Design

### No New Endpoints or Commands

This change removes a backend; it does not add new user-facing APIs or CLI commands.

### Modified Error Messages

In `registry/api/search_routes.py`:

```python
# Before
logger.error("FAISS search service unavailable: %s", exc, exc_info=True)

# After
logger.error("Search service unavailable: %s", exc, exc_info=True)
```

The HTTP 503 response detail can remain backend-agnostic: "Semantic search is temporarily unavailable. Please try again later."

## Configuration Parameters

### Removed Environment Variables / Settings

| Setting | Previously | Change |
|---------|------------|--------|
| `settings.faiss_index_path` | Computed path to `service_index.faiss` | Remove |
| `settings.faiss_metadata_path` | Computed path to `service_index_metadata.json` | Remove |
| `FaissMetadata` schema | Internal metadata format | Remove |

### Deployment Surface Checklist

- [ ] `pyproject.toml`: remove `faiss-cpu`.
- [ ] `uv.lock`: regenerate.
- [ ] `.env.example`: verify no FAISS-specific variables exist (none were found).
- [ ] `docker/Dockerfile.registry` and variants: remove FAISS comments/build steps.
- [ ] `docker-compose*.yml`: remove FAISS comments.
- [ ] `build_and_run.sh`: remove FAISS index file checks and verification block.
- [ ] `terraform/aws-ecs/scripts/service_mgmt.sh`: remove `verify_faiss_metadata` and call sites.
- [ ] `terraform/telemetry-collector/lambda/collector/schemas.py`: narrow `search_backend` enum.

## New Dependencies

This change uses only existing dependencies. `faiss-cpu` is removed.

## Implementation Details

### Step-by-Step Plan

#### Step 1: Remove FAISS service and repository files
**Files to delete:**
- `registry/search/service.py`
- `registry/repositories/file/search_repository.py`
- `tests/fixtures/mocks/mock_faiss.py`
- `tests/unit/search/test_faiss_service.py`

Also delete the `registry/search/` directory if it becomes empty. Check whether `registry/search/__init__.py` contains anything other than `FaissService` re-exports.

#### Step 2: Update repository factory
**File:** `registry/repositories/factory.py`

Change `get_search_repository()` to always return `DocumentDBSearchRepository`:

```python
def get_search_repository() -> SearchRepositoryBase:
    """Get search repository singleton (DocumentDB only)."""
    global _search_repo

    if _search_repo is not None:
        return _search_repo

    logger.info("Creating DocumentDB search repository")
    from .documentdb.search_repository import DocumentDBSearchRepository

    _search_repo = DocumentDBSearchRepository()
    return _search_repo
```

Remove the `backend` check and the `FaissSearchRepository` import branch.

#### Step 3: Update startup flow
**File:** `registry/main.py` (~lines 495-547)

Simplify startup to:

```python
search_repo = get_search_repository()
logger.info("Initializing DocumentDB search service...")
await search_repo.initialize()
logger.info("DocumentDB search index is persistent, skipping startup re-index")
```

Remove the conditional rebuild block that only ran for non-MongoDB backends.

#### Step 4: Replace direct `faiss_service` usage in routes and services
**Files:**
- `registry/api/server_routes.py`
- `registry/api/agent_routes.py`
- `registry/services/agent_batch_item_processor.py`

Pattern:

```python
# Before
from ..search.service import faiss_service
await faiss_service.add_or_update_service(path, server_info, is_enabled)

# After
from ..repositories.factory import get_search_repository
search_repo = get_search_repository()
await search_repo.index_server(path, server_info, is_enabled)
```

For removals:

```python
# Before
await faiss_service.remove_service(path)

# After
search_repo = get_search_repository()
await search_repo.remove_entity(path)
```

For agent routes, use `search_repo.index_agent(path, agent_card, is_enabled)` and `search_repo.remove_entity(path)`.

#### Step 5: Remove settings and schema
**Files:**
- `registry/core/config.py`: delete `faiss_index_path` and `faiss_metadata_path` properties.
- `registry/core/schemas.py`: delete `FaissMetadata`.

#### Step 6: Update telemetry and metrics
**File:** `registry/core/telemetry.py` (~line 731)

```python
search_backend = "documentdb"
```

**File:** `registry/metrics/client.py` (~line 111)

```python
async def emit_discovery_metric(
    self,
    query: str,
    results_count: int,
    duration_ms: float,
    top_k_services: int | None = None,
    top_n_tools: int | None = None,
    embedding_time_ms: float | None = None,
    vector_search_time_ms: float | None = None,
) -> bool:
    metadata={
        "embedding_time_ms": embedding_time_ms,
        "vector_search_time_ms": vector_search_time_ms,
    }
```

**File:** `metrics-service/app/storage/database.py`, `migrations.py`, schema docs, and API docs: rename `faiss_search_time_ms` to `vector_search_time_ms`.

**File:** `terraform/telemetry-collector/lambda/collector/schemas.py`: change `pattern="^(faiss|documentdb)$"` to `pattern="^documentdb$"`.

#### Step 7: Remove FAISS from build and deployment
**Files:**
- `docker/Dockerfile.registry`, `Dockerfile.registry-cpu`, and any other registry Dockerfiles: remove comments/steps mentioning FAISS.
- `docker-compose.yml`, `docker-compose.podman.yml`, `docker-compose.prebuilt.yml`: remove comments mentioning FAISS.
- `build-config.yaml`: remove comments/description mentioning FAISS.
- `build_and_run.sh`: remove lines 242-278 (FAISS file cleanup check) and 620-634 (FAISS index creation verification).
- `terraform/aws-ecs/scripts/service_mgmt.sh`: remove `verify_faiss_metadata()` function and its call sites on lines 631 and 705.
- `terraform/aws-ecs/OPERATIONS.md`: update the registry description table row to remove FAISS.

#### Step 8: Update dependency manifest
**File:** `pyproject.toml`

Remove `"faiss-cpu>=1.7.4"` from `dependencies`.

**File:** `uv.lock`

Run `uv lock` (or `uv sync`) to regenerate. Do not hand-edit.

#### Step 9: Update tests
**Files:**
- `tests/conftest.py`: remove `from tests.fixtures.mocks.mock_faiss import create_mock_faiss_module` and the block that injects `sys.modules["faiss"]`. Update the docstring for the mock search repository comment.
- `tests/unit/conftest.py`: remove `mock_faiss_service` fixture.
- `tests/unit/core/test_config.py`: remove `test_faiss_index_path` and `test_faiss_metadata_path`.
- `tests/unit/core/test_telemetry.py`: update expected `search_backend` from `faiss` to `documentdb` for the file-backend test case.
- `tests/unit/repositories/test_factory_aliases.py`: update the assertion that allows `FaissSearchRepository`; it should now expect only `DocumentDBSearchRepository`.
- `tests/unit/search/`: delete `__init__.py` and `test_faiss_service.py` if the directory is empty.
- `tests/unit/api/test_server_routes.py`, `test_agent_routes.py`, etc.: replace `mock_faiss_service` fixtures and patches with a mock `SearchRepositoryBase` returned by `get_search_repository()`.
- `tests/integration/test_search_integration.py`: replace patches on `registry.api.search_routes.faiss_service.search_mixed` with patches on `get_search_repository().search`.
- `tests/integration/test_tool_level_access.py`: replace `fake_faiss` with a fake repository.
- `tests/test_infrastructure.py`: remove `MockFaissIndex` test.
- `tests/unit/test_safe_eval_arithmetic.py`: remove the FAISS mock spec-fix block; with `faiss` deleted, this workaround is unnecessary.

### Error Handling

- If `get_search_repository()` is called before DocumentDB is available, `DocumentDBSearchRepository.initialize()` or the first search call will raise a connection error. Existing route handlers should surface this as a 503.
- All removed `faiss_service` call sites already catch and log exceptions; preserve that behavior.

### Logging

- Remove all log messages containing "FAISS" from deleted files.
- In remaining files, change messages to be backend-agnostic, e.g. "Indexing server '{path}'" instead of "Adding/updating service '{path}' in FAISS".

## Observability

### Tracing / Metrics / Logging Points

- `registry/main.py`: log "Initializing DocumentDB search service..." at INFO on startup.
- `registry/repositories/factory.py`: log "Creating DocumentDB search repository" at INFO.
- `registry/metrics/client.py`: discovery metric now records `vector_search_time_ms` instead of `faiss_search_time_ms`.
- `registry/core/telemetry.py`: heartbeat always reports `search_backend: documentdb`.

## Scaling Considerations

- DocumentDB HNSW vector search scales with the database rather than process memory, removing the previous in-memory FAISS index size limit.
- There is no startup re-index penalty because embeddings persist in DocumentDB.
- The embeddings client still loads the configured model into memory; this is unchanged.

## File Changes

### Deleted Files

| File Path | Reason |
|-----------|--------|
| `registry/search/service.py` | `FaissService` implementation obsolete |
| `registry/repositories/file/search_repository.py` | `FaissSearchRepository` obsolete |
| `tests/fixtures/mocks/mock_faiss.py` | FAISS mock no longer needed |
| `tests/unit/search/test_faiss_service.py` | FAISS unit tests obsolete |
| `tests/unit/search/__init__.py` | Directory can be removed if empty |

### Modified Files

| File Path | Lines | Change Description |
|-----------|-------|--------------------|
| `pyproject.toml` | ~23 | Remove `faiss-cpu>=1.7.4` |
| `uv.lock` | many | Regenerate after dependency removal |
| `registry/repositories/factory.py` | ~132-151 | Always return `DocumentDBSearchRepository` |
| `registry/core/config.py` | ~996-1001 | Remove `faiss_index_path` and `faiss_metadata_path` |
| `registry/core/schemas.py` | ~505-511 | Remove `FaissMetadata` |
| `registry/main.py` | ~495-547 | Simplify startup; remove file-backend re-index |
| `registry/api/search_routes.py` | ~440 | Update error log message |
| `registry/api/server_routes.py` | many | Replace `faiss_service` calls with repository |
| `registry/api/agent_routes.py` | many | Replace `faiss_service` calls with repository |
| `registry/services/agent_batch_item_processor.py` | ~225, ~338 | Replace `faiss_service` calls with repository |
| `registry/core/telemetry.py` | ~731 | Hard-code `search_backend = "documentdb"` |
| `registry/metrics/client.py` | ~111, ~126 | Rename `faiss_search_time_ms` |
| `metrics-service/app/storage/database.py` | ~172, ~306, ~319 | Rename column and references |
| `metrics-service/app/storage/migrations.py` | ~120 | Rename column |
| `metrics-service/docs/database-schema.md` | ~207 | Rename column |
| `metrics-service/docs/api-reference.md` | ~440, ~666 | Rename field |
| `metrics-service/tests/test_database.py` | ~142 | Update test data key |
| `terraform/telemetry-collector/lambda/collector/schemas.py` | ~267 | Narrow regex |
| `terraform/aws-ecs/scripts/service_mgmt.sh` | ~184-203, 631, 705 | Remove `verify_faiss_metadata` |
| `terraform/aws-ecs/OPERATIONS.md` | ~136 | Remove FAISS from description |
| `docker-compose.yml` | ~71 | Remove FAISS comment |
| `docker-compose.podman.yml` | ~4 | Remove FAISS comment |
| `docker-compose.prebuilt.yml` | ~14 | Remove FAISS comment |
| `build-config.yaml` | ~25, ~30 | Remove FAISS comments/description |
| `build_and_run.sh` | ~242-278, ~620-634 | Remove FAISS file checks |
| `tests/conftest.py` | ~51-149 | Remove FAISS mock injection |
| `tests/unit/conftest.py` | ~16-21 | Remove `mock_faiss_service` fixture |
| `tests/unit/core/test_config.py` | ~555-587 | Remove FAISS path tests |
| `tests/unit/core/test_telemetry.py` | ~340, ~343 | Update expected backend |
| `tests/unit/repositories/test_factory_aliases.py` | ~197-199 | Expect only DocumentDB repository |
| `tests/unit/api/test_search_routes.py` | many | Rename sample fixtures, update mocks |
| `tests/unit/api/test_server_routes.py` | many | Replace `mock_faiss_service` |
| `tests/unit/api/test_agent_routes.py` | many | Replace `mock_faiss_service` |
| `tests/unit/api/test_skill_inline_content.py` | ~126 | Update patch target |
| `tests/unit/services/test_agent_batch_item_processor.py` | ~144, ~258 | Update patch target |
| `tests/unit/services/test_duplicate_check_service.py` | ~585 | Update error text |
| `tests/integration/test_search_integration.py` | many | Patch repository instead of `faiss_service` |
| `tests/integration/test_server_lifecycle.py` | ~54-67 | Replace `mock_faiss_service` fixture |
| `tests/integration/test_tool_level_access.py` | ~168-297, ~394-505 | Replace fake FAISS with fake repository |
| `tests/integration/test_telemetry_e2e.py` | ~312, ~335 | Expect `documentdb` |
| `tests/test_infrastructure.py` | ~15-29 | Remove `MockFaissIndex` test |
| `tests/unit/test_safe_eval_arithmetic.py` | ~25-37 | Remove FAISS spec workaround |
| `tests/unit/api/test_search_routes_local_server.py` | ~51 | Update comment |
| `tests/unit/api/test_check_duplicates_endpoints.py` | ~299 | Update error text |

### Documentation Files

| File Path | Change Description |
|-----------|--------------------|
| `docs/embeddings.md` | Remove FAISS integration section, update links |
| `docs/service-management.md` | Remove FAISS indexing references |
| `docs/server-versioning-operations.md` | Update "FAISS search index" wording |
| `docs/dynamic-tool-discovery.md` | Replace FAISS references with DocumentDB hybrid search |
| `docs/database-design.md` | Update vector search backend table |
| `docs/configuration.md` | Update file-backend cons |
| `docs/design/database-abstraction-layer.md` | Remove FaissSearchRepo references |
| `docs/design/storage-architecture-mongodb-documentdb.md` | Remove FAISS column |
| `docs/design/server-versioning.md` | Update re-index wording |
| `docs/design/a2a-protocol-integration.md` | Replace FAISS indexing with DocumentDB |
| `docs/api-reference.md` | Update agent search description |
| `docs/llms.txt` | Remove FAISS from module/flow descriptions |
| `docs/TELEMETRY.md` | Update `search_backend` description |
| `docs/OBSERVABILITY-LEGACY.md` | Update legacy metrics references (optional) |
| `docs/testing/test-categories.md` | Remove FAISS mock instructions |
| `docs/testing/memory-management.md` | Remove FAISS vector indexes bullet |
| `docs/testing/QUICK-START.md` | Remove FAISS from heavy-dependency list |
| `release-notes/1.24.5.md` (new) | Document breaking change and migration notes |

### Estimated Lines of Code

| Category | Lines |
|----------|-------|
| Deleted code | ~2,300 (FAISS service, repository, tests, mocks) |
| New code | ~50 (factory simplification, startup simplification, renamed fields) |
| Modified code | ~800 (route patches, test updates, docs) |
| **Total net change** | **~-2,500** |

## Testing Strategy

See `testing.md` for the full executable plan.

## Alternatives Considered

### Alternative 1: Keep a simplified keyword-only search for file backend
**Description:** Remove FAISS vector index but keep a text-only search implementation for file-backed storage.
**Pros:** File-backend deployments retain some search capability without DocumentDB.
**Cons:** Adds a third search path to maintain; semantic relevance would differ from DocumentDB; contradicts the goal of consolidating on DocumentDB.
**Why Rejected:** The stated goal is to replace FAISS with DocumentDB hybrid search. A keyword-only fallback would fragment behavior and add maintenance overhead.

### Alternative 2: Make FAISS optional behind a feature flag
**Description:** Keep `faiss-cpu` as an optional extra and only import it when `STORAGE_BACKEND=file`.
**Pros:** Backwards-compatible for file-backend users.
**Cons:** Still requires building and testing the FAISS path; does not reduce deployment complexity; native-library issues remain for anyone who installs the extra.
**Why Rejected:** The problem statement says FAISS is obsolete and should be removed, not hidden.

### Comparison Matrix

| Criteria | Chosen: Delete FAISS | Alt 1: Keyword fallback | Alt 2: Optional flag |
|----------|----------------------|-------------------------|----------------------|
| Code complexity | Lowest | Medium | High |
| Deployment simplicity | Highest | Medium | Low |
| Search behavior consistency | High | Low | Medium |
| Maintenance burden | Lowest | Medium | High |
| Backwards compatibility for file backend | Breaks semantic search | Preserves partial search | Preserves full search |

## Rollout Plan

- **Phase 1: Implementation** (out of scope for this skill): Apply the file changes listed above.
- **Phase 2: Testing**: Run unit and integration tests per `testing.md`; verify DocumentDB hybrid search against representative queries.
- **Phase 3: Documentation and release notes**: Update docs and publish a release note explaining the breaking change for operators still relying on FAISS.
- **Phase 4: Deployment**: Build updated containers, run smoke tests, and monitor search success-rate metrics.

## Open Questions

1. Are there any production deployments still using file-backend + FAISS that need a migration window?
2. Should the telemetry collector schema continue to accept `faiss` as a legacy value for a transition period, or can it be rejected immediately?
3. Do downstream dashboards need a coordinated rename of `faiss_search_time_ms` to `vector_search_time_ms`?

## References

- `registry/repositories/documentdb/search_repository.py` - existing hybrid search implementation
- `docs/design/database-abstraction-layer.md` - prior design describing FAISS vs DocumentDB backends
- `docs/design/storage-architecture-mongodb-documentdb.md` - backend comparison
