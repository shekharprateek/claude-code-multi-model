# Testing Plan: Migrate ECS plaintext secrets to AWS Secrets Manager

*Created: 2026-07-15*
*Related LLD: `./lld.md`*
*Related Issue: `./github-issue.md`*

## Overview

### Scope of Testing
This plan verifies that all secret-like values are removed from ECS task-definition `environment` blocks and referenced via AWS Secrets Manager `secrets` blocks, that IAM permissions allow retrieval, and that application behavior is unchanged.

### Prerequisites
- [ ] AWS credentials with permissions to run `terraform plan` and read ECS task definitions and Secrets Manager secrets.
- [ ] Terraform >= 1.2 installed.
- [ ] A valid `terraform.tfvars` file for a non-production deployment.
- [ ] The target repository is checked out at tag `1.24.4`.

### Shared Variables

```bash
export REPO_ROOT="$(git rev-parse --show-toplevel)"
export TF_DIR="$REPO_ROOT/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/terraform/aws-ecs"
export AWS_REGION="us-east-1"  # adjust to your deployment region
export NAME_PREFIX="mcp-test-v2"  # adjust to match var.name
```

## 1. Functional Tests

### 1.1 Terraform plan shows no plaintext secrets in task definitions

For each migrated secret value, run a targeted plan and grep the rendered container definitions.

```bash
cd "$TF_DIR"
terraform init
terraform plan -out=tfplan -var-file=terraform.tfvars
terraform show -json tfplan > tfplan.json
```

Verify that none of the following environment-variable names appear inside a `values` field of an `environment` block for the auth-server, registry, or grafana containers:

```bash
jq -r '
  .resource_changes[] |
  select(.type == "module") |
  .change.after.container_definitions // empty |
  fromjson? // [] |
  .[] |
  select(.environment != null) |
  .name as $container |
  .environment[] |
  select(.name | IN(
    "REGISTRY_API_TOKEN",
    "REGISTRY_API_KEYS",
    "FEDERATION_STATIC_TOKEN",
    "FEDERATION_ENCRYPTION_KEY",
    "ANS_API_KEY",
    "ANS_API_SECRET",
    "AUTH0_MANAGEMENT_API_TOKEN",
    "REGISTRATION_WEBHOOK_AUTH_TOKEN",
    "REGISTRATION_GATE_AUTH_CREDENTIAL",
    "REGISTRATION_GATE_OAUTH2_CLIENT_SECRET",
    "GITHUB_PAT",
    "GITHUB_APP_PRIVATE_KEY",
    "GF_SECURITY_ADMIN_PASSWORD"
  )) |
  "FOUND plaintext secret in container " + $container + ": " + .name
' tfplan.json
```

**Expected result:** The command produces no output.

### 1.2 Terraform plan shows Secrets Manager references for migrated secrets

```bash
jq -r '
  .resource_changes[] |
  select(.type == "module") |
  .change.after.container_definitions // empty |
  fromjson? // [] |
  .[] |
  select(.secrets != null) |
  .name as $container |
  .secrets[] |
  select(.name | IN(
    "REGISTRY_API_TOKEN",
    "REGISTRY_API_KEYS",
    "FEDERATION_STATIC_TOKEN",
    "FEDERATION_ENCRYPTION_KEY",
    "ANS_API_KEY",
    "ANS_API_SECRET",
    "AUTH0_MANAGEMENT_API_TOKEN",
    "REGISTRATION_WEBHOOK_AUTH_TOKEN",
    "REGISTRATION_GATE_AUTH_CREDENTIAL",
    "REGISTRATION_GATE_OAUTH2_CLIENT_SECRET",
    "GITHUB_PAT",
    "GITHUB_APP_PRIVATE_KEY",
    "GF_SECURITY_ADMIN_PASSWORD"
  )) |
  $container + " -> " + .name + " from " + .valueFrom
' tfplan.json
```

**Expected result:** Each migrated env var is listed with a `valueFrom` ARN containing `secretsmanager`.

### 1.3 AWS CLI verifies Secrets Manager secrets exist after apply

After `terraform apply`:

```bash
for secret_name in \
  "${NAME_PREFIX}-registry-api-token" \
  "${NAME_PREFIX}-registry-api-keys" \
  "${NAME_PREFIX}-federation-static-token" \
  "${NAME_PREFIX}-federation-encryption-key" \
  "${NAME_PREFIX}-ans-api-key" \
  "${NAME_PREFIX}-ans-api-secret" \
  "${NAME_PREFIX}-auth0-management-api-token" \
  "${NAME_PREFIX}-registration-webhook-auth-token" \
  "${NAME_PREFIX}-registration-gate-auth-credential" \
  "${NAME_PREFIX}-registration-gate-oauth2-client-secret" \
  "${NAME_PREFIX}-github-pat" \
  "${NAME_PREFIX}-github-app-private-key" \
  "${NAME_PREFIX}-grafana-admin-password"; do
  aws secretsmanager describe-secret \
    --secret-id "${secret_name}-" \
    --region "$AWS_REGION" >/dev/null 2>&1 \
    && echo "OK: $secret_name" \
    || echo "MISSING: $secret_name"
done
```

**Expected result:** All secrets that have non-empty variable values show `OK`. Optional empty secrets show `MISSING`, which is acceptable.

### 1.4 AWS CLI verifies ECS task definitions use secrets references

```bash
for service in auth-server registry grafana; do
  task_def_arn=$(aws ecs list-task-definitions \
    --family-prefix "${NAME_PREFIX}-${service}" \
    --sort DESC \
    --max-items 1 \
    --region "$AWS_REGION" \
    --query 'taskDefinitionArns[0]' --output text)

  aws ecs describe-task-definition \
    --task-definition "$task_def_arn" \
    --region "$AWS_REGION" \
    --query 'taskDefinition.containerDefinitions[].{
      name: name,
      secrets: secrets[].name,
      env_names: environment[].name
    }'
done
```

**Expected result:** For each container, the migrated names appear under `secrets` and not under `env_names`.

## 2. Backwards Compatibility Tests

### 2.1 Existing Terraform variables still accepted

```bash
cd "$TF_DIR"
cp terraform.tfvars.example terraform.tfvars.backwards-test
terraform validate -var-file=terraform.tfvars.backwards-test
rm terraform.tfvars.backwards-test
```

**Expected result:** `terraform validate` succeeds. The example file may contain old plaintext examples; the module should still accept them.

### 2.2 Empty secret values remain optional

Run a plan with optional secrets set to empty strings and verify no errors:

```bash
cd "$TF_DIR"
terraform plan \
  -var='registry_api_token=""' \
  -var='federation_static_token=""' \
  -var='github_pat=""' \
  -var='ans_integration_enabled=false' \
  -var='registration_gate_enabled=false' \
  -var='registration_webhook_url=""' \
  -var-file=terraform.tfvars
```

**Expected result:** Plan succeeds; no `aws_secretsmanager_secret` resources are created for empty optional secrets, and no ECS `secrets` entries reference them.

### 2.3 Environment-variable names unchanged in containers

```bash
aws ecs describe-task-definition \
  --task-definition "${NAME_PREFIX}-auth-server" \
  --region "$AWS_REGION" \
  --query 'taskDefinition.containerDefinitions[0].secrets[].name' \
  --output table
```

**Expected result:** Names like `REGISTRY_API_TOKEN` and `FEDERATION_STATIC_TOKEN` are unchanged so application code reads the same variables.

## 3. UX Tests

### 3.1 Operator documentation is accurate

```bash
grep -n "Secrets Manager" "$TF_DIR/README.md"
grep -n "plaintext" "$TF_DIR/README.md"
```

**Expected result:** The README states that credentials are stored in Secrets Manager and does not claim that values are passed as plaintext env vars.

### 3.2 Example files guide operators correctly

```bash
grep -A2 "registry_api_token" "$TF_DIR/terraform.tfvars.example"
grep -A2 "grafana_admin_password" "$TF_DIR/terraform.tfvars.example"
```

**Expected result:** Comments indicate the value is stored in Secrets Manager when deployed via ECS, while the variable still accepts a plaintext value.

### 3.3 Terraform plan output masks sensitive values

```bash
cd "$TF_DIR"
terraform plan -var-file=terraform.tfvars 2>&1 | grep -E "(REGISTRY_API_TOKEN|FEDERATION_STATIC_TOKEN|GITHUB_APP_PRIVATE_KEY)" || true
```

**Expected result:** No plaintext secret values are printed in the plan output because variables are marked `sensitive = true`.

## 4. Deployment Surface Tests

### 4.1 Terraform / ECS wiring

Verify the IAM policy resource includes all new secret ARNs:

```bash
terraform state show module.mcp_gateway.aws_iam_policy.ecs_secrets_access | grep -E "(registry-api-token|federation-static-token|github-app-private-key|grafana-admin-password)"
```

**Expected result:** New secret ARNs appear in the policy JSON.

### 4.2 Grafana task execution role wiring

```bash
terraform state show 'module.mcp_gateway.module.ecs_service_grafana[0]' | grep -A5 "task_exec_iam_role_policies"
```

**Expected result:** `SecretsManagerAccess` is listed in the Grafana task execution role policies.

### 4.3 Deploy and verify service health

After `terraform apply`:

```bash
aws ecs describe-services \
  --cluster "${NAME_PREFIX}-cluster" \
  --services "${NAME_PREFIX}-auth" "${NAME_PREFIX}-registry" "${NAME_PREFIX}-grafana" \
  --region "$AWS_REGION" \
  --query 'services[].{name: serviceName, status: status, runningCount: runningCount, desiredCount: desiredCount, failures: failures}'
```

**Expected result:** All services are `ACTIVE` with `runningCount == desiredCount` and no failures.

### 4.4 Rollback verification

To roll back to the previous task definition revision:

```bash
for service in auth registry grafana; do
  aws ecs update-service \
    --cluster "${NAME_PREFIX}-cluster" \
    --service "${NAME_PREFIX}-${service}" \
    --task-definition "${NAME_PREFIX}-${service}:<PREVIOUS_REVISION>" \
    --region "$AWS_REGION" \
    --force-new-deployment
done
```

**Expected result:** Services return to the previous revision and remain stable. This validates that the change is reversible via standard ECS mechanisms.

## 5. End-to-End API Tests

### 5.1 Registry health check after migration

```bash
export REGISTRY_URL="https://$(terraform output -raw registry_domain)"
curl -sf "${REGISTRY_URL}/health" | jq .
```

**Expected result:** HTTP 200 with healthy status.

### 5.2 Auth server health check after migration

```bash
curl -sf "${REGISTRY_URL}/auth/health" | jq .
```

**Expected result:** HTTP 200 with healthy status.

### 5.3 Registry static-token authenticated endpoint

If `registry_static_token_auth_enabled = true`:

```bash
export REGISTRY_API_TOKEN="$(terraform output -raw registry_api_token_value)"
curl -sf -H "Authorization: Bearer ${REGISTRY_API_TOKEN}" \
  "${REGISTRY_URL}/api/servers" | jq '. | length'
```

**Expected result:** Request succeeds, confirming the token was injected correctly.

### 5.4 Grafana login

```bash
export GRAFANA_URL="${REGISTRY_URL}/grafana"
export GRAFANA_ADMIN_PASSWORD="$(terraform output -raw grafana_admin_password_value)"
curl -sf -X POST "${GRAFANA_URL}/api/user/password" \
  -u "admin:${GRAFANA_ADMIN_PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"oldPassword":"'"${GRAFANA_ADMIN_PASSWORD}"'","newPassword":"'"${GRAFANA_ADMIN_PASSWORD}2"'"}'
```

**Expected result:** HTTP 200, confirming Grafana received the admin password via Secrets Manager.

## 6. Test Execution Checklist

- [ ] Section 1.1 passes (no plaintext secrets in plan)
- [ ] Section 1.2 passes (Secrets Manager references present)
- [ ] Section 1.3 passes (secrets exist in AWS)
- [ ] Section 1.4 passes (ECS task definitions use secrets)
- [ ] Section 2.1 passes (existing variables accepted)
- [ ] Section 2.2 passes (optional secrets remain optional)
- [ ] Section 2.3 passes (env-var names unchanged)
- [ ] Section 3.1 passes (README accurate)
- [ ] Section 3.2 passes (example files accurate)
- [ ] Section 3.3 passes (sensitive values masked in plan)
- [ ] Section 4.1 passes (IAM policy includes new secrets)
- [ ] Section 4.2 passes (Grafana execution role has SecretsManagerAccess)
- [ ] Section 4.3 passes (services healthy after apply)
- [ ] Section 4.4 passes (rollback succeeds)
- [ ] Section 5.1 passes (registry health)
- [ ] Section 5.2 passes (auth health)
- [ ] Section 5.3 passes (static-token API works, if enabled)
- [ ] Section 5.4 passes (Grafana login works, if enabled)
- [ ] `terraform validate` succeeds
- [ ] `terraform plan` succeeds with no unexpected changes
