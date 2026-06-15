# Expert Review: Remove EFS from Terraform AWS ECS Configuration

*Created: 2024-06-15*
*Related LLD: `./lld.md`*
*Related Issue: `./github-issue.md`*

## Review Summary

| Reviewer | Verdict | Blockers | Key Recommendations |
|----------|---------|----------|---------------------|
| Frontend (Pixel) | N/A | 0 | No frontend changes required |
| Backend (Byte) | **APPROVED WITH CHANGES** | 2 | Security hardening, IAM scope |
| SRE (Circuit) | **APPROVED WITH CHANGES** | 1 | Migration tooling |
| Security (Cipher) | **APPROVED WITH CHANGES** | 1 | Parameter Store encryption |
| SMTS (Sage) | **APPROVED WITH CHANGES** | 1 | Testing verifications |

## Reviewer: Backend Engineer (Byte)

### Focus: API design, data models, business logic, performance

### Strengths
✓ Clean separation of concerns - removing infrastructure without application changes
✓ Well-analyzed migration path from EFS to AWS-native services
✓ Parameter Store is appropriate for configuration storage
✓ IAM policies correctly scoped to specific parameter paths
✓ Maintains stateless design principles

### Concerns
**Blocker 1: Parameter Store Size Limits**
- Systems Manager Parameter Store has a 4KB limit for standard parameters
- Scopes.yml file currently unknown size - could exceed limit
- **Recommendation**: Use Parameter Store "Advanced" parameters (8KB limit) or Secrets Manager for larger configs

**Blocker 2: Configuration Updates**
- Current design suggests static file upload to Parameter Store
- No mechanism for configuration updates without restart
- **Recommendation**: Add configuration reload mechanism or use dynamic parameter retrieval

**Non-Blocker 1: Performance Impact**
- Slight concern about Parameter Store throttling during startup storms
- Could cache parameters in task definition or use environment variables
- **Recommendation**: Document expected call patterns and add caching suggestions

### Better Alternatives Considered
Considered Secrets Manager vs Parameter Store:
- **Parameter Store**: Lower cost, easier IAM policies, built-in CloudFormation support
- **Secrets Manager**: Automatic rotation, better for sensitive data
- **Verdict**: Parameter Store is correct choice for scopes.yml (non-sensitive config)

### Verdict: APPROVED WITH CHANGES - 2 Blockers
1. Validate scopes.yml fits in Parameter Store limits
2. Add configuration reload or dynamic retrieval

---

## Reviewer: SRE/DevOps Engineer (Circuit)

### Focus: Deployment, monitoring, scaling, infrastructure

### Strengths
✓ Excellent migration path documentation
✓ Clear rollback strategy via infrastructure-as-code
✓ Observability concerns addressed
✓ Performance considerations well-analyzed
✓ EFS removal reduces cross-AZ traffic and costs

### Concerns
**Blocker 1: Migration Tooling**
- No tooling provided to extract existing EFS data
- Users upgrading from EFS version face manual migration
- **Recommendation**: Provide standalone migration script for EFS → Parameter Store + S3

**Non-Blocker 1: Deployment Sequence**
- Concern about applying changes while services are running
- Could cause brief config unavailability during Parameter Store setup
- **Recommendation**: Add "Maintenance Window" recommendation to deployment docs

**Non-Blocker 2: Monitoring**
- No dashboards for Parameter Store health tracking
- Should add CloudWatch alarms for parameter access errors
- **Recommendation**: Include CloudWatch alarm configurations for SSM Parameter Store

### Better Alternatives Considered
**Alternative**: Can we support both EFS and Parameter Store temporarily?
- **Pros**: Zero-downtime migration
- **Cons**: Adds complexity, conditional logic in Terraform
- **Verdict**: Clean break is better for long-term maintainability

### Verdict: APPROVED WITH CHANGES - 1 Blocker  
1. Provide EFS migration tooling for existing users

---

## Reviewer: Security Engineer (Cipher)

### Focus: AuthN/AuthZ, validation, OWASP, data protection

### Strengths
✓ IAM policies follow least-privilege principle
✓ EFS security group removal eliminates NFS attack surface
✓ Parameter Store allows encryption at rest
✓ No sensitive data in scope.yml (public OAuth scopes)
✓ Transit encryption already enabled for CloudWatch Logs

### Concerns
**Blocker 1: Parameter Store Encryption**
- scopes.yml contains OAuth scope definitions - not sensitive but should be protected
- Current design uses standard (non-encrypted) Parameter Store parameters
- **Recommendation**: Enable KMS encryption for scopes.yml parameter despite low sensitivity

```terraform
# Before:
type = "String"

# After:
type = "SecureString"
key_id = aws_kms_key.parameter_store.arn
```

**Non-Blocker 1: File Permissions**
- scopes.yml will be written to `/tmp/scopes.yml` in container
- Should verify file permissions are restricted (600) to prevent container escape
- **Recommendation**: Add chmod 600 in container startup script

**Non-Blocker 2: Parameter Path Naming**
- Using `/mcp-gateway/auth-server/scopes-yml` which is fine but parameter store path structure should be documented
- **Recommendation**: Document parameter naming convention in README

### Better Alternatives Considered
**Alternative**: Use Secrets Manager instead of Parameter Store
- **Pros**: Better audit trail, automatic rotation capabilities
- **Cons**: More complex IAM, additional cost ($0.40/secret/month)
- **Verdict**: Parameter Store adequate for scopes.yml, but Secrets Manager worth considering for future sensitive configs

### Verdict: APPROVED WITH CHANGES - 1 Blocker
1. Enable KMS encryption for scopes.yml parameter

---

## Reviewer: SMTS (Sage)

### Focus: Architecture, code quality, maintainability

### Strengths
✓ Well-structured implementation plan with clear file-by-file changes
✓ Follows existing patterns from registry service migration (issue #1122)
✓ Comprehensive test plan (separate document)
✓ Good analysis of alternatives
✓ Clear acceptance criteria

### Concerns
**Blocker 1: Testing Verifications**
- LLD mentions "Application logging already goes to CloudWatch" but:
  - Need to verify auth-server is actually writing to CloudWatch today
  - Need to verify mcpgw logs are handled similarly
  - Should validate by checking current CloudWatch log groups in running deployments
- **Recommendation**: Add pre-implementation verification step to confirm existing CloudWatch logging

**Non-Blocker 1: Documentation Completeness**
- Excellent infrastructure docs, but missing application-level documentation
- Application code doesn't know how to read from Parameter Store yet
- **Recommendation**: Provide example Python code showing how to fetch and write scopes.yml from Parameter Store

**Non-Blocker 2: Rollback Plan**
- While Terraform enables rollback via state, no explicit rollback procedure documented
- Should mention: `terraform state show`, `terraform state rm` workflows for recovery
- **Recommendation**: Add rollback procedure to deployment docs

**Non-Blocker 3: Scope Creep Risk**
- LLD mentions "Evaluate mcpgw_data for S3 vs ephemeral" - creates unnecessary decision point
- Recommend making the simpler choice (ephemeral) with clear documentation
- **Recommendation**: Simplify by standardizing on ephemeral for demo data

### Better Alternatives Considered
**Alternative**: Micro-batches instead of single big-bang change
- **Pros**: Lower risk, easier debugging, backward compatible at each step
- **Cons**: Takes longer, requires more PR cycles
- **Verdict**: Too slow for infrastructure simplification; big-bang acceptable here

### Verdict: APPROVED WITH CHANGES - 1 Blocker
1. Verify existing CloudWatch logging setup before making changes

---

## Review Summary Table

| Category | Rating | Summary |
|----------|--------|---------|
| **Technical Design** | 8/10 | Well-architected migration to AWS-native services |
| **Security** | 8/10 | Strong IAM policies, needs encryption enabled |
| **Operational Readiness** | 7/10 | Good monitoring, lacks migration tooling |
| **Risk Assessment** | 6/10 | Medium risk - breaking change with mitigation steps |
| **Cost Impact** | 9/10 | Significant cost reduction opportunity |

## Questions for Author

1. **Backend (Byte)**: Can we add a small Python library to abstract Parameter Store access?
2. **SRE (Circuit)**: What Terraform version did you test this with? v1.0+?
3. **Security (Cipher)**: Should we use AWS Config rule to enforce Parameter Store encryption?
4. **All**: Should we create a feature flag to enable/disable EFS during transition period?

## Recommendations Summary

### Must-Fix (Blockers)
1. **Validate Parameter Store limits** - Ensure scopes.yml fits in 8KB advanced parameter limit
2. **Enable KMS encryption** - Use SecureString for scopes.yml parameter
3. **Verify existing CloudWatch logging** - Confirm auth-server and mcpgw already log to CloudWatch
4. **Create migration tooling** - Script to extract scopes.yml from existing EFS

### Should-Fix (Non-Blockers)
1. Add config reload mechanism for scopes.yml changes
2. Document parameter naming conventions
3. Include CloudWatch alarms for SSM Parameter Store
4. Add chmod 600 to container startup for scopes.yml
5. Provide example Python code for Parameter Store integration
6. Add rollback procedure documentation

### Nice-to-Have
1. Consider Secrets Manager for future sensitive configs
2. Add maintenance window recommendation to deployment
3. Create dashboard templates for new monitoring metrics
4. Add feature flag for gradual rollout

## Final Verdict

**Status**: APPROVED FOR IMPLEMENTATION with 4 blockers

**Overall Assessment**: The design direction is sound and follows AWS best practices. The economic benefits (cost reduction + complexity reduction) justify the breaking change. Good analysis of alternatives and clear migration path.

**Risk Level**: MEDIUM - Manageable with proper testing and the recommended changes

**Next Steps**:
1. Address all 4 blockers
2. Implement design in feature branch
3. Execute testing plan from testing.md
4. Code review with blockers resolved
5. Merge with clear release notes and migration guide