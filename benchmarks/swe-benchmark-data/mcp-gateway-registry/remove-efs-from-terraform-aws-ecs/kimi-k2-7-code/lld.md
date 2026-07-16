# Low-Level Design: Remove EFS from Terraform AWS ECS Deployment

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

The AWS ECS Terraform deployment still provisions and mounts an Amazon EFS file system even though the application no longer requires shared file storage. Persistent state for models, scopes, and metadata is now stored in Amazon S3 and DocumentDB. The remaining EFS infrastructure adds cost, complexity, and a task startup dependency without providing value.

The registry service has already been decoupled from EFS (its task definition explicitly uses `mountPoints = []` and `volume = {}`), but the file system itself and the mounts for the auth-server and mcpgw services remain.

### Goals

- Eliminate the EFS file system, mount targets, access points, and NFS security group from the Terraform AWS ECS module.
- Remove EFS volume and mount configurations from all ECS task definitions.
- Clean up EFS-related Terraform variables, outputs, and documentation.
- Update post-deployment automation so it no longer branches to an EFS-based scopes initialization path.
- Ensure `terraform plan` and `terraform validate` succeed after the changes.

### Non-Goals

- Changing how scopes are loaded in the application (DocumentDB-backed scope loading already exists).
- Modifying the Helm chart or Docker Compose deployments (unless shared documentation is affected).
- Migrating data from existing EFS volumes (this is a remove-and-recreate change; Terraform will destroy the EFS resources on the next apply).
- Removing CloudWatch log groups or application logging directories.

## Codebase Analysis

### Key Files Reviewed

| File/Directory | Purpose | Relevance to This Change |
|----------------|---------|--------------------------|
| `terraform/aws-ecs/modules/mcp-gateway/storage.tf` | Defines the EFS module, access points, mount targets, and NFS security group. | Delete entirely. |
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | ECS task definitions for auth-server, registry, mcpgw, and demo servers. | Remove EFS volumes and mount points from auth-server and mcpgw; update environment variables that reference `/efs/...`. |
| `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | Input variables for the mcp-gateway module. | Remove `efs_throughput_mode` and `efs_provisioned_throughput`. |
| `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` | Module outputs. | Remove `efs_id`, `efs_arn`, and `efs_access_points`. |
| `terraform/aws-ecs/modules/mcp-gateway/iam.tf` | IAM policies for ECS task and execution roles. | No EFS-specific permissions currently declared; verify after changes. |
| `terraform/aws-ecs/modules/mcp-gateway/data.tf` | Data sources including `aws_vpc`. | `data.aws_vpc.vpc` is still used by non-EFS security group rules; keep. |
| `terraform/aws-ecs/main.tf` | Root module call for `mcp_gateway`. | No EFS variables currently passed; no changes required unless root variables are removed. |
| `terraform/aws-ecs/outputs.tf` | Root outputs. | Remove `mcp_gateway_efs_id`, `mcp_gateway_efs_arn`, and `mcp_gateway_efs_access_points`. |
| `terraform/aws-ecs/scripts/post-deployment-setup.sh` | Post-apply setup script. | Remove `mcp_gateway_efs_id` from required outputs and remove the EFS scopes init branch. |
| `terraform/aws-ecs/scripts/run-scopes-init-task.sh` | EFS scopes initialization runner. | Remove the script; the DocumentDB init script remains the only path. |
| `docker/Dockerfile.scopes-init` | Docker image for the EFS scopes init task. | Remove. |
| `registry/common/scopes_loader.py` | Application scope loader. | Confirms `SCOPES_CONFIG_PATH` is only used for non-MongoDB backends; unused when `storage_backend = documentdb`. |
| `terraform/aws-ecs/README.md` | ECS deployment documentation. | Remove EFS feature mentions and operator IAM permission. |
| `README.md` | Top-level project README. | Remove EFS from feature list. |
| `terraform/README.md` | Terraform overview documentation. | Remove EFS from feature list. |
| `docs/deployment-modes.md` | Deployment troubleshooting guide. | Rewrite the 403 troubleshooting section to reference DocumentDB scope init instead of EFS. |
| `docs/architecture-diagrams.md` | Architecture diagram text. | Remove EFS from encryption-at-rest list. |

### Existing Patterns Identified

1. **Modular Terraform layout**: The AWS ECS deployment uses a root module at `terraform/aws-ecs/` and a child module at `terraform/aws-ecs/modules/mcp-gateway/`. Changes that remove module outputs must also remove root outputs that consume them.
2. **ECS task definitions via `terraform-aws-modules/ecs/aws//modules/service`**: Volumes are declared at the service level and mount points at the container level. Removing a volume requires removing both the service-level `volume` entry and the container-level `mountPoints` entry.
3. **Environment variables as concatenated lists**: The auth-server container environment is built with `concat([...], var.auth_server_extra_env)`. Removing a hardcoded env var means removing one map element from the first list.
4. **DocumentDB is the default storage backend**: `terraform/aws-ecs/terraform.tfvars.example` sets `storage_backend = "documentdb"`, and `registry/common/scopes_loader.py` loads scopes from DocumentDB when the backend is in `MONGODB_BACKENDS`. This makes the `/efs/auth_config` mount unnecessary in the standard AWS ECS deployment.
5. **Post-deployment scripts branch on `documentdb_cluster_endpoint`**: When the DocumentDB endpoint output is present, the script runs `run-documentdb-init.sh`; otherwise it falls back to `run-scopes-init-task.sh`. After EFS removal, only the DocumentDB path is valid.

### Integration Points

| Component | Integration Type | Details |
|-----------|------------------|---------|
| EFS module (`terraform-aws-modules/efs/aws`) | Removed | Module and all its outputs are deleted. |
| Auth-server ECS service | Modified | Remove `mcp-logs` and `auth-config` volumes/mounts; remove `SCOPES_CONFIG_PATH` env var. |
| mcpgw ECS service | Modified | Remove `mcpgw-data` volume and `/app/data` mount. |
| registry ECS service | No change | Already uses `mountPoints = []` and `volume = {}`. |
| Post-deployment setup script | Modified | Remove EFS output validation and EFS scopes init branch. |
| Operator IAM documentation | Modified | Remove `elasticfilesystem:*` from example IAM policy. |

### Constraints and Limitations Discovered

- The registry service already removed EFS mounts, but the underlying EFS module was left in place. This creates a partial state that must be fully cleaned up.
- `SCOPES_CONFIG_PATH` is hardcoded to `/efs/auth_config/auth_config/scopes.yml` in the auth-server container definition. While the DocumentDB backend does not read this variable, leaving it would be confusing and could mask future misconfigurations.
- The mcpgw `/app/data` mount has no known application consumers, but it is wired into the task definition and must be removed cleanly.
- No explicit `elasticfilesystem:*` IAM permissions are declared for ECS task roles, so no IAM cleanup is required beyond verifying none are introduced by copy-paste.
- `run-scopes-init-task.sh` references a build script (`build-and-push-scopes-init.sh`) that does not exist in the repository, indicating the EFS scopes init path is already unmaintained.

## Architecture

### System Context Diagram

```text
Before:

  Users / Clients
        |
        v
  [CloudFront / ALB]
        |
        +----> [Auth Server Task]  ----> /efs/auth_config (EFS)
        |                              /app/logs (EFS)
        |
        +----> [Registry Task]   ----> DocumentDB / S3
        |      (EFS already removed)
        |
        +----> [mcpgw Task]      ----> /app/data (EFS)

After:

  Users / Clients
        |
        v
  [CloudFront / ALB]
        |
        +----> [Auth Server Task]  ----> DocumentDB / S3
        |
        +----> [Registry Task]   ----> DocumentDB / S3
        |
        +----> [mcpgw Task]      ----> DocumentDB / S3
```

### Component Diagram

```text
terraform/aws-ecs/
|
+-- main.tf                 (no EFS references after cleanup)
+-- outputs.tf              (EFS outputs removed)
+-- variables.tf            (no EFS root variables exist)
+-- modules/mcp-gateway/
    |
    +-- storage.tf          (DELETED)
    +-- ecs-services.tf     (EFS volumes/mounts removed)
    +-- variables.tf        (efs_* variables removed)
    +-- outputs.tf          (efs_* outputs removed)
    +-- data.tf             (unchanged, aws_vpc still used)
    +-- iam.tf              (unchanged, no EFS permissions)
```

## Data Models

No new data models are introduced. Existing Terraform variables and outputs are removed.

## API / CLI Design

No new API endpoints or CLI commands are introduced. The change is purely infrastructure cleanup.

## Configuration Parameters

### Removed Variables

| Variable | File | Reason |
|----------|------|--------|
| `efs_throughput_mode` | `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | No EFS file system to configure. |
| `efs_provisioned_throughput` | `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | No EFS file system to configure. |

### Removed Outputs

| Output | File | Reason |
|--------|------|--------|
| `efs_id` | `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` | No EFS file system exists. |
| `efs_arn` | `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` | No EFS file system exists. |
| `efs_access_points` | `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` | No EFS access points exist. |
| `mcp_gateway_efs_id` | `terraform/aws-ecs/outputs.tf` | No EFS file system exists. |
| `mcp_gateway_efs_arn` | `terraform/aws-ecs/outputs.tf` | No EFS file system exists. |
| `mcp_gateway_efs_access_points` | `terraform/aws-ecs/outputs.tf` | No EFS access points exist. |

### Deployment Surface Checklist

- [ ] `.env.example` - No EFS variables; no change required.
- [ ] `docker-compose.yml` / `docker-compose.*.yml` - No EFS mounts; no change required.
- [ ] `terraform/aws-ecs/terraform.tfvars.example` - No EFS variables; no change required.
- [ ] `terraform/aws-ecs/main.tf` - No EFS variables currently passed; no change required.
- [ ] Helm charts - Out of scope for this task.

## New Dependencies

This change uses only existing dependencies. The `terraform-aws-modules/efs/aws` module dependency is removed.

## Implementation Details

### Step-by-Step Plan

#### Step 1: Delete the EFS module file

**File:** `terraform/aws-ecs/modules/mcp-gateway/storage.tf`
**Action:** Delete the entire file.

This removes:
- `module "efs"` declaration.
- Six EFS access points (`servers`, `models`, `logs`, `agents`, `auth_config`, `mcpgw_data`).
- EFS mount targets and NFS security group.
- `aws_vpc_security_group_egress_rule` attached to the EFS security group.

#### Step 2: Remove EFS variables

**File:** `terraform/aws-ecs/modules/mcp-gateway/variables.tf`
**Lines:** 259-274

Remove:

```hcl
# EFS Configuration
variable "efs_throughput_mode" {
  description = "Throughput mode for EFS (bursting or provisioned)"
  type        = string
  default     = "bursting"
  validation {
    condition     = contains(["bursting", "provisioned"], var.efs_throughput_mode)
    error_message = "EFS throughput mode must be either 'bursting' or 'provisioned'."
  }
}

variable "efs_provisioned_throughput" {
  description = "Provisioned throughput in MiB/s for EFS (only used if throughput_mode is provisioned)"
  type        = number
  default     = 100
}
```

#### Step 3: Remove EFS module outputs

**File:** `terraform/aws-ecs/modules/mcp-gateway/outputs.tf`
**Lines:** 47-69

Remove:

```hcl
# EFS outputs
output "efs_id" {
  description = "MCP Gateway Registry EFS file system ID"
  value       = module.efs.id
  sensitive   = false
}

output "efs_arn" {
  description = "MCP Gateway Registry EFS file system ARN"
  value       = module.efs.arn
  sensitive   = false
}

output "efs_access_points" {
  description = "EFS access point IDs"
  value = {
    servers     = module.efs.access_points["servers"].id
    models      = module.efs.access_points["models"].id
    logs        = module.efs.access_points["logs"].id
    auth_config = module.efs.access_points["auth_config"].id
  }
  sensitive = false
}
```

#### Step 4: Remove root EFS outputs

**File:** `terraform/aws-ecs/outputs.tf`
**Lines:** 67-81

Remove:

```hcl
# EFS Outputs
output "mcp_gateway_efs_id" {
  description = "MCP Gateway EFS file system ID"
  value       = module.mcp_gateway.efs_id
}

output "mcp_gateway_efs_arn" {
  description = "MCP Gateway EFS file system ARN"
  value       = module.mcp_gateway.efs_arn
}

output "mcp_gateway_efs_access_points" {
  description = "MCP Gateway EFS access point IDs"
  value       = module.mcp_gateway.efs_access_points
}
```

#### Step 5: Remove EFS mounts from auth-server task definition

**File:** `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`

**5a. Remove `SCOPES_CONFIG_PATH` environment variable**
**Lines:** 219-222

Delete:

```hcl
        {
          name  = "SCOPES_CONFIG_PATH"
          value = "/efs/auth_config/auth_config/scopes.yml"
        },
```

**5b. Remove EFS mount points**
**Lines:** 482-493

Change from:

```hcl
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

To:

```hcl
      mountPoints = []
```

**5c. Remove EFS volumes**
**Lines:** 542-557

Change from:

```hcl
  volume = {
    mcp-logs = {
      efs_volume_configuration = {
        file_system_id     = module.efs.id
        access_point_id    = module.efs.access_points["logs"].id
        transit_encryption = "ENABLED"
      }
    }
    auth-config = {
      efs_volume_configuration = {
        file_system_id     = module.efs.id
        access_point_id    = module.efs.access_points["auth_config"].id
        transit_encryption = "ENABLED"
      }
    }
  }
```

To:

```hcl
  volume = {}
```

**Rationale:** The auth-server loads scopes from DocumentDB when `storage_backend = "documentdb"` (the default AWS ECS configuration). The `/app/logs` mount is also unnecessary because application logs are sent to CloudWatch via the ECS service module's `enable_cloudwatch_logging` setting.

#### Step 6: Remove EFS mounts from mcpgw task definition

**File:** `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`

**6a. Remove mount points**
**Lines:** 1803-1809

Change from:

```hcl
      mountPoints = [
        {
          sourceVolume  = "mcpgw-data"
          containerPath = "/app/data"
          readOnly      = false
        }
      ]
```

To:

```hcl
      mountPoints = []
```

**6b. Remove EFS volume**
**Lines:** 1859-1867

Change from:

```hcl
  volume = {
    mcpgw-data = {
      efs_volume_configuration = {
        file_system_id     = module.efs.id
        access_point_id    = module.efs.access_points["mcpgw_data"].id
        transit_encryption = "ENABLED"
      }
    }
  }
```

To:

```hcl
  volume = {}
```

**Rationale:** No application code in `servers/mcpgw/`, `api/`, or `docker/` references `/app/data` in the ECS deployment context.

#### Step 7: Update post-deployment setup script

**File:** `terraform/aws-ecs/scripts/post-deployment-setup.sh`

**7a. Remove EFS from required outputs**
**Lines:** 218

Remove `"mcp_gateway_efs_id"` from the `required_outputs` array.

**7b. Remove EFS scopes initialization branch**
**Lines:** 548-568

Replace the entire `else` block (the EFS mode branch) with an error or skip path. Suggested replacement:

```bash
    else
        log_error "No DocumentDB endpoint found in terraform outputs."
        log_error "AWS ECS deployments require DocumentDB as the storage backend."
        STEPS_FAILED=$((STEPS_FAILED + 1))
        return 1
    fi
```

This preserves the DocumentDB init path while making it explicit that EFS is no longer supported.

#### Step 8: Remove EFS scopes init script and Dockerfile

**Files:**
- `terraform/aws-ecs/scripts/run-scopes-init-task.sh`
- `docker/Dockerfile.scopes-init`

**Action:** Delete both files.

#### Step 9: Update documentation

**9a. `terraform/aws-ecs/README.md`**
- Line 16: Remove `- Amazon EFS for persistent storage` from the feature list.
- Line 1056: Remove `"elasticfilesystem:*",` from the operator IAM permissions JSON.

**9b. `README.md`**
- Line 817: Remove `- **EFS Shared Storage** - Persistent storage for models, logs, and configuration` from the feature list.

**9c. `terraform/README.md`**
- Line 16: Remove `- Amazon EFS for persistent storage` from the feature list.

**9d. `docs/deployment-modes.md`**
- Lines 211-219: Rewrite the 403 troubleshooting section to reference DocumentDB scope initialization instead of EFS. Replace:

```markdown
**Cause:** Either the user doesn't have required group memberships, or the MCP scopes haven't been initialized on EFS.

**Solution:**
1. Check user groups in Keycloak Admin → mcp-gateway realm → Users → select user → Groups
2. Ensure user is in `mcp-registry-admin` or `mcp-registry-user` group
3. Run the scopes init task:
   ```bash
   ./scripts/run-scopes-init-task.sh --skip-build
   ```
```

With:

```markdown
**Cause:** Either the user doesn't have required group memberships, or the MCP scopes haven't been initialized in DocumentDB.

**Solution:**
1. Check user groups in Keycloak Admin → mcp-gateway realm → Users → select user → Groups
2. Ensure user is in `mcp-registry-admin` or `mcp-registry-user` group
3. Run the DocumentDB scopes init task:
   ```bash
   ./scripts/run-documentdb-init.sh
   ```
```

**9e. `docs/architecture-diagrams.md`**
- Line 519: Change `Encryption at rest: KMS (DocumentDB, EBS, EFS, S3)` to `Encryption at rest: KMS (DocumentDB, EBS, S3)`.

### Error Handling

- If a developer accidentally references `module.efs` after `storage.tf` is deleted, `terraform validate` will fail with a clear error pointing to the missing resource address.
- If `run-scopes-init-task.sh` is still referenced by external runbooks after deletion, the failure will be obvious because the file no longer exists.

### Logging

No application logging changes are required. Terraform plan/apply output will show the EFS resources being destroyed; this is expected.

## Observability

No new metrics, traces, or alerts are required. The existing CloudWatch logging and monitoring for ECS services remains unchanged.

## Scaling Considerations

- Removing EFS reduces task startup latency slightly because tasks no longer wait for EFS volume attachment.
- No horizontal scaling implications; the services were already stateless with respect to EFS.
- Cost reduction scales with the number of deployments because EFS file systems and throughput are provisioned per deployment.

## File Changes

### Deleted Files

| File Path | Description |
|-----------|-------------|
| `terraform/aws-ecs/modules/mcp-gateway/storage.tf` | EFS module, access points, mount targets, security group |
| `terraform/aws-ecs/scripts/run-scopes-init-task.sh` | EFS scopes initialization runner |
| `docker/Dockerfile.scopes-init` | Docker image for EFS scopes init task |

### Modified Files

| File Path | Lines | Change Description |
|-----------|-------|--------------------|
| `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | ~259-274 | Remove `efs_throughput_mode` and `efs_provisioned_throughput` |
| `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` | ~47-69 | Remove `efs_id`, `efs_arn`, `efs_access_points` |
| `terraform/aws-ecs/outputs.tf` | ~67-81 | Remove `mcp_gateway_efs_id`, `mcp_gateway_efs_arn`, `mcp_gateway_efs_access_points` |
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | ~219-222, ~482-493, ~542-557 | Remove auth-server `SCOPES_CONFIG_PATH`, mount points, and EFS volumes |
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | ~1803-1809, ~1859-1867 | Remove mcpgw mount point and EFS volume |
| `terraform/aws-ecs/scripts/post-deployment-setup.sh` | ~218 | Remove `mcp_gateway_efs_id` from required outputs |
| `terraform/aws-ecs/scripts/post-deployment-setup.sh` | ~548-568 | Replace EFS scopes init branch with an error |
| `terraform/aws-ecs/README.md` | ~16, ~1056 | Remove EFS from features and IAM permissions |
| `README.md` | ~817 | Remove EFS from feature list |
| `terraform/README.md` | ~16 | Remove EFS from feature list |
| `docs/deployment-modes.md` | ~211-219 | Rewrite 403 troubleshooting to reference DocumentDB |
| `docs/architecture-diagrams.md` | ~519 | Remove EFS from encryption-at-rest list |

### Estimated Lines of Code

| Category | Lines |
|----------|-------|
| Deleted code | ~210 (storage.tf) + ~220 (run-scopes-init-task.sh) + ~30 (Dockerfile.scopes-init) |
| Removed Terraform config | ~80 |
| Modified scripts/docs | ~30 |
| **Total reduction** | **~570 lines** |

## Testing Strategy

See `testing.md` for the full executable test plan.

## Alternatives Considered

### Alternative 1: Keep EFS module but make it conditional

**Description:** Add a feature flag (e.g., `enable_efs`) that defaults to `false` and conditionally creates the EFS module and mounts.

**Pros / Cons:**
- Pros: Backward compatible for users who still want EFS.
- Cons: Adds complexity and a new variable. The application no longer needs EFS, so a flag only preserves dead code.

**Why Rejected:** The problem statement explicitly says EFS is no longer needed. Conditional dead code increases maintenance burden without benefit.

### Alternative 2: Keep the EFS file system but remove mounts

**Description:** Remove EFS mounts from task definitions but leave the file system provisioned for potential future use.

**Pros / Cons:**
- Pros: Simpler change.
- Cons: Continues to incur EFS cost and complexity while providing zero value.

**Why Rejected:** Does not achieve the cost and complexity reduction goals.

### Alternative 3: Replace EFS with EBS volumes per task

**Description:** Replace EFS mounts with task-local EBS volumes for any services that still need local state.

**Pros / Cons:**
- Pros: Could provide fast local storage if needed.
- Cons: No application code requires local file storage. Adds unnecessary infrastructure.

**Why Rejected:** No known consumer of local file storage exists in the ECS deployment.

### Comparison Matrix

| Criteria | Chosen (Remove EFS) | Conditional EFS | Keep EFS Unmounted | Replace with EBS |
|----------|---------------------|-----------------|--------------------|------------------|
| Cost reduction | Full | Partial | None | None / adds cost |
| Complexity | Lowest | Medium | Medium | High |
| Maintenance burden | Lowest | Medium | Medium | High |
| Backward compatibility | Breaking (expected) | Preserved | Partial | Breaking |
| Alignment with goals | Best | Poor | Poor | Poor |

## Rollout Plan

- **Phase 1 (out of scope for this skill):** Implement the file changes listed above.
- **Phase 2:** Run `terraform fmt`, `terraform validate`, and `terraform plan` in a non-production workspace. Verify that EFS resources show as "destroy" and no references remain.
- **Phase 3:** Apply the changes in a development environment. Confirm ECS tasks start without EFS mounts and that auth/mcpgw/registry services pass health checks.
- **Phase 4:** Run the DocumentDB scopes init script (`run-documentdb-init.sh`) and verify 403 errors are resolved when users are in the correct Keycloak groups.
- **Phase 5:** Promote to staging and production with standard change management.

## Open Questions

1. Should `terraform/aws-ecs/scripts/post-deployment-setup.sh` keep a `--legacy-efs` escape hatch for existing non-DocumentDB deployments, or should it explicitly fail when DocumentDB is not configured?
   - Recommendation: Explicitly fail. The default AWS ECS deployment uses DocumentDB, and the EFS path is unmaintained.
2. Is there any external runbook or CI pipeline outside this repository that depends on `mcp_gateway_efs_id` or `mcp_gateway_efs_access_points` outputs?
   - Action: Search downstream consumers before applying.
3. Should the `mcp-logs` EFS mount be removed from auth-server, or should it be retained because it previously stored logs?
   - Recommendation: Remove it. CloudWatch logging is already enabled, and the registry service removed its analogous logs mount.

## References

- `terraform/aws-ecs/modules/mcp-gateway/storage.tf`
- `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`
- `terraform/aws-ecs/modules/mcp-gateway/variables.tf`
- `terraform/aws-ecs/modules/mcp-gateway/outputs.tf`
- `terraform/aws-ecs/outputs.tf`
- `terraform/aws-ecs/scripts/post-deployment-setup.sh`
- `terraform/aws-ecs/scripts/run-scopes-init-task.sh`
- `docker/Dockerfile.scopes-init`
- `registry/common/scopes_loader.py`
- `docs/deployment-modes.md`
