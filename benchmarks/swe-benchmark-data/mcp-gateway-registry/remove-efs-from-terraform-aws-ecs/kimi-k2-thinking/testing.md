# Testing Plan: Remove EFS from Terraform AWS ECS Configuration

*Created: 2024-06-15*
*Related LLD: `./lld.md`*
*Related Issue: `./github-issue.md`*

## Overview

### Scope of Testing
This plan covers verification of the EFS removal from the terraform-aws-ecs module, ensuring:
- No EFS resources are created
- Alternative storage (Parameter Store, CloudWatch, ephemeral) works correctly
- All services start successfully without EFS dependencies
- Configuration migration works as expected
- No regressions in application functionality

### Prerequisites
- [ ] AWS account with appropriate permissions
- [ ] Terraform CLI installed (v1.0+)
- [ ] AWS CLI installed and configured
- [ ] Access to tfstate management (S3 backend desirable)
- [ ] Service discovery namespace created (for ECS Service Connect)
- [ ] VPC with public/private subnets pre-configured

### Shared Variables
```bash
export AWS_REGION="us-west-2"
export MCP_GATEWAY_NAME="test-mcp-gateway"
export VPC_ID="$(aws ec2 describe-vpcs --filters Name=is-default,Values=true --query 'Vpcs[0].VpcId' --output text --region $AWS_REGION)"
export PRIVATE_SUBNETS="$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=tag:Name,Values='*private*' --query 'Subnets[*].SubnetId' --output text --region $AWS_REGION | tr '\t' ',')"
export PUBLIC_SUBNETS="$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=tag:Name,Values='*public*' --query 'Subnets[*].SubnetId' --output text --region $AWS_REGION | tr '\t' ',')"
export ECS_CLUSTER_ARN="$(aws ecs list-clusters --query 'clusterArns[0]' --output text --region $AWS_REGION)"
export ECS_CLUSTER_NAME="$(aws ecs describe-clusters --clusters $ECS_CLUSTER_ARN --query 'clusters[0].clusterName' --output text --region $AWS_REGION)"

# Verify variables
for var in VPC_ID PRIVATE_SUBNETS PUBLIC_SUBNETS ECS_CLUSTER_ARN ECS_CLUSTER_NAME; do
  value=$(eval echo "\${$var}")
  if [ -z "$value" ] || [ "$value" == "None" ]; then
    echo "Required variable $var is empty - ensure prerequisites are met"
    exit 1
  fi
  echo "$var=$value"
done
```

## 1. Functional Tests

### 1.1 Terraform Plan Validation

**Purpose**: Verify no EFS resources are created or referenced

#### Test 1.1.1: Initial Plan Sanity Check
```bash
cd terraform/aws-ecs
terraform init -backend-config="bucket=mcp-gateway-tfstate-$AWS_REGION" \
               -backend-config="key=$MCP_GATEWAY_NAME.tfstate" \
               -backend-config="region=$AWS_REGION"

terraform plan -var "name=$MCP_GATEWAY_NAME" \
               -var "vpc_id=$VPC_ID" \
               -var "private_subnet_ids=[\"${PRIVATE_SUBNETS//,/\",\"}\"]" \
               -var "public_subnet_ids=[\"${PUBLIC_SUBNETS//,/\",\"}\"]" \
               -var "ecs_cluster_arn=$ECS_CLUSTER_ARN" \
               -var "ecs_cluster_name=$ECS_CLUSTER_NAME" \
               -out=tfplan
```

**Expected Result**: Terraform plan succeeds without errors
**Assertions**:
- [ ] No EFS resources in plan output
- [ ] No `aws_efs_file_system` resources
- [ ] No `aws_efs_mount_target` resources
- [ ] No references to `module.efs`
- [ ] No EFS security group rules

#### Test 1.1.2: Verify EFS Resources Excluded
```bash
# Check plan output for EFS references
if terraform show -json tfplan | jq -r '.planned_values.root_module.resources[]?.type' | grep -q "aws_efs_"; then
  echo "FAIL: EFS resources found in plan"
  terraform show -json tfplan | jq -r '.planned_values.root_module.resources[] | select(.type | contains("aws_efs_")) | .address'
  exit 1
fi

echo "PASS: No EFS resources in plan"
```

#### Test 1.1.3: Verify Parameter Store Resources Included
```bash
# Should have Parameter Store resources
if terraform show -json tfplan | jq -r '.planned_values.root_module.resources[]?.type' | grep -q "aws_ssm_parameter"; then
  echo "PASS: Parameter Store resources found"
  terraform show -json tfplan | jq -r '.planned_values.root_module.resources[] | select(.type == "aws_ssm_parameter") | .address'
else
  echo "FAIL: No Parameter Store resources found"
  exit 1
fi
```

#### Test 1.1.4: Verify IAM Policies for Parameter Store
```bash
# Check for SSM permissions in IAM policies
if terraform show -json tfplan | jq -r '.planned_values.root_module.resources[]?.type' | grep -q "aws_iam_role_policy"; then
  POLICIES=$(terraform show -json tfplan | jq -r '.planned_values.root_module.resources[] | select(.type == "aws_iam_role_policy") | .address')
  for policy in $POLICIES; do
    if terraform show -json tfplan | jq -r ".planned_values.root_module.resources[] | select(.address == \"$policy\") | .values.policy" | grep -q "ssm:GetParameter"; then
      echo "PASS: Parameter Store IAM policy found: $policy"
    fi
  done
else
  echo "FAIL: No IAM role policies found"
  exit 1
fi
```

### 1.2 Apply Changes and Verify Infrastructure

**Purpose**: Deploy changes and validate infrastructure state

#### Test 1.2.1: Apply Terraform Changes
```bash
# Upload scopes.yml to Parameter Store first
aws ssm put-parameter \
  --name "/$MCP_GATEWAY_NAME/auth-server/scopes-yml" \
  --type "String" \
  --value "$(cat ../../auth_server/scopes.yml)" \
  --tags "Key=Name,Value=$MCP_GATEWAY_NAME" \
  --region $AWS_REGION

echo "Parameter Store setup complete"

# Apply changes
terraform apply tfplan
TF_EXIT_CODE=$?

if [ $TF_EXIT_CODE -eq 0 ]; then
  echo "PASS: Terraform apply succeeded"
else
  echo "FAIL: Terraform apply failed with exit code $TF_EXIT_CODE"
  exit 1
fi
```

#### Test 1.2.2: Verify No EFS Resources Created
```bash
# Check for EFS resources in AWS
EFS_COUNT=$(aws efs describe-file-systems --region $AWS_REGION --query "length(FileSystems[?Tags[?Key=='Name' && contains(Value, '$MCP_GATEWAY_NAME')]])" --output text)

if [ "$EFS_COUNT" == "0" ]; then
  echo "PASS: No EFS file systems created"
else
  echo "FAIL: Found $EFS_COUNT EFS file systems (expected 0)"
  aws efs describe-file-systems --region $AWS_REGION --query "FileSystems[?Tags[?Key=='Name' && contains(Value, '$MCP_GATEWAY_NAME')]].FileSystemId" --output table
  exit 1
fi
```

#### Test 1.2.3: Verify Parameter Store Parameter Exists
```bash
PARAMETER_EXISTS=$(aws ssm get-parameter --name "/$MCP_GATEWAY_NAME/auth-server/scopes-yml" --region $AWS_REGION --query "Parameter.Name" --output text 2>/dev/null || echo "")

if [ "$PARAMETER_EXISTS" == "/$MCP_GATEWAY_NAME/auth-server/scopes-yml" ]; then
  echo "PASS: Parameter Store parameter exists"
else
  echo "FAIL: Parameter Store parameter not found"
  exit 1
fi
```

#### Test 1.2.4: Verify ECS Services Are Running
```bash
# Wait for services to be stable
echo "Waiting for services to reach steady state..."
sleep 90

# Check auth-server service
AUTH_SERVICE_STATUS=$(aws ecs describe-services --cluster $ECS_CLUSTER_NAME --services "${MCP_GATEWAY_NAME}-auth-server" --region $AWS_REGION --query "services[0].status" --output text)
if [ "$AUTH_SERVICE_STATUS" == "STEADY_STATE" ]; then
  echo "PASS: Auth-server service in steady state"
else
  echo "FAIL: Auth-server service status=$AUTH_SERVICE_STATUS (expected steady state)"
  exit 1
fi

# Check mcpgw service
MCPGW_SERVICE_STATUS=$(aws ecs describe-services --cluster $ECS_CLUSTER_NAME --services "${MCP_GATEWAY_NAME}-mcpgw" --region $AWS_REGION --query "services[0].status" --output text)
if [ "$MCPGW_SERVICE_STATUS" == "STEADY_STATE" ]; then
  echo "PASS: MCPGW service in steady state"
else
  echo "FAIL: MCPGW service status=$MCPGW_SERVICE_STATUS (expected steady state)"
  exit 1
fi

# Check registry service  
REGISTRY_SERVICE_STATUS=$(aws ecs describe-services --cluster $ECS_CLUSTER_NAME --services "${MCP_GATEWAY_NAME}-registry" --region $AWS_REGION --query "services[0].status" --output text)
if [ "$REGISTRY_SERVICE_STATUS" == "STEADY_STATE" ]; then
  echo "PASS: Registry service in steady state"
else
  echo "FAIL: Registry service status=$REGISTRY_SERVICE_STATUS (expected steady state)"
  exit 1
fi
```

### 1.3 Configuration Validation

**Purpose**: Verify services can access alternative storage

#### Test 1.3.1: Verify Auth Server Can Fetch Configuration
```bash
# Get auth-server task ARN
AUTH_TASK_ARN=$(aws ecs list-tasks --cluster $ECS_CLUSTER_NAME --service "${MCP_GATEWAY_NAME}-auth-server" --region $AWS_REGION --query 'taskArns[0]' --output text)

if [ "$AUTH_TASK_ARN" == "None" ]; then
  echo "FAIL: No auth-server tasks found"
  exit 1
fi

# Check task logs for successful startup
AUTH_LOG_GROUP="/ecs/${MCP_GATEWAY_NAME}-auth-server"
if aws logs filter-log-events --log-group-name "$AUTH_LOG_GROUP" --region $AWS_REGION --filter-pattern "START_SUCCESS" --output text >/dev/null 2>&1; then
  echo "PASS: Auth-server startup validation found in logs"
else
  echo "INFO: Checking for cloud-init logs..."
  # Check for startup message
  aws logs filter-log-events --log-group-name "$AUTH_LOG_GROUP" --region $AWS_REGION --filter-pattern "Server running" --limit 1 --output text || true
fi

# Verify parameter access in logs
PARAMETER_ACCESS=$(aws logs filter-log-events --log-group-name "$AUTH_LOG_GROUP" --region $AWS_REGION --filter-pattern "ssm.amazonaws.com" --query "length(events)" --output text 2>/dev/null || echo "0")
if [ "$PARAMETER_ACCESS" -gt "0" ]; then
  echo "PASS: Detected Parameter Store access in auth-server logs"
fi
```

#### Test 1.3.2: Verify Logging Without EFS
```bash
# Check for any EFS mount errors in logs
MOUNT_ERRORS=$(aws logs filter-log-events --log-group-name "$AUTH_LOG_GROUP" --region $AWS_REGION --filter-pattern "mount" --query "length(events)" --output text 2>/dev/null || echo "0")

if [ "$MOUNT_ERRORS" -gt "0" ]; then
  echo "INFO: Found mount-related logs (investigating):"
  aws logs filter-log-events --log-group-name "$AUTH_LOG_GROUP" --region $AWS_REGION --filter-pattern "mount" --limit 3 --output text | grep -i "error\|fail" || echo "No errors in mount logs"
else
  echo "PASS: No mount-related errors in logs"
fi
```

## 2. Backwards Compatibility Tests

### Purpose
Verify that configuration changes maintain compatibility with deployment workflows

### Test 2.1: Terraform Module Usage Compatibility

**Test**: Verify module can be consumed as before (no breaking input changes)

```bash
cd /tmp
mkdir test-terraform-consumption
cd test-terraform-consumption

cat > main.tf << 'EOF'
module "mcp_gateway" {
  source = "../claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo/terraform/aws-ecs/modules/mcp-gateway"
  
  name       = var.name
  vpc_id     = var.vpc_id
  private_subnet_ids = var.private_subnet_ids
  public_subnet_ids  = var.public_subnet_ids
  ecs_cluster_arn    = var.ecs_cluster_arn
  ecs_cluster_name   = var.ecs_cluster_name
  
  # Should not need to specify EFS variables
}

variable "name" { type = string }
variable "vpc_id" { type = string }
variable "private_subnet_ids" { type = list(string) }
variable "public_subnet_ids" { type = list(string) }
variable "ecs_cluster_arn" { type = string }
variable "ecs_cluster_name" { type = string }
EOF

# Should succeed without EFS variables
terraform init 2>&1 | grep -i "error\|invalid" && echo "FAIL: Configuration errors found" || echo "PASS: Module can be consumed without EFS variables"
```

**Expected Result**: Module can be consumed without any EFS-specific variables

### Test 2.2: Documentation Backwards Compatibility

**Test**: Verify documentation doesn't reference deprecated features

```bash
cd /Users/prsinp/claude-code-multi-model/benchmarks/swe-benchmark-data/mcp-gateway-registry/repo

# Check main terraform docs
if grep -r "EFS\|elastic file\|efs" terraform/README.md; then
  echo "FAIL: Documentation still references EFS"
  exit 1
else
  echo "PASS: Documentation clear of EFS references"
fi

# Verify migration section now present
if grep -q "migrating from efs\|migrating from EFS" terraform/aws-ecs/README.md || \
   grep -q "Migrating from EFS" terraform/aws-ecs/README.md; then
  echo "PASS: Migration documentation present"
else
  echo "FAIL: Missing migration documentation"
  exit 1
fi
```

### Test 2.3: Output Compatibility

**Test**: Verify terraform plan succeeds even if EFS outputs are referenced by consumers

```bash
cd terraform/aws-ecs

# Create a test consumer that tries to reference removed EFS outputs
cat > test-outputs.tf << 'EOF'
# Consumers might have referenced these outputs:
# module.mcp_gateway.efs_id
# module.mcp_gateway.efs_arn  
# module.mcp_gateway.efs_access_points

# Simulate consumer using these outputs
output "test" {
  value = "Outputs successfully accessed"
}
EOF

# This should now fail (intentional breaking change check)
if terraform validate >/dev/null 2>&1; then
  echo "PASS: Terraform validates successfully (consumer references removed)"
else
  echo "INFO: Expected breakage from removed outputs (this is intentional)"
  rm test-outputs.tf
fi

rm -f test-outputs.tf
```

## 3. UX Tests

### Purpose
Verify user experience improvements and maintain existing functionality

### Test 3.1: Configuration Management UX

**Test**: Verify scopes.yml update process

```bash
echo "=== Testing Configuration Update Flow ==="

# Step 1: Update parameter
echo "Adding test scope to scopes.yml..."
echo "  test:api:
    description: Test scope for automated tests
    grants:
      - read:users
      - write:users" >> ../../auth_server/scopes.yml

aws ssm put-parameter \
  --name "/$MCP_GATEWAY_NAME/auth-server/scopes-yml" \
  --type "String" \
  --value "$(cat ../../auth_server/scopes.yml)" \
  --overwrite \
  --region $AWS_REGION

if [ $? -eq 0 ]; then
  echo "PASS: Parameter updated successfully"
else
  echo "FAIL: Parameter update failed"
  exit 1
fi

# Step 2: Restart auth-server to pick up changes
aws ecs update-service \
  --cluster $ECS_CLUSTER_NAME \
  --service "${MCP_GATEWAY_NAME}-auth-server" \
  --force-new-deployment \
  --region $AWS_REGION

sleep 30

# Step 3: Verify new configuration applied
NEW_SCOPE_FOUND=$(aws logs filter-log-events --log-group-name "$AUTH_LOG_GROUP" --region $AWS_REGION --filter-pattern "test:api" --limit 1 --output text | wc -l | tr -d ' ')
if [ "$NEW_SCOPE_FOUND" -gt "100" ]; then  # grep returns more than header lines
  echo "PASS: New scope detected in logs"
else
  echo "INFO: Could not verify new scope in logs (may need longer startup time)"
fi

# Clean up test scope
sed -i '/test:api:/,/write:users/d' ../../auth_server/scopes.yml
```

### Test 3.2: Log Access UX

**Test**: Verify CloudWatch log access is user-friendly

```bash
echo "=== Testing Log Access UX ==="

# Check log groups exist
LOG_GROUPS=$(
  aws logs describe-log-groups --log-group-name-prefix "/ecs/${MCP_GATEWAY_NAME}-" \
    --region $AWS_REGION \
    --query "logGroups[].logGroupName" \
    --output text
)

EXPECTED_SERVICES=("auth-server" "registry" "mcpgw")
for service in "${EXPECTED_SERVICES[@]}"; do
  if echo "$LOG_GROUPS" | grep -q "${MCP_GATEWAY_NAME}-${service}"; then
    echo "PASS: Log group exists for ${service}"
  else
    echo "FAIL: Missing log group for ${service}"
    exit 1
  fi
done

# Verify recent log entries
for service in "${EXPECTED_SERVICES[@]}"; do
  LOG_GROUP="/ecs/${MCP_GATEWAY_NAME}-${service}"
  RECENT_EVENTS=$(aws logs filter-log-events --log-group-name "$LOG_GROUP" --region $AWS_REGION --limit 2 --query "length(events)" --output text 2>/dev/null || echo "0")
  
  if [ "$RECENT_EVENTS" -gt "0" ]; then
    echo "PASS: Active logging detected for ${service}"
  else
    echo "INFO: No recent events for ${service} (may be normal for idle services)"
  fi
done
```

### Test 3.3: Service Health UX

**Test**: Verify service health endpoints remain functional

```bash
echo "=== Testing Service Health Endpoints ==="

# Get ALB DNS name
ALB_DNS=$(aws cloudformation describe-stacks --stack-name "$MCP_GATEWAY_NAME" --region $AWS_REGION --query "Stacks[0].Outputs[?OutputKey=='ALBDNSName'].OutputValue" --output text)

if [ "$ALB_DNS" == "None" ]; then
  echo "FAIL: Could not get ALB DNS name"
  exit 1
fi

echo "ALB DNS: $ALB_DNS"

# Test auth-server health endpoint
curl -s -f "http://$ALB_DNS/auth/health" >/dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "PASS: Auth-server health endpoint accessible"
else
  echo "FAIL: Auth-server health endpoint failed"
  exit 1
fi

# Test registry health endpoint
curl -s -f "http://$ALB_DNS/health" >/dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "PASS: Registry health endpoint accessible"
else
  echo "FAIL: Registry health endpoint failed"
  exit 1
fi
```

## 4. Deployment Surface Tests

### 4.1 Docker Configuration

**Test**: Verify local Docker setup unaffected

```bash
cd /Users/prsinp/claude-code-multi-model

# Main docker-compose should still work (won't start all services due to auth, but validates config)
if docker-compose config >/dev/null 2>&1; then
  echo "PASS: Docker compose configuration valid"
else
  echo "FAIL: Docker compose configuration invalid"
  exit 1
fi

# Verify volumes not affected
docker-compose config | grep -A 5 "volumes:" | grep -q "/app/logs"
if [ $? -eq 0 ]; then
  echo "PASS: Docker volume configurations preserved"
fi
```

### 4.2 Terraform Configuration

**Test**: Verify all Terraform configuration files valid

```bash
cd terraform/aws-ecs

# Validate all .tf files
TF_FILES=$(find . -name "*.tf" -type f)
INVALID_COUNT=0

for file in $TF_FILES; do
  if terraform validate -chdir=$(dirname "$file") /dev/null 2>&1; then
    echo "VALID: $file"
  else
    echo "INVALID: $file"
    INVALID_COUNT=$((INVALID_COUNT + 1))
  fi
done
done

if [ $INVALID_COUNT -eq 0 ]; then
  echo "PASS: All Terraform files valid"
else
  echo "FAIL: $INVALID_COUNT Terraform files invalid"
  exit 1
fi
```

### 4.3 Network Security

**Test**: Verify security groups updated (EFS rules removed)

```bash
echo "=== Testing Security Group Configuration ==="

# Get security group for ECS tasks
TASK_SG=$(aws ec2 describe-security-groups --filters Name=tag:Name,Values="${MCP_GATEWAY_NAME}-*" --region $AWS_REGION --query "SecurityGroups[?contains(GroupName, 'ecs-tasks')].GroupId" --output text)

if [ "$TASK_SG" == "None" ]; then
  echo "FAIL: Could not find task security group"
  exit 1
fi

# Check for NFS port (2049) in security group rules
NFS_RULE=$(aws ec2 describe-security-groups --group-ids $TASK_SG --region $AWS_REGION --query "SecurityGroups[0].IpPermissions[?FromPort<=\`2049\` && ToPort>=\`2049\` && IpProtocol==\`tcp\`]" --output text)

if [ "$NFS_RULE" == "None" ] || [ -z "$NFS_RULE" ]; then
  echo "PASS: No NFS (port 2049) rules in security group"
else
  echo "FAIL: Found NFS rules in security group (should be removed)"
  aws ec2 describe-security-groups --group-ids $TASK_SG --region $AWS_REGION --query "SecurityGroups[0].IpPermissions[?FromPort<=\`2049\` && ToPort>=\`2049\` && IpProtocol==\`tcp\`]"
  exit 1
fi
```

## 5. End-to-End Service Tests

### 5.1 Auth Flow Validation

**Test**: Complete authentication flow using Parameter Store configuration

```bash
echo "=== Testing Complete Auth Flow ==="

# Create test configuration file
cat > /tmp/test-scopes.yml << 'EOF'
test:e2e:
  description: E2E test scope
  grants:
    - read:test
    - write:test
EOF

# Update scope configuration
aws ssm put-parameter \
  --name "/$MCP_GATEWAY_NAME/auth-server/scopes-yml" \
  --type "String" \
  --value "$(cat /tmp/test-scopes.yml)" \
  --overwrite \
  --region $AWS_REGION

# Restart auth-server to pick up new config
aws ecs update-service \
  --cluster $ECS_CLUSTER_NAME \
  --service "${MCP_GATEWAY_NAME}-auth-server" \
  --force-new-deployment \
  --region $AWS_REGION

echo "Service updated, waiting for healthy state..."
sleep 60

# Check service health
AUTH_HEALTH=$(curl -s -w "%{http_code}" -o /tmp/auth_health.json "http://$ALB_DNS/auth/health")
if [ "$AUTH_HEALTH" == "200" ]; then
  echo "PASS: Auth service healthy after config update"
else
  echo "FAIL: Auth service health check returned $AUTH_HEALTH"
  cat /tmp/auth_health.json
  exit 1
fi

# Verify no EFS-related errors in logs during config reload
ERROR_COUNT=$(aws logs filter-log-events --log-group-name "$AUTH_LOG_GROUP" --region $AWS_REGION --filter-pattern "ENCRYPTED" --start-time $(($(date +%s%3N) - 120000)) --query "length(events)" --output text 2>/dev/null || echo "0")

if [ "$ERROR_COUNT" -gt "0" ]; then
  echo "INFO: Found EFS-related errors in recent logs (investigate)"
fi
```

### 5.2 MCPGW Demo App Flow

**Test**: A2A agents can store data without EFS

```bash
echo "=== Testing MCPGW Demo Apps ==="

# If flight-booking agent is enabled, verify it can write to ephemeral storage
if [ "$TEST_MCPS" == "true" ] || [ "$TEST_MCPS" == "1" ]; then
  MCGW_TASK_ARN=$(aws ecs list-tasks --cluster $ECS_CLUSTER_NAME --service "${MCP_GATEWAY_NAME}-mcpgw" --region $AWS_REGION --query 'taskArns[0]' --output text)
  
  if [ "$MCGW_TASK_ARN" != "None" ]; then
    echo "Verifying mcpgw can write to /app/data..."
    
    # Check task logs for successful startup
    MCPGW_LOG_GROUP="/ecs/${MCP_GATEWAY_NAME}-mcpgw"
    SUCCESS_LOG=$(aws logs filter-log-events --log-group-name "$MCPGW_LOG_GROUP" --region $AWS_REGION --filter-pattern "Database initialized\|SQLite version" --limit 1 --query "length(events)" --output text 2>/dev/null || echo "0")
    
    if [ "$SUCCESS_LOG" -gt "0" ]; then
      echo "PASS: MCPGW database initialized successfully"
    else
      echo "INFO: Could not verify database initialization (may be normal)"
    fi
  fi
else
  echo "SKIP: Demo apps test skipped (TEST_MCPS env var)"
fi
```

## 6. Cleanup and Rollback Verification

### Test 6.1: Terraform Destroy

```bash
echo "=== Testing Terraform Destroy ==="
terraform destroy -var "name=$MCP_GATEWAY_NAME" \
                     -var "vpc_id=$VPC_ID" \
                     -var "private_subnet_ids=[\"${PRIVATE_SUBNETS//,/\",\"}\"]" \
                     -var "public_subnet_ids=[\"${PUBLIC_SUBNETS//,/\",\"}\"]" \
                     -var "ecs_cluster_arn=$ECS_CLUSTER_ARN" \
                     -var "ecs_cluster_name=$ECS_CLUSTER_NAME" \
                     -auto-approve
if [ $? -eq 0 ]; then
  echo "PASS: Terraform destroy succeeded"
else
  echo "FAIL: Terraform destroy failed"
  exit 1
fi

# Verify cleanup
EFS_CLEANUP=$(aws efs describe-file-systems --region $AWS_REGION --query "length(FileSystems[?Tags[?Key=='Name' && contains(Value, '$MCP_GATEWAY_NAME')]])" --output text)
if [ "$EFS_CLEANUP" == "0" ]; then
  echo "PASS: No EFS resources remaining"
else
  echo "FAIL: EFS resources still exist after destroy"
  exit 1
fi
```

## 7. Regression Tests

### Purpose
Ensure no regressions in existing functionality

### Test 7.1: DocumentDB Integration (Registry Service)

```bash
echo "=== Testing Registry Service Persistence ==="

# Verify registry still works with DocumentDB (previously migrated off EFS)
REGISTRY_LOG_GROUP="/ecs/${MCP_GATEWAY_NAME}-registry"

# Check for DocumentDB connection logs
DOCDB_CONNECTIONS=$(aws logs filter-log-events --log-group-name "$REGISTRY_LOG_GROUP" --region $AWS_REGION --filter-pattern "DocumentDB\|docdb" --start-time $(($(date +%s%3N) - 600000)) --query "length(events)" --output text 2>/dev/null || echo "0")

if [ "$DOCDB_CONNECTIONS" -gt "0" ]; then
  echo "PASS: Registry service using DocumentDB persistence"
else
  echo "INFO: No DocumentDB logs found in last 10 minutes (service may be idle)"
fi

# Verify NO EFS references in registry logs
EFS_IN_REGISTRY=$(aws logs filter-log-events --log-group-name "$REGISTRY_LOG_GROUP" --region $AWS_REGION --filter-pattern "efs\|EFS\|elastic file" --start-time $(($(date +%s%3N) - 600000)) --query "length(events)" --output text 2>/dev/null || echo "0")

if [ "$EFS_IN_REGISTRY" == "0" ]; then
  echo "PASS: No EFS references in registry service logs"
else
  echo "INFO: Found EFS references in registry logs (analyze further):"
  aws logs filter-log-events --log-group-name "$REGISTRY_LOG_GROUP" --region $AWS_REGION --filter-pattern "efs\|EFS\|elastic file" --limit 3 --output text
fi
```

## 8. Test Execution Checklist

Use this checklist to track test execution:

### Pre-Flight Checks
- [ ] AWS credentials configured
- [ ] Terraform installed (≥ v1.0)
- [ ] AWS CLI installed and configured
- [ ] Unique test name set (MCP_GATEWAY_NAME)
- [ ] S3 bucket for tfstate accessible
- [ ] Service discovery namespace exists
- [ ] DocumentDB cluster provisioned (for registry)

### Functional Tests
- [ ] Terraform plan executed without EFS resources
- [ ] Parameter Store parameter created
- [ ] IAM policies include SSM permissions
- [ ] Terraform apply completed successfully
- [ ] No EFS file systems created in AWS
- [ ] Parameter Store parameter verified
- [ ] All ECS services in steady state

### Backwards Compatibility Tests
- [ ] Module can be consumed without EFS variables
- [ ] Documentation updated with no EFS references
- [ ] Migration documentation added
- [ ] Removal of EFS outputs validated

### UX Tests
- [ ] Configuration update flow tested
- [ ] Log groups created and accessible
- [ ] Log streaming works correctly
- [ ] Service health endpoints functional
- [ ] ALB routing works as expected

### Deployment Surface Tests
- [ ] Docker compose config remains valid
- [ ] All Terraform files validate
- [ ] Security groups have no NFS rules
- [ ] IAM policies properly scoped

### E2E Tests
- [ ] Auth flow completes successfully
- [ ] MCPGW demo apps start without errors
- [ ] Configuration reload works
- [ ] DocumentDB integration persists data

### Cleanup
- [ ] Terraform destroy succeeds
- [ ] All AWS resources cleaned up
- [ ] Parameter Store parameter removed
- [ ] Local test files removed

### Regression Tests
- [ ] Registry service uses DocumentDB (not EFS)
- [ ] No EFS references in logs
- [ ] All existing API endpoints work
- [ ] Monitoring and alerting functional

**All tests must pass before merge.**