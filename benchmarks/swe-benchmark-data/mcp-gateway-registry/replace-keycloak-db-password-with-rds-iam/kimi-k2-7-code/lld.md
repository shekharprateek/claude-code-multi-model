# Low-Level Design: Replace Keycloak static RDS password with RDS IAM authentication

*Created: 2026-07-15*  
*Author: Claude*  
*Status: Draft*

## Table of Contents

1. [Overview](#overview)
2. [Codebase Analysis](#codebase-analysis)
3. [Architecture](#architecture)
4. [Configuration Parameters](#configuration-parameters)
5. [Terraform Resource Changes](#terraform-resource-changes)
6. [Container/Image Changes](#containerimage-changes)
7. [IAM Changes](#iam-changes)
8. [Database User Bootstrap](#database-user-bootstrap)
9. [Implementation Steps](#implementation-steps)
10. [Observability](#observability)
11. [Scaling Considerations](#scaling-considerations)
12. [File Changes](#file-changes)
13. [Testing Strategy](#testing-strategy)
14. [Alternatives Considered](#alternatives-considered)
15. [Rollout Plan](#rollout-plan)
16. [Open Questions](#open-questions)

## Overview

### Problem Statement
The Terraform/ECS deployment currently authenticates Keycloak to its Aurora MySQL backend using a static master password stored in AWS Secrets Manager and rotated by a Lambda. This creates long-lived credentials in the runtime environment and a Secrets Manager dependency solely for the database password. The goal is to allow operators to switch Keycloak to RDS IAM database authentication, which uses short-lived IAM-signed tokens and removes the static password from the container runtime, while preserving password auth as a fallback.

### Goals
- Add an opt-in feature flag `keycloak_db_use_iam` that enables RDS IAM auth for Keycloak.
- Keep the existing password auth path fully functional when the flag is disabled.
- Avoid a Keycloak version upgrade.
- Focus changes on `terraform/aws-ecs`; do not modify Helm/EKS deployment surfaces.

### Non-Goals
- Removing the Aurora master password entirely (still required for cluster creation and fallback).
- Modifying the local/Docker Compose Keycloak setup.
- Automating zero-downtime migration of existing password users to IAM users.
- Supporting IAM auth for DocumentDB (already implemented separately).

## Codebase Analysis

### Key Files Reviewed

| File | Purpose | Relevance |
|------|---------|-----------|
| `terraform/aws-ecs/keycloak-database.tf` | Aurora cluster, RDS Proxy, KMS, Secrets Manager secret, SSM URL parameter | Core RDS/IAM auth target |
| `terraform/aws-ecs/keycloak-ecs.tf` | Keycloak ECS cluster, task definition, task/exec roles, container secrets | Where `KC_DB_*` env/secrets are wired |
| `terraform/aws-ecs/variables.tf` | Root module input variables | New feature flag and IAM username variable |
| `terraform/aws-ecs/locals.tf` | Computed locals | No DB-specific locals currently |
| `terraform/aws-ecs/main.tf` | Root module orchestration and `mcp_gateway` module call | Pass new flag to Keycloak resources, not to `mcp_gateway` module |
| `terraform/aws-ecs/secret-rotation.tf` | IAM role and Lambda function for secret rotation | Existing password rotation; no IAM token support |
| `terraform/aws-ecs/secret-rotation-config.tf` | Secrets Manager rotation schedules | Keeps `keycloak/database` rotation when password fallback is needed |
| `terraform/aws-ecs/lambda/rotate-rds/index.py` | RDS password rotation Lambda | Rotates native master password; not used for IAM auth |
| `terraform/aws-ecs/modules/mcp-gateway/variables.tf` | Module inputs | Confirmed module has no Keycloak DB inputs |
| `terraform/aws-ecs/modules/mcp-gateway/ecs-services.tf` | Registry/auth-server container env | Contains the `documentdb_use_iam` feature-flag pattern to follow |
| `docker/keycloak/Dockerfile` | Keycloak image build | Must be extended to include AWS JDBC wrapper |
| `terraform/aws-ecs/terraform.tfvars.example` | Operator-facing variable template | Document new flag |
| `terraform/aws-ecs/README.md` | Deployment docs | Add IAM auth section |

### Existing Patterns Identified

1. **Feature-flag convention**: Boolean flags such as `documentdb_use_iam`, `entra_enabled`, `okta_enabled`, and `registry_static_token_auth_enabled` are declared in `variables.tf`, passed through `main.tf`, rendered as `tostring(var.flag)` container environment variables, and used to conditionally render secrets and IAM statements.
2. **Secrets Manager secret with key extraction**: The ECS task reads `KC_DB_USERNAME` and `KC_DB_PASSWORD` from a Secrets Manager secret using the `valueFrom = "<arn>:<key>::"` syntax (`keycloak-ecs.tf:97-104`).
3. **Rotation Lambda**: `lambda/rotate-rds/index.py` rotates the native master password via `rds.modify_db_cluster` and Secrets Manager version stages.
4. **Module boundary**: Keycloak-specific resources live outside `modules/mcp-gateway`. The module only receives Keycloak admin credentials and domain; it does not manage the database.

### Constraints and Limitations Discovered

- The official `quay.io/keycloak/keycloak:25.0` image does not contain the AWS CLI or boto3, so it cannot generate RDS IAM tokens natively.
- RDS IAM tokens expire after 15 minutes. Keycloak is a long-running process with a connection pool, so a one-time token written into an environment variable at startup will become stale on reconnect.
- AWS RDS IAM auth requires SSL/TLS. The current RDS Proxy is configured with `require_tls = false` and the JDBC URL has no SSL parameters.
- Aurora MySQL does not allow the same database user to use both native password auth and IAM auth. A fallback path needs two distinct database users.
- Terraform cannot directly create Aurora MySQL users; a bootstrap Lambda or documented manual step is required.

## Architecture

### System Context Diagram

```text
┌─────────────────────────────────────────────────────────────────────┐
│                           AWS Cloud                                  │
│                                                                      │
│  ┌──────────────┐      IAM auth token      ┌─────────────────────┐  │
│  │   Keycloak   │◄────────────────────────►│    RDS Proxy        │  │
│  │   ECS task   │  JDBC wrapper driver     │  (auth_scheme=AWS_IAM)│  │
│  └──────┬───────┘                          └──────────┬──────────┘  │
│         │                                              │             │
│         │ rds-db:connect                               │ rds-db:connect│
│         ▼                                              ▼             │
│  ┌─────────────────────┐                    ┌─────────────────────┐  │
│  │   IAM Role          │                    │   IAM Role          │  │
│  │   keycloak-task-role│                    │   rds-proxy-role    │  │
│  └─────────────────────┘                    └─────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │           Aurora MySQL cluster (iam_database_authentication_enabled=true)│ │
│  │  Native user: keycloak (password fallback)                     │ │
│  │  IAM user:    keycloak_iam                                     │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram: IAM-auth connection establishment

```text
Keycloak container        AWS JDBC wrapper         ECS task metadata        RDS Proxy           Aurora
       │                         │                         │                  │                  │
       │ open connection         │                         │                  │                  │
       │────────────────────────>│                         │                  │                  │
       │                         │ read task role creds    │                  │                  │
       │                         │────────────────────────>│                  │                  │
       │                         │ creds                   │                  │                  │
       │                         │<────────────────────────│                  │                  │
       │                         │ generate RDS auth token │                  │                  │
       │                         │───────────────────────────────────────────>│ connect with token│
       │                         │                         │                  │                  │
       │                         │                         │                  │ validate token   │
       │                         │                         │                  │─────────────────>│
       │                         │                         │                  │                  │
```

## Configuration Parameters

### New Terraform Variables

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `keycloak_db_use_iam` | bool | `false` | No | When `true`, Keycloak uses RDS IAM auth; when `false`, password auth is used. |
| `keycloak_rds_iam_username` | string | `"keycloak_iam"` | No | Aurora MySQL database user created with `AWSAuthenticationPlugin`. |
| `keycloak_iam_auth_image_uri` | string | `""` | Conditional | Custom Keycloak image URI when `keycloak_db_use_iam = true`. Required when flag is true. |

### Existing Variables Retained

| Variable | Notes |
|----------|-------|
| `keycloak_database_username` | Native password fallback user. Keep as master/cluster user. |
| `keycloak_database_password` | Native password fallback. Still stored in Secrets Manager. |
| `keycloak_image_uri` | Default public image used when `keycloak_db_use_iam = false`. |

### Container Environment Variables

When `keycloak_db_use_iam = true`:

| Name | Source | Notes |
|------|--------|-------|
| `KC_DB_URL` | SSM parameter `/keycloak/database/url` | JDBC URL uses `jdbc:aws-wrapper:mysql://...` and SSL params. |
| `KC_DB_USERNAME` | SSM parameter `/keycloak/database/iam_username` | New SSM parameter with IAM DB username. |
| `KC_DB_PASSWORD` | **Not passed** | Generated per connection by the JDBC wrapper. |
| `AWS_REGION` | Plain env var | Already present; required by wrapper for token signing. |

When `keycloak_db_use_iam = false`, keep the current secrets sourced from `aws_secretsmanager_secret.keycloak_db_secret`.

### Deployment Surface Checklist

- [ ] `terraform/aws-ecs/variables.tf`
- [ ] `terraform/aws-ecs/main.tf` (pass flag to Keycloak resources, not to `mcp_gateway` module)
- [ ] `terraform/aws-ecs/keycloak-database.tf`
- [ ] `terraform/aws-ecs/keycloak-ecs.tf`
- [ ] `terraform/aws-ecs/keycloak-iam.tf` (new file for bootstrap Lambda and IAM policies)
- [ ] `terraform/aws-ecs/lambda/keycloak-rds-iam-init/` (new Lambda)
- [ ] `docker/keycloak/Dockerfile` or new `Dockerfile.rds-iam`
- [ ] `terraform/aws-ecs/terraform.tfvars.example`
- [ ] `terraform/aws-ecs/README.md`

## Terraform Resource Changes

### `terraform/aws-ecs/keycloak-database.tf`

1. Conditional IAM auth on the Aurora cluster.

```hcl
resource "aws_rds_cluster" "keycloak" {
  cluster_identifier = "keycloak"
  engine             = "aurora-mysql"
  engine_version     = "8.0.mysql_aurora.3.10.3"
  database_name      = "keycloak"
  master_username    = var.keycloak_database_username
  master_password    = var.keycloak_database_password

  iam_database_authentication_enabled = var.keycloak_db_use_iam

  # ... remaining existing fields unchanged
}
```

2. Conditional RDS Proxy auth block.

```hcl
resource "aws_db_proxy" "keycloak" {
  name          = "keycloak-proxy"
  engine_family = "MYSQL"

  auth {
    auth_scheme               = var.keycloak_db_use_iam ? "AWS_IAM" : "SECRETS"
    secret_arn                = var.keycloak_db_use_iam ? null : aws_secretsmanager_secret.keycloak_db_secret.arn
    client_password_auth_type = var.keycloak_db_use_iam ? null : "MYSQL_CACHING_SHA2_PASSWORD"
    iam_auth                  = var.keycloak_db_use_iam ? "REQUIRED" : "DISABLED"
  }

  role_arn               = aws_iam_role.rds_proxy_role.arn
  vpc_subnet_ids         = module.vpc.private_subnets
  vpc_security_group_ids = [aws_security_group.keycloak_db.id]
  require_tls            = var.keycloak_db_use_iam ? true : false

  # ... tags and depends_on unchanged
}
```

3. Update the SSM URL parameter to include SSL parameters when IAM auth is enabled.

```hcl
resource "aws_ssm_parameter" "keycloak_database_url" {
  name   = "/keycloak/database/url"
  type   = "SecureString"
  key_id = aws_kms_key.rds.id
  value  = var.keycloak_db_use_iam ? (
    "jdbc:aws-wrapper:mysql://${aws_db_proxy.keycloak.endpoint}:3306/keycloak?sslMode=REQUIRE&useSSL=true&wrapperPlugins=iam"
  ) : (
    "jdbc:mysql://${aws_db_proxy.keycloak.endpoint}:3306/keycloak"
  )
  tags = local.common_tags
}
```

4. Add an SSM parameter for the IAM database username.

```hcl
resource "aws_ssm_parameter" "keycloak_database_iam_username" {
  count  = var.keycloak_db_use_iam ? 1 : 0
  name   = "/keycloak/database/iam_username"
  type   = "SecureString"
  key_id = aws_kms_key.rds.id
  value  = var.keycloak_rds_iam_username
  tags   = local.common_tags
}
```

### `terraform/aws-ecs/keycloak-ecs.tf`

1. Conditional container secrets.

```hcl
locals {
  keycloak_container_secrets = concat(
    [
      {
        name      = "KEYCLOAK_ADMIN"
        valueFrom = aws_ssm_parameter.keycloak_admin.arn
      },
      {
        name      = "KEYCLOAK_ADMIN_PASSWORD"
        valueFrom = aws_ssm_parameter.keycloak_admin_password.arn
      },
      {
        name      = "KC_DB_URL"
        valueFrom = aws_ssm_parameter.keycloak_database_url.arn
      },
    ],
    var.keycloak_db_use_iam ? [
      {
        name      = "KC_DB_USERNAME"
        valueFrom = aws_ssm_parameter.keycloak_database_iam_username[0].arn
      }
    ] : [
      {
        name      = "KC_DB_USERNAME"
        valueFrom = "${aws_secretsmanager_secret.keycloak_db_secret.arn}:username::"
      },
      {
        name      = "KC_DB_PASSWORD"
        valueFrom = "${aws_secretsmanager_secret.keycloak_db_secret.arn}:password::"
      }
    ]
  )
}
```

2. Conditional task definition image.

```hcl
container_definitions = jsonencode([
  {
    name  = "keycloak"
    image = var.keycloak_db_use_iam ? var.keycloak_iam_auth_image_uri : var.keycloak_image_uri
    # ... rest unchanged
  }
])
```

3. Validation precondition on the task definition to enforce a custom image when IAM auth is enabled.

```hcl
resource "aws_ecs_task_definition" "keycloak" {
  # ... existing fields

  lifecycle {
    precondition {
      condition     = !var.keycloak_db_use_iam || var.keycloak_iam_auth_image_uri != ""
      error_message = "keycloak_iam_auth_image_uri must be set when keycloak_db_use_iam is true."
    }
  }
}
```

4. Update task execution role policy to allow reading the new IAM username SSM parameter when IAM auth is enabled.

```hcl
resource "aws_iam_role_policy" "keycloak_task_exec_ssm_policy" {
  # ... existing statements

  statement {
    effect = "Allow"
    actions = ["ssm:GetParameter"]
    resources = concat(
      [
        aws_ssm_parameter.keycloak_admin.arn,
        aws_ssm_parameter.keycloak_admin_password.arn,
        aws_ssm_parameter.keycloak_database_url.arn,
      ],
      var.keycloak_db_use_iam ? [aws_ssm_parameter.keycloak_database_iam_username[0].arn] : []
    )
  }
}
```

## Container/Image Changes

### `docker/keycloak/Dockerfile`

Extend the existing Dockerfile to optionally include the AWS Advanced JDBC Wrapper. A single Dockerfile can stay backward-compatible by adding the JAR; the wrapper is only activated by the JDBC URL.

```dockerfile
FROM quay.io/keycloak/keycloak:25.0 as builder

ENV KC_HEALTH_ENABLED=true
ENV KC_METRICS_ENABLED=true
ENV KC_FEATURES=token-exchange
ENV KC_DB=mysql

WORKDIR /opt/keycloak

# Add AWS Advanced JDBC Wrapper for RDS IAM authentication.
# Pin the version via build argument.
ARG AWS_JDBC_WRAPPER_VERSION=2.3.7
ADD --chmod=644 \
  https://repo1.maven.org/maven2/software/amazon/jdbc/aws-advanced-jdbc-wrapper/${AWS_JDBC_WRAPPER_VERSION}/aws-advanced-jdbc-wrapper-${AWS_JDBC_WRAPPER_VERSION}.jar \
  /opt/keycloak/providers/aws-advanced-jdbc-wrapper.jar

RUN keytool -genkeypair -storepass password -storetype PKCS12 -keyalg RSA -keysize 2048 -dname "CN=server" -alias server -ext "SAN:c=DNS:localhost,IP:127.0.0.1" -keystore conf/server.keystore
RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:25.0

COPY --from=builder /opt/keycloak/ /opt/keycloak/

WORKDIR /opt/keycloak

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/health/ready || exit 1

USER keycloak
ENTRYPOINT ["/opt/keycloak/bin/kc.sh", "start", "--optimized"]
```

When the wrapper JAR is present but the JDBC URL uses the standard MySQL protocol, Keycloak continues to use the default MySQL driver, preserving password-auth behavior.

## IAM Changes

### Keycloak Task Role

Add an inline policy attached to `aws_iam_role.keycloak_task_role` when IAM auth is enabled.

```hcl
resource "aws_iam_role_policy" "keycloak_task_rds_iam_policy" {
  count = var.keycloak_db_use_iam ? 1 : 0
  name  = "keycloak-task-rds-iam-policy"
  role  = aws_iam_role.keycloak_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = "rds-db:connect"
        Resource = "arn:aws:rds-db:${var.aws_region}:${data.aws_caller_identity.current.account_id}:dbuser:${aws_rds_cluster.keycloak.cluster_resource_id}/${var.keycloak_rds_iam_username}"
      }
    ]
  })
}
```

### RDS Proxy Role

Add `rds-db:connect` to the existing `aws_iam_role_policy.rds_proxy_policy` when IAM auth is enabled.

```hcl
resource "aws_iam_role_policy" "rds_proxy_policy" {
  name = "keycloak-rds-proxy-policy"
  role = aws_iam_role.rds_proxy_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = concat(
      [
        {
          Effect = "Allow"
          Action = ["secretsmanager:GetSecretValue"]
          Resource = aws_secretsmanager_secret.keycloak_db_secret.arn
        }
      ],
      var.keycloak_db_use_iam ? [
        {
          Effect = "Allow"
          Action = "rds-db:connect"
          Resource = "arn:aws:rds-db:${var.aws_region}:${data.aws_caller_identity.current.account_id}:dbuser:${aws_rds_cluster.keycloak.cluster_resource_id}/${var.keycloak_rds_iam_username}"
        }
      ] : []
    )
  })
}
```

## Database User Bootstrap

Terraform cannot create Aurora MySQL users directly. Use a VPC Lambda that runs once after the cluster is available.

### New Lambda: `terraform/aws-ecs/lambda/keycloak-rds-iam-init/`

Files:
- `index.py` - Connects to Aurora with the native master password and creates/replaces the IAM user.
- `requirements.txt` - `boto3`, `pymysql`.

Core logic:

```python
def lambda_handler(event, context):
    secret = get_secret("keycloak/database")
    host = get_ssm("/keycloak/database/url").split("//")[1].split(":")[0]
    iam_username = get_ssm("/keycloak/database/iam_username")

    conn = pymysql.connect(
        host=host,
        user=secret["username"],
        password=secret["password"],
        database="mysql",
        ssl={"required": True},
    )
    try:
        with conn.cursor() as cur:
            cur.execute(f"CREATE USER IF NOT EXISTS '{iam_username}' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';")
            cur.execute(f"ALTER USER '{iam_username}'@'%' REQUIRE SSL;")
            cur.execute(f"GRANT ALL PRIVILEGES ON keycloak.* TO '{iam_username}'@'%';")
            cur.execute("FLUSH PRIVILEGES;")
        conn.commit()
    finally:
        conn.close()
    return {"statusCode": 200, "body": "IAM database user initialized"}
```

### Terraform wiring

Create a new file `terraform/aws-ecs/keycloak-iam.tf` (or add to `keycloak-database.tf`):

```hcl
resource "aws_iam_role" "keycloak_rds_iam_init_lambda" {
  count = var.keycloak_db_use_iam ? 1 : 0
  name  = "keycloak-rds-iam-init-${var.aws_region}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "keycloak_rds_iam_init_lambda" {
  count = var.keycloak_db_use_iam ? 1 : 0
  name  = "keycloak-rds-iam-init-policy"
  role  = aws_iam_role.keycloak_rds_iam_init_lambda[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["secretsmanager:GetSecretValue"]
        Resource = aws_secretsmanager_secret.keycloak_db_secret.arn
      },
      {
        Effect = "Allow"
        Action = ["ssm:GetParameter"]
        Resource = [
          aws_ssm_parameter.keycloak_database_url.arn,
          aws_ssm_parameter.keycloak_database_iam_username[0].arn,
        ]
      },
      {
        Effect = "Allow"
        Action = ["kms:Decrypt"]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ec2:CreateNetworkInterface",
          "ec2:DescribeNetworkInterfaces",
          "ec2:DeleteNetworkInterface",
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_lambda_function" "keycloak_rds_iam_init" {
  count = var.keycloak_db_use_iam ? 1 : 0

  filename         = data.archive_file.keycloak_rds_iam_init[0].output_path
  function_name    = "${var.name}-keycloak-rds-iam-init"
  role             = aws_iam_role.keycloak_rds_iam_init_lambda[0].arn
  handler          = "index.lambda_handler"
  source_code_hash = data.archive_file.keycloak_rds_iam_init[0].output_base64sha256
  runtime          = "python3.13"
  timeout          = 300
  memory_size      = 256

  vpc_config {
    subnet_ids         = module.vpc.private_subnets
    security_group_ids = [aws_security_group.rotation_lambda.id]
  }

  depends_on = [
    aws_iam_role_policy.keycloak_rds_iam_init_lambda,
    aws_iam_role_policy_attachment.lambda_vpc_execution,
  ]
}

resource "aws_lambda_invocation" "keycloak_rds_iam_init" {
  count = var.keycloak_db_use_iam ? 1 : 0

  function_name = aws_lambda_function.keycloak_rds_iam_init[0].function_name
  input         = jsonencode({})

  lifecycle {
    replace_triggered_by = [
      aws_lambda_function.keycloak_rds_iam_init[0]
    ]
  }
}
```

Note: `aws_lambda_invocation` is provided by the Terraform AWS provider. It invokes the function once during apply. Alternatively, use a `null_resource` with a `local-exec` provisioner that calls `aws lambda invoke`.

## Implementation Steps

For a future implementer, the recommended order is:

1. Add variables `keycloak_db_use_iam`, `keycloak_rds_iam_username`, and `keycloak_iam_auth_image_uri` to `variables.tf`.
2. Extend `docker/keycloak/Dockerfile` to download the AWS Advanced JDBC Wrapper JAR into `/opt/keycloak/providers/`.
3. Build and push the custom image to ECR; capture the URI.
4. Update `keycloak-database.tf`:
   - Add `iam_database_authentication_enabled`.
   - Make the RDS Proxy auth block conditional.
   - Update `KC_DB_URL` SSM parameter with wrapper URL and SSL params.
   - Add `KC_DB_USERNAME` SSM parameter for the IAM user.
5. Update `keycloak-ecs.tf`:
   - Make container secrets conditional.
   - Make task image conditional.
   - Add precondition validating custom image.
   - Update task exec role policy for the new SSM parameter.
6. Add IAM policies for `rds-db:connect` to the Keycloak task role and RDS Proxy role.
7. Create `lambda/keycloak-rds-iam-init/` and wire it in a new `keycloak-iam.tf`.
8. Add a security group ingress rule allowing the init Lambda to reach Aurora (reuse `rotation_lambda` security group, which already has access).
9. Update `terraform.tfvars.example` and `README.md`.
10. Run `terraform plan` with `keycloak_db_use_iam = false` to confirm no drift.
11. Run `terraform apply` with `keycloak_db_use_iam = true` in a non-production environment and verify Keycloak reaches `healthy` status.

## Observability

- Add CloudWatch Logs output in the init Lambda for user creation/replacement.
- Log the value of `keycloak_db_use_iam` in the task environment (the flag itself, not credentials) for debugging.
- Monitor RDS Proxy `ClientConnections` and `DatabaseConnections` metrics after cutover.
- Add an alarm on Keycloak task health-check failures after switching auth modes.

## Scaling Considerations

- IAM auth tokens are generated per connection inside the JDBC wrapper, so no centralized token service is needed.
- The init Lambda runs once per apply; it is not in the request path.
- Aurora MySQL IAM auth has a maximum number of connections per second per IAM user. For high-traffic deployments, monitor `Connections` metrics and consider connection pool tuning in Keycloak/Quarkus.
- The custom image adds one JAR (~2 MB) and does not change Keycloak memory requirements.

## File Changes

### New Files

| File Path | Description |
|-----------|-------------|
| `terraform/aws-ecs/keycloak-iam.tf` | Bootstrap Lambda IAM role, function, and invocation for IAM DB user creation |
| `terraform/aws-ecs/lambda/keycloak-rds-iam-init/index.py` | Lambda that creates the Aurora MySQL IAM user |
| `terraform/aws-ecs/lambda/keycloak-rds-iam-init/requirements.txt` | Lambda dependencies |

### Modified Files

| File Path | Lines | Change Description |
|-----------|-------|--------------------|
| `terraform/aws-ecs/variables.tf` | ~+25 | Add `keycloak_db_use_iam`, `keycloak_rds_iam_username`, `keycloak_iam_auth_image_uri` |
| `terraform/aws-ecs/main.tf` | ~+5 | Pass new flag only to Keycloak resources |
| `terraform/aws-ecs/keycloak-database.tf` | ~+30 | Conditional IAM auth on cluster/proxy, new SSM parameter, SSL URL |
| `terraform/aws-ecs/keycloak-ecs.tf` | ~+40 | Conditional secrets/image, precondition, task exec policy update |
| `terraform/aws-ecs/secret-rotation.tf` | ~+15 | Add `rds-db:connect` to proxy policy when IAM enabled |
| `docker/keycloak/Dockerfile` | ~+5 | Download AWS JDBC wrapper JAR |
| `terraform/aws-ecs/terraform.tfvars.example` | ~+15 | Document new variables |
| `terraform/aws-ecs/README.md` | ~+30 | Security/deployment section for IAM auth |

### Estimated Lines of Code

| Category | Lines |
|----------|-------|
| New Terraform | ~150 |
| New Python (Lambda) | ~60 |
| Modified Terraform | ~120 |
| Modified Dockerfile/docs | ~40 |
| **Total** | **~370** |

## Testing Strategy

See `testing.md` for the complete executable test plan.

## Alternatives Considered

### Alternative 1: Sidecar that generates tokens and restarts Keycloak

A lightweight sidecar container generates an RDS IAM token every 10 minutes and writes it to a shared volume. The Keycloak entrypoint reads the token once at startup. To refresh, the sidecar would need to restart the Keycloak container, causing periodic downtime. This was rejected because it is operationally fragile and conflicts with a long-running identity service.

### Alternative 2: Custom entrypoint with periodic re-exec

A wrapper script runs as PID 1, reads the token, starts `kc.sh`, and re-execs `kc.sh` before the token expires. This adds process-supervision complexity and still causes brief restarts. Rejected in favor of the JDBC wrapper, which refreshes tokens transparently per connection.

### Alternative 3: AWS Advanced JDBC Wrapper (chosen)

The wrapper is added to the Keycloak providers directory and activated only by the JDBC URL. It uses the ECS task role credentials to generate a fresh RDS IAM token for each connection. This avoids sidecars, restarts, and token lifetime concerns. The trade-off is a custom image build and a small amount of Quarkus/JDBC configuration.

### Comparison Matrix

| Criteria | Sidecar + restart | Wrapper re-exec | AWS JDBC wrapper |
|----------|-------------------|-----------------|------------------|
| Operational complexity | Medium | High | Low |
| Token lifetime risk | High | Medium | Low |
| Downtime on refresh | Yes | Yes | No |
| Custom image required | No | No | Yes |
| Production readiness | Poor | Poor | Good |

## Rollout Plan

- Phase 1 (implementation): Merge Terraform, Lambda, Dockerfile, and documentation changes. No runtime impact because the default is `keycloak_db_use_iam = false`.
- Phase 2 (validation): Deploy to a non-production environment with `keycloak_db_use_iam = true`, build and push the custom image, verify Keycloak health checks and login flows.
- Phase 3 (production cutover): In a maintenance window, set `keycloak_db_use_iam = true`, update `keycloak_iam_auth_image_uri`, run `terraform apply`. Validate password fallback by toggling the flag back to `false`.
- Phase 4 (cleanup): Once IAM auth is stable, consider deprecating the rotation Lambda or keeping it for fallback only.

## Open Questions

1. Should the bootstrap Lambda be idempotent and skip creation if the IAM user already exists, or should it always `CREATE USER IF NOT EXISTS`?
2. Is the AWS JDBC wrapper version `2.3.7` compatible with Keycloak 25's MySQL driver expectations, or does a different version need to be pinned?
3. Should the custom Keycloak image be pre-built and published to the project's public ECR, or should operators build and push it themselves?
4. How should existing deployments migrate the database state from the native `keycloak` user to the new `keycloak_iam` user without re-creating the cluster?
