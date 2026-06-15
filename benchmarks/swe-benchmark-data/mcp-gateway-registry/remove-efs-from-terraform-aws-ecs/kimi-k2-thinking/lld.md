# Low-Level Design: Remove EFS from Terraform AWS ECS Configuration

*Created: 2024-06-15*
*Author: Claude*
*Status: Draft*
*Model: moonshotai.kimi-k2-thinking*

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

## Overview

### Problem Statement
The `terraform-aws-ecs` module currently creates and uses Amazon EFS (Elastic File System) for persistent storage, which adds unnecessary complexity, cost, and AWS dependencies. The registry service has already been successfully migrated off EFS to use ephemeral storage combined with DocumentDB. We need to complete this migration by removing EFS usage from auth-server and mcpgw services, then eliminate the EFS infrastructure entirely.

### Goals
- Remove all EFS infrastructure from terraform-aws-ecs module
- Migrate auth-server storage needs to AWS-native services (CloudWatch, Parameter Store)
- Migrate mcpgw data storage to appropriate alternatives (ephemeral/S3)
- Maintain backward compatibility where feasible
- Minimize application code changes
- Provide clear migration path for users

### Non-Goals
- Change application logic or features
- Modify Helm charts or Kubernetes deployment patterns
- Migrate existing EFS data (greenfield focus)
- Change local developer workflows (docker-compose)

## Codebase Analysis

### Key Files Reviewed

| File Path | Purpose | Relevance to This Change |
|-----------|---------|--------------------------|
| `terraform/aws-ecs/modules/mcp-gateway/storage.tf` | Defines EFS module and security group | Primary file to delete |
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | ECS task definitions with EFS volume mounts | Remove volume configs |
| `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | EFS-related variables | Variables to delete |
| `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` | EFS resource outputs | Outputs to delete |
| `terraform/aws-ecs/modules/mcp-gateway/iam.tf` | IAM policies (may reference EFS) | Review for EFS references |
| `terraform/README.md` | High-level documentation | Update to remove EFS mentions |
| `auth_server/scopes.yml` | Auth server scopes configuration | Migrate to Parameter Store |

### Current EFS Usage Pattern

```
EFS File System (module.efs)
├── Access Points (6 total)
│   ├── servers (unused)
│   ├── models (unused)
│   ├── logs → mounted by auth-server at /app/logs
│   ├── auth_config → mounted by auth-server at /efs/auth_config/scopes.yml
│   ├── agents (unused)
│   └── mcpgw_data → mounted by mcpgw at /app/data
└── Security Group (NFS port 2049)
```

### Services Using EFS

#### auth-server
- **mcp-logs volume**: Mounted at `/app/logs` for audit logs
  - Already writes to CloudWatch Logs simultaneously
  - Can safely remove EFS logging
  - **Migration**: Rely solely on CloudWatch Logs

- **auth-config volume**: Mounted at `/efs/auth_config` for `scopes.yml`
  - Contains OAuth/OIDC scope definitions
  - Static configuration file
  - **Migration**: AWS Systems Manager Parameter Store

#### mcpgw
- **mcpgw-data volume**: Mounted at `/app/data` 
  - Used by demo A2A agents for SQLite databases (bookings.db, flights.db)
  - Non-critical demo data
  - **Migration**: Use ephemeral storage (data can be recreated)

#### registry
- Already migrated OFF EFS (lines 1367-1420 in ecs-services.tf)
- Uses ephemeral storage + DocumentDB
- Pattern to follow for other services

### Integration Points

| Component | Integration Type | Details |
|-----------|------------------|---------|
| CloudWatch Logs | Currently using alongside EFS | Will become primary logging mechanism |
| AWS Systems Manager Parameter Store | New integration | Store scopes.yml configuration |
| Security Groups | EFS security group creation | Remove entirely |
| Terraform outputs | EFS resource outputs | Remove outputs |

### Constraints and Limitations Discovered

1. **Application Dependencies**: Code references to `/app/logs` and `/efs/auth_config` exist in Dockerfiles and start scripts
2. **Configuration Files**: `scopes.yml` must be accessible to auth-server at startup
3. **Demo Data**: A2A agents expect SQLite databases at `/app/data`
4. **Terraform Version**: Cannot change provider versions as part of this work
5. **Backward Compatibility**: Removing outputs is a breaking change - document clearly

## Architecture

### System Context Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Before (with EFS)                             │
└─────────────────────────────────────────────────────────────────┘

Users → ALB → ECS Service → Container
                    ↓
              ┌─────┴─────┐
              ↓           ↓
        CloudWatch    EFS Volume
         Logs

┌─────────────────────────────────────────────────────────────────┐
│                    After (EFS removed)                           │
└─────────────────────────────────────────────────────────────────┘

Users → ALB → ECS Service → Container
                    ↓
              CloudWatch Logs
                    ↑
              Parameter Store (config)
```

### Sequence Diagram

```
Before (auth-server startup):
1. ECS Task starts
2. Mount EFS volume
3. Read /efs/auth_config/scopes.yml
4. Start application
5. Write logs to both EFS and CloudWatch

After (auth-server startup):
1. ECS Task starts
2. Fetch parameters from Parameter Store
3. Write scopes.yml to local filesystem
4. Start application
5. Write logs only to CloudWatch
```

### Component Diagram

```
┌─────────────────────────────────────────────────────────┐
│              terraform-aws-ecs-module                   │
└─────────────────────────────────────────────────────────┘
                        │
        ┌──────────────┬──────────────┬───────────────┐
        ↓              ↓              ↓               ↓
    ┌────────┐    ┌──────────┐   ┌──────────┐  ┌──────────┐
    │ Registry│   │ Auth-Server│  │   MCPGW  │  │   ALB    │
    └────────┘    └──────────┘   └──────────┘  └──────────┘
                      │                │
                 CloudWatch        Ephemeral
                    Logs           + S3 (if needed)
                      │
            ┌─────────────────┐
            │ Parameter Store │
            └─────────────────┘
```

## Data Models

### New AWS Resources

#### Parameter Store Parameters

```terraform
resource "aws_ssm_parameter" "scopes_yml" {
  name        = "/${var.name}/auth-server/scopes-yml"
  description = "OAuth/OIDC scopes configuration for auth-server"
  type        = "String"
  value       = file("${path.module}/scopes.yml")
  
  tags = local.common_tags
}
```

### Configuration Data Changes

| Data Location Before | Data Location After | Type |
|---------------------|---------------------|------|
| EFS: `/app/logs` | CloudWatch Logs | Log Data |
| EFS: `/efs/auth_config/scopes.yml` | Parameter Store + Ephemeral | Configuration |
| EFS: `/app/data/*.db` | Ephemeral/S3 | Demo Database |

## API / CLI Design

No API or CLI changes required. This is infrastructure-only change.

## Configuration Parameters

### Removed Environment Variables
| Variable | Type | Description |
|----------|------|-------------|
| `SCOPES_CONFIG_PATH` | string | Path to scopes.yml file (remove) |

### New Environment Variables
| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `PARAMETER_STORE_PATH` | string | `/mcp-gateway/auth-server/scopes-yml` | Yes | Parameter Store path for scopes config |
| `CONFIG_REFRESH_INTERVAL` | int | 300 | No | How often to refresh config (seconds) |

### Modified Variables
- `enable_efs_logging`: Remove variable entirely
- `efs_throughput_mode`: Remove variable entirely  
- `efs_provisioned_throughput`: Remove variable entirely

### Deployment Surface Changes

| File | Change Type | Description |
|------|------------|-------------|
| `.env.example` | Remove | EFS-related variables |
| `docker-compose.yml` | No change | Local development unchanged |
| `terraform.tfvars.example` | Remove | EFS configuration examples |

## New Dependencies

**None.** This change removes infrastructure dependencies rather than adding them.

**Removed Dependencies:**
- `terraform-aws-modules/efs/aws` Terraform module

**Existing Dependencies (Already in Use):**
- AWS Systems Manager (Parameter Store) - already used for other configs
- CloudWatch Logs - already configured for registry service

## Implementation Details

### Step-by-Step Implementation Plan

#### Step 1: Remove EFS Module (storage.tf)
**File:** `terraform/aws-ecs/modules/mcp-gateway/storage.tf`
**Action:** Delete entire file
**Lines affected:** 1-183 (entire file)

**Verification:**
```bash
grep -r "module \"efs\"" terraform/
# Should return no results
```

#### Step 2: Update Auth-Server Task Definition
**File:** `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`
**Location:** Search for "auth-server" service definition (~lines 490-610)

**Changes:**
1. Remove volume definitions (lines ~542-556):
```terraform
volume = {
  mcp-logs = { ... }
  auth-config = { ... }
}
```

2. Remove mountPoints from container definition (lines within auth-server container):
```terraform
mountPoints = [
  {
    sourceVolume  = "mcp-logs"
    containerPath = "/app/logs"
    readOnly      = false
  },
  {
    sourceVolume  = "auth-config"
    containerPath = "/efs/auth_config"
    readOnly      = false
  }
]
```

3. Update SCOPES_CONFIG_PATH environment variable:
```terraform
# Before:
{
  name  = "SCOPES_CONFIG_PATH"
  value = "/efs/auth_config/auth_config/scopes.yml"
}

# After:
{
  name  = "SCOPES_CONFIG_PATH"
  value = "/tmp/scopes.yml"  # Local file created from Parameter Store
}
```

4. Add IAM policy for Parameter Store access:
```terraform
# Add to auth-server task execution role in iam.tf
resource "aws_iam_role_policy" "auth_server_parameter_store" {
  name = "${local.name_prefix}-auth-server-parameter-store"
  role = aws_iam_role.auth_server_task_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath"
        ]
        Resource = "arn:aws:ssm:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:parameter/${var.name}/auth-server/*"
      }
    ]
  })
}
```

#### Step 3: Update MCPGW Task Definition
**File:** `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`
**Location:** Search for "mcpgw" service definition (~lines 1800-1890)

**Changes:**
1. Remove volume definition (lines ~1860-1866):
```terraform
volume = {
  mcpgw-data = {
    efs_volume_configuration = { ... }
  }
}
```

2. Remove mountPoints from container definition:
```terraform
# Remove this section:
mountPoints = [
  {
    sourceVolume  = "mcpgw-data"
    containerPath = "/app/data"
    readOnly      = false
  }
]
```

3. Add environment variable to configure data path:
```terraform
{
  name  = "DATA_PATH"
  value = "/tmp/data"  # Use ephemeral storage
}
```

#### Step 4: Create Parameter Store Resource
**File:** `terraform/aws-ecs/modules/mcp-gateway/secrets.tf` (or new file)

```terraform
# Add to existing secrets management
resource "aws_ssm_parameter" "scopes_yml" {
  name        = "/${var.name}/auth-server/scopes-yml"
  description = "OAuth/OIDC scopes configuration for auth-server"
  type        = "String"
  value       = file("${path.module}/scopes.yml")
  
  tags = local.common_tags
}
```

#### Step 5: Remove EFS Variables
**File:** `terraform/aws-ecs/modules/mcp-gateway/variables.tf`
**Lines to remove:** ~259-273 (efs_throughput_mode, efs_provisioned_throughput)

```terraform
# Remove these variable blocks:
variable "efs_throughput_mode" { ... }
variable "efs_provisioned_throughput" { ... }
```

#### Step 6: Remove EFS Outputs
**File:** `terraform/aws-ecs/modules/mcp-gateway/outputs.tf`
**Lines to remove:** ~30-50 (efs_id, efs_arn, efs_access_points)

```terraform
# Remove these output blocks:
output "efs_id" { ... }
output "efs_arn" { ... }
output "efs_access_points" { ... }
```

#### Step 7: Update terraform/README.md
**File:** `terraform/README.md`

**Change:**
```markdown
# Before:
**Features:**
- Amazon EFS for persistent storage
- ...

# After:
**Features:**
- AWS Systems Manager Parameter Store for configuration
- ...
```

#### Step 8: Update terraform/aws-ecs/README.md
**File:** `terraform/aws-ecs/README.md`

**Add migration section:**
```markdown
## Migrating from EFS

### For version < 1.25.0 (with EFS)
EFS has been removed in favor of AWS-native services:

1. **Configuration**: Use Parameter Store instead of EFS file system
   ```bash
   aws ssm put-parameter --name "/mcp-gateway/auth-server/scopes-yml" \
     --type String --file /path/to/scopes.yml
   ```

2. **Logs**: Already using CloudWatch, no changes needed

3. **Demo data**: Use ephemeral storage (non-persistent) or configure S3
```

### Error Handling

1. **Parameter Store errors**: Fallback to defaults if SSM unavailable
2. **Missing scopes.yml**: Log warning, use default scopes
3. **Write permission errors**: Log to stderr, continue startup

### Logging Changes
All services already log to CloudWatch. Remove double-logging to EFS.

**Before:** Logs written to EFS and CloudWatch
**After:** Logs written to CloudWatch only

## Observability

### Metrics Tracking
- **CloudWatch Logs**: Track error rates, log volume
- **Parameter Store calls**: Monitor via CloudTrail
- **ECS Task starts**: Track success/failure rates

### Alerts
1. **Parameter Store unavailable**: Alert on SSM API errors
2. **Missing configuration**: Alert if scopes.yml not found
3. **Task startup failures**: Standard ECS alerting

### Dashboard Updates
Remove EFS CloudWatch metrics from dashboards
Add Parameter Store health metrics

## Scaling Considerations

### Current Load Assumptions
- EFS throughput: ~10 MB/s (mostly idle)
- Parameter Store: < 1 request per minute (configuration only)
- CloudWatch Logs: ~100 KB/s per service

### Performance Impact
**Positive**: Remove EFS mount latency (~10-20ms per mount)
**Positive**: Reduce cross-AZ traffic for EFS
**Neutral**: Parameter Store adds minimal latency (<5ms) on startup only

### Alternatives Considered

#### Alternative 1: Replace EFS with EBS Volumes
**Description**: Use EBS volumes mounted to ECS tasks
**Pros**: 
- Familiar block storage
- Lower latency than EFS
**Cons**:
- Still adds cost and complexity
- Not shared across tasks without additional work
- Doesn't solve the root simplicity goal

#### Alternative 2: Replace EFS with S3
**Description**: Use S3 for all file storage needs
**Pros**:
- Very cost-effective
- Highly durable
**Cons**:
- Requires application changes (S3 SDK)
- Different API than file system
- Higher latency for small files

#### Alternative 3: Use only ephemeral storage
**Description**: All data is ephemeral, no persistence
**Pros**:
- Simplest possible solution
- Zero additional cost
- Best for stateless applications
**Cons**:
- Configuration must be fetched each restart
- Demo data lost on restart

**Decision**: Chosen hybrid approach:
- Configuration in Parameter Store (Alternative 3 + 2 hybrid)
- Logs in CloudWatch (Alternative 3)
- Demo data ephemeral (Alternative 3)

### Comparison Matrix

| Criteria | Chosen | EBS Alt | S3 Alt | Ephemeral Only |
|----------|--------|---------|--------|----------------|
| Complexity | Low | Medium | Medium | Very Low |
| Cost | Low | Medium | Low | Lowest |
| Performance | Good | Good | Fair | Best |
| Data Durability | Good | Good | Excellent | None |
| Migration Effort | Medium | High | High | Low |
| AWS Best Practice | ✓ | ✗ | ✓ | ✓ |

## Testing Strategy

See separate `testing.md` for detailed test plan.

### High-Level Testing Approach
1. Unit tests: Parameter Store IAM policies
2. Integration tests: Terraform plan validation
3. E2E tests: Full deployment without EFS
4. Backward compatibility: Verify config migration

## Rollout Plan

- **Phase 1**: Implementation (this document)
- **Phase 2**: Testing (see testing.md)
- **Phase 3**: Documentation updates
- **Phase 4**: Breaking release with migration guide

## Open Questions

1. Should we support both Parameter Store and environment variables for scopes config?
2. Do we need to support importing existing EFS data for brownfield deployments?
3. Should mcpgw_data use S3 instead of ephemeral for demo deployments?

## References

- [AWS Systems Manager Parameter Store Docs](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [ECS Task Definition Storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definition_storage)
- [MCP Gateway Registry #1122](https://github.com/agentic-community/mcp-gateway-registry/issues/1122) - Registry EFS migration