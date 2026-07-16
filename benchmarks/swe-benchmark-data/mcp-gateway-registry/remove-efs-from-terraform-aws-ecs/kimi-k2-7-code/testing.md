# Testing Plan: Remove EFS from Terraform AWS ECS Deployment

*Created: 2026-07-15*
*Related LLD: `./lld.md`*
*Related Issue: `./github-issue.md`*

## Overview

### Scope of Testing

This plan verifies that all Amazon EFS resources, volume mounts, variables, outputs, post-deployment script branches, and documentation references are removed from the Terraform AWS ECS deployment, and that the deployment remains valid and functional after the cleanup.

### Prerequisites

- [ ] Terraform >= 1.2 installed.
- [ ] AWS credentials configured with permissions to run `terraform plan` (or a local mock backend).
- [ ] The target repository is checked out at tag `1.24.4` at `benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/`.

### Shared Variables

```bash
export REPO_ROOT="$(git rev-parse --show-toplevel)"
export REPO_PATH="$REPO_ROOT/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo"
export TF_DIR="$REPO_PATH/terraform/aws-ecs"
```

## 1. Functional Tests

### 1.1 Verify No Remaining EFS References in Terraform Code

**Purpose:** Confirm that no EFS module, variable, output, or resource reference remains under `terraform/aws-ecs/`.

**Command:**

```bash
cd "$REPO_PATH"
grep -Rin "efs\|elasticfilesystem\|EFS" terraform/aws-ecs --include="*.tf" 2>/dev/null || true
```

**Expected Result:** No matches. The only permitted exceptions are comments that explain historical removal (e.g., the existing `# EFS volumes removed` comments in `ecs-services.tf`), which should be reviewed and optionally removed.

**Assertion:**

```bash
if grep -Rin "module\.efs\|efs_id\|efs_arn\|efs_access_points\|efs_throughput\|efs_provisioned\|elasticfilesystem" terraform/aws-ecs --include="*.tf" 2>/dev/null; then
    echo "FAIL: EFS references remain in Terraform code"
    exit 1
else
    echo "PASS: No EFS references in Terraform code"
fi
```

### 1.2 Terraform Format Check

**Purpose:** Ensure all modified Terraform files are correctly formatted.

**Command:**

```bash
cd "$TF_DIR"
terraform fmt -check -recursive
```

**Expected Result:** Exit code 0.

### 1.3 Terraform Validation

**Purpose:** Verify that the Terraform configuration is syntactically and structurally valid after removing EFS.

**Command:**

```bash
cd "$TF_DIR"
terraform init -backend=false
terraform validate
```

**Expected Result:** `Success! The configuration is valid.`

**Negative Case:** If `module.efs` is still referenced anywhere, `terraform validate` will fail with an error such as `Reference to undeclared module`.

### 1.4 Terraform Plan Review

**Purpose:** Verify that Terraform plans cleanly and shows EFS resources as destroyed.

**Command:**

```bash
cd "$TF_DIR"
terraform plan -var-file=terraform.tfvars.example -out=tfplan
```

**Expected Result:**

- Plan succeeds without errors.
- At least the following resources appear with `# module.mcp_gateway.module.efs will be destroyed`:
  - EFS file system
  - EFS access points
  - EFS mount targets
  - EFS security group
- No resources show `module.efs` as a dependency in `will be created` or `will be updated` blocks.

**Assertion:**

```bash
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions[] | contains("delete")) | .address' | grep -i efs
```

This should list EFS resource addresses.

## 2. Backwards Compatibility Tests

### 2.1 Removed Outputs

**Purpose:** Confirm that the EFS outputs are no longer exported.

**Command:**

```bash
cd "$TF_DIR"
terraform output -json | jq 'keys' | grep -i efs || true
```

**Expected Result:** No output keys contain `efs`.

### 2.2 Removed Variables

**Purpose:** Confirm that EFS variables are no longer accepted by the mcp-gateway module.

**Command:**

```bash
cd "$TF_DIR"
grep -Rin "efs_throughput_mode\|efs_provisioned_throughput" modules/mcp-gateway/variables.tf terraform.tfvars.example 2>/dev/null || true
```

**Expected Result:** No matches.

### 2.3 Existing DocumentDB Path Preserved

**Purpose:** Ensure the DocumentDB-backed scopes initialization path is unaffected.

**Command:**

```bash
cd "$REPO_PATH"
grep -n "run-documentdb-init.sh" terraform/aws-ecs/scripts/post-deployment-setup.sh
```

**Expected Result:** The DocumentDB init branch is still present and reachable when `documentdb_cluster_endpoint` exists.

## 3. UX Tests

### 3.1 Documentation Accuracy

**Purpose:** Verify that documentation no longer presents EFS as an active component.

**Test 3.1.1 - Top-level README**

```bash
grep -n "EFS Shared Storage" "$REPO_PATH/README.md" || true
```

**Expected Result:** No match.

**Test 3.1.2 - Terraform README**

```bash
grep -n "Amazon EFS" "$REPO_PATH/terraform/README.md" "$REPO_PATH/terraform/aws-ecs/README.md" || true
```

**Expected Result:** No match (except historical release notes or changelogs, which may remain).

**Test 3.1.3 - Troubleshooting Guide**

```bash
grep -n "initialized on EFS\|run-scopes-init-task.sh" "$REPO_PATH/docs/deployment-modes.md" || true
```

**Expected Result:** No match.

**Test 3.1.4 - Architecture Diagrams**

```bash
grep -n "Encryption at rest: KMS" "$REPO_PATH/docs/architecture-diagrams.md"
```

**Expected Result:** The line should read `Encryption at rest: KMS (DocumentDB, EBS, S3)` without `EFS`.

### 3.2 Operator IAM Documentation

**Purpose:** Confirm that the operator IAM policy example no longer includes EFS permissions.

**Command:**

```bash
grep -n "elasticfilesystem" "$REPO_PATH/terraform/aws-ecs/README.md" || true
```

**Expected Result:** No match.

### 3.3 Script README

**Purpose:** Verify that the scripts README does not reference removed EFS scripts.

**Command:**

```bash
if [ -f "$REPO_PATH/terraform/aws-ecs/scripts/README.md" ]; then
    grep -n "run-scopes-init-task\|EFS\|efs" "$REPO_PATH/terraform/aws-ecs/scripts/README.md" || true
fi
```

**Expected Result:** If the file exists, no EFS references remain.

## 4. Deployment Surface Tests

### 4.1 Terraform / ECS Wiring

**Purpose:** Verify that ECS task definitions no longer declare EFS volumes or mount points.

**Test 4.1.1 - No `efs_volume_configuration` blocks**

```bash
grep -Rin "efs_volume_configuration" "$REPO_PATH/terraform/aws-ecs" --include="*.tf" || true
```

**Expected Result:** No match.

**Test 4.1.2 - Auth-server mount points are empty**

```bash
python3 - <<'PY'
import re
with open("$REPO_PATH/terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf") as f:
    text = f.read()
# Find the auth-server mountPoints block
match = re.search(r'module "ecs_service_auth".*?container_definitions.*?mountPoints\s*=\s*(\[.*?\])', text, re.DOTALL)
if match:
    block = match.group(1).replace("\n", " ").replace(" ", "")
    assert block == "[]", f"Expected empty mountPoints for auth-server, got: {block}"
    print("PASS: auth-server mountPoints is empty")
else:
    print("FAIL: Could not find auth-server mountPoints block")
    exit(1)
PY
```

**Test 4.1.3 - mcpgw mount points are empty**

```bash
python3 - <<'PY'
import re
with open("$REPO_PATH/terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf") as f:
    text = f.read()
match = re.search(r'module "ecs_service_mcpgw".*?container_definitions.*?mountPoints\s*=\s*(\[.*?\])', text, re.DOTALL)
if match:
    block = match.group(1).replace("\n", " ").replace(" ", "")
    assert block == "[]", f"Expected empty mountPoints for mcpgw, got: {block}"
    print("PASS: mcpgw mountPoints is empty")
else:
    print("FAIL: Could not find mcpgw mountPoints block")
    exit(1)
PY
```

**Test 4.1.4 - No `/efs/...` paths in container definitions**

```bash
grep -Rin "/efs/" "$REPO_PATH/terraform/aws-ecs" --include="*.tf" || true
```

**Expected Result:** No match.

### 4.2 Post-Deployment Script

**Purpose:** Verify that the post-deployment setup script no longer depends on EFS outputs or branches to EFS init.

**Test 4.2.1 - No EFS output requirement**

```bash
grep -n "mcp_gateway_efs_id\|mcp_gateway_efs_access_points" "$REPO_PATH/terraform/aws-ecs/scripts/post-deployment-setup.sh" || true
```

**Expected Result:** No match.

**Test 4.2.2 - EFS branch removed**

```bash
grep -n "run-scopes-init-task.sh\|EFS mode" "$REPO_PATH/terraform/aws-ecs/scripts/post-deployment-setup.sh" || true
```

**Expected Result:** No match.

**Test 4.2.3 - DocumentDB branch preserved**

```bash
grep -n "run-documentdb-init.sh" "$REPO_PATH/terraform/aws-ecs/scripts/post-deployment-setup.sh"
```

**Expected Result:** The DocumentDB init script is referenced.

### 4.3 Removed Files

**Purpose:** Verify that the EFS-specific files are deleted.

**Command:**

```bash
for f in \
    "$REPO_PATH/terraform/aws-ecs/modules/mcp-gateway/storage.tf" \
    "$REPO_PATH/terraform/aws-ecs/scripts/run-scopes-init-task.sh" \
    "$REPO_PATH/docker/Dockerfile.scopes-init"; do
    if [ -e "$f" ]; then
        echo "FAIL: File still exists: $f"
        exit 1
    else
        echo "PASS: Removed: $f"
    fi
done
```

**Expected Result:** All three files are reported as removed.

## 5. End-to-End API Tests

### 5.1 Green-Field Deployment Verification

**Purpose:** After applying the changes, confirm that the services start and pass health checks.

**Prerequisites:** An AWS account and a Terraform workspace configured for non-production.

**Steps:**

1. Apply the Terraform changes:

   ```bash
   cd "$TF_DIR"
   terraform apply tfplan
   ```

2. Save Terraform outputs:

   ```bash
   terraform output -json > terraform-outputs.json
   ```

3. Wait for services to stabilize:

   ```bash
   aws ecs wait services-stable \
     --cluster "$(terraform output -raw mcp_gateway_ecs_cluster_name)" \
     --services "mcp-gateway-v2-auth" "mcp-gateway-v2-registry" "mcp-gateway-v2-mcpgw"
   ```

4. Verify no EFS mounts in running tasks:

   ```bash
   aws ecs describe-tasks \
     --cluster "$(terraform output -raw mcp_gateway_ecs_cluster_name)" \
     --tasks "$(aws ecs list-tasks --cluster "$(terraform output -raw mcp_gateway_ecs_cluster_name)" --service-name mcp-gateway-v2-auth --query 'taskArns[0]' --output text)" \
     --query 'tasks[0].containers[0].mounts'
   ```

   **Expected Result:** Empty list `[]` or no EFS mount entries.

5. Initialize DocumentDB scopes:

   ```bash
   cd "$REPO_PATH/terraform/aws-ecs/scripts"
   ./run-documentdb-init.sh
   ```

6. Verify health endpoints:

   ```bash
   curl -f "http://$(terraform output -raw mcp_gateway_url)/health" || true
   curl -f "http://$(terraform output -raw mcp_gateway_auth_url)/health" || true
   ```

### 5.2 Login and Scope Authorization Verification

**Purpose:** Confirm that scopes work correctly from DocumentDB after EFS removal.

**Steps:**

1. Obtain an access token via Keycloak:

   ```bash
   TOKEN=$(curl -s -X POST \
     "https://$(terraform output -raw keycloak_url)/realms/mcp-gateway/protocol/openid-connect/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=password" \
     -d "client_id=mcp-gateway-web" \
     -d "username=$TEST_USER" \
     -d "password=$TEST_PASSWORD" \
     | jq -r '.access_token')
   ```

2. Call a registry API endpoint that requires scopes:

   ```bash
   curl -f -H "Authorization: Bearer $TOKEN" \
     "http://$(terraform output -raw mcp_gateway_url)/api/v1/servers"
   ```

   **Expected Result:** HTTP 200 with a list of servers (or empty list), not HTTP 403.

## 6. Test Execution Checklist

- [ ] Section 1.1 (no EFS references) passes
- [ ] Section 1.2 (`terraform fmt -check -recursive`) passes
- [ ] Section 1.3 (`terraform validate`) passes
- [ ] Section 1.4 (`terraform plan`) succeeds and shows EFS resources destroyed
- [ ] Section 2.1 (removed outputs) verified
- [ ] Section 2.2 (removed variables) verified
- [ ] Section 2.3 (DocumentDB path preserved) verified
- [ ] Section 3.1 (docs accuracy) verified
- [ ] Section 3.2 (operator IAM docs) verified
- [ ] Section 3.3 (script README) verified or marked Not Applicable
- [ ] Section 4.1 (ECS wiring) verified
- [ ] Section 4.2 (post-deployment script) verified
- [ ] Section 4.3 (removed files) verified
- [ ] Section 5.1 (green-field deployment) verified or skipped for non-AWS environments
- [ ] Section 5.2 (login/scope authorization) verified or skipped for non-AWS environments
- [ ] A new release note entry is drafted under `release-notes/`
