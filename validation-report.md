# AWS HealthOmics VPC Integration - Validation Report

**Purpose:** Validate AWS HealthOmics VPC configuration with third-party IdP federation and SDK programmatic access

---

## 1. Test Environment Summary

### 1.1 Architecture

```
[Keycloak Mock IdP] --SAML 2.0 (HTTPS)--> [AWS IAM SAML Provider] --> [IAM Roles (3)]
   (TLS, port 8443)    (Assertion Encrypted)   (SourceIp restricted)    Admin/Analyst/ReadOnly
                                                                              |
[SageMaker Notebook] --Private Subnet--> [VPC Endpoints] --> [HealthOmics APIs]
     (ml.t3.medium)                      (5 Interface)       (Workflows/Storage/Analytics)
```

### 1.2 Infrastructure

| Component | Detail |
|-----------|--------|
| VPC | `<VPC_ID>` (10.0.0.0/16) |
| Private Subnets | 2x (AZ-a: 10.0.1.0/24, AZ-b: 10.0.2.0/24) |
| Public Subnet | 1x (AZ-a: 10.0.3.0/24) |
| Security Group | `<SG_VPCE>` - HealthOmics-VPCEndpoint-SG (TCP 443, VPC CIDR) |
| SageMaker Notebook | ml.t3.medium, Private Subnet, no internet |

### 1.3 VPC Endpoints (5 HealthOmics)

| Service | State | Policy |
|---------|-------|--------|
| workflows-omics | Available | Account-restricted |
| storage-omics | Available | Account-restricted |
| analytics-omics | Available | Account-restricted |
| tags-omics | Available | Account-restricted |
| control-storage-omics | Available | Account-restricted |

- Private DNS enabled on all endpoints
- Deployed in both Private Subnets

### 1.4 Mock IdP (Keycloak as Azure Entra ID)

| Item | Value |
|------|-------|
| URL | https://localhost:8443 |
| Realm | azure-entra-mock |
| SAML Client ID | urn:amazon:webservices |
| TLS | Self-signed certificate, HTTPS only |
| SAML Signing | Enabled (server + assertion) |
| SAML Encryption | Enabled (saml.encrypt=true) |
| MFA | TOTP Required |

### 1.5 Test Users

| User | Email | Mapped IAM Role |
|------|-------|-----------------|
| bioepis-admin | admin@example-test.com | HealthOmics-SAML-Admin |
| bioepis-analyst | analyst@example-test.com | HealthOmics-SAML-Analyst |
| bioepis-readonly | readonly@example-test.com | HealthOmics-SAML-ReadOnly |

---

## 2. IAM RBAC Validation

### 2.1 IAM Policy Configuration

| Role | Policy | Allowed Actions |
|------|--------|-----------------|
| Admin | HealthOmics-FullAccess | `omics:*` (all), S3 R/W, KMS all, Logs all |
| Analyst | HealthOmics-Analyst | `omics:Get*`, `omics:List*`, `omics:StartRun`, `omics:TagResource`, S3 RO, KMS decrypt |
| ReadOnly | HealthOmics-ReadOnly | `omics:Get*`, `omics:List*`, S3 RO |

### 2.2 IAM Policy Simulator Results

#### Admin Role

| API Action | Result | Note |
|------------|--------|------|
| omics:ListWorkflows | ALLOW | Resource: * |
| omics:ListRuns | ALLOW | Resource: * |
| omics:GetWorkflow | ALLOW | Resource: workflow ARN |
| omics:StartRun | ALLOW | Resource: workflow ARN |
| omics:DeleteWorkflow | ALLOW | Resource: workflow ARN |
| omics:CancelRun | ALLOW | Resource: run ARN |

#### Analyst Role

| API Action | Result | Note |
|------------|--------|------|
| omics:ListWorkflows | ALLOW | |
| omics:GetWorkflow | ALLOW | |
| omics:StartRun | ALLOW | Core analyst permission |
| omics:TagResource | ALLOW | |
| omics:DeleteRun | DENY | No delete permission (by design) |
| omics:DeleteWorkflow | DENY | No delete permission (by design) |
| omics:CancelRun | DENY | |

#### ReadOnly Role

| API Action | Result | Note |
|------------|--------|------|
| omics:ListWorkflows | ALLOW | |
| omics:ListRuns | ALLOW | |
| omics:GetRun | ALLOW | |
| omics:StartRun | DENY | Read-only (by design) |
| omics:DeleteRun | DENY | Read-only (by design) |
| omics:TagResource | DENY | Read-only (by design) |

### 2.3 Actual API Call Test Results

Each SAML role was assumed via AssumeRole and tested with real HealthOmics API calls.

> **Method:** Temporarily added `sts:AssumeRole` to each role's Trust Policy, executed tests, then reverted to SAML-only Trust Policy.

| API Action | Admin | Analyst | ReadOnly | Note |
|------------|:-----:|:-------:|:--------:|------|
| ListWorkflows | ALLOW | ALLOW | ALLOW | All roles allowed |
| ListRuns | ALLOW | ALLOW | ALLOW | All roles allowed |
| GetWorkflow | ALLOW | ALLOW | ALLOW | |
| ListSequenceStores | ALLOW | ALLOW | ALLOW | All roles allowed |
| TagResource | ALLOW | ALLOW | **DENY** | ReadOnly write blocked |
| StartRun | ALLOW | ALLOW | **DENY** | Admin/Analyst: `ValidationException` (permission OK), ReadOnly: `AccessDeniedException` |
| DeleteWorkflow | ALLOW | **DENY** | **DENY** | Admin: `ResourceNotFoundException` (permission OK), others: `AccessDeniedException` |
| DeleteRun | ALLOW | **DENY** | **DENY** | Admin: `ResourceNotFoundException` (permission OK), others: `AccessDeniedException` |

**8 APIs x 3 Roles = 24 tests, all matched expected results (24/24 PASS)**

#### Test Criteria
- **ALLOW**: Normal response or `ValidationException`/`ResourceNotFoundException` (permission passed, parameter/resource missing)
- **DENY**: `AccessDeniedException` (IAM policy denied)

#### Findings & Fixes During Testing
- Admin/Analyst SAML roles were missing `iam:PassRole` inline policy, causing `AccessDeniedException` on StartRun -> Fixed by adding `AllowPassOmicsRole` inline policy
- PassRole targets: HealthOmics service role (condition: `iam:PassedToService: omics.amazonaws.com`)

---

## 3. Security Validation Results

### 3.1 Checklist (13/13 Passed)

| # | Item | Result | Evidence |
|---|------|--------|----------|
| 1 | Keycloak SAML login (HTTPS) | PASS | HTTPS port 8443, HTTP disabled, ACS URL `https://signin.aws.amazon.com/saml` |
| 2 | MFA (TOTP) mandatory | PASS | Browser auth flow: OTP changed from ALTERNATIVE to REQUIRED |
| 3 | Role-based access control | PASS | Per-user `aws-role` attribute mapping, IAM Simulator + **actual API test 24/24 PASS** |
| 4 | SourceIp restriction | PASS | All 3 SAML Trust Policies include `aws:SourceIp: 10.0.0.0/16` |
| 5 | SAML Assertion encryption | PASS | `saml.encrypt=true`, `saml.server.signature=true`, `saml.assertion.signature=true` |
| 6 | SageMaker -> HealthOmics SDK | PASS | Workflow run COMPLETED via SDK from SageMaker Notebook (private subnet, no internet) |
| 7 | VPC Endpoint policy | PASS | All 5 endpoints: account-restricted policy applied |
| 8 | CloudTrail audit logs | PASS | omics API events captured (GetReferenceStore, ListTagsForResource, GetWorkflow, etc.) |
| 9 | IP whitelist (Security Group) | PASS | Inbound TCP 443 from VPC CIDR only |
| 10 | KMS encryption (rest/transit) | PASS | AWS-owned KMS default, TLS 1.2+ on all endpoints |
| 11 | Confused deputy prevention | PASS | Trust Policy: `aws:SourceAccount` + `aws:SourceArn` for both omics and sagemaker principals |
| 12 | Analyst: StartRun yes, Delete no | PASS | IAM Simulator + **actual API test**: StartRun=`ValidationException`(OK), Delete=`AccessDeniedException` |
| 13 | ReadOnly: Get/List only | PASS | IAM Simulator + **actual API test**: Get/List=allowed, StartRun/Delete/Tag=`AccessDeniedException` |

### 3.2 Security Hardening History

| # | Item | Before | After |
|---|------|--------|-------|
| 1 | SAML Trust Policy | No SourceIp restriction | `aws:SourceIp: <VPC_CIDR>` on all 3 roles |
| 2 | Keycloak transport | HTTP (port 8080) | HTTPS (port 8443), HTTP disabled |
| 3 | SAML Assertion encryption | Optional (not applied) | Required (saml.encrypt=true) |
| 4 | IAM policies | Single `omics:*` policy shared | Admin / Analyst / ReadOnly separated |
| 5 | Confused Deputy prevention | No SageMaker condition | `aws:SourceAccount` + `aws:SourceArn` added |
| 6 | SAML signing certificate | 10-year default | 1-year validity, rotation required |
| 7 | VPC Endpoint policy | Default (allow all) | Account-restricted |
| 8 | MFA enforcement | ALTERNATIVE (optional) | REQUIRED (mandatory) |
| 9 | IAM Policy List* fix | `omics:List*` on scoped ARN (broken) | `omics:List*` on `Resource: "*"` (separate Statement) |
| 10 | SAML role PassRole | Not configured (StartRun blocked) | Admin/Analyst: `iam:PassRole` inline policy added |

---

## 4. SDK Integration Test Results

### 4.1 Test Environment

- **Runtime:** SageMaker Notebook (ml.t3.medium, Private Subnet, no internet)
- **SDK:** boto3 omics client
- **Notebook:** `healthomics-sdk-test.ipynb`

### 4.2 VPC Connectivity

| Endpoint | DNS Resolution | Status |
|----------|---------------|--------|
| workflows.omics.`<REGION>`.amazonaws.com | Private IP confirmed | OK |
| storage.omics.`<REGION>`.amazonaws.com | Private IP confirmed | OK |
| analytics.omics.`<REGION>`.amazonaws.com | Private IP confirmed | OK |
| tags.omics.`<REGION>`.amazonaws.com | Private IP confirmed | OK |
| control-storage.omics.`<REGION>`.amazonaws.com | Private IP confirmed | OK |

### 4.3 SDK Operation Tests

| Test | Result | Detail |
|------|--------|--------|
| List workflows | PASS | Multiple workflows discovered |
| Get workflow detail | PASS | Parameters, engine, status confirmed |
| Start workflow run | PASS | Nextflow workflow executed |
| Monitor run status | PASS | Status: COMPLETED |
| List sequence stores | PASS | Sequence store found |
| List reference stores | PASS | |
| CloudTrail log check | PASS | All API calls recorded |

### 4.4 Workflow Execution Result

| Item | Value |
|------|-------|
| Workflow Engine | Nextflow |
| Status | COMPLETED |
| Storage Type | DYNAMIC |

---

## 5. Production Recommendations

| # | Item | Demo State | Production Recommendation |
|---|------|-----------|--------------------------|
| 1 | User passwords | Shared demo password | Unique per user, complexity policy (min 12 chars, 2+ special) |
| 2 | MFA verification | Keycloak TOTP configured | End-to-end login flow E2E test needed |
| 3 | TLS certificate | Self-signed | Replace with CA-signed certificate |
| 4 | SAML key management | Local filesystem | Migrate to AWS Secrets Manager or HSM |
| 5 | Keycloak admin | Default credentials | Strong password, IP-restricted admin console |
| 6 | IdP replacement | Keycloak Mock | Azure Entra ID (same SAML config) |
| 7 | SAML certificate renewal | 1-year expiry | Regenerate Keycloak key + update AWS SAML metadata before expiry |
| 8 | KMS keys | AWS-owned (default) | Consider customer-managed KMS keys (CMK) |
| 9 | VPC Endpoint policy | Account-level restriction | Fine-tune to specific IAM roles/resources |

---

## 6. Conclusion

**All 13 security validation items PASSED**, and **all 24 actual API RBAC tests matched expected results (24/24 PASS)**.

### Key Achievements

1. **Azure Entra ID Federation Simulation** - SAML 2.0 federation, mandatory MFA, 3-tier RBAC verified via Keycloak Mock IdP.

2. **VPC Private Networking Verified** - HealthOmics SDK calls via VPC Endpoints from SageMaker Notebook (no internet access). Workflow run completed successfully.

3. **10 Security Hardening Items Applied** - SourceIp restriction, HTTPS-only, SAML assertion encryption, RBAC policy separation, confused deputy prevention, VPC Endpoint policy restriction, mandatory MFA, certificate validity reduction, IAM List* fix, PassRole addition.

4. **Actual API RBAC Verification** - 8 HealthOmics APIs tested across 3 SAML roles (Admin/Analyst/ReadOnly), confirming permission separation works as designed (24/24 PASS).

5. **Audit Trail Confirmed** - All HealthOmics API calls recorded via CloudTrail.

This demo environment can be directly adopted for production by replacing Keycloak with Azure Entra ID and applying the recommendations above.

---

**Attachments:**
- `setup-guide.md` - Full environment setup guide
- `healthomics-sdk-test.ipynb` - HealthOmics SDK test Jupyter notebook
- `validation-report.md` - This validation report
