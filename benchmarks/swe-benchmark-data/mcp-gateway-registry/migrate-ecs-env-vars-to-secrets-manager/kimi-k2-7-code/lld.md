# Low-Level Design: Migrate ECS plaintext secrets to AWS Secrets Manager

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
The Terraform ECS deployment in `terraform/aws-ecs/` passes sensitive values such as API tokens, encryption keys, OAuth secrets, and private keys as plaintext container `environment` variables. This exposes them to any principal with `ecs:DescribeTaskDefinition`, the ECS console, or AWS CLI read access, and prevents centralized audit and rotation.

### Goals
- Move all secret-like ECS `environment` variables into AWS Secrets Manager.
- Reference those secrets via the ECS container `secrets` block so values are injected at runtime.
- Preserve the existing environment-variable names so application code is unchanged.
- Update IAM policies and documentation to reflect the new secure-by-default behavior.

### Non-Goals
- Helm / EKS parity.
- Automatic rotation of migrated secrets.
- Changing SSM Parameter Store usage for Keycloak admin/database URL.
- Modifying application Python source code (the env-var interface is unchanged).

## Codebase Analysis

### Key Files Reviewed

| File/Directory | Purpose | Relevance to This Change |
|----------------|---------|--------------------------|
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | ECS task definitions for auth-server, registry, mcpgw, demo servers | Source of plaintext `environment` variables that must move to `secrets` |
| `terraform/aws-ecs/modules/mcp-gateway/secrets.tf` | Existing Secrets Manager resources and KMS key | Pattern to follow for new secrets; IAM target ARNs |
| `terraform/aws-ecs/modules/mcp-gateway/iam.tf` | ECS task execution/task role policies | Must be expanded to allow `GetSecretValue` on new secrets |
| `terraform/aws-ecs/modules/mcp-gateway/observability.tf` | Grafana and metrics-service ECS tasks | Grafana admin password is plaintext and must be migrated |
| `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | Module input variables | Existing sensitive variables are reused; descriptions may be updated |
| `terraform/aws-ecs/main.tf` | Root module calling `mcp_gateway` | Passes sensitive values through unchanged |
| `terraform/aws-ecs/variables.tf` | Root module input variables | Existing sensitive variables are reused; docs may be updated |
| `terraform/aws-ecs/README.md` | Deployment documentation | Security claims need updating |
| `terraform/aws-ecs/terraform.tfvars.example` | Example inputs | Needs notes on Secrets Manager-only variables |
| `.env.example` | Local/Docker Compose env reference | Needs comments about ECS behavior |
| `registry/core/config.py` | Registry Pydantic settings loader | Reads secrets from env vars; no change required |
| `auth_server/server.py` | Auth server startup config | Reads secrets from env vars; no change required |
| `servers/mcpgw/server.py` | MCPGW startup config | Reads env vars but no secret wiring currently exists in Terraform |

### Existing Patterns Identified

1. **Secrets Manager resource pattern**
   - Files: `terraform/aws-ecs/modules/mcp-gateway/secrets.tf`
   - Pattern: `aws_secretsmanager_secret` with `name_prefix`, `kms_key_id = aws_kms_key.secrets.id`, `recovery_window_in_days = 0`, plus `aws_secretsmanager_secret_version` with `lifecycle { ignore_changes = [secret_string] }` for externally managed or user-supplied values.
   - How to follow: create one secret/version pair per migrated plaintext env var.

2. **ECS `secrets` block pattern**
   - Files: `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` lines 413–480 and 1288–1365
   - Pattern: `secrets = concat([...fixed secrets...], conditional([...]))` where each entry is `{ name = "ENV_VAR_NAME", valueFrom = aws_secretsmanager_secret.xxx.arn }`.
   - How to follow: append new conditional secret entries to the existing `concat` structure.

3. **IAM policy pattern**
   - File: `terraform/aws-ecs/modules/mcp-gateway/iam.tf` lines 4–52
   - Pattern: `aws_iam_policy.ecs_secrets_access` grants `secretsmanager:GetSecretValue` on a concatenated list of secret ARNs and `kms:Decrypt`/`DescribeKey` on the application KMS key.
   - How to follow: add new secret ARNs to the `Resource` list, preserving conditional logic.

4. **Lifecycle/Checkov suppression pattern**
   - File: `terraform/aws-ecs/modules/mcp-gateway/secrets.tf`
   - Pattern: `#checkov:skip=CKV2_AWS_57` on each secret with a justification comment explaining why rotation is external or requires coordinated restart.
   - How to follow: copy the suppression style and wording.

### Integration Points

| Component | Integration Type | Details |
|-----------|------------------|---------|
| `aws_kms_key.secrets` | Uses | New secrets are encrypted with the existing application KMS key. |
| `aws_iam_policy.ecs_secrets_access` | Extends | New secret ARNs are added to the policy `Resource` list. |
| ECS task execution role | Uses policy | Created automatically by `terraform-aws-modules/ecs/aws//modules/service`; the policy ARN is attached via `task_exec_iam_role_policies`. |
| Registry/auth-server containers | Consumes env vars | ECS injects secret values as env vars at container startup; applications read them via existing `os.environ`/Pydantic settings. |
| Grafana container | Consumes env vars | Grafana reads `GF_SECURITY_ADMIN_PASSWORD`; the sidecar reads the same variable for API calls. |

### Constraints and Limitations Discovered

- **No application code changes required**: ECS `secrets` and `environment` both produce the same runtime environment-variable interface. The Python config loaders already read from `os.environ` or Pydantic `BaseSettings`.
- **Optional values must remain optional**: Many secrets are empty by default. New Secrets Manager resources must be conditionally created with `count` so users who do not enable a feature are not forced to create empty secrets.
- **Grafana task role currently lacks Secrets Manager access**: The Grafana service only has `EcsExecTaskExecution` and `GrafanaAMPAccess` policies today. It needs `SecretsManagerAccess` after migration.
- **Reserved-name validation**: `registry_extra_env`, `auth_server_extra_env`, and `mcpgw_extra_env` are validated against reserved-name lists in `charts/*/reserved-env-names.txt`. The migrated names must remain on those lists so users cannot override them.
- **mcpgw has no current secret wiring**: The `mcpgw` container does not currently receive `REGISTRY_API_TOKEN`, `SECRET_KEY`, or IdP secrets. This design does not add new secrets to mcpgw unless an existing feature requires it.

## Architecture

### System Context Diagram

```
                     +---------------------------+
                     |   AWS Secrets Manager     |
                     |  (KMS-encrypted secrets)  |
                     +-------------+-------------+
                                   |
                    GetSecretValue | IAM
                                   v
+------------+   secrets block   +------------------+
| Terraform  | ----------------> |  ECS Task Def    |
|  (ecs task |                   | (auth/registry/  |
|   secrets) |                   |  grafana)        |
+------------+                   +--------+---------+
                                          |
                           env var injection
                                          v
                              +---------------------+
                              |  Application        |
                              |  (unchanged code)   |
                              +---------------------+
```

### Sequence Diagram

```
Operator -> Terraform: terraform apply
Terraform -> AWS Secrets Manager: create/update secret versions
Terraform -> AWS ECS: register task definition with secrets[] references
ECS Agent -> AWS Secrets Manager: GetSecretValue at container start
AWS Secrets Manager -> ECS Agent: decrypted secret value
ECS Agent -> Container: inject as environment variable
Container -> App: read env var via existing config loader
```

### Component Diagram

```
terraform/aws-ecs/modules/mcp-gateway/
├── secrets.tf          # new aws_secretsmanager_secret resources
├── ecs-services.tf     # move env vars to secrets[]
├── observability.tf    # migrate GF_SECURITY_ADMIN_PASSWORD
├── iam.tf              # expand ecs_secrets_access policy
├── variables.tf        # update descriptions (no new inputs needed)
└── locals.tf           # unchanged

caller: terraform/aws-ecs/main.tf passes existing var.* values through
```

## Data Models

### New Models
No new Terraform provider resources or data sources are introduced. The change uses the existing `aws_secretsmanager_secret`, `aws_secretsmanager_secret_version`, `aws_iam_policy`, and ECS task definition schemas.

### Model Changes

#### `aws_secretsmanager_secret` additions in `secrets.tf`
Example resource shape for a user-supplied optional secret:

```hcl
#checkov:skip=CKV2_AWS_57:User-provided API token - rotation managed by operator outside Terraform
resource "aws_secretsmanager_secret" "registry_api_token" {
  count = var.registry_api_token != "" ? 1 : 0

  name_prefix             = "${local.name_prefix}-registry-api-token-"
  description             = "Static API token for Registry API access"
  recovery_window_in_days = 0
  kms_key_id              = aws_kms_key.secrets.id
  tags                    = local.common_tags
}

resource "aws_secretsmanager_secret_version" "registry_api_token" {
  count = var.registry_api_token != "" ? 1 : 0

  secret_id     = aws_secretsmanager_secret.registry_api_token[0].id
  secret_string = var.registry_api_token

  lifecycle {
    ignore_changes = [secret_string]
  }
}
```

For feature-gated secrets, use the feature flag in `count`. Example:

```hcl
resource "aws_secretsmanager_secret" "ans_api_key" {
  count = var.ans_integration_enabled ? 1 : 0
  ...
}
```

For secrets that are always created when a parent feature is enabled but may contain empty defaults, keep the existing variable default and create the secret only when non-empty.

#### ECS `secrets` block additions in `ecs-services.tf`
Example addition to the auth-server `secrets` concat list:

```hcl
var.registry_api_token != "" ? [
  {
    name      = "REGISTRY_API_TOKEN"
    valueFrom = aws_secretsmanager_secret.registry_api_token[0].arn
  }
] : [],
var.registry_api_keys != "" ? [
  {
    name      = "REGISTRY_API_KEYS"
    valueFrom = aws_secretsmanager_secret.registry_api_keys[0].arn
  }
] : [],
```

The same pattern is repeated for `registry` and, where applicable, for `grafana`/`grafana-config`.

## API / CLI Design

No new API endpoints or CLI commands are introduced. Operators continue to run the standard Terraform workflow:

```bash
cd terraform/aws-ecs
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

After apply, verify that no secret values appear in the task definition:

```bash
aws ecs describe-task-definition \
  --task-definition "<name-prefix>-auth-server" \
  --query 'taskDefinition.containerDefinitions[].secrets'
```

## Configuration Parameters

### New Environment Variables
No new Terraform variables are required. Existing sensitive variables are reused.

### Updated Terraform Variables

| Variable | File | Change |
|----------|------|--------|
| `registry_api_token` | `variables.tf` | Update description to note it is stored in Secrets Manager when deployed via ECS |
| `registry_api_keys` | `variables.tf` | Same |
| `federation_static_token` | `variables.tf` | Same |
| `federation_encryption_key` | `variables.tf` | Same |
| `ans_api_key` | `variables.tf` | Same |
| `ans_api_secret` | `variables.tf` | Same |
| `auth0_management_api_token` | `variables.tf` | Same |
| `registration_webhook_auth_token` | `variables.tf` | Same |
| `registration_gate_auth_credential` | `variables.tf` | Same |
| `registration_gate_oauth2_client_secret` | `variables.tf` | Same |
| `github_pat` | `variables.tf` | Same |
| `github_app_private_key` | `variables.tf` | Same |
| `grafana_admin_password` | `variables.tf` | Same |
| `mongodb_connection_string` | `variables.tf` | Add deprecation note directing users to `mongodb_connection_string_secret_arn` |

### Deployment Surface Checklist

- [ ] `terraform/aws-ecs/modules/mcp-gateway/secrets.tf` - new secret resources
- [ ] `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` - auth-server and registry `secrets` blocks
- [ ] `terraform/aws-ecs/modules/mcp-gateway/observability.tf` - grafana/grafana-config `secrets` block
- [ ] `terraform/aws-ecs/modules/mcp-gateway/iam.tf` - expanded policy
- [ ] `terraform/aws-ecs/modules/mcp-gateway/variables.tf` - updated descriptions
- [ ] `terraform/aws-ecs/variables.tf` - updated descriptions
- [ ] `terraform/aws-ecs/README.md` - updated Security Considerations
- [ ] `terraform/aws-ecs/terraform.tfvars.example` - updated comments
- [ ] `.env.example` - updated comments

## New Dependencies

This change uses only existing dependencies:
- `hashicorp/aws` provider >= 5.0
- `terraform-aws-modules/ecs/aws//modules/service` ~> 6.0

No new Terraform providers, modules, or Python packages are required.

## Implementation Details

### Step-by-Step Plan

#### Step 1: Add Secrets Manager resources
**File:** `terraform/aws-ecs/modules/mcp-gateway/secrets.tf`
**Lines:** after existing `otlp_exporter_headers` resources (around line 377)

Add one secret/version pair per migrated plaintext env var. Use `count` for optional values. Follow the existing Checkov suppression and `lifecycle { ignore_changes = [secret_string] }` pattern for user-supplied or externally managed values.

Secrets to create:
- `registry_api_token`
- `registry_api_keys`
- `federation_static_token`
- `federation_encryption_key`
- `ans_api_key` (conditional on `var.ans_integration_enabled`)
- `ans_api_secret` (conditional on `var.ans_integration_enabled`)
- `auth0_management_api_token` (conditional on `var.auth0_enabled`)
- `registration_webhook_auth_token` (conditional on `var.registration_webhook_url != ""`)
- `registration_gate_auth_credential` (conditional on `var.registration_gate_enabled`)
- `registration_gate_oauth2_client_secret` (conditional on `var.registration_gate_enabled`)
- `github_pat` (conditional on `var.github_pat != ""`)
- `github_app_private_key` (conditional on `var.github_app_private_key != ""`)
- `grafana_admin_password` (conditional on `var.enable_observability`)

#### Step 2: Move auth-server env vars to secrets
**File:** `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`
**Lines:** 97–411 (environment) and 413–480 (secrets)

Remove the following entries from the `auth-server` `environment` list:
- `REGISTRY_API_TOKEN`
- `REGISTRY_API_KEYS`
- `FEDERATION_STATIC_TOKEN`
- `FEDERATION_ENCRYPTION_KEY`
- `ANS_API_KEY`
- `ANS_API_SECRET`
- `AUTH0_MANAGEMENT_API_TOKEN`

Add corresponding entries to the `auth-server` `secrets` concat list using conditional ternary lists.

#### Step 3: Move registry env vars to secrets
**File:** `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`
**Lines:** 698–1286 (environment) and 1288–1365 (secrets)

Remove the same entries as auth-server, plus registry-only secrets:
- `REGISTRATION_WEBHOOK_AUTH_TOKEN`
- `REGISTRATION_GATE_AUTH_CREDENTIAL`
- `REGISTRATION_GATE_OAUTH2_CLIENT_SECRET`
- `GITHUB_PAT`
- `GITHUB_APP_PRIVATE_KEY`

Add corresponding entries to the registry `secrets` concat list.

#### Step 4: Migrate Grafana admin password
**File:** `terraform/aws-ecs/modules/mcp-gateway/observability.tf`
**Lines:** 544–597 (grafana environment) and 642–647 (grafana-config environment)

Remove `GF_SECURITY_ADMIN_PASSWORD` from both containers' `environment` lists. Add a `secrets` block to the Grafana task definition referencing `aws_secretsmanager_secret.grafana_admin_password[0].arn` for both containers.

Example Grafana container change:

```hcl
container_definitions = {
  grafana = {
    ...
    environment = [
      ...
      # remove GF_SECURITY_ADMIN_PASSWORD entry
    ]
    secrets = [
      {
        name      = "GF_SECURITY_ADMIN_PASSWORD"
        valueFrom = aws_secretsmanager_secret.grafana_admin_password[0].arn
      }
    ]
    ...
  }
  grafana-config = {
    ...
    environment = [
      ...
      # remove GF_SECURITY_ADMIN_PASSWORD entry
    ]
    secrets = [
      {
        name      = "GF_SECURITY_ADMIN_PASSWORD"
        valueFrom = aws_secretsmanager_secret.grafana_admin_password[0].arn
      }
    ]
    ...
  }
}
```

#### Step 5: Update IAM policy
**File:** `terraform/aws-ecs/modules/mcp-gateway/iam.tf`
**Lines:** 4–52

Expand the `Resource` list in `aws_iam_policy.ecs_secrets_access` to include:

```hcl
concat(
  [
    aws_secretsmanager_secret.secret_key.arn,
    ...existing secrets...
  ],
  ...existing conditionals...,
  var.registry_api_token != "" ? [aws_secretsmanager_secret.registry_api_token[0].arn] : [],
  var.registry_api_keys != "" ? [aws_secretsmanager_secret.registry_api_keys[0].arn] : [],
  var.federation_static_token != "" ? [aws_secretsmanager_secret.federation_static_token[0].arn] : [],
  var.federation_encryption_key != "" ? [aws_secretsmanager_secret.federation_encryption_key[0].arn] : [],
  var.ans_integration_enabled ? [
    aws_secretsmanager_secret.ans_api_key[0].arn,
    aws_secretsmanager_secret.ans_api_secret[0].arn
  ] : [],
  var.auth0_enabled && var.auth0_management_api_token != "" ? [aws_secretsmanager_secret.auth0_management_api_token[0].arn] : [],
  var.registration_webhook_url != "" ? [aws_secretsmanager_secret.registration_webhook_auth_token[0].arn] : [],
  var.registration_gate_enabled ? [
    aws_secretsmanager_secret.registration_gate_auth_credential[0].arn,
    aws_secretsmanager_secret.registration_gate_oauth2_client_secret[0].arn
  ] : [],
  var.github_pat != "" ? [aws_secretsmanager_secret.github_pat[0].arn] : [],
  var.github_app_private_key != "" ? [aws_secretsmanager_secret.github_app_private_key[0].arn] : [],
  var.enable_observability ? [aws_secretsmanager_secret.grafana_admin_password[0].arn] : []
)
```

#### Step 6: Grant Grafana task execution role access
**File:** `terraform/aws-ecs/modules/mcp-gateway/observability.tf`
**Lines:** 505–513

Change Grafana's `task_exec_iam_role_policies` from:

```hcl
task_exec_iam_role_policies = {
  EcsExecTaskExecution = aws_iam_policy.ecs_exec_task_execution.arn
}
```

to:

```hcl
task_exec_iam_role_policies = {
  SecretsManagerAccess = aws_iam_policy.ecs_secrets_access.arn
  EcsExecTaskExecution = aws_iam_policy.ecs_exec_task_execution.arn
}
```

#### Step 7: Update Terraform variable descriptions
**File:** `terraform/aws-ecs/modules/mcp-gateway/variables.tf` and `terraform/aws-ecs/variables.tf`

Append a note such as:

```hcl
variable "registry_api_token" {
  description = "Static API key for Registry API. When deploying via terraform/aws-ecs, this value is stored in AWS Secrets Manager and injected into the container at runtime; it is not rendered as a plaintext environment variable."
  ...
}
```

For `mongodb_connection_string`, add a deprecation pointer:

```hcl
variable "mongodb_connection_string" {
  description = "Optional full MongoDB connection string override (plain text). Deprecated for ECS deployments; prefer mongodb_connection_string_secret_arn to avoid storing credentials in Terraform state and ECS task definitions."
  ...
}
```

#### Step 8: Update documentation
**File:** `terraform/aws-ecs/README.md`

Update the Security Considerations / Secrets Management section to accurately state that all credentials are stored in Secrets Manager, and update the "Avoid hardcoding credentials" best practice to reference the migrated variables.

**File:** `terraform/aws-ecs/terraform.tfvars.example`

Add comments above each migrated variable indicating that in ECS the value is stored in Secrets Manager. Keep the examples because the variable still accepts a plaintext value locally or in Docker Compose.

**File:** `.env.example`

Add similar comments for local development clarity.

### Error Handling

- If a required secret variable is empty, the corresponding `aws_secretsmanager_secret_version` is not created and the conditional `secrets` entry is omitted. This preserves existing optional behavior.
- If `terraform plan` is run with a non-empty secret value, the plan must not show that value in `environment` blocks. Any regression that renders a secret value in plain text should be caught during review.
- Existing `precondition` and `validation` blocks continue to function unchanged.

### Logging

No new application logging is required. Terraform already logs resource creation. Operational teams can use AWS CloudTrail to audit `secretsmanager:GetSecretValue` calls.

## Observability

### Tracing / Metrics / Logging Points

- **CloudTrail**: `GetSecretValue` calls from ECS task execution roles are logged automatically.
- **Terraform plan output**: Ensure secret values are masked because variables are already marked `sensitive = true`.
- **No application metrics changes**: The change is purely at the infrastructure layer.

## Scaling Considerations

- Secrets Manager read throughput is high and supports caching by the ECS agent; no scaling concerns for normal registry workloads.
- Each ECS task still receives the same set of environment variables, so container startup time is unchanged.
- The number of Secrets Manager resources increases by up to 13 per deployment, well within account limits.

## File Changes

### New Resources (within existing files)

| File Path | Description |
|-----------|-------------|
| `terraform/aws-ecs/modules/mcp-gateway/secrets.tf` | New `aws_secretsmanager_secret` and `aws_secretsmanager_secret_version` resources for migrated secrets |

### Modified Files

| File Path | Lines | Change Description |
|-----------|-------|--------------------|
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | ~97–411, ~698–1286 | Remove secret-like variables from `environment` blocks |
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | ~413–480, ~1288–1365 | Add conditional `secrets` entries for migrated variables |
| `terraform/aws-ecs/modules/mcp-gateway/observability.tf` | ~544–597, ~642–647 | Remove `GF_SECURITY_ADMIN_PASSWORD` from `environment` |
| `terraform/aws-ecs/modules/mcp-gateway/observability.tf` | ~527–660 | Add `secrets` blocks to grafana and grafana-config containers |
| `terraform/aws-ecs/modules/mcp-gateway/observability.tf` | ~505–513 | Add `SecretsManagerAccess` to Grafana task execution role |
| `terraform/aws-ecs/modules/mcp-gateway/iam.tf` | ~4–52 | Expand `aws_iam_policy.ecs_secrets_access` Resource list |
| `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | multiple | Update descriptions for migrated variables |
| `terraform/aws-ecs/variables.tf` | multiple | Update descriptions for migrated variables |
| `terraform/aws-ecs/README.md` | ~69–70, ~1036–1040, ~1112 | Update Security Considerations and best practices |
| `terraform/aws-ecs/terraform.tfvars.example` | multiple | Add comments about Secrets Manager storage |
| `.env.example` | multiple | Add comments about ECS Secrets Manager behavior |

### Estimated Lines of Code

| Category | Lines |
|----------|-------|
| New Terraform resources | ~250 |
| Modified Terraform task definitions | ~150 |
| Modified IAM policy | ~40 |
| Documentation updates | ~50 |
| Tests (see testing.md) | ~100 |
| **Total** | **~590** |

## Testing Strategy

See `testing.md` for the complete executable test plan.

## Alternatives Considered

### Alternative 1: Use SSM Parameter Store SecureString for all secrets
**Description:** Store every migrated value in SSM Parameter Store instead of Secrets Manager.
**Pros:** Slightly lower cost; consistent with existing Keycloak admin credentials.
**Cons:** Secrets Manager is already used for the majority of credentials, has better rotation support, and is the stated direction in the README. Mixing two secret stores increases operator confusion.
**Why Rejected:** Prefer Secrets Manager for consistency with existing IdP, metrics, and DocumentDB secrets.

### Alternative 2: Require users to pre-create secrets and pass ARNs
**Description:** Replace plaintext variables with ARN variables; users create secrets outside Terraform.
**Pros:** Eliminates secret values from Terraform state entirely.
**Cons:** Adds operational friction; breaks backwards compatibility for existing `terraform.tfvars` files; many secrets are optional and would require complex conditional ARN handling.
**Why Rejected:** The current project pattern creates Secrets Manager resources from user-supplied variables (e.g., `entra_client_secret`, `okta_client_secret`). Following that pattern preserves consistency and backwards compatibility.

### Alternative 3: Keep plaintext env vars and rely on IAM alone
**Description:** Leave secrets in `environment` and tighten IAM on `DescribeTaskDefinition`.
**Pros:** Minimal code change.
**Cons:** Does not address the core security risk; secrets remain visible in AWS Console, CloudTrail, and task definition snapshots; fails the stated acceptance criteria.
**Why Rejected:** Does not solve the problem.

### Comparison Matrix

| Criteria | Chosen (Secrets Manager) | SSM Parameter Store | Pre-created ARNs | Keep plaintext |
|----------|--------------------------|---------------------|------------------|----------------|
| Security | High | Medium-High | High | Low |
| Consistency with existing code | High | Medium | Low | N/A |
| Operator friction | Low | Low | High | Lowest |
| Backwards compatibility | High | High | Low | High |
| Future rotation support | High | Medium | High | None |

## Rollout Plan

- Phase 1: Implementation (out of scope for this skill)
  - Implement Steps 1–8 above.
  - Run `terraform fmt` and `terraform validate`.
- Phase 2: Testing
  - Execute the plan in `testing.md`.
  - Verify no plaintext secrets in task definitions.
- Phase 3: Deployment
  - Apply to a non-production environment first.
  - Confirm all services start and authenticate correctly.
  - Apply to production.

## Open Questions

1. Should the `mongodb_connection_string` plaintext variable be formally deprecated with a `validation` warning, or only a description note?
2. Should the mcpgw service receive `REGISTRY_API_TOKEN` or other secrets to support future OIDC/M2M features, even though it does not consume them today?
3. Should the Grafana admin password be randomly generated when not supplied, or should the variable remain required when observability is enabled?

## References

- [AWS ECS secrets documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data-secrets.html)
- [AWS Secrets Manager best practices](https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html)
- Existing `aws_secretsmanager_secret` resources in `terraform/aws-ecs/modules/mcp-gateway/secrets.tf`
- Existing ECS `secrets` blocks in `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`
