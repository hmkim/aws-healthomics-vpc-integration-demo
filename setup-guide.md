# AWS HealthOmics VPC Integration - Setup Guide

## Architecture Overview

```
[Keycloak Mock IdP] --SAML 2.0 (HTTPS)--> [AWS IAM SAML Provider] --> [IAM Roles (3)]
   (TLS, port 8443)    (Assertion Encrypted)   (SourceIp restricted)    Admin/Analyst/ReadOnly
                                                                              |
[SageMaker Notebook] --Private Subnet--> [VPC Endpoints] --> [HealthOmics APIs]
     (ml.t3.medium)                      (5x Interface)      (Workflows/Storage/Analytics)
```

---

## 1. Keycloak Mock IdP (Azure Entra ID Replacement)

### Service Info

| Item | Value |
|------|-------|
| Keycloak URL | https://localhost:8443 |
| Admin Console | https://localhost:8443/admin |
| Admin ID/PW | admin / admin (demo only, change in production) |
| Realm | azure-entra-mock |
| SSL Required | all (HTTP disabled) |
| SAML Metadata | https://localhost:8443/realms/azure-entra-mock/protocol/saml/descriptor |
| SAML SSO Endpoint | https://localhost:8443/realms/azure-entra-mock/protocol/saml |
| Entity ID | https://localhost:8443/realms/azure-entra-mock |
| SAML Client ID | urn:amazon:webservices |
| TLS Certificate | ./keycloak-certs/keycloak-tls-cert.pem |
| TLS Private Key | ./keycloak-certs/keycloak-tls-key.pem |

### Test Users

| Username | Email | Password | AWS IAM Role |
|----------|-------|----------|--------------|
| bioepis-admin | admin@example-test.com | (set your own) | HealthOmics-SAML-Admin |
| bioepis-analyst | analyst@example-test.com | (set your own) | HealthOmics-SAML-Analyst |
| bioepis-readonly | readonly@example-test.com | (set your own) | HealthOmics-SAML-ReadOnly |

### SAML Mappers

| Mapper | SAML Attribute | Value |
|--------|---------------|-------|
| aws-role-mapper | https://aws.amazon.com/SAML/Attributes/Role | User attribute `aws-role` (per-user IAM Role ARN) |
| aws-session-duration | https://aws.amazon.com/SAML/Attributes/SessionDuration | 3600 (1 hour) |
| aws-session-name-email | https://aws.amazon.com/SAML/Attributes/RoleSessionName | User email address |

### User -> IAM Role Mapping (aws-role attribute)

| User | aws-role Attribute Value |
|------|------------------------|
| bioepis-admin | `arn:aws:iam::<ACCOUNT_ID>:saml-provider/AzureEntraMock-Keycloak,arn:aws:iam::<ACCOUNT_ID>:role/HealthOmics-SAML-Admin` |
| bioepis-analyst | `arn:aws:iam::<ACCOUNT_ID>:saml-provider/AzureEntraMock-Keycloak,arn:aws:iam::<ACCOUNT_ID>:role/HealthOmics-SAML-Analyst` |
| bioepis-readonly | `arn:aws:iam::<ACCOUNT_ID>:saml-provider/AzureEntraMock-Keycloak,arn:aws:iam::<ACCOUNT_ID>:role/HealthOmics-SAML-ReadOnly` |

### MFA Policy
- Type: TOTP (Time-based One-Time Password)
- Algorithm: HmacSHA1, Digits: 6, Period: 30 seconds
- Enforcement: **Required** (Browser Auth Flow -> OTP = REQUIRED)

### SAML Security Settings
| Item | Value |
|------|-------|
| saml.encrypt | true (Assertion encryption enabled) |
| saml.server.signature | true (Server signature enabled) |
| saml.assertion.signature | true (Assertion signature enabled) |
| saml.force.post.binding | true (POST binding enforced) |

---

## 2. AWS Infrastructure

### Account & Region

| Item | Value |
|------|-------|
| Account ID | `<AWS_ACCOUNT_ID>` |
| Region | `<AWS_REGION>` (e.g., us-west-2) |
| VPC | `<VPC_ID>` (CIDR: 10.0.0.0/16) |

### VPC Subnets

| Subnet | CIDR | AZ | Type |
|--------|------|----|------|
| `<PRIVATE_SUBNET_1>` | 10.0.1.0/24 | AZ-a | Private |
| `<PRIVATE_SUBNET_2>` | 10.0.2.0/24 | AZ-b | Private |
| `<PUBLIC_SUBNET_1>` | 10.0.3.0/24 | AZ-a | Public |

### VPC Endpoints (HealthOmics) - 5 Required

Ref: https://docs.aws.amazon.com/omics/latest/dev/vpc-interface-endpoints.html

| Service | Type | Policy |
|---------|------|--------|
| com.amazonaws.`<REGION>`.workflows-omics | Interface | Account-restricted |
| com.amazonaws.`<REGION>`.storage-omics | Interface | Account-restricted |
| com.amazonaws.`<REGION>`.analytics-omics | Interface | Account-restricted |
| com.amazonaws.`<REGION>`.tags-omics | Interface | Account-restricted |
| com.amazonaws.`<REGION>`.control-storage-omics | Interface | Account-restricted |

- Security Group: `<SG_VPCE>` (HealthOmics-VPCEndpoint-SG)
  - Inbound: TCP 443 from VPC CIDR (e.g., 10.0.0.0/16)
- Private DNS enabled on all endpoints
- All endpoints deployed in both private subnets

#### VPC Endpoint Policy (Account-Restricted)

```json
{
  "Statement": [{
    "Sid": "AllowAccountOnly",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<ACCOUNT_ID>:root"},
    "Action": "*",
    "Resource": "*"
  }]
}
```

### VPC Endpoints (Other - as needed)

| Service | Type |
|---------|------|
| com.amazonaws.`<REGION>`.s3 | Gateway |
| com.amazonaws.`<REGION>`.sagemaker.api | Interface |
| com.amazonaws.`<REGION>`.sagemaker.runtime | Interface |

---

## 3. IAM Configuration

### SAML Provider

| Item | Value |
|------|-------|
| Provider Name | AzureEntraMock-Keycloak |
| Provider ARN | `arn:aws:iam::<ACCOUNT_ID>:saml-provider/AzureEntraMock-Keycloak` |
| Metadata Source | Keycloak SAML descriptor (HTTPS) |
| Assertion Encryption | **Required** (Private Key registered) |
| Signing Certificate Validity | 1 year (rotation required before expiry) |

### SAML Federated Roles

All SAML roles include `aws:SourceIp` condition restricting access to VPC CIDR.

| Role | Policy | Purpose |
|------|--------|---------|
| HealthOmics-SAML-Admin | HealthOmics-FullAccess | Full HealthOmics admin access |
| HealthOmics-SAML-Analyst | HealthOmics-Analyst | Read + StartRun access |
| HealthOmics-SAML-ReadOnly | HealthOmics-ReadOnly | Read-only access |

### SAML Trust Policy (common to all 3 roles)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:saml-provider/AzureEntraMock-Keycloak"
    },
    "Action": "sts:AssumeRoleWithSAML",
    "Condition": {
      "StringEquals": {
        "SAML:aud": "https://signin.aws.amazon.com/saml"
      },
      "IpAddress": {
        "aws:SourceIp": ["10.0.0.0/16"]
      }
    }
  }]
}
```

### Service Roles

| Role | Purpose |
|------|---------|
| HealthOmics-ServiceRole | HealthOmics service role (confused deputy prevention) |
| SageMaker-Notebook-Role | SageMaker Notebook execution role |

### IAM Policies (Role-Based Access Control)

Ref: https://docs.aws.amazon.com/omics/latest/dev/security_iam_service-with-iam.html

#### Admin Policy (HealthOmics-FullAccess)
- `omics:*` on `arn:aws:omics:<REGION>:<ACCOUNT_ID>:*`
- `omics:List*` on `*` (List actions require unscoped resource)
- S3 Read/Write for omics/genomics buckets
- KMS full access via `kms:ViaService` condition
- CloudWatch Logs full access for `/aws/omics/*`
- Inline: `AllowPassOmicsRole` - `iam:PassRole` for HealthOmics service role

#### Analyst Policy (HealthOmics-Analyst)
- `omics:Get*`, `omics:StartRun`, `omics:TagResource` on `arn:aws:omics:<REGION>:<ACCOUNT_ID>:*`
- `omics:List*` on `*` (List actions require unscoped resource)
- S3 Read-only for omics/genomics buckets
- KMS Decrypt/DescribeKey via `kms:ViaService` condition
- CloudWatch Logs read-only for `/aws/omics/*`
- Inline: `AllowPassOmicsRole` - `iam:PassRole` for HealthOmics service role

#### ReadOnly Policy (HealthOmics-ReadOnly)
- `omics:Get*` on `arn:aws:omics:<REGION>:<ACCOUNT_ID>:*`
- `omics:List*` on `*` (List actions require unscoped resource)
- S3 Read-only for omics/genomics buckets
- CloudWatch Logs read-only for `/aws/omics/*`

#### Example: Admin Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "HealthOmicsAdminAccess",
      "Effect": "Allow",
      "Action": "omics:*",
      "Resource": "arn:aws:omics:<REGION>:<ACCOUNT_ID>:*"
    },
    {
      "Sid": "HealthOmicsListAccess",
      "Effect": "Allow",
      "Action": "omics:List*",
      "Resource": "*"
    },
    {
      "Sid": "S3AccessForOmics",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": ["arn:aws:s3:::*omics*", "arn:aws:s3:::*omics*/*"]
    },
    {
      "Sid": "KMSAccessForOmics",
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:DescribeKey", "kms:Encrypt", "kms:GenerateDataKey*"],
      "Resource": "*",
      "Condition": {"StringEquals": {"kms:ViaService": "omics.<REGION>.amazonaws.com"}}
    },
    {
      "Sid": "LogsAccess",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", "logs:GetLogEvents", "logs:DescribeLogGroups", "logs:DescribeLogStreams"],
      "Resource": "arn:aws:logs:*:<ACCOUNT_ID>:log-group:/aws/omics/*"
    }
  ]
}
```

#### Example: PassRole Inline Policy (for Admin/Analyst)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowPassOmicsRunRole",
    "Effect": "Allow",
    "Action": "iam:PassRole",
    "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/<OMICS_SERVICE_ROLE>",
    "Condition": {
      "StringEquals": {"iam:PassedToService": "omics.amazonaws.com"}
    }
  }]
}
```

### Confused Deputy Prevention

Ref: https://docs.aws.amazon.com/omics/latest/dev/cross-service-confused-deputy-prevention.html

HealthOmics service role trust policy:
```json
{
  "Statement": [
    {
      "Sid": "OmicsServiceRole",
      "Effect": "Allow",
      "Principal": {"Service": "omics.amazonaws.com"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {"aws:SourceAccount": "<ACCOUNT_ID>"},
        "ArnLike": {"aws:SourceArn": "arn:aws:omics:<REGION>:<ACCOUNT_ID>:*"}
      }
    },
    {
      "Sid": "SageMakerAssumeRole",
      "Effect": "Allow",
      "Principal": {"Service": "sagemaker.amazonaws.com"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {"aws:SourceAccount": "<ACCOUNT_ID>"},
        "ArnLike": {"aws:SourceArn": "arn:aws:sagemaker:<REGION>:<ACCOUNT_ID>:*"}
      }
    }
  ]
}
```

### Data Protection

Ref: https://docs.aws.amazon.com/omics/latest/dev/data-protection.html

- Encryption at rest: AWS-owned KMS keys (default) / Customer-managed keys supported
- Encryption in transit: TLS 1.2+ (TLS 1.3 recommended)
- Workflow temp storage: XTS-AES-256 block cipher (automatic)
- SAML Assertion: Encrypted (Required mode)

---

## 4. SageMaker Notebook

| Item | Value |
|------|-------|
| Name | HealthOmics-BioEpis-Notebook |
| Instance Type | ml.t3.medium |
| Subnet | `<PRIVATE_SUBNET_1>` (Private) |
| Security Groups | `<SG_DEFAULT>`, `<SG_VPCE>` |
| Direct Internet | Disabled (VPC-only) |
| Volume | 20 GB |
| Execution Role | SageMaker-Notebook-Role |

---

## 5. Docker Management (Keycloak)

```bash
# Start (HTTPS, port 8443, HTTP disabled)
docker run -d --name keycloak \
  -p 8443:8443 \
  -v $(pwd)/keycloak-certs/keycloak-tls-cert.pem:/opt/keycloak/conf/server.crt.pem:ro \
  -v $(pwd)/keycloak-certs/keycloak-tls-key.pem:/opt/keycloak/conf/server.key.pem:ro \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -e KC_HOSTNAME=localhost \
  -e KC_HTTPS_CERTIFICATE_FILE=/opt/keycloak/conf/server.crt.pem \
  -e KC_HTTPS_CERTIFICATE_KEY_FILE=/opt/keycloak/conf/server.key.pem \
  quay.io/keycloak/keycloak:latest start --https-port=8443 --http-enabled=false

# Stop
docker stop keycloak

# Start (existing container)
docker start keycloak

# Logs
docker logs -f keycloak

# Remove
docker rm -f keycloak
```

---

## 6. Security Hardening Summary

| # | Item | Before | After |
|---|------|--------|-------|
| 1 | SAML Trust Policy | No SourceIp restriction | `aws:SourceIp: <VPC_CIDR>` on all 3 SAML roles |
| 2 | Keycloak Transport | HTTP (port 8080) | HTTPS (port 8443), HTTP disabled |
| 3 | SAML Assertion Encryption | Allowed (optional) | Required (`saml.encrypt=true`) |
| 4 | IAM Policies | `omics:*` on `*` shared by all roles | Role-based: Admin / Analyst / ReadOnly separated |
| 5 | SageMaker Confused Deputy | No condition on SageMaker trust | `aws:SourceAccount` + `aws:SourceArn` condition added |
| 6 | SAML Signing Certificate | 10-year default validity | 1-year validity, rotation required |
| 7 | VPC Endpoint Policy | Default (allow all) | Account-restricted (`Principal: <ACCOUNT_ID>:root`) |
| 8 | MFA Enforcement | ALTERNATIVE (optional) | REQUIRED (mandatory) |
| 9 | IAM Policy List* Fix | `omics:List*` on scoped ARN (broken) | `omics:List*` on `Resource: "*"` (separate Statement) |
| 10 | PassRole for SAML Roles | Not configured (StartRun blocked) | Admin/Analyst: `iam:PassRole` inline policy added |

### Production Recommendations

| # | Item | Recommendation |
|---|------|---------------|
| 1 | Test User Passwords | Enforce unique passwords per user with complexity policy (min 12 chars, special chars, no reuse) |
| 2 | MFA Enforcement | Verify end-to-end TOTP login flow works correctly |
| 3 | TLS Certificate | Replace self-signed certificate with CA-signed certificate |
| 4 | SAML Key Storage | Move keys to AWS Secrets Manager or HSM |
| 5 | Keycloak Admin | Change default credentials, restrict admin console access by IP |
| 6 | IdP Migration | Replace Keycloak Mock with Azure Entra ID (same SAML config) |
| 7 | KMS Keys | Consider customer-managed KMS keys (CMK) |
| 8 | VPC Endpoint Policy | Fine-tune to specific IAM roles/resources |

---

## 7. Actual API Permission Test Results

Each SAML role was tested via AssumeRole with actual HealthOmics API calls.

| API Action | Admin | Analyst | ReadOnly | Note |
|------------|:-----:|:-------:|:--------:|------|
| ListWorkflows | ALLOW | ALLOW | ALLOW | All roles allowed |
| ListRuns | ALLOW | ALLOW | ALLOW | All roles allowed |
| GetWorkflow | ALLOW | ALLOW | ALLOW | |
| ListSequenceStores | ALLOW | ALLOW | ALLOW | All roles allowed |
| TagResource | ALLOW | ALLOW | **DENY** | ReadOnly write blocked |
| StartRun | ALLOW | ALLOW | **DENY** | ValidationException = permission OK |
| DeleteWorkflow | ALLOW | **DENY** | **DENY** | ResourceNotFound = permission OK |
| DeleteRun | ALLOW | **DENY** | **DENY** | ResourceNotFound = permission OK |

**8 APIs x 3 Roles = 24 tests, all matched expected results (24/24 PASS)**

### Test Method
- Temporarily added `sts:AssumeRole` to each SAML role's Trust Policy for testing
- Restored to SAML-only Trust Policy after testing
- **ALLOW**: Normal response or `ValidationException`/`ResourceNotFoundException` (permission passed, parameter/resource missing)
- **DENY**: `AccessDeniedException` (IAM policy denied)

---

## 8. Test Validation Checklist

- [x] Keycloak SAML login (HTTPS) -> AWS Console access via federated role
- [x] MFA (TOTP) enforcement on Keycloak login
- [x] Role-based access: Admin/Analyst/ReadOnly mapped to correct IAM roles (IAM Simulator + actual API test 24/24)
- [x] SourceIp restriction: SAML federation denied from outside VPC CIDR
- [x] SAML assertion encryption validation (Required mode)
- [x] SageMaker Notebook -> HealthOmics SDK calls via VPC Endpoints (workflow run COMPLETED)
- [x] VPC Endpoint policy enforcement (account-restricted)
- [x] CloudTrail audit log capture for all API calls
- [x] IP whitelist validation via Security Group rules
- [x] KMS encryption verification (at rest / in transit)
- [x] Confused deputy prevention condition validation
- [x] Analyst role: can StartRun but cannot delete resources (actual API test verified)
- [x] ReadOnly role: can only Get/List, no write operations (actual API test verified)
