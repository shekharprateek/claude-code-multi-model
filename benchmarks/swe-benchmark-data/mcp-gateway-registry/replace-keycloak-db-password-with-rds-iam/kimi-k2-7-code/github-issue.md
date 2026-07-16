# GitHub Issue: Replace Keycloak static RDS password with RDS IAM authentication

## Title
Replace Keycloak static Aurora MySQL password with RDS IAM database authentication (Terraform/ECS)

## Labels
- enhancement
- security
- terraform
- keycloak
- aws

## Description

### Problem Statement
The current Terraform/ECS deployment stores and rotates a static master password for the Keycloak Aurora MySQL database. Long-lived credentials increase the blast radius of credential leakage and require Secrets Manager rotation (lambda/rotate-rds) to mitigate password age. AWS RDS IAM database authentication removes the need for a static application password by using short-lived IAM-signed auth tokens scoped to the ECS task role.

### Proposed Solution
Add an opt-in Terraform flag `keycloak_db_use_iam` that switches the Keycloak ECS service from password-based authentication to RDS IAM authentication while keeping password auth available as a fallback.

When the flag is enabled:
1. Enable `iam_database_authentication_enabled` on `aws_rds_cluster.keycloak`.
2. Create a dedicated IAM database user in Aurora MySQL (e.g. `keycloak_iam`) via a VPC Lambda bootstrap function.
3. Reconfigure the RDS Proxy to use `auth_scheme = "AWS_IAM"` and `iam_auth = "REQUIRED"` with TLS enforced.
4. Grant the Keycloak ECS task role and the RDS Proxy role `rds-db:connect` on the IAM database user.
5. Provide a custom Keycloak container image that includes the AWS Advanced JDBC Wrapper with the IAM Authentication Plugin so the driver generates a fresh token for each new connection.
6. Update `KC_DB_URL` to reference the wrapper driver class and require SSL; stop passing `KC_DB_PASSWORD` into the Keycloak container when IAM auth is active.

When the flag is disabled, the existing password flow (Secrets Manager secret `keycloak/database`, SSM URL parameter, rotation Lambda) must continue to work unchanged.

### User Stories
- As a platform operator deploying on AWS ECS, I want to eliminate the static Keycloak database password from the runtime container so that credential leakage risk is reduced.
- As a platform operator, I want a feature flag for IAM auth so that I can roll back to password auth without rebuilding infrastructure.
- As a security reviewer, I want Keycloak database connections to use short-lived IAM tokens so that the deployment aligns with AWS least-privilege credential recommendations.

### Acceptance Criteria
- [ ] New Terraform variable `keycloak_db_use_iam` (bool, default `false`) is added to `terraform/aws-ecs/variables.tf` and wired through `main.tf`.
- [ ] When `keycloak_db_use_iam = false`, `terraform plan`/`apply` produces no changes to the existing Keycloak password flow.
- [ ] When `keycloak_db_use_iam = true`:
  - [ ] `aws_rds_cluster.keycloak` has `iam_database_authentication_enabled = true`.
  - [ ] A dedicated IAM database user exists in Aurora MySQL with the `AWSAuthenticationPlugin`.
  - [ ] `aws_db_proxy.keycloak` uses `auth_scheme = "AWS_IAM"` and `iam_auth = "REQUIRED"` and `require_tls = true`.
  - [ ] The Keycloak ECS task role has an inline policy allowing `rds-db:connect` scoped to the IAM DB user resource.
  - [ ] The RDS Proxy IAM role has an inline policy allowing `rds-db:connect` scoped to the IAM DB user resource.
  - [ ] The Keycloak task definition no longer passes `KC_DB_PASSWORD` from the Secrets Manager password secret.
  - [ ] `KC_DB_URL` uses the AWS JDBC wrapper driver class and SSL parameters required for IAM auth.
- [ ] A custom Keycloak Dockerfile (or extension of `docker/keycloak/Dockerfile`) is provided that bundles the AWS Advanced JDBC Wrapper.
- [ ] `terraform.tfvars.example` documents the new flag and image override.
- [ ] The `terraform/aws-ecs/README.md` security section is updated to describe the IAM auth option and fallback behavior.
- [ ] A test plan verifies both password fallback and IAM auth paths.

### Out of Scope
- Helm/EKS deployment changes.
- Upgrading the Keycloak version (remains on Keycloak 25.x).
- Removing the Aurora master password entirely; it is still required for cluster creation and password fallback.
- Changes to the Docker Compose/local Keycloak setup (`docker-compose.yml`, `.env.example`, `keycloak/setup/*`) unless needed for image build documentation.
- Automatic migration of existing deployments from password user to IAM user; this will be documented as a manual cutover step.

### Dependencies
- AWS provider supports `iam_database_authentication_enabled` on `aws_rds_cluster` and IAM auth settings on `aws_db_proxy` (available in current `>= 5.0` provider).
- Keycloak 25.x Quarkus datasource configuration must allow overriding the JDBC driver class and URL.

### Related Issues
- Issue #1026 (existing password rotation and SSM removal).
