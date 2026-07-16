# GitHub Issue: Migrate ECS plaintext secrets to AWS Secrets Manager

## Title
Migrate ECS task-definition secrets from plaintext `environment` to AWS Secrets Manager `secrets` references

## Labels
- security
- terraform
- aws
- enhancement
- infra

## Description

### Problem Statement
The Terraform ECS deployment under `terraform/aws-ecs/` currently passes many sensitive values as plaintext container `environment` variables. Anyone with `ecs:DescribeTaskDefinition` or read access to the ECS console / AWS CLI can retrieve DB passwords, API keys, bearer tokens, OAuth client secrets, and private keys. This violates the principle of least exposure and complicates audit, rotation, and encryption-at-rest guarantees.

Moving these values to AWS Secrets Manager and referencing them via the ECS `secrets` block provides:
- KMS encryption at rest
- Fine-grained IAM access control
- CloudTrail audit logging
- A consistent path for future automatic rotation

### Proposed Solution
1. Create new AWS Secrets Manager resources (KMS-encrypted with the existing `aws_kms_key.secrets`) for every sensitive value that is currently passed as a plaintext ECS `environment` variable.
2. Move those variables from the `environment` block to the `secrets` block in the affected ECS task definitions (`auth-server`, `registry`, `grafana`, and `grafana-config`).
3. Update the ECS task execution IAM policy (`aws_iam_policy.ecs_secrets_access`) to include the ARNs of the new secrets.
4. Add Grafana task execution role access to Secrets Manager so the Grafana admin password can also be migrated.
5. Keep the existing environment-variable names unchanged so application code continues to work without modification.
6. Update `terraform/aws-ecs/README.md`, `.env.example`, and `terraform.tfvars.example` to document the new secure-by-default behavior.

### User Stories
- As an AWS operator deploying the registry, I want sensitive configuration to be retrieved from Secrets Manager so that plaintext secrets are not visible in task definitions or the ECS console.
- As a security reviewer, I want all credentials stored in Secrets Manager so that access, rotation, and audit can be managed in one place.
- As a platform engineer, I want the migration to keep the same environment-variable interface so that no application code changes are required.

### Acceptance Criteria
- [ ] `REGISTRY_API_TOKEN`, `REGISTRY_API_KEYS`, `FEDERATION_STATIC_TOKEN`, `FEDERATION_ENCRYPTION_KEY`, `ANS_API_KEY`, `ANS_API_SECRET`, `AUTH0_MANAGEMENT_API_TOKEN`, `REGISTRATION_WEBHOOK_AUTH_TOKEN`, `REGISTRATION_GATE_AUTH_CREDENTIAL`, `REGISTRATION_GATE_OAUTH2_CLIENT_SECRET`, `GITHUB_PAT`, and `GITHUB_APP_PRIVATE_KEY` are no longer present in plaintext `environment` blocks for the `auth-server` or `registry` services.
- [ ] New `aws_secretsmanager_secret` and `aws_secretsmanager_secret_version` resources are created for each migrated secret, using the existing `aws_kms_key.secrets` KMS key and following the naming/lifecycle patterns already used for `entra_client_secret`, `okta_client_secret`, etc.
- [ ] The ECS `secrets` blocks for `auth-server` and `registry` reference the new Secrets Manager ARNs using the same environment-variable names.
- [ ] `GF_SECURITY_ADMIN_PASSWORD` is migrated from plaintext to Secrets Manager for both the `grafana` and `grafana-config` containers.
- [ ] The `aws_iam_policy.ecs_secrets_access` policy includes all new secret ARNs.
- [ ] The Grafana task execution role is granted `SecretsManagerAccess` so it can read the Grafana admin password secret.
- [ ] `terraform plan` succeeds with no plaintext secret values rendered in task definitions.
- [ ] `terraform/aws-ecs/README.md` Security Considerations section is updated to state that all credentials are stored in Secrets Manager.
- [ ] `.env.example` and `terraform.tfvars.example` are updated to note which variables are Secrets Manager-only in ECS and may still be used locally/Docker Compose.
- [ ] All existing functionality remains intact; no application source-code changes are required.

### Out of Scope
- Helm / EKS deployment surfaces (no parity required).
- Automatic rotation of the newly migrated secrets (only storage and referencing change; rotation can be added later).
- Changes to SSM Parameter Store usage in `keycloak-ecs.tf` (Keycloak admin username/password and database URL remain in SSM for now).
- Adding new secrets to the `mcpgw` service unless an existing feature flag or wiring already requires them.
- Changes to the application Python config loaders, because ECS `secrets` injection preserves the existing env-var interface.

### Dependencies
- Existing `aws_kms_key.secrets` and `aws_iam_policy.ecs_secrets_access` resources in `terraform/aws-ecs/modules/mcp-gateway/`.
- Existing `terraform-aws-modules/ecs/aws//modules/service` module v6.x used for ECS service definitions.

### Related Issues
- PR #947 introduced `mongodb_connection_string_secret_arn` as the Secrets Manager variant of a MongoDB URI; this issue extends that pattern to all remaining plaintext secrets.
- Issue #1000 added `extra_env` support; the reserved-name lists should continue to block users from overriding migrated secret names.
