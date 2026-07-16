# Expert Review: Remove FAISS from the codebase

## Review Summary

| Reviewer | Verdict | Blockers | Key Recommendations |
|----------|---------|----------|---------------------|
| Frontend (Pixel) | APPROVED | 0 | No UI changes; ensure search API responses remain unchanged. |
| Backend (Byte) | APPROVED WITH CHANGES | 2 | (1) Clarify file-backend search fallback. (2) Add migration path for telemetry field rename. |
| SRE (Circuit) | APPROVED WITH CHANGES | 1 | Remove FAISS index file cleanup from `build_and_run.sh` and document disk cleanup for operators. |
| Security (Cipher) | APPROVED | 0 | No security impact; removal of native dependency reduces attack surface. |
| SMTS (Sage) | APPROVED WITH CHANGES | 2 | (1) Confirm file backend deprecation scope. (2) Provide data migration guidance. |

---

## Frontend Engineer - Pixel

### Strengths
- The change is backend-only from the user's perspective; all existing search endpoints keep their contracts.
- DocumentDB hybrid search already returns the same grouped result shape (`servers`, `agents`, `tools`, etc.), so UI consumers do not need updates.

### Concerns
- None significant for the frontend layer.

### New libraries / infra dependencies required
- None for the frontend.

### Better alternatives considered
- N/A: no frontend alternatives needed.

### Recommendations
- Verify that the search response schema (`SemanticSearchResponse`) is unchanged after removing FAISS-shaped fallback data in tests.
- Keep the existing OpenAPI description for `/semantic` backend-agnostic.

### Questions for author
- Are there any frontend tests or Storybook fixtures that reference FAISS result shapes?

### Verdict
**APPROVED**

---

## Backend Engineer - Byte

### Strengths
- The design correctly identifies the redundant backend and provides a concrete deletion list.
- Reusing `DocumentDBSearchRepository` avoids re-implementing search.
- Replacing lazy `faiss_service` imports with `get_search_repository()` follows the existing repository pattern.

### Concerns
1. **File backend semantic search is silently lost.** The design says DocumentDB becomes the required search backend, but the repo still advertises a file backend for server storage. If `STORAGE_BACKEND=file` is used, search will still route to DocumentDB, which may surprise operators who expected a fully file-based deployment.
2. **Telemetry field rename is breaking.** Changing `faiss_search_time_ms` to `vector_search_time_ms` without a transition plan will break existing metrics consumers and dashboards.
3. **Test patch target churn is large.** Many tests patch `registry.search.service.faiss_service`. Updating all of them is error-prone.

### New libraries / infra dependencies required
- None. `faiss-cpu` is removed.

### Better alternatives considered
- Keep the old metric key as an alias for one release to give consumers time to migrate.

### Recommendations
1. Explicitly document that file-backend deployments now require DocumentDB for semantic search, or add a startup guard that fails fast if DocumentDB is unavailable.
2. For the metrics rename, either:
   a. Emit both `faiss_search_time_ms` and `vector_search_time_ms` for one release, or
   b. Bump the metrics schema version and document the breaking change in release notes.
3. Introduce a single `search_repository` fixture in `tests/conftest.py` so future test updates do not need to patch multiple targets.

### Questions for author
- What happens at startup if `STORAGE_BACKEND=file` but DocumentDB is not configured?
- Is there a plan to deprecate the file backend entirely, or only its search component?

### Verdict
**APPROVED WITH CHANGES**

---

## SRE / DevOps Engineer - Circuit

### Strengths
- Removing `faiss-cpu` will reduce container image size and eliminate platform-specific wheel issues.
- Deleting the FAISS index file checks from `build_and_run.sh` simplifies startup.
- DocumentDB is a managed service, which improves operational reliability compared to an in-memory index.

### Concerns
1. **Leftover FAISS index files.** Existing deployments may have `service_index.faiss` and `service_index_metadata.json` on disk. The design does not say whether these should be cleaned up.
2. **Startup behavior change.** File-backend deployments previously rebuilt the FAISS index on boot; after the change, they rely on DocumentDB persistence. Operators need to know this.
3. **Terraform and compose comments** mention FAISS in descriptions that are user-visible. These need to be updated before the next release.

### New libraries / infra dependencies required
- None. `faiss-cpu` is removed.

### Better alternatives considered
- N/A.

### Recommendations
1. Add a one-time cleanup note in the release notes instructing operators to delete `service_index.faiss` and `service_index_metadata.json` from persistent volumes.
2. Update all user-facing descriptions in Docker Compose, build config, and Terraform to remove FAISS.
3. Verify that container health checks do not depend on FAISS index files being present.

### Questions for author
- Are there any volume mounts in Docker Compose or Terraform that exist solely for FAISS index files?
- Should the registry delete legacy FAISS files automatically on first startup?

### Verdict
**APPROVED WITH CHANGES**

---

## Security Engineer - Cipher

### Strengths
- Removing a native C++ dependency reduces the supply-chain and native-code attack surface.
- No new user input paths are introduced.
- The change does not affect authN/authZ flows.

### Concerns
- None.

### New libraries / infra dependencies required
- None.

### Better alternatives considered
- N/A.

### Recommendations
- Run `bandit` and dependency scanning after `faiss-cpu` removal to confirm no unused imports or vulnerable transitive dependencies remain.

### Questions for author
- Are there any secrets or environment variables that were only used for FAISS model caching?

### Verdict
**APPROVED**

---

## SMTS - Sage

### Strengths
- The design is focused and addresses the root problem: FAISS is duplicated capability.
- The file deletion list is comprehensive.
- The test update plan covers unit, integration, and infrastructure tests.

### Concerns
1. **Scope of file-backend deprecation.** Removing FAISS search while keeping file server storage creates an awkward split: data lives in files, search lives in DocumentDB. This may confuse the abstraction.
2. **Data migration guidance is missing.** Operators who have a file backend with an existing FAISS index need to know how to move to DocumentDB-only search. The design says this is out of scope, but it is a real operational concern.
3. **Metrics backwards compatibility.** The telemetry collector schema and metrics field rename need a clear transition strategy.

### New libraries / infra dependencies required
- None.

### Better alternatives considered
- A phased deprecation where FAISS is first marked deprecated in logs and docs, then removed in the following release.

### Recommendations
1. Add an explicit architectural decision record (ADR) or release note stating that semantic search is now DocumentDB-only and file backend is storage-only.
2. Provide a short migration checklist for operators moving from FAISS to DocumentDB search.
3. Consider keeping the old metrics field as a deprecated alias for one release to avoid breaking dashboards.
4. Update the repository factory docstring to explain that search is now backend-agnostic from the caller's point of view but always resolves to DocumentDB.

### Questions for author
- Does the team intend to remove the file backend entirely in a future release?
- How will the registry behave if DocumentDB hybrid search is unavailable at runtime?

### Verdict
**APPROVED WITH CHANGES**

---

## Consolidated Next Steps

1. **Clarify file backend semantics** in the LLD and release notes: DocumentDB is required for search.
2. **Add migration guidance** for operators with legacy FAISS index files.
3. **Decide on metrics field transition**: alias for one release vs hard rename.
4. **Update all user-facing deployment descriptions** (Docker Compose, Terraform, build config) to remove FAISS.
5. **Run the test plan in `testing.md`** after implementation.
