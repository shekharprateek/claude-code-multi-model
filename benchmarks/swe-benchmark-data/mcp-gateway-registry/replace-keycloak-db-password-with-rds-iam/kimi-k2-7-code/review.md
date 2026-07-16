# Expert Review: Replace Keycloak static RDS password with RDS IAM authentication

*Date: 2026-07-15*

## Reviewers

| Role | Reviewer | Focus |
|------|----------|-------|
| Frontend Engineer | Pixel | Operator-facing documentation and UX of the feature flag |
| Backend Engineer | Byte | Terraform design, IAM policies, Lambda logic, JDBC integration |
| SRE/DevOps Engineer | Circuit | Deployment, rollback, observability, image lifecycle |
| Security Engineer | Cipher | IAM least privilege, secrets, TLS, bootstrap risk |
| SMTS (Overall) | Sage | Architecture, maintainability, migration path |

---

## Frontend Engineer - Pixel

### Strengths
- The feature flag is simple and aligns with existing flags like `documentdb_use_iam`, so operators will recognize the pattern.
- `terraform.tfvars.example` and `README.md` are explicitly called out for updates, which reduces confusion.

### Concerns
1. **No single source of truth for the auth mode**: The flag lives in Terraform, but the container also infers the mode from the absence of `KC_DB_PASSWORD`. If someone manually adds `KC_DB_PASSWORD` via `keycloak_extra_env` (which does not exist today but could be added later), the behavior becomes ambiguous.
2. **Operator error on image URI**: The precondition catches a missing custom image, but the error message should explicitly tell the operator how to build the image.
3. **Documentation gap**: The README needs a clear "before you enable IAM auth" checklist, including building the image and understanding the 15-minute token behavior.

### Recommendations
- Render an explicit `KEYCLOAK_DB_USE_IAM=true/false` environment variable into the Keycloak container so the mode is visible in the task definition and logs.
- Add a README section titled "Enabling RDS IAM authentication" with numbered steps.
- Consider adding a validation rule that rejects `keycloak_iam_auth_image_uri` when `keycloak_db_use_iam = false` to avoid silent misconfigurations.

### Verdict
**APPROVED WITH CHANGES** - 2 blockers (explicit env var, README checklist).

---

## Backend Engineer - Byte

### Strengths
- The conditional logic in Terraform is clean and keeps the default path unchanged.
- Using the AWS Advanced JDBC Wrapper is the correct technical answer to the 15-minute token lifetime problem.
- The bootstrap Lambda is a pragmatic way to create an Aurora MySQL user that Terraform cannot create directly.

### Concerns
1. **JDBC URL format uncertainty**: `jdbc:aws-wrapper:mysql://...` is the documented wrapper protocol, but Keycloak/Quarkus may require additional `quarkus.datasource.jdbc.driver` configuration. The LLD does not show how to set the driver class.
2. **Lambda idempotency**: The proposed `CREATE USER IF NOT EXISTS` is fine, but `GRANT` statements should also be idempotent or guarded to avoid accumulating duplicate grants.
3. **Proxy role scope**: The RDS Proxy role needs `rds-db:connect` only when the proxy itself authenticates with IAM. If the proxy uses SECRETS for backend auth and the client-to-proxy connection uses IAM, the proxy still needs to validate the IAM token but may not need `rds-db:connect`. The exact AWS requirement should be verified.
4. **SSM parameter for username**: Storing the IAM username in SSM is fine, but it is not a secret. Using a plain `String` type instead of `SecureString` would reduce KMS decrypt calls and cost.

### Recommendations
- Add a Quarkus property override such as `QUARKUS_DATASOURCE_JDBC_DRIVER=software.amazon.jdbc.Driver` when IAM auth is enabled, or confirm that the wrapper URL prefix is sufficient.
- Make the Lambda idempotent by checking user existence before granting.
- Validate the exact IAM actions required for RDS Proxy IAM auth and document the finding.
- Consider changing `aws_ssm_parameter.keycloak_database_iam_username` to type `String`.

### Verdict
**APPROVED WITH CHANGES** - 3 blockers (driver config, Lambda grants, proxy IAM research).

---

## SRE/DevOps Engineer - Circuit

### Strengths
- Default-off feature flag allows safe, gradual rollout.
- The design explicitly avoids changes to Helm/EKS, focusing on the Terraform/ECS surface that operators already use.
- Keeping the master password path intact provides a clear rollback.

### Concerns
1. **Custom image build is a new operational step**: Today the stack works with public images. Requiring operators to build and push a custom Keycloak image increases onboarding friction and CI/CD complexity.
2. **Lambda bootstrap is an apply-time dependency**: If the Lambda fails (e.g., network partition or the cluster is still `creating`), the apply fails. There is no retry or graceful degradation except re-running `terraform apply`.
3. **Rollback during incident**: Toggling `keycloak_db_use_iam` back to `false` changes the task definition and proxy auth. The time to stabilize depends on ECS service deployment speed and Aurora proxy reconfiguration.
4. **Monitoring gaps**: The LLD mentions CloudWatch metrics but does not specify which alarm thresholds or dashboards need updates.

### Recommendations
- Provide a `Makefile` target or script to build and push the custom Keycloak image, similar to the existing `make build-push` workflow.
- Add a `ignore_errors` or retry wrapper around `aws_lambda_invocation`, or document that a failed invocation requires re-running apply.
- Add an explicit rollback runbook section in the README.
- Add a CloudWatch alarm on Keycloak health-check failures with a clear description referencing the auth mode change.

### Verdict
**APPROVED WITH CHANGES** - 3 blockers (image build automation, Lambda resilience, rollback runbook).

---

## Security Engineer - Cipher

### Strengths
- The design removes the static password from the Keycloak container when IAM auth is enabled.
- TLS is enforced on the RDS Proxy when IAM auth is active.
- IAM policies are scoped to the specific DB user resource rather than using wildcard ARNs.
- The master password remains in Secrets Manager with rotation for fallback, rather than being duplicated.

### Concerns
1. **Bootstrap Lambda has broad KMS decrypt**: The Lambda policy allows `kms:Decrypt` on `*`. It should be scoped to the RDS KMS key (`aws_kms_key.rds.arn`).
2. **Bootstrap Lambda can read the master password**: By design, the Lambda must read `keycloak/database` to create the IAM user. Compromise of this Lambda allows password retrieval. This is unavoidable for a bootstrap step but should be documented as a temporary privilege.
3. **Native master user remains**: Even with IAM auth enabled, the native `keycloak` user with a static password still exists in Aurora. The Secrets Manager rotation continues, which is good, but the account is still a persistent credential.
4. **SQL injection risk in Lambda**: The Lambda constructs SQL strings from SSM parameters. While SSM is controlled by Terraform, f-string SQL should be replaced with parameterized statements where possible (MySQL DDL does not support placeholders for identifiers, so allowlist validation is needed).

### Recommendations
- Scope the Lambda KMS policy to `aws_kms_key.rds.arn`.
- Add a comment and documentation warning that the bootstrap Lambda has temporary read access to the master password.
- Validate `keycloak_rds_iam_username` with a regex in `variables.tf` to prevent SQL injection through the Lambda.
- Consider adding a checkov/terrascan suppression comment with justification for the `rds-db:connect` wildcard-like resource (the cluster resource ID is dynamic but scoped).

### Verdict
**APPROVED WITH CHANGES** - 3 blockers (KMS scope, Lambda warning, username validation).

---

## SMTS - Sage

### Strengths
- The design correctly identifies the hardest part of the problem (15-minute token lifetime) and selects the AWS-recommended JDBC wrapper solution.
- Feature-flag gating keeps the change backwards-compatible and minimizes blast radius.
- The design respects the existing module boundary by not pushing Keycloak DB concerns into `modules/mcp-gateway`.

### Concerns
1. **Migration path for existing deployments is vague**: The LLD says migration is out of scope, but operators will need guidance on how to cut over an existing cluster from password user to IAM user without data loss.
2. **Custom image ownership**: The project currently publishes public ECR images. If IAM auth requires a custom image, the project should either publish the custom image or clearly state that operators must maintain it.
3. **Dependency on a third-party JAR**: The AWS JDBC wrapper is an external dependency that must be tracked for security updates. The current design pins a version via build arg, which is good, but there is no mention of automated update checks.
4. **Test coverage**: The LLD points to `testing.md` but the design itself does not describe how to unit-test the conditional Terraform logic or the Lambda SQL execution.

### Recommendations
- Include a migration runbook in the README that covers: (1) build image, (2) apply with flag off to create IAM user, (3) verify IAM user, (4) flip flag on.
- Decide whether the project will publish the custom Keycloak image; if yes, update CI/CD docs.
- Add a dependabot/renovate configuration or a scheduled task to check the AWS JDBC wrapper version.
- Add pytest unit tests for the Lambda and Terraform plan tests for both flag states.

### Verdict
**APPROVED WITH CHANGES** - 3 blockers (migration runbook, image ownership, test plan details).

---

## Review Summary Table

| Reviewer | Verdict | Blockers | Key Recommendations |
|----------|---------|----------|---------------------|
| Pixel (Frontend/UX) | APPROVED WITH CHANGES | 2 | Explicit `KEYCLOAK_DB_USE_IAM` env var; README checklist |
| Byte (Backend) | APPROVED WITH CHANGES | 3 | Confirm Quarkus driver config; idempotent Lambda grants; research proxy IAM |
| Circuit (SRE) | APPROVED WITH CHANGES | 3 | Image build script; Lambda resilience; rollback runbook |
| Cipher (Security) | APPROVED WITH CHANGES | 3 | Scope Lambda KMS; document temporary password read; validate username |
| Sage (SMTS) | APPROVED WITH CHANGES | 3 | Migration runbook; custom image ownership; dependency update tracking |

## Next Steps

1. Address the blockers above and update the LLD.
2. Confirm Quarkus/JDBC wrapper configuration with a small proof-of-concept build.
3. Flesh out `testing.md` with Terraform plan tests and Lambda unit tests.
