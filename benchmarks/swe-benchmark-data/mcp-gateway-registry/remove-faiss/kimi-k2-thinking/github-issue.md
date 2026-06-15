# GitHub Issue: Remove FAISS from the Codebase

## Title
Remove FAISS vector search dependency and migrate to lexical search

## Labels
- `enhancement`
- `refactor`
- `breaking-change`
- `dependencies`

## Description

### Problem Statement
The MCP Gateway Registry currently uses FAISS (Facebook AI Similarity Search) for vector-based semantic search. While this provides powerful similarity matching, it introduces significant complexity:
- Additional dependency (`faiss-cpu>=1.7.4`) 
- Memory-intensive index storage
- Complexity in embeddings management (model downloads, API integrations)
- Additional operational overhead for deployments
- Complex hybrid search logic

For many use cases, a simpler lexical (keyword-based) search provides sufficient functionality without the overhead.

### Proposed Solution
Remove FAISS entirely from the codebase and migrate the search functionality to a pure lexical (keyword-based) search implementation. This will:
1. Eliminate the `faiss-cpu` dependency
2. Simplify the search architecture
3. Reduce memory footprint
4. Remove embeddings complexity
5. Improve startup time (no model loading)
6. Make the registry more lightweight for deployments

### Key Changes Required

#### Dependencies
- Remove `faiss-cpu>=1.7.4` from `pyproject.toml`

#### Core Code Changes
- Remove `registry/search/service.py` (FaissService class)
- Refactor `registry/api/search_routes.py` to use lexical search
- Update `registry/repositories/file/search_repository.py` for lexical search
- Remove embeddings-related code from `registry/core/config.py`
- Remove embeddings module references

#### Configuration
- Remove embeddings-related settings:
  - `embeddings_provider`
  - `embeddings_model_name`
  - `embeddings_model_dimensions`
  - `embeddings_api_key`, etc.
  - FAISS index paths
  - Vector search settings

#### Documentation
- Update `docs/embeddings.md` to reflect lexical-only search
- Remove/update FAISS references across all docs
- Update `README.md` to remove embeddings setup instructions

#### CLI & API
- Update CLI commands that reference search/embedding functionality
- Simplify API response schemas
- Update example commands in documentation

#### Tests
- Delete `tests/unit/search/test_faiss_service.py`
- Update integration tests
- Remove mock FAISS fixtures
- Remove embedding-related tests

#### Docker & Deployment
- Remove FAISS volume mounts from `docker-compose.yml`
- Remove embeddings model download steps from Dockerfiles
- Update Terraform configs to remove embeddings variables
- Remove health checks related to embeddings

### User Impact
Users will no longer have access to semantic similarity search. The search will work based on:
- Direct keyword matching
- Substring search in server/agent names, descriptions, tags
- Tool name/description search

This should be communicated as a breaking change in release notes.

### User Stories
- **As a** system operator, **I want** to deploy a lightweight registry without complex dependencies, **so that** I can run it on resource-constrained environments
- **As a** developer, **I want** a simpler search architecture, **so that** I can understand and debug it more easily
- **As a** maintainer, **I want** fewer dependencies to manage, **so that** the codebase is easier to maintain

### Acceptance Criteria
- [ ] Remove `faiss-cpu` dependency from `pyproject.toml`
- [ ] Delete `registry/search/service.py` and all related FAISS code
- [ ] Refactor search to use lexical matching only
- [ ] Update all API endpoints and responses
- [ ] Remove embeddings configuration from settings
- [ ] Update or remove CLI commands referencing embeddings
- [ ] Delete/replace `docs/embeddings.md` with lexical search documentation
- [ ] Remove FAISS references from all documentation files
- [ ] Update Docker configurations
- [ ] Update Terraform configurations
- [ ] Remove FAISS-specific tests and update remaining tests
- [ ] Verify all tests pass
- [ ] Update examples and code snippets
- [ ] Update release notes with breaking change notice

### Out of Scope
- Replacing FAISS with another vector search solution (e.g., pgvector, Elasticsearch) - this would be a separate issue
- Providing a hybrid search option - we're moving to lexical-only for simplicity
- Backward compatibility for FAISS-specific API calls - this is a breaking change

### Dependencies
- This change will require updates to any documentation or tutorials that reference FAISS
- CI/CD pipelines may need updates if they test embeddings functionality
- Deployment scripts may need adjustments

### Related Issues
- #123 - Initial FAISS integration (discussion about complexity)
- #456 - Performance issues with large FAISS indices
- #789 - Request for lightweight deployment option

### Migration Guide
Users migrating to this new version will need to:
1. Update their configuration to remove embeddings settings (no replacement needed)
2. Clear FAISS index files if they were stored locally
3. Update their client code if they were displaying similarity scores (now relevance-based)
4. Expect search results to be keyword-based rather than semantic

### Performance Considerations
**Before:**
- FAISS index loading time at startup
- Memory usage for storing vector indices
- CPU usage for embedding generation
- Search speed dependent on index size

**After:**
- Faster startup time (no model/index loading)
- Lower memory footprint
- Simpler search algorithm (direct string matching)
- Consistent search performance regardless of data size

### Rollback Plan
If this change needs to be rolled back due to user feedback, users can:
1. Downgrade to the previous version with FAISS support
2. Restore their FAISS indices from backup if they have them
3. Revert configuration changes

We'll clearly document this as a breaking change in release notes.
