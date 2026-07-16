# Expert Review: Migrate ECS plaintext secrets to AWS Secrets Manager

## Review Context
- **Design document:** `./lld.md`
- **GitHub issue:** `./github-issue.md`
- **Scope:** Terraform ECS deployment only; no Helm/EKS parity.

---

## Frontend Engineer — Pixel

### Strengths
- The design preserves the same environment-variable names, so no frontend or Grafana configuration URL changes are required.
- Grafana admin password migration is explicitly covered, including the `grafana-config` sidecar that talks to the Grafana HTTP API.

### Concerns
- The Grafana `grafana-config` sidecar interpolates `GF_SECURITY_ADMIN_PASSWORD` into shell `curl` commands. Moving the value to `secrets` still leaves it exposed in the sidecar's process command line and potentially in CloudWatch logs if `set -x` or error output is enabled. The design does not mention masking or using a credentials file.
- No UI or documentation is proposed to tell operators where to find or rotate secrets after deployment.

### New libraries / infra dependencies
None.

### Better alternatives considered
- For Grafana specifically, consider using a post-deployment Job or Lambda instead of an inline sidecar shell script that passes the password on the command line.

### Recommendations
1. Verify that the `grafana-config` sidecar does not log `GF_SECURITY_ADMIN_PASSWORD`; if it does, redirect stderr and avoid `set -x`.
2. Add a short operator-facing note in the README showing how to update a secret value after initial deployment (`aws secretsmanager put-secret-value`).

### Questions for author
1. Does the `grafana-config` sidecar log the Grafana admin password on failure today, and will moving it to `secrets` change that behavior?
2. Is there a UI or CLI workflow planned for rotating these secrets without a full Terraform apply?

### Verdict
**APPROVED WITH CHANGES** — address Grafana sidecar command-line exposure and add operator documentation.

---

## Backend Engineer — Byte

### Strengths
- The design correctly identifies that ECS `secrets` injection uses the same runtime env-var interface, so no Python source changes are needed.
- The conditional `count` approach keeps optional features optional without forcing users to create empty secrets.
- The plan reuses existing patterns (`name_prefix`, `kms_key_id`, `lifecycle { ignore_changes = [secret_string] }`), making the change consistent with already-reviewed code.

### Concerns
- The `registry_api_keys` variable is a JSON string containing multiple keys and group mappings. Storing it as a single `secret_string` is simple, but any partial rotation requires rewriting the whole JSON blob. There is no schema validation at the Terraform layer.
- `GITHUB_APP_PRIVATE_KEY` is a multi-line PEM value. `aws_secretsmanager_secret_version.secret_string` accepts multiline strings, but special care is needed during `terraform plan` to avoid spurious diffs caused by newline normalization.
- The design does not explicitly handle the case where a user later sets a migrated variable to `""`. The secret version resource would be destroyed; ensure the ECS `secrets` conditional also removes the reference so the task definition remains valid.

### New libraries / infra dependencies
None.

### Better alternatives considered
- Splitting `registry_api_keys` into a structured Secrets Manager secret with separate keys per group. This would improve rotation granularity but adds complexity and is not required for this security hardening issue.

### Recommendations
1. Add a validation block or `precondition` to reject obviously invalid `registry_api_keys` values when the feature is enabled.
2. Use `trimspace()` or a `lifecycle { ignore_changes }` strategy for `GITHUB_APP_PRIVATE_KEY` to avoid newline drift.
3. Explicitly test the "empty value after previously non-empty" path to ensure Terraform destroys the secret version and removes the ECS secret reference cleanly.

### Questions for author
1. Should `registry_api_keys` be stored as a JSON object in Secrets Manager instead of an opaque string to enable future per-key rotation?
2. How will the implementation handle newline characters in `GITHUB_APP_PRIVATE_KEY` during `terraform plan`?

### Verdict
**APPROVED WITH CHANGES** — add validation and newline handling for multiline secrets.

---

## SRE / DevOps Engineer — Circuit

### Strengths
- The IAM policy expansion is explicit and follows the existing conditional concat pattern.
- The design includes a deployment checklist covering all surfaces (module files, root variables, README, examples).
- No new providers or modules are introduced, reducing rollout risk.

### Concerns
- Adding up to 13 new Secrets Manager resources increases Terraform plan/apply time and state size. The impact is small but should be noted.
- The Grafana task execution role change is a breaking IAM change for existing observability deployments; operators applying this will see a task execution role policy attachment update. The design does not call out a rolling-restart strategy.
- The design does not mention drift detection. If an operator updates a secret value directly in Secrets Manager after deployment, `lifecycle { ignore_changes = [secret_string] }` means Terraform will not revert it, which is correct but should be documented.
- `mcpgw` is mentioned as out of scope, but if future features add secret wiring there, the `ecs_secrets_access` policy pattern will need to be repeated.

### New libraries / infra dependencies
None.

### Better alternatives considered
- Use Terraform `moved` blocks if secret resource addresses change during implementation. Not needed for net-new resources but useful if refactoring occurs.

### Recommendations
1. Document that existing ECS tasks will receive the new secret references only after a task definition revision is created and services are restarted.
2. Add a note to the README explaining that direct secret updates in the AWS Console are preserved on subsequent Terraform applies due to `ignore_changes`.
3. Consider adding a `terraform_data` resource or `precondition` to warn when both `mongodb_connection_string` and `mongodb_connection_string_secret_arn` are set.

### Questions for author
1. What is the recommended rollout order: update Terraform first, then cycle services, or vice versa?
2. How will the change affect existing task definitions that already reference some of these values in `environment`? Will AWS create a new revision automatically on apply?

### Verdict
**APPROVED WITH CHANGES** — add rollout and drift-detection notes.

---

## Security Engineer — Cipher

### Strengths
- The design directly addresses the core risk: plaintext secrets in ECS task definitions.
- KMS encryption via the existing `aws_kms_key.secrets` is preserved.
- The IAM policy uses resource-level ARNs rather than wildcard `Resource: "*"` for `secretsmanager:GetSecretValue`.
- Checkov suppression patterns and `lifecycle` handling are consistent with existing secrets.

### Concerns
- `aws_kms_key.secrets` policy currently allows `"AWS": "*"` with a condition matching `*task-exec*` role ARNs. This is broad but consistent with existing design; new secrets do not worsen the posture. However, the KMS key policy should ideally be scoped to the exact task execution role ARNs rather than a wildcard.
- The design stores user-supplied secrets in Terraform state because the variables accept plaintext and pass them to `aws_secretsmanager_secret_version`. While better than plaintext task definitions, sensitive values still exist in Terraform state. The design acknowledges this in the `mongodb_connection_string` deprecation note but does not for the newly migrated secrets.
- `github_app_private_key` is a private key; storing it in Terraform state and then in Secrets Manager creates two encrypted copies. This is acceptable but should be documented.
- No threat model or audit requirement is specified beyond CloudTrail.

### New libraries / infra dependencies
None.

### Better alternatives considered
- Accept secret ARNs instead of plaintext values to remove secrets from Terraform state entirely. Rejected in the LLD for consistency and backwards compatibility, but this remains the stronger security posture.

### Recommendations
1. Add a prominent security note in the README and issue stating that secret values still reside in Terraform state and that state should be encrypted and access-restricted.
2. Consider scoping the KMS key policy condition from `*task-exec*` to the exact ECS task execution role ARNs created by the module.
3. Mark all newly created secret version resources with `sensitive = true` where applicable (the `secret_string` argument is already sensitive by provider default).
4. Add CloudTrail audit instructions to the README.

### Questions for author
1. Is Terraform state currently stored in an S3 bucket with encryption and versioning enabled?
2. Has the team evaluated accepting ARNs for all secrets as a follow-up hardening step?

### Verdict
**APPROVED WITH CHANGES** — document Terraform-state risk and KMS policy scope.

---

## SMTS / Overall — Sage

### Strengths
- The design is simple, follows established repository patterns, and has a clear acceptance-criteria list.
- The scope is well bounded: Terraform-only, no application changes, no Helm/EKS parity.
- The rollout plan and file-change inventory are detailed enough for an entry-level developer to execute.

### Concerns
- The phrase "app config loader changes" appeared in the user's scope statement, but the LLD correctly concludes no Python changes are needed. This should be explicitly reconciled with the user so expectations are aligned.
- The LLD does not include a rollback plan. If a secret is created but the ECS task cannot read it due to an IAM gap, services will fail to start.
- The estimated effort may be optimistic given the number of conditional branches and the need to keep `environment` and `secrets` blocks in sync across auth-server and registry.

### New libraries / infra dependencies
None.

### Better alternatives considered
None beyond those already evaluated in the LLD.

### Recommendations
1. Add a rollback section to the LLD: how to revert a task definition revision or restore an old secret value if deployment fails.
2. Include a pre-apply validation script or `terraform plan` grep that fails the build if any migrated variable name still appears in an `environment` block.
3. Reconcile the "app config loader changes" requirement with the user before implementation begins; if documentation counts as a loader change, clarify that.

### Questions for author
1. Should the implementation include a CI check that scans rendered task definitions for plaintext secret values?
2. Is there a preference for one large PR or smaller per-service PRs?

### Verdict
**APPROVED WITH CHANGES** — add rollback plan and reconcile scope wording with the user.

---

## Review Summary

| Reviewer | Verdict | Blockers | Key Recommendations |
|----------|---------|----------|---------------------|
| Frontend (Pixel) | APPROVED WITH CHANGES | 1 | Address Grafana sidecar command-line exposure; add operator docs |
| Backend (Byte) | APPROVED WITH CHANGES | 2 | Validate `registry_api_keys`; handle PEM newlines |
| SRE (Circuit) | APPROVED WITH CHANGES | 1 | Document rollout order and drift behavior |
| Security (Cipher) | APPROVED WITH CHANGES | 2 | Document Terraform-state risk; scope KMS key policy |
| SMTS (Sage) | APPROVED WITH CHANGES | 1 | Add rollback plan; reconcile "app config loader" scope |

### Next Steps
1. Decide whether to accept ARNs for secrets as a future hardening follow-up.
2. Update the LLD with rollback, validation, and documentation additions requested by reviewers.
3. Confirm with the user that no Python application changes are required.
