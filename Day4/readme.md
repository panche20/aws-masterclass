# 🔐 AWS Phase 3 — Security, Multi-Account & Advanced Networking

## PART 1 — AWS Organizations & Multi-Account Strategy

Every serious company runs multiple AWS accounts. Not multiple VPCs — multiple accounts. Account-level isolation is the strongest security boundary AWS provides.

**Why Multiple Accounts?**

```
Single Account Problems:
├── One IAM mistake = blast radius is EVERYTHING
├── Dev workloads share quota limits with prod
├── Billing is opaque — which team spent what?
├── Compliance audit scope = entire account
└── One compromised credential = all environments exposed

Multi-Account Model:
├── Blast radius is contained per account
├── Service quotas are per account
├── Cost allocation is clear
├── Audit scope is limited
└── SCPs enforce hard guardrails at org level
```

**AWS Organizations — The Structure**

```
Root (Management Account — never run workloads here)
│
├── OU: Security
│   ├── Account: Log Archive        ← all CloudTrail, Config logs flow here
│   └── Account: Security Tooling   ← GuardDuty master, Security Hub master
│
├── OU: Infrastructure
│   ├── Account: Network            ← Transit Gateway, Direct Connect, DNS
│   └── Account: Shared Services    ← ECR, artifact repos, internal tools
│
├── OU: Workloads
│   ├── OU: Production
│   │   ├── Account: prod-url-shortener
│   │   └── Account: prod-payments
│   ├── OU: Staging
│   │   └── Account: staging-url-shortener
│   └── OU: Dev
│       └── Account: dev-url-shortener
│
└── OU: Sandbox
    └── Account: engineers-playground  ← SCPs prevent production actions
```

**Hands-On: Build the Organization**

```
# Enable Organizations (run from management account)
aws organizations create-organization \
  --feature-set ALL

# Get root ID
ROOT_ID=$(aws organizations list-roots \
  --query 'Roots[0].Id' --output text)

echo "Root ID: $ROOT_ID"

# Create OUs
SECURITY_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name Security \
  --query 'OrganizationalUnit.Id' --output text)

INFRA_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name Infrastructure \
  --query 'OrganizationalUnit.Id' --output text)

WORKLOADS_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name Workloads \
  --query 'OrganizationalUnit.Id' --output text)

PROD_OU=$(aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU \
  --name Production \
  --query 'OrganizationalUnit.Id' --output text)

DEV_OU=$(aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU \
  --name Development \
  --query 'OrganizationalUnit.Id' --output text)

SANDBOX_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name Sandbox \
  --query 'OrganizationalUnit.Id' --output text)

# Create member accounts
aws organizations create-account \
  --email prod-url-shortener@yourcompany.com \
  --account-name "prod-url-shortener" \
  --iam-user-access-to-billing ALLOW \
  --role-name OrganizationAccountAccessRole

aws organizations create-account \
  --email log-archive@yourcompany.com \
  --account-name "log-archive" \
  --iam-user-access-to-billing DENY \
  --role-name OrganizationAccountAccessRole

# List all accounts
aws organizations list-accounts \
  --query 'Accounts[].[Name,Id,Status,Email]' \
  --output table

# Move account to correct OU
PROD_ACCOUNT_ID=$(aws organizations list-accounts \
  --query 'Accounts[?Name==`prod-url-shortener`].Id' \
  --output text)

aws organizations move-account \
  --account-id $PROD_ACCOUNT_ID \
  --source-parent-id $ROOT_ID \
  --destination-parent-id $PROD_OU

# Assume role into member account (cross-account access)
aws sts assume-role \
  --role-arn "arn:aws:iam::${PROD_ACCOUNT_ID}:role/OrganizationAccountAccessRole" \
  --role-session-name "chetan-prod-access" \
  --duration-seconds 3600

# Export creds and work in prod account
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

aws sts get-caller-identity  # verify you're in prod account
```

**Service Control Policies (SCPs) — The Hardest Guardrails**

SCPs are IAM policies attached to OUs/accounts. They define the maximum permissions any identity in that account can have — even root.

```
# Enable SCP policy type
aws organizations enable-policy-type \
  --root-id $ROOT_ID \
  --policy-type SERVICE_CONTROL_POLICY

# ── SCP 1: Deny everything outside approved regions ───────
cat > scp-region-restriction.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "route53:*",
        "budgets:*",
        "waf:*",
        "cloudfront:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotIn": {
          "aws:RequestedRegion": [
            "ap-south-1",
            "eu-west-1",
            "eu-central-1"
          ]
        }
      }
    }
  ]
}
EOF

aws organizations create-policy \
  --name DenyNonApprovedRegions \
  --description "Restrict all workloads to approved regions only" \
  --type SERVICE_CONTROL_POLICY \
  --content file://scp-region-restriction.json

# ── SCP 2: Prevent leaving the organization ───────────────
cat > scp-prevent-leave.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeaveOrganization",
      "Effect": "Deny",
      "Action": ["organizations:LeaveOrganization"],
      "Resource": "*"
    }
  ]
}
EOF

# ── SCP 3: Deny root user actions ─────────────────────────
cat > scp-deny-root.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUser",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
EOF

# ── SCP 4: Require MFA for sensitive actions ──────────────
cat > scp-require-mfa.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyHighRiskWithoutMFA",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeleteUser",
        "iam:CreateAccessKey",
        "iam:AttachRolePolicy",
        "iam:DeleteRolePolicy",
        "organizations:*",
        "ec2:DeleteVpc",
        "s3:DeleteBucket",
        "rds:DeleteDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
EOF

# ── SCP 5: Sandbox — no production services ───────────────
cat > scp-sandbox.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyExpensiveServicesInSandbox",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotLike": {
          "ec2:InstanceType": [
            "t3.*",
            "t3a.*",
            "t2.*"
          ]
        }
      }
    },
    {
      "Sid": "DenyRDSProductionClasses",
      "Effect": "Deny",
      "Action": ["rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "rds:DatabaseClass": "db.t3.*"
        }
      }
    }
  ]
}
EOF

# Create and attach all SCPs
for SCP_FILE in scp-region-restriction scp-prevent-leave scp-deny-root scp-require-mfa; do
  POLICY_ID=$(aws organizations create-policy \
    --name "${SCP_FILE}" \
    --description "${SCP_FILE} guardrail" \
    --type SERVICE_CONTROL_POLICY \
    --content file://${SCP_FILE}.json \
    --query 'Policy.PolicySummary.Id' --output text)

  # Attach to root (applies to all accounts)
  aws organizations attach-policy \
    --policy-id $POLICY_ID \
    --target-id $ROOT_ID

  echo "Attached: $SCP_FILE ($POLICY_ID)"
done

# Attach sandbox SCP specifically to sandbox OU
SANDBOX_SCP_ID=$(aws organizations create-policy \
  --name scp-sandbox \
  --description "Sandbox account restrictions" \
  --type SERVICE_CONTROL_POLICY \
  --content file://scp-sandbox.json \
  --query 'Policy.PolicySummary.Id' --output text)

aws organizations attach-policy \
  --policy-id $SANDBOX_SCP_ID \
  --target-id $SANDBOX_OU
```

**AWS Control Tower (Production Multi-Account Bootstrap)**

Control Tower automates org setup with landing zones. Know this for architect interviews.

```
# Control Tower is console-driven for initial setup.
# But you manage Account Factory via CLI:

# Provision a new account via Account Factory
aws servicecatalog provision-product \
  --product-name "AWS Control Tower Account Factory" \
  --provisioning-artifact-name "AWS Control Tower Account Factory" \
  --provisioned-product-name "prod-payments" \
  --provisioning-parameters \
    Key=AccountName,Value=prod-payments \
    Key=AccountEmail,Value=prod-payments@yourcompany.com \
    Key=SSOUserEmail,Value=admin@yourcompany.com \
    Key=SSOUserFirstName,Value=Admin \
    Key=SSOUserLastName,Value=User \
    Key=ManagedOrganizationalUnit,Value="Production (ou-xxxx-yyyy)"

# Control Tower automatically:
# - Creates the account
# - Applies guardrails (SCPs)
# - Sets up CloudTrail in log archive account
# - Configures AWS Config in all accounts
# - Sets up SSO access
```

## PART 2 — IAM Advanced (IRSA, Permission Boundaries, Identity Center)

**Permission Boundaries — Advanced IAM Control**

A permission boundary is a managed policy that sets the **maximum permissions** a role/user can ever have — even if someone attaches AdministratorAccess to them.

```
# Scenario: Allow devs to create their own roles,
# but those roles can NEVER exceed developer permissions.
# Without boundary → dev creates admin role → privilege escalation.
# With boundary → any role a dev creates is capped.

cat > dev-permission-boundary.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServices",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "logs:*",
        "cloudwatch:*",
        "ec2:Describe*",
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyPrivilegeEscalation",
      "Effect": "Deny",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy",
        "iam:CreateUser",
        "iam:AttachUserPolicy",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowIAMOnlyWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:PutRolePolicy",
        "iam:AttachRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::ACCOUNT_ID:policy/dev-permission-boundary"
        }
      }
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name dev-permission-boundary \
  --policy-document file://dev-permission-boundary.json

# Create a developer role WITH the boundary
aws iam create-role \
  --role-name developer-role \
  --assume-role-policy-document file://dev-trust.json \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/dev-permission-boundary

# Even if you attach AdministratorAccess, the boundary limits it
aws iam attach-role-policy \
  --role-name developer-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
# ^ This attaches — but effective permissions are STILL limited by the boundary
```

**IAM Identity Center (SSO) — Enterprise Authentication**

```
# IAM Identity Center replaces individual IAM users.
# One login → access to all AWS accounts via SSO.

# Enable Identity Center (console-driven first time)
# Then manage via CLI:

# List instances
aws sso-admin list-instances \
  --query 'Instances[0].{InstanceArn:InstanceArn,IdentityStoreId:IdentityStoreId}'

INSTANCE_ARN="arn:aws:sso:::instance/ssoins-xxxxxxxxx"
IDENTITY_STORE_ID="d-xxxxxxxxxx"

# Create a group in Identity Center
GROUP_ID=$(aws identitystore create-group \
  --identity-store-id $IDENTITY_STORE_ID \
  --display-name "DevOps-Engineers" \
  --description "DevOps team with prod access" \
  --query 'GroupId' --output text)

# Create a user
USER_ID=$(aws identitystore create-user \
  --identity-store-id $IDENTITY_STORE_ID \
  --user-name chetan \
  --name Formatted=Chetan,GivenName=Chetan,FamilyName=DevOps \
  --emails Value=chetan@company.com,Type=work,Primary=true \
  --query 'UserId' --output text)

# Add user to group
aws identitystore create-group-membership \
  --identity-store-id $IDENTITY_STORE_ID \
  --group-id $GROUP_ID \
  --member-id UserId=$USER_ID

# Create Permission Set (what access does this group have in each account)
PERMSET_ARN=$(aws sso-admin create-permission-set \
  --instance-arn $INSTANCE_ARN \
  --name "DevOpsEngineerAccess" \
  --description "Full DevOps access" \
  --session-duration PT8H \
  --query 'PermissionSet.PermissionSetArn' --output text)

# Attach managed policy to permission set
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn $INSTANCE_ARN \
  --permission-set-arn $PERMSET_ARN \
  --managed-policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# Inline policy for specific restrictions
aws sso-admin put-inline-policy-to-permission-set \
  --instance-arn $INSTANCE_ARN \
  --permission-set-arn $PERMSET_ARN \
  --inline-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Deny",
      "Action": ["organizations:*","account:*"],
      "Resource": "*"
    }]
  }'

# Assign permission set to account + group
aws sso-admin create-account-assignment \
  --instance-arn $INSTANCE_ARN \
  --target-id $PROD_ACCOUNT_ID \
  --target-type AWS_ACCOUNT \
  --permission-set-arn $PERMSET_ARN \
  --principal-type GROUP \
  --principal-id $GROUP_ID

# Now Chetan can log in at:
# https://your-org.awsapps.com/start
# And assume DevOpsEngineerAccess in prod-url-shortener account
# No IAM users. No access keys. SSO tokens only.
```

**Cross-Account Role Assumption Pattern**

```
# The standard pattern for tools (Terraform, CI/CD) accessing multiple accounts:

# In each member account — create a role that management account can assume
cat > cross-account-trust.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${MANAGEMENT_ACCOUNT_ID}:role/cicd-role"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "url-shortener-cicd-secret"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name cross-account-deploy-role \
  --assume-role-policy-document file://cross-account-trust.json

aws iam attach-role-policy \
  --role-name cross-account-deploy-role \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# In CodePipeline/CodeBuild — assume role per account
# This is how a single pipeline deploys to staging AND production accounts
aws sts assume-role \
  --role-arn "arn:aws:iam::${PROD_ACCOUNT_ID}:role/cross-account-deploy-role" \
  --role-session-name "pipeline-prod-deploy" \
  --external-id "url-shortener-cicd-secret"
```

## PART 3 — GuardDuty, Security Hub, Inspector & Macie

This is the detection layer. Your SCPs and IAM are preventive. These services are detective.

**GuardDuty — Threat Detection**

GuardDuty analyzes CloudTrail, VPC Flow Logs, DNS logs, and EKS audit logs using ML to detect threats.

```
# Enable GuardDuty (enable in EVERY account, in EVERY region you use)
aws guardduty create-detector \
  --enable \
  --data-sources '{
    "S3Logs": {"Enable": true},
    "Kubernetes": {"AuditLogs": {"Enable": true}},
    "MalwareProtection": {
      "ScanEc2InstanceWithFindings": {"EbsVolumes": true}
    }
  }' \
  --finding-publishing-frequency FIFTEEN_MINUTES

DETECTOR_ID=$(aws guardduty list-detectors \
  --query 'DetectorIds[0]' --output text)

echo "GuardDuty Detector: $DETECTOR_ID"

# Enable GuardDuty as organization-wide (from management account)
aws guardduty create-organization-admin-account \
  --admin-account-id $SECURITY_ACCOUNT_ID

# From security account — auto-enable for all org members
aws guardduty update-organization-configuration \
  --detector-id $DETECTOR_ID \
  --auto-enable-organization-members ALL \
  --data-sources '{
    "S3Logs": {"AutoEnable": true},
    "Kubernetes": {"AuditLogs": {"AutoEnable": true}}
  }'

# List findings
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "severity": {"Gte": 7},
      "service.archived": {"Eq": ["false"]}
    }
  }' \
  --sort-criteria '{"AttributeName":"severity","OrderBy":"DESC"}'

# Get finding details
FINDING_ID="abc123..."
aws guardduty get-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids $FINDING_ID \
  --query 'Findings[0].{Type:Type,Severity:Severity,Description:Description,Region:Region}'

# Archive false positives (suppress known-good behavior)
aws guardduty create-filter \
  --detector-id $DETECTOR_ID \
  --name "suppress-codebuild-findings" \
  --action ARCHIVE \
  --finding-criteria '{
    "Criterion": {
      "service.userFeedback": {"Eq": ["NOT_USEFUL"]},
      "resource.instanceDetails.tags.key": {"Eq": ["aws:cloudformation:stack-name"]},
      "resource.instanceDetails.tags.value": {"Eq": ["codebuild"]}
    }
  }'

# Automate response with EventBridge → Lambda
cat > guardduty-response.py << 'EOF'
import boto3
import json

def handler(event, context):
    """
    Auto-respond to GuardDuty findings:
    - Severity >= 7 (High): isolate the instance
    - Severity >= 4 (Medium): notify + snapshot
    """
    finding = event['detail']
    severity = finding['severity']
    finding_type = finding['type']

    ec2 = boto3.client('ec2')
    sns = boto3.client('sns')
    ALERT_TOPIC = "arn:aws:sns:ap-south-1:ACCOUNT_ID:security-alerts"

    # Extract instance ID if finding involves EC2
    instance_id = None
    resource = finding.get('resource', {})
    if resource.get('resourceType') == 'Instance':
        instance_id = resource['instanceDetails']['instanceId']

    if severity >= 7.0 and instance_id:
        print(f"HIGH severity finding: {finding_type}. Isolating {instance_id}")

        # Create forensic snapshot before isolation
        volumes = ec2.describe_volumes(
            Filters=[{'Name': 'attachment.instance-id', 'Values': [instance_id]}]
        )['Volumes']

        for vol in volumes:
            ec2.create_snapshot(
                VolumeId=vol['VolumeId'],
                Description=f"GuardDuty-forensic-{finding['id']}"
            )

        # Isolate: apply empty security group
        QUARANTINE_SG = "sg-quarantine-id"
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=[QUARANTINE_SG]
        )

        # Notify security team
        sns.publish(
            TopicArn=ALERT_TOPIC,
            Subject=f"🚨 CRITICAL: Instance {instance_id} ISOLATED",
            Message=json.dumps({
                'finding': finding_type,
                'severity': severity,
                'instance': instance_id,
                'action': 'ISOLATED - quarantine SG applied + forensic snapshot taken'
            }, indent=2)
        )

    elif severity >= 4.0:
        sns.publish(
            TopicArn=ALERT_TOPIC,
            Subject=f"⚠️ MEDIUM GuardDuty: {finding_type}",
            Message=json.dumps(finding, indent=2, default=str)
        )

    return {'statusCode': 200}
EOF
```

**Security Hub — Centralized Security Posture**

```
# Enable Security Hub with all standards
aws securityhub enable-security-hub \
  --enable-default-standards \
  --control-finding-generator SECURITY_CONTROL

# Enable specific standards
aws securityhub batch-enable-standards \
  --standards-subscription-requests \
    StandardsArn=arn:aws:securityhub:ap-south-1::standards/aws-foundational-security-best-practices/v/1.0.0 \
    StandardsArn=arn:aws:securityhub:ap-south-1::standards/cis-aws-foundations-benchmark/v/1.4.0 \
    StandardsArn=arn:aws:securityhub:ap-south-1::standards/pci-dss/v/3.2.1

# Organization-wide Security Hub (from security account)
aws securityhub create-finding-aggregator \
  --linking-mode ALL_REGIONS

# Get security score
aws securityhub get-insights \
  --insight-arns \
    "arn:aws:securityhub:::insight/securityhub/default/1" \
  --query 'Insights[0].{Name:Name,Filters:Filters}'

# Get failed controls
aws securityhub get-findings \
  --filters '{
    "ComplianceStatus": [{"Value": "FAILED", "Comparison": "EQUALS"}],
    "WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}],
    "SeverityLabel": [
      {"Value": "CRITICAL", "Comparison": "EQUALS"},
      {"Value": "HIGH", "Comparison": "EQUALS"}
    ]
  }' \
  --sort-criteria \
    Field=SeverityLabel,SortOrder=asc \
  --query 'Findings[].[Title,SeverityLabel,ProductName,ResourcesId]' \
  --output table

# Suppress a known false positive
aws securityhub batch-update-findings \
  --finding-identifiers Id=finding-id,ProductArn=product-arn \
  --workflow Status=SUPPRESSED \
  --note '{"Text":"Known false positive - internal scanner", "UpdatedBy":"chetan"}'
```

**Amazon Inspector v2 — Vulnerability Management**

```
# Enable Inspector across org
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA LAMBDA_CODE \
  --account-ids $PROD_ACCOUNT_ID $STAGING_ACCOUNT_ID

# Get critical findings for ECR images
aws inspector2 list-findings \
  --filter-criteria '{
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_ECR_CONTAINER_IMAGE"}],
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}],
    "fixAvailable": [{"comparison": "EQUALS", "value": "YES"}]
  }' \
  --query 'findings[].[title,severity,packageVulnerabilityDetails.vulnerabilityId,resourcesAffected[0].awsEcrContainerImage.repositoryName]' \
  --output table

# Get SBOM (software bill of materials) for an image
aws inspector2 create-sbom-export \
  --report-format CYCLONEDX_1_4 \
  --s3-destination bucketName=$ARTIFACT_BUCKET,keyPrefix=sbom/ \
  --filter-criteria '{
    "ecrImageRepositoryName": [{
      "comparison": "EQUALS",
      "value": "url-shortener/api"
    }]
  }'
```

**Amazon Macie — S3 Data Security**

```
# Enable Macie (discovers PII in S3)
aws macie2 enable-macie

# Create a classification job to scan for sensitive data
aws macie2 create-classification-job \
  --job-type SCHEDULED \
  --name "url-shortener-pii-scan" \
  --schedule-frequency '{"weeklySchedule": {"dayOfWeek": "MONDAY"}}' \
  --s3-job-definition '{
    "bucketDefinitions": [{
      "accountId": "'$ACCOUNT_ID'",
      "buckets": ["url-shortener-assets-'$ACCOUNT_ID'"]
    }],
    "scoping": {
      "includes": {
        "and": [{
          "simpleScopeTerm": {
            "comparator": "EQ",
            "key": "OBJECT_SIZE",
            "values": ["1024"]
          }
        }]
      }
    }
  }' \
  --managed-data-identifier-selector ALL

# Get findings (PII discovered)
aws macie2 list-findings \
  --finding-criteria '{
    "criterion": {
      "severity.score": {
        "gte": 50
      }
    }
  }'
```

## PART 4 — AWS WAF, Shield & Network Security

**WAF (Web Application Firewall)**

```
# Create WAF Web ACL for ALB
WAF_ACL=$(aws wafv2 create-web-acl \
  --name url-shortener-waf \
  --scope REGIONAL \
  --region $AWS_REGION \
  --default-action Allow={} \
  --description "WAF for url-shortener ALB" \
  --visibility-config \
    SampledRequestsEnabled=true,\
    CloudWatchMetricsEnabled=true,\
    MetricName=url-shortener-waf \
  --rules '[
    {
      "Name": "AWS-AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet",
          "ExcludedRules": []
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSet"
      }
    },
    {
      "Name": "AWS-AWSManagedRulesKnownBadInputsRuleSet",
      "Priority": 2,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesKnownBadInputsRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "KnownBadInputs"
      }
    },
    {
      "Name": "AWS-AWSManagedRulesSQLiRuleSet",
      "Priority": 3,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesSQLiRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "SQLiRuleSet"
      }
    },
    {
      "Name": "RateLimitPerIP",
      "Priority": 10,
      "Action": {"Block": {}},
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP",
          "ScopeDownStatement": {
            "ByteMatchStatement": {
              "SearchString": "/api/",
              "FieldToMatch": {"UriPath": {}},
              "TextTransformations": [{"Priority": 0, "Type": "LOWERCASE"}],
              "PositionalConstraint": "STARTS_WITH"
            }
          }
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimitPerIP"
      }
    },
    {
      "Name": "BlockBadBots",
      "Priority": 5,
      "Action": {"Block": {}},
      "Statement": {
        "ByteMatchStatement": {
          "SearchString": "python-requests",
          "FieldToMatch": {
            "SingleHeader": {"Name": "user-agent"}
          },
          "TextTransformations": [{"Priority": 0, "Type": "LOWERCASE"}],
          "PositionalConstraint": "CONTAINS"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "BlockBadBots"
      }
    },
    {
      "Name": "GeoBlockHighRisk",
      "Priority": 7,
      "Action": {"Block": {}},
      "Statement": {
        "GeoMatchStatement": {
          "CountryCodes": ["KP", "IR", "CU", "SY"]
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "GeoBlock"
      }
    }
  ]' \
  --query 'Summary.ARN' --output text)

echo "WAF ACL: $WAF_ACL"

# Associate WAF with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn $WAF_ACL \
  --resource-arn $ALB_ARN \
  --region $AWS_REGION

# Enable WAF logging to S3
aws wafv2 put-logging-configuration \
  --logging-configuration \
    ResourceArn=$WAF_ACL,\
    LogDestinationConfigs=arn:aws:s3:::$ARTIFACT_BUCKET,\
    RedactedFields=[]

# Enable AWS Shield Advanced (for DDoS protection)
# NOTE: Shield Advanced costs $3000/month — know it conceptually
aws shield create-subscription   # enables Shield Advanced

aws shield create-protection \
  --name url-shortener-alb-protection \
  --resource-arn $ALB_ARN

aws shield create-protection \
  --name url-shortener-cloudfront-protection \
  --resource-arn "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/EDFDVBD6EXAMPLE"
```

## PART 5 — Advanced VPC (Transit Gateway, PrivateLink, VPC Peering)

**VPC Peering vs Transit Gateway**

```
VPC PEERING:
  VPC-A ←──────────────────→ VPC-B
  VPC-A ←──────────────────→ VPC-C
  VPC-B ←──────────────────→ VPC-C
  
  Problem: N*(N-1)/2 peering connections.
  10 VPCs = 45 peering connections. Not scalable.
  Also: NOT TRANSITIVE. A→B and B→C does NOT mean A→C.

TRANSIT GATEWAY:
  VPC-A ──┐
  VPC-B ──┤
  VPC-C ──┼── Transit Gateway ── On-Premises (via VPN/Direct Connect)
  VPC-D ──┤
  VPC-E ──┘
  
  Hub-and-spoke. Fully transitive. Scales to thousands of VPCs.
  Cross-account. Cross-region (peered TGWs).
```

**Hands-On: Transit Gateway**

```
# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
  --description "Central hub for url-shortener multi-VPC" \
  --options '{
    "AmazonSideAsn": 64512,
    "AutoAcceptSharedAttachments": "disable",
    "DefaultRouteTableAssociation": "enable",
    "DefaultRouteTablePropagation": "enable",
    "VpnEcmpSupport": "enable",
    "DnsSupport": "enable",
    "MulticastSupport": "disable"
  }' \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=url-shortener-tgw}]' \
  --query 'TransitGateway.TransitGatewayId' \
  --output text)

echo "Transit Gateway: $TGW_ID"

# Wait for TGW to be available
aws ec2 wait transit-gateway-available \
  --transit-gateway-ids $TGW_ID

# Attach production VPC
TGW_ATTACH_PROD=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $VPC_ID \
  --subnet-ids $PRIV_SUBNET_A $PRIV_SUBNET_B \
  --options "DnsSupport=enable,Ipv6Support=disable" \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=prod-vpc-attachment}]' \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
  --output text)

# Share Transit Gateway across org accounts
aws ram create-resource-share \
  --name tgw-org-share \
  --resource-arns "arn:aws:ec2:${AWS_REGION}:${ACCOUNT_ID}:transit-gateway/${TGW_ID}" \
  --principals "arn:aws:organizations::${MANAGEMENT_ACCOUNT_ID}:organization/o-xxxxxxxxxxxx" \
  --allow-external-principals false

# Add routes to private route tables pointing to TGW
# (other VPC CIDRs go through Transit Gateway)
aws ec2 create-route \
  --route-table-id $PRIV_RT \
  --destination-cidr-block 10.1.0.0/16 \  # staging VPC CIDR
  --transit-gateway-id $TGW_ID

aws ec2 create-route \
  --route-table-id $PRIV_RT \
  --destination-cidr-block 10.2.0.0/16 \  # shared services VPC CIDR
  --transit-gateway-id $TGW_ID

# Create separate route table for segmentation (prod cannot reach dev)
TGW_PROD_RT=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=prod-rt}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
  --output text)

# Associate prod attachment with prod route table
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id $TGW_PROD_RT \
  --transit-gateway-attachment-id $TGW_ATTACH_PROD

# Only propagate routes from shared services — NOT from dev
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $TGW_PROD_RT \
  --transit-gateway-attachment-id $TGW_ATTACH_SHARED_SERVICES

# Verify TGW route table
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id $TGW_PROD_RT \
  --filters Name=state,Values=active \
  --query 'Routes[].[DestinationCidrBlock,TransitGatewayAttachments[0].ResourceId,Type]' \
  --output table
```

**AWS PrivateLink — Private Service Exposure**

PrivateLink lets you expose a service to other VPCs or accounts **without peering, without internet**. Traffic never leaves AWS backbone.

```
# ── Create PrivateLink Endpoint Service ───────────────────
# Your url-shortener API can be consumed by other teams' VPCs
# privately, without VPC peering

# 1. You need an NLB in front of your service first
NLB_ARN=$(aws elbv2 create-load-balancer \
  --name url-shortener-nlb-private \
  --subnets $PRIV_SUBNET_A $PRIV_SUBNET_B \
  --scheme internal \
  --type network \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# 2. Create endpoint service backed by the NLB
ENDPOINT_SERVICE=$(aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns $NLB_ARN \
  --acceptance-required \
  --private-dns-name api-internal.yourdomain.com \
  --tag-specifications 'ResourceType=vpc-endpoint-service,Tags=[{Key=Name,Value=url-shortener-privatelink}]' \
  --query 'ServiceConfiguration.ServiceId' \
  --output text)

# 3. Allow specific accounts to use this service
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id $ENDPOINT_SERVICE \
  --add-allowed-principals \
    "arn:aws:iam::${CONSUMER_ACCOUNT_ID}:root"

# ── Consumer side (in another account/VPC) ────────────────
# Create an interface endpoint to connect to your service
INTERFACE_ENDPOINT=$(aws ec2 create-vpc-endpoint \
  --vpc-endpoint-type Interface \
  --service-name "com.amazonaws.vpce.ap-south-1.${ENDPOINT_SERVICE}" \
  --vpc-id $CONSUMER_VPC_ID \
  --subnet-ids $CONSUMER_PRIV_SUBNET \
  --security-group-ids $CONSUMER_SG \
  --private-dns-enabled \
  --query 'VpcEndpoint.VpcEndpointId' \
  --output text)

# Now the consumer can reach your API at the private DNS name
# without peering, internet gateway, or NAT
# Traffic stays entirely within AWS network
```

**VPC Flow Logs — Network Forensics**

```
# Enable Flow Logs for the entire VPC
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::${ACCOUNT_ID}:role/flow-logs-role \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${windowstart} ${windowend} ${action} ${flowDirection} ${traffic-path}'

# Also send to S3 for long-term retention and Athena querying
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination "arn:aws:s3:::${ARTIFACT_BUCKET}/vpc-flow-logs/" \
  --destination-options \
    FileFormat=parquet,HiveCompatiblePartitions=true,PerHourPartition=true

# Query Flow Logs with Athena (create the table first)
cat > create-flow-logs-table.sql << 'EOF'
CREATE EXTERNAL TABLE vpc_flow_logs (
  version int,
  account_id string,
  interface_id string,
  srcaddr string,
  dstaddr string,
  srcport int,
  dstport int,
  protocol bigint,
  packets bigint,
  bytes bigint,
  windowstart bigint,
  windowend bigint,
  action string,
  flowdirection string
)
PARTITIONED BY (dt string)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ' '
LOCATION 's3://url-shortener-artifacts-ACCOUNT_ID/vpc-flow-logs/'
TBLPROPERTIES ("skip.header.line.count"="1");
EOF

# Find top talkers (who's sending most traffic)
cat > top-talkers.sql << 'EOF'
SELECT srcaddr, dstaddr, sum(bytes) as total_bytes,
       sum(packets) as total_packets, action
FROM vpc_flow_logs
WHERE dt = '2024-01-15'
  AND action = 'REJECT'
GROUP BY srcaddr, dstaddr, action
ORDER BY total_bytes DESC
LIMIT 20;
EOF

# Detect port scanning (many different dstports from same src)
cat > port-scan-detection.sql << 'EOF'
SELECT srcaddr, COUNT(DISTINCT dstport) as unique_ports,
       COUNT(*) as connection_attempts
FROM vpc_flow_logs
WHERE dt = '2024-01-15'
  AND action = 'REJECT'
  AND windowend - windowstart < 60
GROUP BY srcaddr
HAVING COUNT(DISTINCT dstport) > 100
ORDER BY unique_ports DESC;
EOF
```

## PART 6 — CloudTrail, AWS Config & Compliance Automation

**CloudTrail — Complete API Audit Trail**

```
# Create organization trail (logs ALL accounts in one S3 bucket)
aws cloudtrail create-trail \
  --name org-wide-audit-trail \
  --s3-bucket-name log-archive-bucket \
  --is-multi-region-trail \
  --is-organization-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:ap-south-1:${ACCOUNT_ID}:alias/cloudtrail-key \
  --cloud-watch-logs-log-group-arn \
    "arn:aws:logs:ap-south-1:${ACCOUNT_ID}:log-group:/cloudtrail/org" \
  --cloud-watch-logs-role-arn \
    "arn:aws:iam::${ACCOUNT_ID}:role/cloudtrail-cloudwatch-role"

# Enable the trail
aws cloudtrail start-logging --name org-wide-audit-trail

# Enable data events (S3 object-level + Lambda invocations)
aws cloudtrail put-event-selectors \
  --trail-name org-wide-audit-trail \
  --advanced-event-selectors '[
    {
      "Name": "S3-data-events",
      "FieldSelectors": [
        {"Field": "eventCategory", "Equals": ["Data"]},
        {"Field": "resources.type", "Equals": ["AWS::S3::Object"]},
        {"Field": "resources.ARN", "StartsWith": [
          "arn:aws:s3:::url-shortener-"
        ]}
      ]
    },
    {
      "Name": "Lambda-invocations",
      "FieldSelectors": [
        {"Field": "eventCategory", "Equals": ["Data"]},
        {"Field": "resources.type", "Equals": ["AWS::Lambda::Function"]}
      ]
    }
  ]'

# Query CloudTrail with Athena — find who deleted an S3 bucket
cat > find-deletion.sql << 'EOF'
SELECT eventtime, useridentity.arn, sourceipaddress, useragent, requestparameters
FROM cloudtrail_logs
WHERE eventsource = 's3.amazonaws.com'
  AND eventname IN ('DeleteBucket', 'DeleteObject', 'PutBucketPolicy')
  AND eventtime > '2024-01-01'
ORDER BY eventtime DESC
LIMIT 50;
EOF

# Detect root login (should never happen)
cat > detect-root-login.sql << 'EOF'
SELECT eventtime, sourceipaddress, useragent, responseelements
FROM cloudtrail_logs
WHERE useridentity.type = 'Root'
  AND eventname = 'ConsoleLogin'
ORDER BY eventtime DESC;
EOF

# Detect IAM privilege escalation attempts
cat > detect-privesc.sql << 'EOF'
SELECT eventtime, useridentity.arn, eventname, requestparameters
FROM cloudtrail_logs
WHERE eventname IN (
  'AttachRolePolicy', 'PutRolePolicy', 'CreateRole',
  'AddUserToGroup', 'AttachUserPolicy', 'CreateAccessKey'
)
AND useridentity.type != 'AWSService'
ORDER BY eventtime DESC;
EOF
```

**AWS Config — Resource Compliance**

```
# Enable Config (records every resource change)
aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::'$ACCOUNT_ID':role/config-role",
    "recordingGroup": {
      "allSupported": true,
      "includeGlobalResourceTypes": true
    }
  }'

aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "log-archive-bucket",
    "s3KeyPrefix": "aws-config",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }'

aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# Enable Config Rules (compliance checks)

# Rule 1: EC2 instances must use IMDSv2
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ec2-imdsv2-required",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "EC2_IMDSV2_REQUIRED"
    }
  }'

# Rule 2: S3 buckets must block public access
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-access-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_ACCESS_PROHIBITED"
    }
  }'

# Rule 3: RDS must be encrypted
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "rds-storage-encrypted",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "RDS_STORAGE_ENCRYPTED"
    }
  }'

# Rule 4: EKS cluster logging enabled (custom rule via Lambda)
cat > eks-logging-check.py << 'EOF'
import boto3
import json

def evaluate_compliance(config_item, rule_params):
    eks = boto3.client('eks')

    if config_item['resourceType'] != 'AWS::EKS::Cluster':
        return 'NOT_APPLICABLE'

    cluster_name = config_item['resourceId']

    try:
        cluster = eks.describe_cluster(name=cluster_name)['cluster']
        logging_config = cluster.get('logging', {}).get('clusterLogging', [])

        enabled_types = []
        for log_setup in logging_config:
            if log_setup.get('enabled'):
                enabled_types.extend(log_setup.get('types', []))

        required = {'api', 'audit', 'authenticator', 'controllerManager', 'scheduler'}

        if required.issubset(set(enabled_types)):
            return {
                'complianceType': 'COMPLIANT',
                'annotation': 'All logging types enabled'
            }
        else:
            missing = required - set(enabled_types)
            return {
                'complianceType': 'NON_COMPLIANT',
                'annotation': f'Missing log types: {missing}'
            }
    except Exception as e:
        return {
            'complianceType': 'NON_COMPLIANT',
            'annotation': str(e)
        }

def handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    config_item = invoking_event['configurationItem']
    rule_params = json.loads(event.get('ruleParameters', '{}'))

    result = evaluate_compliance(config_item, rule_params)

    config = boto3.client('config')
    config.put_evaluations(
        Evaluations=[{
            'ComplianceResourceType': config_item['resourceType'],
            'ComplianceResourceId': config_item['resourceId'],
            'ComplianceType': result['complianceType'],
            'Annotation': result.get('annotation', ''),
            'OrderingTimestamp': config_item['configurationItemCaptureTime']
        }],
        ResultToken=event['resultToken']
    )
EOF

# Auto-remediate non-compliant resources with SSM Automation
aws configservice put-remediation-configurations \
  --remediation-configurations '[{
    "ConfigRuleName": "s3-bucket-public-access-prohibited",
    "TargetType": "SSM_DOCUMENT",
    "TargetId": "AWS-DisableS3BucketPublicReadWrite",
    "Parameters": {
      "AutomationAssumeRole": {
        "StaticValue": {
          "Values": ["arn:aws:iam::'$ACCOUNT_ID':role/config-remediation-role"]
        }
      },
      "S3BucketName": {
        "ResourceValue": {"Value": "RESOURCE_ID"}
      }
    },
    "Automatic": true,
    "MaximumAutomaticAttempts": 3,
    "RetryAttemptSeconds": 60
  }]'

# Get compliance summary
aws configservice describe-compliance-by-config-rule \
  --query 'ComplianceByConfigRules[].[ConfigRuleName,Compliance.ComplianceType]' \
  --output table

# Get non-compliant resources for a rule
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-access-prohibited \
  --compliance-types NON_COMPLIANT \
  --query 'EvaluationResults[].[EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId,ComplianceType]' \
  --output table
```

## PART 7 — KMS, Secrets Manager & Parameter Store

**KMS — Key Management Deep Dive**

```
# Create Customer Managed Key (CMK)
CMK_ARN=$(aws kms create-key \
  --description "url-shortener production encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "Enable root account",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::'$ACCOUNT_ID':root"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "Allow ECS tasks to use this key",
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::'$ACCOUNT_ID':role/ecs-task-execution-role"
        },
        "Action": ["kms:Decrypt","kms:DescribeKey"],
        "Resource": "*"
      },
      {
        "Sid": "Allow CloudTrail to encrypt logs",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": ["kms:GenerateDataKey*","kms:DescribeKey"],
        "Resource": "*"
      },
      {
        "Sid": "Allow Key Administrators",
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::'$ACCOUNT_ID':role/SecurityTeamRole"
        },
        "Action": [
          "kms:Create*","kms:Describe*","kms:Enable*","kms:List*",
          "kms:Put*","kms:Update*","kms:Revoke*","kms:Disable*",
          "kms:Get*","kms:Delete*","kms:ScheduleKeyDeletion","kms:CancelKeyDeletion"
        ],
        "Resource": "*"
      }
    ]
  }' \
  --query 'KeyMetadata.KeyId' --output text)

# Create alias
aws kms create-alias \
  --alias-name alias/url-shortener-prod \
  --target-key-id $CMK_ARN

# Enable automatic rotation (every year)
aws kms enable-key-rotation --key-id $CMK_ARN

# Encrypt a secret manually
PLAINTEXT="my-secret-value"
ENCRYPTED=$(aws kms encrypt \
  --key-id alias/url-shortener-prod \
  --plaintext "$PLAINTEXT" \
  --query 'CiphertextBlob' --output text)

# Decrypt
aws kms decrypt \
  --ciphertext-blob $ENCRYPTED \
  --query 'Plaintext' --output text | base64 --decode

# Grant access to another role (without modifying key policy)
aws kms create-grant \
  --key-id $CMK_ARN \
  --grantee-principal "arn:aws:iam::${ACCOUNT_ID}:role/eks-url-shortener-role" \
  --operations Decrypt DescribeKey \
  --name "eks-app-grant" \
  --constraints EncryptionContextSubset={Application=url-shortener}
```

**Secrets Manager — Production Pattern**

```
# Create secrets with rotation
aws secretsmanager create-secret \
  --name /url-shortener/prod/rds-credentials \
  --description "RDS PostgreSQL credentials" \
  --secret-string '{
    "username": "devopsadmin",
    "password": "InitialPass123!",
    "engine": "postgres",
    "host": "devops-postgres.xxx.ap-south-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "urlshortener"
  }' \
  --kms-key-id alias/url-shortener-prod \
  --tags Key=Project,Value=url-shortener Key=Environment,Value=production

# Enable automatic rotation (every 30 days using AWS-managed Lambda)
aws secretsmanager rotate-secret \
  --secret-id /url-shortener/prod/rds-credentials \
  --rotation-lambda-arn \
    arn:aws:lambda:ap-south-1:${ACCOUNT_ID}:function:SecretsManagerRDSPostgreSQLRotationSingleUser \
  --rotation-rules AutomaticallyAfterDays=30

# Cross-account secret access
aws secretsmanager put-resource-policy \
  --secret-id /url-shortener/prod/rds-credentials \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::'$STAGING_ACCOUNT_ID':role/staging-app-role"
      },
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "secretsmanager:VersionStage": "AWSCURRENT"
        }
      }
    }]
  }'

# Retrieve in application code
aws secretsmanager get-secret-value \
  --secret-id /url-shortener/prod/rds-credentials \
  --query 'SecretString' --output text | python3 -m json.tool

# Parameter Store — for non-sensitive config (free for Standard tier)
aws ssm put-parameter \
  --name /url-shortener/prod/config/log-level \
  --value INFO \
  --type String \
  --tier Standard

# For sensitive values — SecureString (encrypted with KMS)
aws ssm put-parameter \
  --name /url-shortener/prod/api-key \
  --value "sk-live-xxxxxxxxxxxx" \
  --type SecureString \
  --key-id alias/url-shortener-prod \
  --tier Advanced

# Get with decryption
aws ssm get-parameter \
  --name /url-shortener/prod/api-key \
  --with-decryption \
  --query 'Parameter.Value' --output text

# Get all params in a path (hierarchy)
aws ssm get-parameters-by-path \
  --path /url-shortener/prod/ \
  --recursive \
  --with-decryption \
  --query 'Parameters[].[Name,Type,Value]' \
  --output table

# Secrets Manager vs Parameter Store — know the difference
# Secrets Manager: rotation, cross-account, $0.40/secret/month
# Parameter Store Standard: no rotation, free up to 10k params
# Parameter Store Advanced: larger values, higher throughput, $0.05/param/month
# Rule: Passwords → Secrets Manager. Config → Parameter Store.
```

## PART 8 — Cost Optimization & Governance

**AWS Budgets + Cost Anomaly Detection**

```
# Create budget with alerts
aws budgets create-budget \
  --account-id $ACCOUNT_ID \
  --budget '{
    "BudgetName": "url-shortener-monthly",
    "BudgetLimit": {
      "Amount": "500",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "TagKeyValue": ["user:Project$url-shortener"]
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{
        "SubscriptionType": "EMAIL",
        "Address": "chetan@company.com"
      },{
        "SubscriptionType": "SNS",
        "Address": "arn:aws:sns:ap-south-1:'$ACCOUNT_ID':cost-alerts"
      }]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{
        "SubscriptionType": "EMAIL",
        "Address": "chetan@company.com"
      }]
    }
  ]'

# Cost Anomaly Detection (ML-based)
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "url-shortener-anomaly-monitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }' \
  --query 'MonitorArn' --output text

aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "url-shortener-anomaly-alerts",
    "MonitorArnList": ["arn:aws:ce::ACCOUNT_ID:anomalymonitor/xxx"],
    "Subscribers": [{
      "Address": "arn:aws:sns:ap-south-1:ACCOUNT_ID:cost-alerts",
      "Type": "SNS"
    }],
    "Threshold": 50,
    "Frequency": "DAILY"
  }'

# Tag-based cost allocation
# ALL resources must be tagged. Enforce with Config rule:
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "InputParameters": "{\"tag1Key\":\"Project\",\"tag2Key\":\"Environment\",\"tag3Key\":\"Owner\"}"
  }'

# Enable Cost Explorer and get breakdown
aws ce get-cost-and-usage \
  --time-period Start=$(date -d "30 days ago" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost UnblendedCost UsageQuantity \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[].[Keys[0],Metrics.BlendedCost.Amount]' \
  --output table | sort -k2 -nr | head -10
```

## Phase 3 — Complete Security Architecture

```
AWS Organization (Management Account)
├── SCPs: region lock, deny root, require MFA, sandbox limits
├── Control Tower: landing zone, guardrails, account vending
└── IAM Identity Center: SSO → all accounts, no IAM users

Security Account
├── GuardDuty master → all accounts → EventBridge → Lambda (auto-isolate)
├── Security Hub → aggregate findings → CRITICAL/HIGH → PagerDuty
├── CloudTrail org trail → S3 (log-archive) → Athena queries
└── Config aggregator → compliance dashboard

Network Account
├── Transit Gateway → hub for all VPCs (prod, staging, shared services)
│   └── Segmented route tables (prod cannot reach dev)
├── PrivateLink → expose services without internet
└── VPC Flow Logs → S3 (parquet) → Athena (forensics)

Production Account (prod-url-shortener)
├── WAF → ALB (OWASP rules, rate limiting, geo-block, SQLi/XSS)
├── Shield Advanced → DDoS protection on ALB + CloudFront
├── KMS CMK → encrypts EBS, RDS, S3, Secrets Manager, CloudTrail
├── Secrets Manager → RDS creds (auto-rotate every 30 days)
├── Parameter Store → non-sensitive config hierarchy
├── Inspector v2 → ECR image scanning + EC2 CVE detection
├── Macie → PII discovery in S3
└── Config Rules → auto-remediate non-compliant resources
```

## Phase 3 Interview Cheat Sheet

<img width="916" height="448" alt="image" src="https://github.com/user-attachments/assets/50df55c5-b8a4-4bd8-ba7a-13ecc6065aa1" />

<img width="923" height="382" alt="image" src="https://github.com/user-attachments/assets/55970012-940a-4a09-b47b-50b1dc5ad341" />






