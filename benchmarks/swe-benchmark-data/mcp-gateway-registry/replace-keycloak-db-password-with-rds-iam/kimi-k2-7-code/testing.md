# Testing Plan: Replace Keycloak static RDS password with RDS IAM authentication

*Created: 2026-07-15*  
*Related LLD: `./lld.md`*  
*Related Issue: `./github-issue.md`*

## Overview

### Scope of Testing
This plan verifies that the new `keycloak_db_use_iam` feature flag correctly toggles between the existing password-auth path and the new RDS IAM-auth path without breaking existing deployments. Tests cover Terraform validation, IAM policy generation, container wiring, custom image requirements, and end-to-end Keycloak health.

### Prerequisites
- [ ] Repository cloned and checked out at tag `1.24.4`.
- [ ] Terraform `>= 1.5.0` installed.
- [ ] AWS credentials configured with permissions to run `terraform plan`.
- [ ] Docker installed (for custom Keycloak image build test).
- [ ] A non-production AWS account/region for apply tests.

### Shared Variables

```bash
export REPO_ROOT="$(git rev-parse --show-toplevel)"
export TF_DIR="$REPO_ROOT/terraform/aws-ecs"
export AWS_REGION="us-east-1"
```

## 1. Functional Tests

### 1.1 Terraform plan with IAM auth disabled (default)

**Purpose:** Confirm the default path is unchanged.

```bash
cd "$TF_DIR"
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars to set required values (region, domain, passwords, etc.)
# Ensure keycloak_db_use_iam is NOT set (defaults to false).

terraform init -backend=false
terraform plan -var="name=mcp-gateway-iam-test" -out=tfplan-password
```

**Expected result:**
- Exit code `0`.
- Plan output shows no changes to `aws_db_proxy.keycloak` auth block.
- `aws_rds_cluster.keycloak` does not include `iam_database_authentication_enabled` or it is `false`.

**Assertions:**

```bash
terraform show -json tfplan-password | jq '
  .resource_changes[]
  | select(.address == "aws_db_proxy.keycloak")
  | .change.after.auth[0]
' | grep -E '"auth_scheme": "SECRETS"|"iam_auth": "DISABLED"'
```

### 1.2 Terraform plan with IAM auth enabled

**Purpose:** Confirm IAM-auth resources are created and password secret is removed from the container.

```bash
cd "$TF_DIR"
terraform plan \
  -var="name=mcp-gateway-iam-test" \
  -var="keycloak_db_use_iam=true" \
  -var="keycloak_iam_auth_image_uri=123456789012.dkr.ecr.us-east-1.amazonaws.com/keycloak-rds-iam:1.24.4" \
  -out=tfplan-iam
```

**Expected result:**
- Exit code `0`.
- Plan includes `iam_database_authentication_enabled = true` on the cluster.
- RDS Proxy `auth_scheme` becomes `AWS_IAM` and `iam_auth` becomes `REQUIRED`.
- New Lambda and invocation resources appear.

**Assertions:**

```bash
terraform show -json tfplan-iam | jq '
  .resource_changes[]
  | select(.address == "aws_db_proxy.keycloak")
  | .change.after.auth[0]
' | grep -E '"auth_scheme": "AWS_IAM"|"iam_auth": "REQUIRED"'

terraform show -json tfplan-iam | jq '
  .resource_changes[]
  | select(.address == "aws_rds_cluster.keycloak")
  | .change.after.iam_database_authentication_enabled
' | grep true

terraform show -json tfplan-iam | jq '
  .resource_changes[]
  | select(.address == "aws_lambda_function.keycloak_rds_iam_init[0]")
  | .change.actions
' | grep -E '\["create"\]|\["create"\]'
```

### 1.3 Precondition rejects missing custom image

**Purpose:** Ensure operators cannot enable IAM auth without providing a compatible image.

```bash
cd "$TF_DIR"
terraform plan \
  -var="name=mcp-gateway-iam-test" \
  -var="keycloak_db_use_iam=true" \
  -var="keycloak_iam_auth_image_uri=" \
  2>&1 | tee tfplan-missing-image.log
```

**Expected result:**
- Non-zero exit code.
- Error message contains: `keycloak_iam_auth_image_uri must be set when keycloak_db_use_iam is true`.

**Assertion:**

```bash
grep -q "keycloak_iam_auth_image_uri must be set" tfplan-missing-image.log && echo "PASS" || echo "FAIL"
```

### 1.4 Lambda unit test: IAM user creation SQL

**Purpose:** Verify the bootstrap Lambda constructs valid, idempotent SQL.

```bash
cd "$REPO_ROOT/terraform/aws-ecs/lambda/keycloak-rds-iam-init"
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt pytest
pytest tests/
```

**Expected result:**
- All tests pass, including tests for:
  - User creation with `AWSAuthenticationPlugin`.
  - SSL requirement.
  - Grant on `keycloak` database.
  - Idempotency when user already exists.

**Example test to add under `terraform/aws-ecs/lambda/keycloak-rds-iam-init/tests/test_index.py`:**

```python
from unittest.mock import Mock, patch

import index


def test_create_iam_user_executes_expected_sql():
    mock_conn = Mock()
    mock_cur = Mock()
    mock_conn.cursor.return_value.__enter__ = Mock(return_value=mock_cur)
    mock_conn.cursor.return_value.__exit__ = Mock(return_value=False)

    with patch("index.get_secret", return_value={"username": "admin", "password": "pw"}), \
         patch("index.get_ssm", side_effect=["proxy.cluster-xxx.us-east-1.rds.amazonaws.com", "keycloak_iam"]), \
         patch("pymysql.connect", return_value=mock_conn):
        index.lambda_handler({}, {})

    executed = " ".join(call.args[0] for call in mock_cur.execute.call_args_list)
    assert "CREATE USER IF NOT EXISTS 'keycloak_iam'" in executed
    assert "IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS'" in executed
    assert "REQUIRE SSL" in executed
    assert "GRANT ALL PRIVILEGES ON keycloak.*" in executed
```

## 2. Backwards Compatibility Tests

### 2.1 Password-auth plan shows no drift

**Purpose:** Ensure existing deployments with `keycloak_db_use_iam = false` see no changes after the code update.

```bash
cd "$TF_DIR"
terraform plan \
  -var="name=mcp-gateway-iam-test" \
  -var="keycloak_db_use_iam=false" \
  -var="keycloak_database_password=SuperSecret123!" \
  -detailed-exitcode
```

**Expected result:**
- Exit code `0` (no changes) if the state already matches.
- If this is a fresh plan, verify the resources match the pre-change baseline.

### 2.2 Container secrets still include password when flag is false

**Purpose:** Confirm the fallback path still passes `KC_DB_PASSWORD`.

```bash
terraform show -json tfplan-password | jq '
  .resource_changes[]
  | select(.address == "aws_ecs_task_definition.keycloak")
  | .change.after.container_definitions
  | fromjson
  | .[0].secrets
' | grep -q '"KC_DB_PASSWORD"' && echo "PASS" || echo "FAIL"
```

### 2.3 Container secrets omit password when flag is true

**Purpose:** Confirm IAM auth path does not leak the static password into the container.

```bash
terraform show -json tfplan-iam | jq '
  .resource_changes[]
  | select(.address == "aws_ecs_task_definition.keycloak")
  | .change.after.container_definitions
  | fromjson
  | .[0].secrets
' | grep -q '"KC_DB_PASSWORD"' && echo "FAIL" || echo "PASS"
```

## 3. UX Tests

### 3.1 README clarity

**Purpose:** Operators can follow the docs to enable IAM auth without reading the source code.

- [ ] Open `terraform/aws-ecs/README.md`.
- [ ] Locate a section titled "RDS IAM authentication for Keycloak".
- [ ] Verify the section contains:
  - [ ] A warning that a custom Keycloak image is required.
  - [ ] Exact build/push commands.
  - [ ] The variable names `keycloak_db_use_iam` and `keycloak_iam_auth_image_uri`.
  - [ ] A rollback command (`terraform apply -var="keycloak_db_use_iam=false"`).

### 3.2 Error message clarity

Run the precondition failure test from `1.3`. A non-technical operator should understand that a custom image URI is missing.

## 4. Deployment Surface Tests

### 4.1 Docker image build

**Purpose:** Confirm the custom Keycloak image includes the AWS JDBC wrapper.

```bash
cd "$REPO_ROOT/docker/keycloak"
docker build -t keycloak-rds-iam:local .
docker run --rm --entrypoint ls keycloak-rds-iam:local /opt/keycloak/providers/ | grep aws-advanced-jdbc-wrapper
```

**Expected result:**
- Image builds successfully.
- `aws-advanced-jdbc-wrapper-*.jar` is present in `/opt/keycloak/providers/`.

### 4.2 Terraform validate

**Purpose:** Catch syntax errors in the modified Terraform files.

```bash
cd "$TF_DIR"
terraform validate
```

**Expected result:**
- Exit code `0`.

### 4.3 IAM policy scoping

**Purpose:** Verify `rds-db:connect` is scoped to the specific cluster resource ID and IAM username.

```bash
terraform show -json tfplan-iam | jq '
  .resource_changes[]
  | select(.address | contains("keycloak_task_rds_iam_policy"))
  | .change.after.policy
  | fromjson
  | .Statement[0].Resource
'
```

**Expected result:**
- Resource contains the IAM username (e.g., `.../keycloak_iam`).
- Resource does not contain `*`.

### 4.4 RDS Proxy TLS enforcement

**Purpose:** Confirm `require_tls = true` when IAM auth is enabled.

```bash
terraform show -json tfplan-iam | jq '
  .resource_changes[]
  | select(.address == "aws_db_proxy.keycloak")
  | .change.after.require_tls
' | grep true
```

### 4.5 Deploy and verify (non-production)

**Prerequisites:** Non-production AWS environment, custom image pushed to ECR.

```bash
cd "$TF_DIR"
terraform apply \
  -var="name=mcp-gateway-iam-test" \
  -var="keycloak_db_use_iam=true" \
  -var="keycloak_iam_auth_image_uri=YOUR_ACCOUNT.dkr.ecr.YOUR_REGION.amazonaws.com/keycloak-rds-iam:1.24.4" \
  -auto-approve
```

**Expected result:**
- Apply completes without errors.
- Init Lambda invocation succeeds.
- Keycloak ECS service reaches `steady` state with `runningCount == desiredCount`.

```bash
aws ecs describe-services \
  --cluster keycloak \
  --services keycloak \
  --region "$AWS_REGION" \
  --query 'services[0].{running:runningCount, desired:desiredCount, status:status}'
```

### 4.6 Rollback verification

**Purpose:** Confirm password fallback can be restored.

```bash
cd "$TF_DIR"
terraform apply \
  -var="name=mcp-gateway-iam-test" \
  -var="keycloak_db_use_iam=false" \
  -auto-approve
```

**Expected result:**
- Apply completes.
- Keycloak ECS service returns to healthy state.
- `aws_db_proxy.keycloak` auth block reverts to `SECRETS` / `DISABLED`.

## 5. End-to-End API Tests

### 5.1 Keycloak health endpoint

After IAM-auth deployment, verify Keycloak responds to health checks.

```bash
export KEYCLOAK_URL=$(terraform output -raw keycloak_url)
curl -fsS "$KEYCLOAK_URL/health/ready"
```

**Expected result:**
- HTTP `200` with `{"status": "up"}`.

### 5.2 Registry login via Keycloak

After IAM-auth deployment, verify the registry login flow works end-to-end.

```bash
export REGISTRY_URL=$(terraform output -raw mcp_gateway_url)
curl -fsS "$REGISTRY_URL/health"
```

**Expected result:**
- HTTP `200`.

### 5.3 RDS Proxy connection metrics

After IAM-auth deployment, verify RDS Proxy is serving client connections.

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ClientConnections \
  --dimensions Name=ProxyName,Value=keycloak-proxy Name=Role,Value=CLIENT \
  --start-time "$(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 60 \
  --statistics Sum \
  --region "$AWS_REGION"
```

**Expected result:**
- Non-zero `Sum` values after Keycloak tasks start.

## 6. Test Execution Checklist

- [ ] Section 1.1 (password plan) passes.
- [ ] Section 1.2 (IAM plan) passes.
- [ ] Section 1.3 (precondition) passes.
- [ ] Section 1.4 (Lambda unit tests) passes.
- [ ] Section 2.1 (no drift) verified.
- [ ] Section 2.2 (password secret present) verified.
- [ ] Section 2.3 (password secret absent in IAM mode) verified.
- [ ] Section 3.1 (README clarity) verified.
- [ ] Section 3.2 (error message clarity) verified.
- [ ] Section 4.1 (image build) passes.
- [ ] Section 4.2 (terraform validate) passes.
- [ ] Section 4.3 (IAM scoping) verified.
- [ ] Section 4.4 (TLS enforcement) verified.
- [ ] Section 4.5 (IAM deploy) passes in non-production.
- [ ] Section 4.6 (rollback) passes.
- [ ] Section 5.1 (Keycloak health) passes.
- [ ] Section 5.2 (registry health) passes.
- [ ] Section 5.3 (RDS Proxy metrics) passes.
- [ ] Unit tests added under `terraform/aws-ecs/lambda/keycloak-rds-iam-init/tests/`.
- [ ] Terraform plan tests added for both flag states (optional but recommended).
