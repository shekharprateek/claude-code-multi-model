# GitHub Issue: Remove FAISS from the codebase

## Title
Remove FAISS vector search dependency and code paths

## Labels
- refactor
- dependencies
- search
- documentation
- breaking-change

## Description

### Problem Statement
The registry currently carries `faiss-cpu` as a runtime dependency and maintains a file-backed `FaissService` for vector similarity search. DocumentDB now provides native hybrid search (text + vector via HNSW), which is already implemented in `DocumentDBSearchRepository`. FAISS therefore adds deployment complexity (native C++ libraries, large wheels, numpy version constraints) and duplicate code without adding unique capability.

Removing FAISS simplifies the codebase, shrinks container images, eliminates a native dependency, and aligns all deployments on the maintained DocumentDB search path.

### Proposed Solution
1. Delete `registry/search/service.py` (the `FaissService` implementation).
2. Delete `registry/repositories/file/search_repository.py` (`FaissSearchRepository`).
3. Update `registry/repositories/factory.py` so `get_search_repository()` always returns `DocumentDBSearchRepository`.
4. Remove `faiss-cpu` from `pyproject.toml` and regenerate `uv.lock`.
5. Remove FAISS-related settings (`faiss_index_path`, `faiss_metadata_path`) and the `FaissMetadata` schema.
6. Replace all direct `faiss_service` imports/calls in routes and services with calls through `get_search_repository()`.
7. Remove FAISS build steps, comments, and verification logic from Dockerfiles, compose files, `build_and_run.sh`, and Terraform scripts.
8. Update metrics and telemetry to replace `faiss_search_time_ms` with a backend-agnostic name (`vector_search_time_ms` / `search_time_ms`) and remove the `faiss` option from the `search_backend` enum.
9. Delete FAISS-specific tests and mocks; update remaining tests to use `DocumentDBSearchRepository` or repository mocks.
10. Update documentation to remove FAISS references and mark file-backend semantic search as deprecated/removed.

### User Stories
- As an operator deploying the registry, I no longer need to worry about FAISS native-library compatibility so that container builds are faster and more portable.
- As a developer, I want a single search backend so that I do not have to maintain two parallel indexing and search implementations.
- As an end-user, I want semantic search results to remain available and equivalent to the previous FAISS-based behavior.

### Acceptance Criteria
- [ ] `faiss-cpu` is removed from `pyproject.toml` and `uv.lock`.
- [ ] `import faiss`, `FaissService`, `FaissSearchRepository`, and `FaissMetadata` no longer exist in the codebase.
- [ ] `get_search_repository()` always returns `DocumentDBSearchRepository`, regardless of `STORAGE_BACKEND`.
- [ ] All route and service code that previously imported `faiss_service` directly now uses `get_search_repository()`.
- [ ] Dockerfiles, compose files, `build_and_run.sh`, and Terraform scripts no longer reference FAISS index files or build steps.
- [ ] Telemetry/metrics fields referencing FAISS are renamed or removed with backwards-compatible handling documented.
- [ ] All FAISS-specific tests and mocks are deleted and the remaining test suite passes.
- [ ] Documentation no longer describes FAISS as a supported or active search backend.
- [ ] DocumentDB hybrid search covers the FAISS use cases: server search, agent search, tool search, tag search, and mixed semantic search.

### Out of Scope
- Rewriting the DocumentDB hybrid search algorithm (it already exists and should be reused).
- Removing the file-based server/agent repositories; only search is being consolidated.
- Changing embedding model configuration or the embeddings client.
- Production data migration from FAISS index files to DocumentDB (assumed already completed or not required because DocumentDB already owns persistence).

### Dependencies
- DocumentDB native hybrid search must be confirmed working before this change merges.

### Related Issues
- #452 (FAISS removal tracking)
- #1285 (this issue)
