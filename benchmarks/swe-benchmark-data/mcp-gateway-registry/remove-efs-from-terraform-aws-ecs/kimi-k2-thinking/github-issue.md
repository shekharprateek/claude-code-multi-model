# GitHub Issue: Remove EFS from Terraform AWS ECS Configuration

## Title
Remove Amazon EFS dependency from terraform-aws-ecs module

## Labels
- enhancement
- infrastructure
- breaking-change
- terraform

## Description

### Problem Statement
The terraform-aws-ecs module currently creates and uses Amazon EFS (Elastic File System) for persistent storage across multiple services. However, EFS adds unnecessary complexity and cost for the MCP Gateway Registry deployment pattern. The registry service has already been successfully migrated off EFS to use ephemeral storage combined with DocumentDB. We need to complete this migration by removing EFS usage from the remaining services (auth-server and mcpgw) and eliminate the EFS infrastructure entirely.

### Current State
The module `terraform-aws-ecs/modules/mcp-gateway/storage.tf` creates an EFS file system with:
- 6 access points (servers, models, logs, agents, auth_config, mcpgw_data)
- Security group with NFS rules
- Encrypted storage with transit encryption

EFS is currently mounted by:
1. **auth-server**: mounts logs and auth_config volumes
2. **mcpgw**: mounts mcpgw_data volume
3. **registry**: Already migrated OFF EFS (uses ephemeral storage + DocumentDB)

### Proposed Solution
Remove EFS entirely from the terraform-aws-ecs module by:
1. Removing the EFS module from `storage.tf`
2. Modifying auth-server to use alternatives for logs and auth config
3. Modifying mcpgw to use alternatives for data storage
4. Removing EFS-related outputs
5. Removing EFS-related variables
6. Updating documentation

### Alternative Solutions for EFS Data

**For auth-server:**
- **Logs**: Already using CloudWatch Logs (double logging to both EFS and CloudWatch), so can just remove EFS logging
- **Auth config (scopes.yml)**: Migrate to AWS Systems Manager Parameter Store for the scopes configuration

**For mcpgw:**
- **mcpgw_data**: Evaluate usage patterns to determine appropriate storage (likely S3 for file-based data or database for structured data)

### Acceptance Criteria
- [ ] Remove EFS module from `terraform/aws-ecs/modules/mcp-gateway/storage.tf`
- [ ] Update auth-server ECS task definition to remove EFS volume mounts
- [ ] Migrate auth-server scopes.yml configuration to Parameter Store
- [ ] Update mcpgw ECS task definition to remove EFS volume mount
- [ ] Remove or replace mcpgw_data storage (use S3 or ephemeral as appropriate)
- [ ] Remove EFS outputs from `outputs.tf`
- [ ] Remove EFS-related variables from `variables.tf`
- [ ] Remove EFS security group resources
- [ ] Update `terraform/README.md` to remove EFS references
- [ ] Update `terraform/aws-ecs/README.md` with migration guide
- [ ] Verify terraform plan shows no EFS resources being created
- [ ] Test deployment without EFS

### Out of Scope
- Changes to terraform/telemetry-collector (separate module)
- Changes to Helm charts or EKS deployments
- Changes to application code logging configuration (handled at app level)
- Migration path for existing EFS data (greenfield deployments only in scope)

### User Stories
- As a DevOps engineer, I want to deploy MCP Gateway Registry without EFS to reduce infrastructure costs and complexity
- As a platform administrator, I want fewer AWS dependencies to simplify our architecture and reduce operational overhead
- As a developer, I want clearer storage patterns that use standard AWS services (S3, Parameter Store, CloudWatch) rather than file-based storage

### Related Issues
- Issue #1122: Previous work migrating registry service off EFS to ephemeral storage
- Issue #1050: Cost reduction initiative (EFS removal contributes to this goal)

### Completion Notes
This is a breaking change for terraform-aws-ecs deployments. Users upgrading will need to:
1. Migrate any data from EFS volumes before applying
2. Update their configuration to use new storage patterns
3. Apply the Terraform changes which will destroy EFS resources

Migration documentation should be provided in the README.md file.