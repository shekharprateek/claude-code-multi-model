# GitHub Issue: Remove EFS from Terraform AWS ECS deployment

## Title
Remove Amazon EFS from the AWS ECS Terraform modules

## Labels
- refactor
- infra
- terraform
- aws
- cost

## Description

### Problem Statement
The MCP Gateway Registry AWS ECS deployment still provisions an Amazon EFS file system and mounts it into the ECS task definitions, even though the application no longer relies on shared file storage. All persistent state now lives in Amazon S3 and DocumentDB. Keeping EFS adds:

- **Unnecessary cost** - EFS throughput, capacity, and mount-target charges.
- **Operational complexity** - An extra managed service with security groups, access points, and mount targets.
- **Task startup dependency** - Containers wait for EFS volume attachment before they can start.
- **Configuration drift** - EFS variables, outputs, and mounts are still referenced in Terraform despite the registry service already having its EFS volumes removed in the current release.

### Proposed Solution
Complete the EFS removal that was started for the registry service by:

1. Removing the `module "efs"` declaration and the dedicated EFS security-group egress rule from `terraform/aws-ecs/modules/mcp-gateway/storage.tf`.
2. Removing the remaining EFS volume mounts from the auth-server and mcpgw ECS service definitions in `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf`.
3. Updating environment variables and container paths that currently point to `/efs/...` so they use local/ephemeral paths or rely on DocumentDB-backed configuration loading.
4. Removing EFS-related variables from `terraform/aws-ecs/modules/mcp-gateway/variables.tf`.
5. Removing EFS-related outputs from `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` and `terraform/aws-ecs/outputs.tf`.
6. Updating Terraform and project documentation that mention EFS.

### User Stories
- As an operator deploying via Terraform, I want the infrastructure to provision only the storage services the application actually uses, so that my monthly AWS bill is lower and the blast radius is smaller.
- As an SRE, I want ECS tasks to start without an EFS mount dependency, so that task startup is faster and less brittle.
- As a platform engineer, I want obsolete Terraform variables and outputs removed, so that the configuration surface is easier to understand and maintain.

### Acceptance Criteria
- [ ] `terraform/aws-ecs/modules/mcp-gateway/storage.tf` no longer contains EFS resources.
- [ ] `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` no longer declares EFS volumes or mount points for any service.
- [ ] Auth-server and mcpgw container definitions no longer reference `/efs/...` paths.
- [ ] `terraform/aws-ecs/modules/mcp-gateway/variables.tf` no longer contains `efs_throughput_mode` or `efs_provisioned_throughput`.
- [ ] `terraform/aws-ecs/modules/mcp-gateway/outputs.tf` and `terraform/aws-ecs/outputs.tf` no longer export EFS IDs, ARNs, or access points.
- [ ] `terraform/aws-ecs/main.tf` no longer passes EFS-related variables to the module.
- [ ] No `module.efs` references remain anywhere under `terraform/aws-ecs`.
- [ ] Documentation in `terraform/README.md`, `README.md`, and `docs/deployment-modes.md` no longer describes EFS as a required or active component.
- [ ] `terraform plan` succeeds against a fresh workspace after the changes.
- [ ] Existing DocumentDB/S3-backed functionality continues to work without the EFS mounts.

### Out of Scope
- Replacing the application's scope-loading mechanism with a new persistence layer (the DocumentDB-backed path already exists).
- Changes to the Helm chart or Docker Compose deployment surfaces unless they share Terraform-managed docs.
- Migrating data out of existing EFS file systems (this change targets green-field/destroy-and-recreate deployments; existing EFS resources will be destroyed by Terraform on the next apply).

### Dependencies
- Confirm that the auth-server `SCOPES_CONFIG_PATH` fallback is not required when `storage_backend = documentdb` (the default for AWS ECS).
- Confirm that the mcpgw `/app/data` mount is unused.

### Related Issues
- Partial EFS removal for the registry service (existing comments in `ecs-services.tf`).
