# Expert Review: Remove EFS from Terraform AWS ECS Deployment

*Review Date: 2026-07-15*
*Design Under Review: `lld.md`*

## Reviewers

| Role | Reviewer | Focus |
|------|----------|-------|
| Frontend Engineer | Pixel | Documentation clarity, operator-facing output and runbooks |
| Backend Engineer | Byte | Terraform module structure, reference integrity, state migration |
| SRE / DevOps Engineer | Circuit | Deployment safety, rollback, operational impact |
| Security Engineer | Cipher | IAM, network segmentation, data protection |
| SMTS | Sage | Architecture, maintainability, scope completeness |

---

## Frontend Engineer - Pixel

### Strengths

- The documentation updates are scoped clearly, including line numbers for every doc file that mentions EFS.
- The 403 troubleshooting rewrite in `docs/deployment-modes.md` gives operators a single, correct command to run.

### Concerns

1. **Operator experience for existing deployments:** Removing `mcp_gateway_efs_id` from `post-deployment-setup.sh` will cause the script to fail on first run for anyone upgrading an existing stack until they re-run `terraform output`. This is not called out as a prerequisite.
2. **Documentation discoverability:** The LLD does not propose updating `terraform/aws-ecs/scripts/README.md` if that file references `run-scopes-init-task.sh` or EFS steps. A stale script README is worse than no README.
3. **No mention of `terraform output -json` refresh:** After the outputs are removed, any saved `terraform-outputs.json` files in operator workspaces will contain stale EFS keys. Operators need to regenerate the file before running post-deployment scripts.

### New Libraries / Infra Dependencies

None.

### Better Alternatives Considered

- Add an explicit "Before you upgrade" note in the top-level README and `terraform/aws-ecs/README.md` telling operators to refresh `terraform-outputs.json` and destroy EFS data before applying.

### Recommendations

1. Verify whether `terraform/aws-ecs/scripts/README.md` exists and references the EFS init path; update or remove those references.
2. Add a pre-upgrade checklist to the documentation artifacts.

### Questions for Author

- Is there a migration guide planned for operators who have data in the existing EFS file system?

### Verdict

**APPROVED WITH CHANGES** - Minor documentation/operator-experience gaps.

---

## Backend Engineer - Byte

### Strengths

- The file-by-file change list is precise and maps directly to the locations found during analysis.
- The design correctly identifies that the registry service already removed EFS mounts and avoids redundant changes there.
- Removing `SCOPES_CONFIG_PATH` from the auth-server container is the right call because the default AWS ECS backend is DocumentDB.

### Concerns

1. **Reference completeness in `ecs-services.tf`:** The LLD correctly targets the auth-server and mcpgw blocks, but it should explicitly instruct the implementer to run `grep -R "module.efs" terraform/aws-ecs` after editing to catch any missed references.
2. **`mcp-logs` volume removal:** The design removes the `mcp-logs` EFS mount from auth-server based on the assumption that CloudWatch logging is sufficient. This is reasonable, but if any application code writes to `/app/logs` and expects the directory to exist on a writable filesystem, Fargate ephemeral storage is writable so this should be fine. The LLD should explicitly mention this verification.
3. **No root variable removal:** The LLD states that root variables do not exist, but it should also confirm that no `terraform.tfvars.example` lines reference EFS, just to be safe.
4. **`run-scopes-init-task.sh` references:** The script references `build-and-push-scopes-init.sh`, which does not exist in the repo. The design does not clarify whether the missing build script should also be removed or if it exists elsewhere. The agent found no such file, so the EFS init path is effectively already broken.

### New Libraries / Infra Dependencies

None.

### Better Alternatives Considered

- Leave `SCOPES_CONFIG_PATH` pointing to a local path such as `/app/auth_server/scopes.yml` to preserve the non-DocumentDB backend fallback. Rejected because the AWS ECS deployment defaults to DocumentDB and the environment variable is unused.

### Recommendations

1. Add a verification grep step to the implementation plan.
2. Confirm `/app/logs` is not required to be an EFS mount by checking the auth-server logging setup.
3. Explicitly state that the missing `build-and-push-scopes-init.sh` confirms the EFS path is unmaintained and safe to remove.

### Questions for Author

- Have you verified that no other module output (for example, outputs consumed by Keycloak or telemetry-collector modules) references the EFS outputs?

### Verdict

**APPROVED WITH CHANGES** - Add a final grep verification and clarify the `/app/logs` assumption.

---

## SRE / DevOps Engineer - Circuit

### Strengths

- The design treats EFS removal as a destroy-and-recreate change, which is honest. Many designs pretend stateful removal is risk-free.
- The post-deployment script cleanup is included, which prevents a dangling EFS branch from failing silently.
- The plan preserves the DocumentDB scopes initialization path, which is the only supported path after EFS removal.

### Concerns

1. **Terraform destroy behavior:** When `storage.tf` is deleted, Terraform will plan to destroy the EFS file system, access points, mount targets, and security group. This is a destructive operation. The LLD should require a non-production `terraform plan` review before apply and suggest `terraform plan -destroy` or targeted review.
2. **Task definition rollouts:** Removing `mountPoints` and `volume` from ECS task definitions will trigger a new deployment. The design should note that services will restart and that operators should expect brief downtime unless a blue/green rollout is used.
3. **Post-deployment script failure mode:** The proposed replacement for the EFS branch is to log an error and increment `STEPS_FAILED`. This is correct, but the script should also return a non-zero exit code so CI/CD pipelines fail fast.
4. **No rollback procedure:** The design mentions a rollout plan but does not describe rollback. If services fail to start after removing mounts, the fastest rollback is to revert the Terraform changes and re-apply. This should be documented.
5. **State file cleanup:** If the EFS module is deleted, operators with existing state may see orphaned resources if Terraform state is not refreshed. The design should mention `terraform refresh` or `terraform plan` review.

### New Libraries / Infra Dependencies

None.

### Better Alternatives Considered

- Use `terraform state rm` to remove EFS resources from state without destroying them, then delete them manually. Rejected because it leaves unmanaged resources and contradicts the goal of reducing cost.

### Recommendations

1. Add a pre-apply checklist that includes reviewing the destroy plan and backing up any data that might still be on EFS.
2. Document the expected service restart behavior.
3. Verify that the post-deployment script exits non-zero when `STEPS_FAILED` increments.
4. Add a rollback section to the rollout plan.

### Questions for Author

- Is there a process for draining or backing up the EFS file system before Terraform destroys it?

### Verdict

**APPROVED WITH CHANGES** - Add destroy-plan review, service-restart notes, and rollback guidance.

---

## Security Engineer - Cipher

### Strengths

- The design removes an NFS security group and an unnecessary network-accessible managed service, which reduces the attack surface.
- The IAM section correctly notes that no explicit `elasticfilesystem:*` permissions are granted to ECS tasks, so no task-level IAM cleanup is required.
- Removing EFS also removes a data-at-rest encryption boundary that had to be managed separately; all persistent data will now be in DocumentDB/S3, which have consistent KMS encryption.

### Concerns

1. **EFS security group egress rule:** The `aws_vpc_security_group_egress_rule` attached to the EFS module allows all outbound (`0.0.0.0/0`). Removing the entire module eliminates this rule, which is good, but the design should verify that no other security group rule references the EFS security group ID.
2. **`SCOPES_CONFIG_PATH` removal:** The auth-server will no longer be able to load scopes from a local file path in ECS. This is acceptable because DocumentDB is the configured backend, but any future deployment that sets `storage_backend` to a non-MongoDB value will fail to load scopes. The LLD should flag this as a deployment constraint.
3. **Documentation IAM example:** Removing `elasticfilesystem:*` from the operator IAM policy is correct, but the design should also verify that no other documentation or example policy still includes it.
4. **Transit encryption:** The EFS mounts used `transit_encryption = "ENABLED"`. After removal, traffic between ECS tasks and DocumentDB/S3 uses TLS by default, which is equivalent or better, but this should be stated explicitly for auditors.

### New Libraries / Infra Dependencies

None.

### Better Alternatives Considered

- Restrict the auth-server to use a non-EFS local scope path even when not using DocumentDB. Rejected because it is outside the Terraform scope and the application already has a file-backend fallback.

### Recommendations

1. Add a security-impact note confirming that all persistent data is now encrypted by DocumentDB/S3 KMS and that no NFS surface remains.
2. Document that AWS ECS deployments must use a MongoDB-compatible backend for scopes.
3. Search all example IAM policies across docs for `elasticfilesystem`.

### Questions for Author

- Are there any compliance or audit artifacts that list EFS as an in-scope data store and need updating?

### Verdict

**APPROVED WITH CHANGES** - Add security-impact note and backend constraint documentation.

---

## SMTS - Sage

### Strengths

- The design is cohesive: it removes EFS from infrastructure, task definitions, variables, outputs, scripts, and documentation.
- The codebase analysis accurately reflects the current partial-removal state.
- The alternatives comparison clearly justifies full removal over conditional retention.

### Concerns

1. **Scope creep risk:** The LLD deletes three files and modifies eleven. This is a medium-sized change. The implementer should be careful not to accidentally remove the wrong volumes (for example, the registry service's empty `volume = {}` should remain empty, not be removed).
2. **Testing coverage:** The design points to `testing.md` but does not specify how to verify that no EFS references remain. A grep-based test should be part of the acceptance criteria.
3. **Historical release notes:** The design correctly leaves `release-notes/v1.0.6.md` untouched but does not propose adding a new release note entry for this change. A release note should be added under `release-notes/` as part of implementation.
4. **Cross-module dependency check:** The LLD assumes no other module consumes the EFS outputs. This should be verified by grepping the entire `terraform/` tree for `mcp_gateway.efs_*` and `module.efs`.

### New Libraries / Infra Dependencies

None.

### Better Alternatives Considered

- Implement a deprecation period with warning logs before removing EFS. Rejected because the application already does not use EFS and the cost is ongoing.

### Recommendations

1. Add a mandatory grep verification to the implementation checklist.
2. Include a release note template in the LLD or testing plan.
3. Verify cross-module references before implementation.

### Questions for Author

- How will the benchmark judge the quality of the final implementation? Will a clean `terraform validate` and zero EFS grep hits be sufficient?

### Verdict

**APPROVED WITH CHANGES** - Add grep-based verification and release note guidance.

---

## Review Summary

| Reviewer | Verdict | Blockers | Key Recommendations |
|----------|---------|----------|---------------------|
| Frontend (Pixel) | APPROVED WITH CHANGES | 0 | Update script README; add pre-upgrade checklist for `terraform-outputs.json` refresh |
| Backend (Byte) | APPROVED WITH CHANGES | 0 | Add final grep verification; confirm `/app/logs` ephemeral-storage assumption |
| SRE (Circuit) | APPROVED WITH CHANGES | 0 | Add destroy-plan review, service-restart notes, and rollback guidance |
| Security (Cipher) | APPROVED WITH CHANGES | 0 | Document encryption impact and MongoDB-compatible backend constraint |
| SMTS (Sage) | APPROVED WITH CHANGES | 0 | Add grep-based verification and release note guidance |

## Next Steps

1. Address the five review feedback themes:
   - Documentation/operator-experience (Pixel)
   - Verification grep and `/app/logs` confirmation (Byte)
   - Destroy-plan and rollback guidance (Circuit)
   - Security-impact note (Cipher)
   - Grep verification and release note (Sage)
2. Update `testing.md` to include the grep-based acceptance tests and destroy-plan review checklist.
3. Proceed to implementation only after the artifacts are revised and the review feedback is incorporated.
