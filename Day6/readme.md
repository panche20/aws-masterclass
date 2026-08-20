# 🏛️ AWS Phase 5 — Cloud Architect Capstone

## PART 1 — AWS Well-Architected Framework (All 6 Pillars)

The Well-Architected Framework is AWS's codified body of knowledge for building production systems. Every architect interview tests this. Every AWS review uses this. Know it cold.

**The 6 Pillars**

```
1. Operational Excellence   — run and improve systems
2. Security                 — protect data and systems
3. Reliability              — recover from failures
4. Performance Efficiency   — use resources efficiently
5. Cost Optimization        — eliminate waste
6. Sustainability           — minimize environmental impact
```

## Pillar 1 — Operational Excellence

```
Core principle: Treat operations as code. Everything automated, nothing manual.

Key practices:
├── Infrastructure as Code (CloudFormation / Terraform)
├── Annotated runbooks in code (not Wiki pages nobody reads)
├── Make frequent, small, reversible changes
├── Anticipate failure — game days, chaos engineering
├── Learn from failures — blameless post-mortems
└── Measure everything — define business and technical KPIs
```

```
# Operational Excellence hands-on: Runbook Automation with SSM

# Create an SSM Automation runbook for incident response
cat > runbook-high-cpu.yaml << 'EOF'
description: "Automated response to high CPU on EC2 instances"
schemaVersion: "0.3"
assumeRole: "{{AutomationAssumeRole}}"

parameters:
  InstanceId:
    type: String
    description: "EC2 instance with high CPU"
  AutomationAssumeRole:
    type: String
    description: "IAM role for automation"

mainSteps:
  - name: CollectDiagnostics
    action: aws:runCommand
    inputs:
      DocumentName: AWS-RunShellScript
      InstanceIds: ["{{InstanceId}}"]
      Parameters:
        commands:
          - "top -bn1 | head -20"
          - "ps aux --sort=-%cpu | head -20"
          - "free -m"
          - "df -h"
          - "netstat -tuln | head -20"
    outputs:
      - Name: diagnostics
        Selector: $.Output
        Type: String

  - name: TakeMemorySnapshot
    action: aws:createSnapshot
    inputs:
      VolumeId: "{{InstanceId}}"
      Description: "Pre-remediation snapshot {{global:DATE}}"

  - name: CheckAutoScaling
    action: aws:executeAwsApi
    inputs:
      Service: autoscaling
      Api: DescribeAutoScalingInstances
      InstanceIds: ["{{InstanceId}}"]
    outputs:
      - Name: asgName
        Selector: $.AutoScalingInstances[0].AutoScalingGroupName
        Type: String

  - name: TriggerScaleOut
    action: aws:executeAwsApi
    inputs:
      Service: autoscaling
      Api: SetDesiredCapacity
      AutoScalingGroupName: "{{CollectDiagnostics.asgName}}"
      DesiredCapacity: 0   # fetched dynamically
      HonorCooldown: false

  - name: NotifyOpsTeam
    action: aws:publishSNS
    inputs:
      TopicArn: "arn:aws:sns:ap-south-1:ACCOUNT_ID:ops-alerts"
      Message: |
        High CPU Auto-Remediation Complete
        Instance: {{InstanceId}}
        Diagnostics: {{CollectDiagnostics.diagnostics}}
        Action: Scale-out triggered
EOF

aws ssm create-document \
  --name HighCPURemediation \
  --content file://runbook-high-cpu.yaml \
  --document-type Automation \
  --document-format YAML

# Trigger runbook from CloudWatch Alarm action
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions \
    "arn:aws:ssm:ap-south-1:${ACCOUNT_ID}:automation-definition/HighCPURemediation"
```

## Pillar 2 — Security

You mastered this in Phase 3. The architect-level additions:

```
Defense in depth model:
┌─────────────────────────────────────────────────────┐
│  Edge: CloudFront + WAF + Shield                    │
│  ┌───────────────────────────────────────────────┐  │
│  │  Network: VPC + NACLs + Security Groups       │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  Identity: IAM + SCPs + SSO             │  │  │
│  │  │  ┌───────────────────────────────────┐  │  │  │
│  │  │  │  Application: Input validation    │  │  │  │
│  │  │  │  ┌───────────────────────────┐    │  │  │  │
│  │  │  │  │  Data: KMS + Macie + TLS  │    │  │  │  │
│  │  │  │  └───────────────────────────┘    │  │  │  │
│  │  │  └───────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

Detective controls running continuously:
├── GuardDuty (threat detection)
├── Security Hub (posture scoring)
├── CloudTrail (API audit)
├── Config (compliance drift)
└── Inspector + Macie (vuln + PII)
```

## Pillar 3 — Reliability

This is the most technically complex pillar. Every senior/architect interview goes here.

```
Reliability hierarchy:
├── Foundations        (quotas, network topology)
├── Workload architecture (distributed system design)
├── Change management  (deploys without downtime)
└── Failure management (detect, respond, recover)

Key metrics:
├── RTO — Recovery Time Objective
│         How long can you be down? (1 min? 4 hours? 24 hours?)
└── RPO — Recovery Point Objective
          How much data can you lose? (0 seconds? 1 hour? 1 day?)
```

```
# Reliability hands-on: Multi-AZ health validation

# Check your current multi-AZ coverage
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=url-shortener" \
  --query 'Reservations[].Instances[].[InstanceId,Placement.AvailabilityZone,State.Name]' \
  --output table

# Verify RDS Multi-AZ
aws rds describe-db-instances \
  --db-instance-identifier devops-postgres \
  --query 'DBInstances[0].{MultiAZ:MultiAZ,Engine:Engine,Status:DBInstanceStatus,AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone}'

# Simulate AZ failure — terminate all instances in one AZ
# (DO THIS IN STAGING ONLY)
AZ_TO_SIMULATE="ap-south-1a"

INSTANCES_IN_AZ=$(aws ec2 describe-instances \
  --filters \
    "Name=placement-availability-zone,Values=${AZ_TO_SIMULATE}" \
    "Name=tag:Environment,Values=staging" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text)

echo "Would terminate: $INSTANCES_IN_AZ"
# aws ec2 terminate-instances --instance-ids $INSTANCES_IN_AZ

# AWS Resilience Hub — formal resilience assessment
aws resiliencehub create-app \
  --name url-shortener \
  --description "URL Shortener resilience assessment" \
  --policy-arn arn:aws:resiliencehub:${AWS_REGION}:${ACCOUNT_ID}:resiliency-policy/url-shortener-policy

aws resiliencehub import-resources-to-draft-app-version \
  --app-arn arn:aws:resiliencehub:${AWS_REGION}:${ACCOUNT_ID}:app/url-shortener \
  --source-arns \
    "arn:aws:cloudformation:${AWS_REGION}:${ACCOUNT_ID}:stack/url-shortener-prod"

aws resiliencehub resolve-app-version-resources \
  --app-arn arn:aws:resiliencehub:${AWS_REGION}:${ACCOUNT_ID}:app/url-shortener \
  --app-version draft

aws resiliencehub create-app-version-assessment \
  --app-arn arn:aws:resiliencehub:${AWS_REGION}:${ACCOUNT_ID}:app/url-shortener \
  --app-version release
```

## Pillar 4 — Performance Efficiency

```
Four areas:
├── Selection    — right resource type for the workload
├── Review       — keep up with new AWS service launches
├── Monitoring   — measure actual performance, not assumed
└── Trade-offs   — consistency vs latency, cost vs speed

Key patterns:
├── Caching hierarchy:
│   CloudFront (edge) → ElastiCache (regional) → DAX (DDB) → Application cache
│
├── Read vs write separation:
│   ALB → Read replicas (SELECT queries)
│       → Primary (INSERT/UPDATE/DELETE)
│
└── Async over sync:
    Sync: User waits for entire operation
    Async: User gets immediate ack, work happens in background
```

```
# Performance benchmarking
# Install k6 for load testing
sudo apt-get install -y gnupg
sudo gpg --dearmor -o /usr/share/keyrings/k6-archive-keyring.gpg \
  https://dl.k6.io/key.gpg
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] \
  https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6

# Load test script
cat > load-test.js << 'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const redirectLatency = new Trend('redirect_latency');

export const options = {
  stages: [
    { duration: '2m', target: 100  },  // ramp up
    { duration: '5m', target: 1000 },  // sustain peak
    { duration: '2m', target: 5000 },  // spike
    { duration: '3m', target: 1000 },  // recovery
    { duration: '2m', target: 0    },  // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // 99th percentile < 500ms
    errors: ['rate<0.01'],              // error rate < 1%
    redirect_latency: ['p(95)<200'],    // redirects < 200ms
  },
};

const BASE_URL = 'https://api.yourdomain.com';

export default function() {
  // Test redirect (hot path)
  const redirectStart = Date.now();
  const redirectRes = http.get(`${BASE_URL}/abc123`, {
    redirects: 0,  // don't follow redirect — just measure 301
    tags: { name: 'redirect' },
  });
  redirectLatency.add(Date.now() - redirectStart);

  check(redirectRes, {
    'redirect status 301': (r) => r.status === 301,
    'has location header': (r) => r.headers['Location'] !== undefined,
  }) || errorRate.add(1);

  sleep(0.1);

  // Test URL creation
  const createRes = http.post(
    `${BASE_URL}/urls`,
    JSON.stringify({ url: 'https://google.com', ttl_days: 7 }),
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
      },
      tags: { name: 'create' },
    }
  );

  check(createRes, {
    'create status 201': (r) => r.status === 201,
    'has short_url': (r) => JSON.parse(r.body).short_url !== undefined,
  }) || errorRate.add(1);

  sleep(1);
}
EOF

# Run the test
k6 run --env JWT_TOKEN=your-test-token load-test.js

# CloudWatch Performance Insights for RDS
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-XXXXXXXXXXXXXXX \
  --metric-queries \
    '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.sql","Limit":5}}]' \
  --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period-in-seconds 60
```

## Pillar 5 — Cost Optimization

```
Cost optimization layers:
├── Right-sizing    — use what you actually need
├── Pricing models  — on-demand vs reserved vs spot vs savings plans
├── Storage tiers   — S3 intelligent tiering, EBS gp3 vs io2
├── Waste elimination — idle resources, unattached EBS, old snapshots
└── Architecture    — serverless eliminates idle compute cost entirely

Cost hierarchy (most impactful first):
1. Architecture decisions (Lambda vs EC2 = 10x difference)
2. Purchase options (Savings Plans = 40-60% off)
3. Right-sizing (oversized instances)
4. Storage optimization (S3 lifecycle, EBS types)
5. Data transfer (same-region, same-AZ = free)
```

```
# Comprehensive cost analysis
# Find idle resources (running but doing nothing)
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,LaunchTime,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Find unattached EBS volumes (paying for nothing)
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[].[VolumeId,Size,VolumeType,CreateTime]' \
  --output table

# Find old snapshots (> 90 days)
aws ec2 describe-snapshots \
  --owner-ids self \
  --query "Snapshots[?StartTime<='$(date -d '90 days ago' +%Y-%m-%d)'].[SnapshotId,VolumeSize,StartTime,Description]" \
  --output table

# Find unused Elastic IPs (costs $0.005/hr when unattached)
aws ec2 describe-addresses \
  --query 'Addresses[?!InstanceId].[PublicIp,AllocationId]' \
  --output table

# Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].[instanceArn,finding,recommendationOptions[0].instanceType,recommendationOptions[0].estimatedMonthlySavings.value]' \
  --output table

# Savings Plans analysis
aws savingsplans describe-savings-plans \
  --query 'savingsPlans[].[savingsPlanType,committedAmount,currency,status,end]' \
  --output table

# Cost Explorer: top services this month
aws ce get-cost-and-usage \
  --time-period \
    Start=$(date -d "$(date +%Y-%m-01)" +%Y-%m-%d),\
    End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups' \
  --output table | sort -k4 -rn | head -15

# Auto-delete idle resources with Lambda
cat > cost-janitor.py << 'EOF'
import boto3
from datetime import datetime, timezone, timedelta

ec2 = boto3.client('ec2')
cloudwatch = boto3.client('cloudwatch')

def handler(event, context):
    """Weekly janitor: find and report idle resources."""
    findings = []
    now = datetime.now(timezone.utc)

    # Find instances with < 5% CPU for 7 days
    instances = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )['Reservations']

    for reservation in instances:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']

            # Get average CPU for last 7 days
            response = cloudwatch.get_metric_statistics(
                Namespace='AWS/EC2',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                StartTime=now - timedelta(days=7),
                EndTime=now,
                Period=604800,
                Statistics=['Average']
            )

            if response['Datapoints']:
                avg_cpu = response['Datapoints'][0]['Average']
                if avg_cpu < 5.0:
                    name = next(
                        (t['Value'] for t in instance.get('Tags', [])
                         if t['Key'] == 'Name'), 'unnamed'
                    )
                    findings.append({
                        'type': 'idle_ec2',
                        'id': instance_id,
                        'name': name,
                        'avg_cpu': avg_cpu,
                        'instance_type': instance['InstanceType']
                    })

    # Find unattached EBS volumes
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'status', 'Values': ['available']}]
    )['Volumes']

    for vol in volumes:
        days_idle = (now - vol['CreateTime']).days
        if days_idle > 7:
            findings.append({
                'type': 'unattached_ebs',
                'id': vol['VolumeId'],
                'size_gb': vol['Size'],
                'days_idle': days_idle,
                'monthly_cost': vol['Size'] * 0.08
            })

    # Report findings
    if findings:
        sns = boto3.client('sns')
        sns.publish(
            TopicArn=os.environ['COST_ALERTS_TOPIC'],
            Subject=f'Cost Janitor: {len(findings)} idle resources found',
            Message=json.dumps(findings, indent=2, default=str)
        )

    return {'findings': len(findings)}
EOF
```

## Pillar 6 — Sustainability

```
Key practices:
├── Right-size to eliminate idle capacity
├── Use managed services (AWS optimizes the hardware)
├── Graviton (arm64) processors — 60% better perf/watt
├── Choose regions with renewable energy
│   (eu-west-1 Ireland, eu-north-1 Stockholm — renewable heavy)
└── Spot instances — use spare capacity, reduce waste
```

## PART 2 — Disaster Recovery Strategies

This is the single most important architect topic. Know all four strategies, their RTO/RPO, and when to use each.

**The Four DR Strategies**

```
STRATEGY 1: BACKUP AND RESTORE
  RPO: Hours       RTO: Hours
  Cost: $          Complexity: Low
  
  What: Regular backups to S3. Restore from scratch on failure.
  Architecture:
    Primary Region                    DR Region
    ├── RDS → automated backup ──────→ S3 (cross-region replication)
    ├── EC2 AMI ─────────────────────→ AMI copied to DR region
    ├── S3 → versioning + replication→ S3 replica bucket
    └── CloudFormation template ─────→ redeploy everything from scratch
  
  Use for: Dev/test, non-critical internal tools

─────────────────────────────────────────────────────────────

STRATEGY 2: PILOT LIGHT
  RPO: Minutes     RTO: 10-30 minutes
  Cost: $$         Complexity: Medium
  
  What: Core data layer always running in DR. Compute turned off.
        On failure: scale up compute, point DNS to DR.
  Architecture:
    Primary Region                    DR Region
    ├── Full app stack (running)      ├── RDS replica (running, standby)
    ├── RDS primary (running)    ──→  ├── ElastiCache (stopped/min)
    ├── ECS service (running)         ├── ECS service (0 tasks — scale up)
    └── ALB (active)                  └── ALB (exists, no targets)
  
  Use for: Applications with moderate criticality

─────────────────────────────────────────────────────────────

STRATEGY 3: WARM STANDBY
  RPO: Seconds     RTO: Minutes
  Cost: $$$        Complexity: High
  
  What: Scaled-down version running in DR at all times.
        On failure: scale up to full capacity, shift DNS.
  Architecture:
    Primary Region                    DR Region
    ├── ECS: 10 tasks (full)     ──→  ├── ECS: 2 tasks (minimal, live)
    ├── RDS Multi-AZ (full)      ──→  ├── RDS read replica (promotes to primary)
    ├── ElastiCache 3 nodes      ──→  ├── ElastiCache 1 node
    └── Route53 primary               └── Route53 secondary (failover routing)
  
  Use for: Business-critical applications

─────────────────────────────────────────────────────────────

STRATEGY 4: MULTI-SITE ACTIVE-ACTIVE
  RPO: Near zero   RTO: Near zero (seconds)
  Cost: $$$$       Complexity: Very High
  
  What: Full production in 2+ regions simultaneously.
        Traffic split across regions. Immediate failover.
  Architecture:
    ap-south-1 (active)               eu-west-1 (active)
    ├── ECS: 10 tasks            ──→  ├── ECS: 10 tasks
    ├── Aurora Global primary    ──→  ├── Aurora Global replica
    ├── ElastiCache              ──→  ├── ElastiCache
    └── Route53 latency routing  ──→  └── Route53 latency routing
    
    Data replication: Aurora Global (<1s replication lag)
    Conflict resolution: last-write-wins or application-level
  
  Use for: Financial services, healthcare, tier-1 SaaS
```

**Hands-On: Pilot Light DR Implementation**

```
export PRIMARY_REGION=ap-south-1
export DR_REGION=eu-west-1

# ── Step 1: S3 Cross-Region Replication ──────────────────
aws s3api put-bucket-replication \
  --bucket url-shortener-assets-${ACCOUNT_ID} \
  --replication-configuration '{
    "Role": "arn:aws:iam::'$ACCOUNT_ID':role/s3-replication-role",
    "Rules": [{
      "ID": "replicate-all",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Destination": {
        "Bucket": "arn:aws:s3:::url-shortener-assets-dr-'$ACCOUNT_ID'",
        "ReplicaKmsKeyID": "arn:aws:kms:'$DR_REGION':'$ACCOUNT_ID':alias/url-shortener-dr",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {"Minutes": 15}
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {"Minutes": 15}
        }
      },
      "DeleteMarkerReplication": {"Status": "Enabled"}
    }]
  }'

# ── Step 2: RDS Cross-Region Read Replica ────────────────
aws rds create-db-instance-read-replica \
  --db-instance-identifier devops-postgres-dr \
  --source-db-instance-identifier \
    arn:aws:rds:${PRIMARY_REGION}:${ACCOUNT_ID}:db:devops-postgres \
  --db-instance-class db.t3.medium \
  --region $DR_REGION \
  --kms-key-id arn:aws:kms:${DR_REGION}:${ACCOUNT_ID}:alias/url-shortener-dr \
  --publicly-accessible false \
  --tags Key=Environment,Value=dr Key=Role,Value=standby

# Monitor replication lag
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-DR-Replication-Lag" \
  --metric-name ReplicaLag \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=devops-postgres-dr \
  --statistic Average \
  --period 60 \
  --threshold 30 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:${DR_REGION}:${ACCOUNT_ID}:dr-alerts \
  --region $DR_REGION

# ── Step 3: Copy AMIs to DR region ────────────────────────
SOURCE_AMI="ami-0f58b397bc5c1f2e8"

DR_AMI=$(aws ec2 copy-image \
  --source-image-id $SOURCE_AMI \
  --source-region $PRIMARY_REGION \
  --region $DR_REGION \
  --name "url-shortener-app-dr-$(date +%Y%m%d)" \
  --encrypted \
  --kms-key-id arn:aws:kms:${DR_REGION}:${ACCOUNT_ID}:alias/url-shortener-dr \
  --query 'ImageId' --output text)

echo "DR AMI: $DR_AMI"

# ── Step 4: ECR Cross-Region Replication ─────────────────
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [{
        "region": "'$DR_REGION'",
        "registryId": "'$ACCOUNT_ID'"
      }],
      "repositoryFilters": [{
        "filter": "url-shortener/",
        "filterType": "PREFIX_MATCH"
      }]
    }]
  }' \
  --region $PRIMARY_REGION

# ── Step 5: Pre-deploy CloudFormation in DR (pilot light) ─
# Deploy minimal stack — VPC, subnets, security groups, ALB
# ECS service at 0 tasks, RDS in standby mode
aws cloudformation deploy \
  --template-file url-shortener-stack.yaml \
  --stack-name url-shortener-dr \
  --parameter-overrides \
    Environment=dr \
    DesiredCount=0 \
    AppVersion=latest \
    DBInstanceClass=db.t3.micro \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $DR_REGION

# ── Step 6: Route53 Failover Setup ────────────────────────
# Primary health check (ap-south-1)
PRIMARY_HC=$(aws route53 create-health-check \
  --caller-reference $(date +%s)-primary \
  --health-check-config '{
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "api-primary.yourdomain.com",
    "Port": 443,
    "RequestInterval": 10,
    "FailureThreshold": 2,
    "Regions": ["ap-southeast-1", "eu-west-1", "us-east-1"]
  }' \
  --query 'HealthCheck.Id' --output text)

# Failover routing (primary → secondary on health check failure)
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "primary-ap-south-1",
          "Failover": "PRIMARY",
          "HealthCheckId": "'$PRIMARY_HC'",
          "AliasTarget": {
            "HostedZoneId": "ZP97RAFLXTNZK",
            "DNSName": "url-shortener-alb.ap-south-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "secondary-eu-west-1",
          "Failover": "SECONDARY",
          "AliasTarget": {
            "HostedZoneId": "Z32O12XQLNTSW2",
            "DNSName": "url-shortener-alb-dr.eu-west-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

**DR Failover Runbook (Pilot Light)**

```
#!/bin/bash
# dr-failover.sh — execute when primary region fails
# RTO target: 15 minutes

set -euo pipefail

DR_REGION="eu-west-1"
CLUSTER="url-shortener-dr"
SERVICE="url-shortener-api"

echo "=== DR FAILOVER INITIATED: $(date) ==="
echo "Target region: $DR_REGION"

# Step 1: Promote RDS read replica to primary (3-5 min)
echo "Step 1: Promoting RDS replica..."
aws rds promote-read-replica \
  --db-instance-identifier devops-postgres-dr \
  --region $DR_REGION

aws rds wait db-instance-available \
  --db-instance-identifier devops-postgres-dr \
  --region $DR_REGION

echo "RDS promoted ✓"

# Step 2: Update Secrets Manager in DR region
DR_RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier devops-postgres-dr \
  --region $DR_REGION \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

aws secretsmanager update-secret \
  --secret-id /url-shortener/prod/db-password \
  --secret-string '{"host":"'$DR_RDS_ENDPOINT'","username":"devopsadmin","password":"SecurePass123!"}' \
  --region $DR_REGION

echo "Secrets updated ✓"

# Step 3: Scale up ECS to full capacity
echo "Step 3: Scaling up ECS service..."
aws ecs update-service \
  --cluster $CLUSTER \
  --service $SERVICE \
  --desired-count 10 \
  --region $DR_REGION

aws ecs wait services-stable \
  --cluster $CLUSTER \
  --services $SERVICE \
  --region $DR_REGION

echo "ECS scaled to 10 tasks ✓"

# Step 4: Run smoke tests
echo "Step 4: Running smoke tests..."
DR_ALB=$(aws elbv2 describe-load-balancers \
  --names url-shortener-alb-dr \
  --region $DR_REGION \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

for i in {1..5}; do
  STATUS=$(curl -so /dev/null -w "%{http_code}" \
    --max-time 10 "https://${DR_ALB}/health")
  echo "  Health check $i: HTTP $STATUS"
  [ "$STATUS" = "200" ] || { echo "HEALTH CHECK FAILED"; exit 1; }
  sleep 2
done

echo "Smoke tests passed ✓"

# Step 5: Route53 has already failed over automatically
# (health check detected failure, switched to secondary)
echo "Step 5: Verifying DNS failover..."
RESOLVED_IP=$(dig +short api.yourdomain.com | head -1)
echo "  api.yourdomain.com resolves to: $RESOLVED_IP"

# Step 6: Notify stakeholders
aws sns publish \
  --topic-arn arn:aws:sns:${DR_REGION}:${ACCOUNT_ID}:executive-alerts \
  --subject "DR FAILOVER COMPLETE — eu-west-1 is now serving production" \
  --message "Failover completed at $(date). RTO: $SECONDS seconds. All systems operational in eu-west-1." \
  --region $DR_REGION

echo "=== FAILOVER COMPLETE: $(date) Total time: ${SECONDS}s ==="
```

## PART 3 — Aurora Global Database (Active-Active)

Aurora Global is the production-grade active-active data layer. Sub-second replication across regions.

```
# Create Aurora Global Cluster
aws rds create-global-cluster \
  --global-cluster-identifier url-shortener-global \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --storage-encrypted \
  --deletion-protection

# Create primary cluster in ap-south-1
aws rds create-db-cluster \
  --db-cluster-identifier url-shortener-primary \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username devopsadmin \
  --master-user-password "SecurePass123!" \
  --global-cluster-identifier url-shortener-global \
  --db-subnet-group-name devops-db-subnets \
  --vpc-security-group-ids $RDS_SG \
  --storage-encrypted \
  --kms-key-id alias/url-shortener-prod \
  --deletion-protection \
  --enable-cloudwatch-logs-exports postgresql \
  --region $PRIMARY_REGION

# Create primary instance
aws rds create-db-instance \
  --db-instance-identifier url-shortener-primary-1 \
  --db-cluster-identifier url-shortener-primary \
  --db-instance-class db.r6g.xlarge \
  --engine aurora-postgresql \
  --region $PRIMARY_REGION

# Add secondary region (eu-west-1) to global cluster
aws rds create-db-cluster \
  --db-cluster-identifier url-shortener-secondary \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --global-cluster-identifier url-shortener-global \
  --db-subnet-group-name devops-db-subnets-dr \
  --vpc-security-group-ids $DR_RDS_SG \
  --storage-encrypted \
  --kms-key-id alias/url-shortener-dr \
  --region $DR_REGION

aws rds create-db-instance \
  --db-instance-identifier url-shortener-secondary-1 \
  --db-cluster-identifier url-shortener-secondary \
  --db-instance-class db.r6g.xlarge \
  --engine aurora-postgresql \
  --region $DR_REGION

# Check global cluster replication lag
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraGlobalDBReplicationLag \
  --dimensions Name=DBClusterIdentifier,Value=url-shortener-secondary \
  --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average \
  --region $DR_REGION

# Managed planned failover (no data loss — RPO = 0)
aws rds failover-global-cluster \
  --global-cluster-identifier url-shortener-global \
  --target-db-cluster-identifier \
    arn:aws:rds:${DR_REGION}:${ACCOUNT_ID}:cluster:url-shortener-secondary
```

## PART 4 — The 6 R's of Migration

Every cloud architect must know migration strategies. This comes up in every principal-level interview.

```
THE 6 R's — Migration Strategy Selection

R1: REHOST (Lift and Shift)
    What:  Move VMs as-is to EC2. No code changes.
    Speed: Fast (days-weeks)
    Savings: ~20-30% (same architecture, just on AWS)
    Tools: AWS Application Migration Service (MGN)
    Use for: Legacy apps, tight deadlines, large VM estates
    Example: Move 500 on-prem VMs to EC2 in 3 months

R2: REPLATFORM (Lift, Tinker, and Shift)
    What:  Minor optimizations without core code changes.
    Speed: Medium (weeks-months)
    Savings: 40-60%
    Examples:
      ├── Oracle DB → RDS PostgreSQL (managed, no patching)
      ├── Tomcat on EC2 → Elastic Beanstalk
      ├── Self-managed Redis → ElastiCache
      └── On-prem MySQL → Aurora
    Use for: Apps where managed services give quick wins

R3: REPURCHASE (Drop and Shop)
    What:  Abandon existing app, buy SaaS replacement.
    Speed: Fast (weeks, but change management is hard)
    Examples:
      ├── Custom CRM → Salesforce
      ├── Self-hosted JIRA → Atlassian Cloud
      └── Self-hosted email → Google Workspace
    Use for: Commodity software where build vs buy favors buy

R4: REFACTOR / RE-ARCHITECT
    What:  Redesign the application using cloud-native services.
    Speed: Slow (months-years)
    Savings: 60-80%
    Examples:
      ├── Monolith → Microservices on ECS/EKS
      ├── Cron jobs → Lambda + EventBridge
      ├── FTP file processing → S3 events + Lambda
      └── Batch jobs → Fargate Spot
    Use for: Apps with scalability problems, high operational cost
    Risk: Highest — most code changes, most testing

R5: RETIRE
    What:  Decommission. Just turn it off.
    Speed: Immediate
    Savings: 100% of that workload
    Reality: 10-20% of on-prem estate is actually unused
    Tools: AWS Migration Evaluator identifies candidates

R6: RETAIN (Revisit)
    What:  Keep on-prem for now. Revisit later.
    When:  Recently upgraded hardware, compliance constraints,
           latency requirements, or not worth the effort yet
    Examples:
      ├── Mainframe workloads (migration risk too high)
      ├── Heavily licensed software (Oracle licensing nightmares)
      └── Edge computing requirements
```

**Migration Assessment Hands-On**

```
# AWS Migration Evaluator — analyze on-prem estate
# (Typically involves installing a collector agent on-prem)
# CLI for managing assessments:

aws migrationhub-config create-home-region-control \
  --home-region ap-south-1 \
  --target '{"Type":"ACCOUNT"}'

# Application Discovery Service — auto-discover on-prem
aws discovery start-data-collection-by-agent-ids \
  --agent-ids agent-id-1 agent-id-2

# Get discovered servers
aws discovery describe-agents \
  --filters Name=hostName,Values=prod-app-server-01 \
  --query 'agentsInfo[].[agentId,hostName,agentState,health]' \
  --output table

# AWS Application Migration Service (MGN) — Rehost
# Initialize MGN in account
aws mgn initialize-service

# Create replication configuration template
aws mgn create-replication-configuration-template \
  --staging-area-subnet-id $PRIV_SUBNET_A \
  --replication-server-instance-type t3.small \
  --use-dedicated-replication-server false \
  --default-large-staging-disk-type GP3 \
  --replication-servers-security-groups-i-ds $REPLICATION_SG \
  --bandwidth-throttling 0

# After agent install on source server — launch test instance
aws mgn start-test \
  --source-server-ids s-xxxxxxxxxxxxxxxxx

# Validate test instance, then:
aws mgn start-cutover \
  --source-server-ids s-xxxxxxxxxxxxxxxxx

# Database Migration Service (DMS) — Replatform
# Migrate Oracle to Aurora PostgreSQL

# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier url-shortener-dms \
  --replication-instance-class dms.r5.xlarge \
  --allocated-storage 100 \
  --vpc-security-group-ids $DMS_SG \
  --replication-subnet-group-identifier devops-db-subnets \
  --multi-az \
  --publicly-accessible false

# Create source endpoint (Oracle on-prem)
aws dms create-endpoint \
  --endpoint-identifier oracle-source \
  --endpoint-type source \
  --engine-name oracle \
  --username admin \
  --password "SourcePass" \
  --server-name on-prem-oracle.internal \
  --port 1521 \
  --database-name ORCL \
  --oracle-settings '{
    "UseLogminerReader": true,
    "ReadAheadBlocks": 1000
  }'

# Create target endpoint (Aurora PostgreSQL)
aws dms create-endpoint \
  --endpoint-identifier aurora-target \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --username devopsadmin \
  --password "TargetPass" \
  --server-name url-shortener-primary.cluster.ap-south-1.rds.amazonaws.com \
  --port 5432 \
  --database-name urlshortener

# Create migration task (full load + CDC for zero-downtime)
aws dms create-replication-task \
  --replication-task-identifier oracle-to-aurora \
  --source-endpoint-arn $SOURCE_ARN \
  --target-endpoint-arn $TARGET_ARN \
  --replication-instance-arn $REPLICATION_INSTANCE_ARN \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [{
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all",
      "object-locator": {"schema-name": "APP", "table-name": "%"},
      "rule-action": "include"
    }]
  }' \
  --replication-task-settings '{
    "TargetMetadata": {"TargetSchema": "public"},
    "FullLoadSettings": {"TargetTablePrepMode": "DO_NOTHING"},
    "Logging": {
      "EnableLogging": true,
      "LogComponents": [
        {"Id": "SOURCE_UNLOAD", "Severity": "LOGGER_SEVERITY_DEFAULT"},
        {"Id": "TARGET_LOAD", "Severity": "LOGGER_SEVERITY_DEFAULT"},
        {"Id": "TASK_MANAGER", "Severity": "LOGGER_SEVERITY_DEFAULT"}
      ]
    }
  }'

# Start migration
aws dms start-replication-task \
  --replication-task-arn $TASK_ARN \
  --start-replication-task-type start-replication

# Monitor progress
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=$TASK_ARN \
  --query 'ReplicationTasks[0].{Status:Status,Progress:ReplicationTaskStats.FullLoadProgressPercent,Latency:ReplicationTaskStats.CDCLatencySource}'
```

## PART 5 — Final Capstone Architecture

This is the crown jewel — the complete production architecture for URL Shortener as a global, highly available, multi-region SaaS platform. This is what you'd design in a principal engineer or cloud architect interview.

```
═══════════════════════════════════════════════════════════════════
           URL SHORTENER — GLOBAL PRODUCTION ARCHITECTURE
                       Designed by Chetan
═══════════════════════════════════════════════════════════════════

AWS Organizations
├── Management Account (billing, SCPs, Control Tower)
├── Security Account (GuardDuty, Security Hub, CloudTrail org trail)
├── Network Account (Transit Gateway, DNS)
├── Shared Services (ECR, artifact repos, internal tools)
├── prod-url-shortener (ap-south-1 PRIMARY)
└── dr-url-shortener   (eu-west-1  DR/SECONDARY)

═══════════════════════════════════════════════════════════════════
LAYER 0 — EDGE (Global)
═══════════════════════════════════════════════════════════════════

Users (India/Europe/Global)
         │
         ▼
    Route 53 (Latency routing)
    ├── India users   → ap-south-1  (CloudFront)
    └── Europe users  → eu-west-1   (CloudFront)
         │
         ▼
    CloudFront (Global CDN)
    ├── Cache: redirect responses (301) — TTL 300s
    ├── Cache: static assets — TTL 86400s
    ├── Origin: ALB in VPC (via origin access control)
    ├── WAF: OWASP rules, rate limit, geo-block, SQLi/XSS
    └── Shield Advanced: DDoS protection

═══════════════════════════════════════════════════════════════════
LAYER 1 — LOAD BALANCING & INGRESS
═══════════════════════════════════════════════════════════════════

Application Load Balancer
├── HTTPS 443 (TLS 1.3, ACM cert, HSTS header)
├── HTTP 80 → 301 redirect to HTTPS
├── /r/{code}    → ECS redirect service (path-based)
├── /api/v1/*    → ECS API service
├── /admin/*     → ECS admin service (IP restricted)
└── Access logs → S3 → Athena

═══════════════════════════════════════════════════════════════════
LAYER 2 — COMPUTE (ECS Fargate)
═══════════════════════════════════════════════════════════════════

ECS Cluster: url-shortener-prod
├── Service: redirect (hot path — most traffic)
│   ├── Fargate 1024 CPU / 2048 MB
│   ├── arm64 (Graviton — 20% cheaper, faster)
│   ├── Min: 5, Max: 100 tasks
│   ├── HPA: ALB requests/target < 1000
│   ├── Spread: across 3 AZs
│   └── IRSA: DAX + DynamoDB read
│
├── Service: api (create/update/delete)
│   ├── Fargate 512 CPU / 1024 MB
│   ├── Min: 3, Max: 30 tasks
│   ├── HPA: CPU < 60%
│   ├── Blue/Green via CodeDeploy (Canary10Percent5Min)
│   └── IRSA: DynamoDB write + SQS + Secrets Manager
│
└── Service: worker (async processors)
    ├── Fargate Spot (70% cheaper — tolerant to interruption)
    ├── Min: 1, Max: 20 tasks
    ├── Scales on SQS queue depth
    └── Processes: clicks, analytics, abuse checks

═══════════════════════════════════════════════════════════════════
LAYER 3 — SERVERLESS LAYER
═══════════════════════════════════════════════════════════════════

API Gateway (REST) — public API for external developers
├── Usage plans + API keys (rate limit per customer)
├── Lambda authorizer (JWT validation)
└── Routes to same Lambda functions as ECS

Lambda Functions:
├── url-shortener-create      (512MB, arm64, DLQ, SQS output)
├── url-shortener-redirect    (1024MB, arm64, Provisioned=10)
├── url-shortener-analytics   (SQS trigger, batch=100)
├── url-shortener-abuse-check (EventBridge trigger)
├── url-shortener-stats-cron  (Scheduled 01:00 UTC)
└── url-shortener-stream      (DynamoDB stream trigger)

Step Functions: AbuseDetection
├── Analyze URL (Lambda) → risk score
├── < 60: approve (DDB update)
├── 60-90: SQS → human review queue (WaitForTaskToken 24h)
└── > 90: parallel (disable + ban owner + alert security)

═══════════════════════════════════════════════════════════════════
LAYER 4 — MESSAGING & EVENTS
═══════════════════════════════════════════════════════════════════

EventBridge (url-shortener-bus):
├── URLCreated → Lambda:notifier + analytics
├── URLClicked → SQS:clicks (batched counter updates)
├── AbuseDetected → Step Functions + SNS:security
└── Scheduled → Lambda:stats + Lambda:cleanup

SQS Queues:
├── url-shortener-clicks (standard, DLQ, batch processing)
├── url-shortener-analytics.fifo (ordered, deduplicated)
├── url-shortener-review (human review queue)
└── *-dlq (DLQ for every queue, 14-day retention)

SNS Topics:
├── url-shortener-events (domain events fanout)
├── security-alerts (GuardDuty + abuse)
├── ops-alerts (CloudWatch alarms)
└── cost-alerts (budget breaches)

Kinesis:
├── url-shortener-clickstream (real-time analytics)
└── → Firehose → S3 (parquet) → Athena → QuickSight

═══════════════════════════════════════════════════════════════════
LAYER 5 — DATA
═══════════════════════════════════════════════════════════════════

Aurora Global (PostgreSQL 15):
├── Primary: ap-south-1 (r6g.xlarge, Multi-AZ)
├── Secondary: eu-west-1 (r6g.large, read-only)
├── Replication lag: < 1 second
├── RPO: near-zero, RTO: < 60s
└── Auto backups: 30 days, PITR enabled

DynamoDB (url-shortener-v2):
├── Single-table design
├── PAY_PER_REQUEST
├── DAX cluster (microsecond reads, 3-node)
├── Global Tables: ap-south-1 ↔ eu-west-1
├── Streams → Lambda (real-time sync)
├── PITR enabled + KMS encryption
└── TTL on short URLs (auto-expire)

ElastiCache Redis (cluster mode):
├── 3 shards × 2 nodes (primary + replica per shard)
├── Multi-AZ
├── In-transit + at-rest encryption
└── Caches: session tokens, rate limit counters, hot URLs

S3:
├── url-shortener-assets (versioned, replicated to eu-west-1)
├── url-shortener-logs (ALB + CloudFront + WAF logs)
├── url-shortener-backups (Velero, CloudFormation, etcd)
├── url-shortener-analytics (parquet clickstream data)
└── All: KMS encrypted, Block public access, Access logging

═══════════════════════════════════════════════════════════════════
LAYER 6 — SECURITY
═══════════════════════════════════════════════════════════════════

Identity & Access:
├── IAM Identity Center (SSO) — no IAM users
├── Permission boundaries on all dev-created roles
├── IRSA for all ECS tasks and Lambda functions
└── SCPs: region lock, deny root, require MFA

Secrets & Encryption:
├── KMS CMK per environment (auto-rotate yearly)
├── Secrets Manager: DB creds, API keys (auto-rotate 30d)
├── Parameter Store: non-sensitive config hierarchy
└── TLS everywhere: ALB termination, in-transit encryption

Detection & Response:
├── GuardDuty (all accounts) → EventBridge → Lambda (auto-isolate)
├── Security Hub (findings aggregator, posture scoring)
├── Inspector v2 (ECR + EC2 CVE scanning)
├── Macie (PII discovery in S3)
├── CloudTrail (org trail → log-archive account, Athena)
├── Config (compliance rules, auto-remediation)
└── VPC Flow Logs → S3 (parquet) → Athena (forensics)

Network:
├── WAF (ALB + CloudFront): OWASP, rate limit, geo-block
├── Shield Advanced: DDoS on ALB + CloudFront + Route53
├── Private subnets for all compute and data
├── Transit Gateway (multi-VPC connectivity)
├── PrivateLink (internal service exposure)
└── VPC Endpoints (S3, DynamoDB — no NAT cost)

═══════════════════════════════════════════════════════════════════
LAYER 7 — OBSERVABILITY
═══════════════════════════════════════════════════════════════════

Metrics:
├── CloudWatch Container Insights (ECS + EKS)
├── Prometheus (EKS workloads)
├── CloudWatch custom metrics (business KPIs)
└── Grafana dashboards (stitches all sources)

Logs:
├── CloudWatch Logs (Lambda, ECS, EKS, RDS, CloudTrail)
├── Log Insights (ad-hoc analysis)
├── S3 archive (long-term retention, Athena queries)
└── Fluentbit (log routing from containers)

Tracing:
├── X-Ray (Lambda + API Gateway + ECS)
├── X-Ray Service Map (visual distributed trace)
└── Lambda Powertools (structured logging + tracing)

Alerting:
├── CloudWatch Alarms → SNS → PagerDuty (P1/P2)
├── Anomaly detection alarms (ML baseline)
├── Composite alarms (don't alert on single signal)
└── SLO monitoring (error rate, latency p99, availability)

═══════════════════════════════════════════════════════════════════
LAYER 8 — CI/CD
═══════════════════════════════════════════════════════════════════

GitHub (main branch push)
    → CodePipeline
        ├── Source (CodeStar → GitHub)
        ├── Build (CodeBuild: test + scan + push ECR)
        │   ├── Unit tests (pytest, >80% coverage gate)
        │   ├── SAST (Bandit, Safety)
        │   ├── Container scan (Trivy — block on CRITICAL)
        │   └── Dockerfile lint (hadolint)
        ├── Deploy Staging (CodeDeploy ECS Blue/Green)
        ├── Integration Tests (Newman/k6)
        ├── Manual Approval (SNS email → team lead)
        └── Deploy Production (CodeDeploy Canary 10% → 5min → 100%)

Rollback: automatic on CloudWatch alarm breach
Notifications: SNS → Slack webhook

═══════════════════════════════════════════════════════════════════
DISASTER RECOVERY
═══════════════════════════════════════════════════════════════════

Strategy: Warm Standby
Primary: ap-south-1 (ACTIVE — 100% traffic)
DR:      eu-west-1  (WARM — scaled-down, ready)

RPO: < 30 seconds (Aurora Global replication lag)
RTO: < 10 minutes (automated failover runbook)

Data replication:
├── Aurora Global: < 1 second lag (automatic)
├── DynamoDB Global Tables: < 1 second (automatic)
├── ElastiCache: warm standby in DR (pre-provisioned)
├── S3: Cross-Region Replication (async, SLA 15 min)
└── ECR: Cross-region replication (automatic)

DNS failover:
└── Route53 health check (10s interval, 2 failures = failover)
    └── Primary fails → DR automatically receives traffic

Backup:
├── Velero (EKS): daily → S3, 30-day retention
├── RDS: automated daily + PITR 30 days
├── DynamoDB: PITR + daily exports to S3
└── CloudFormation: all stacks in git (IaC is the backup)
```

## PART 6 — Architecture Decision Records (ADRs)

In principal/architect roles you document WHY decisions were made. This is what separates architects from engineers.

```
# ADR-001: Use DynamoDB Global Tables for URL metadata

**Status:** Accepted
**Date:** 2024-01-15

**Context:**
URL shortener needs sub-millisecond reads globally.
Redirects are 95% of traffic.
Must work in both ap-south-1 and eu-west-1.

**Decision:**
Use DynamoDB Global Tables with DAX in each region.
Aurora Global for relational data (user accounts, billing).

**Rationale:**
- DynamoDB: single-digit ms reads. Global Tables = multi-region
  active-active with < 1s replication. No schema migrations.
- DAX: microsecond reads for hot short codes (top 1% of URLs
  get 80% of clicks — classic power law distribution).
- Aurora for relational: user accounts need ACID transactions,
  foreign keys, and complex queries (billing reports).

**Consequences:**
- Single-table design required — access patterns locked in upfront.
- No ad-hoc queries — must design GSIs for every access pattern.
- DAX adds $0.27/hr per node — justified by redirect latency SLA.

**Alternatives considered:**
- RDS PostgreSQL globally: too slow for redirect hot path.
- Redis only: no persistence guarantee, complex replication.
- ElastiSearch: overkill, expensive for key-value lookups.

---

# ADR-002: Fargate Spot for worker services

**Status:** Accepted

**Context:**
Worker tasks (click processing, analytics) are stateless
and can tolerate 2-minute interruption warnings.

**Decision:**
Use FARGATE_SPOT for worker service.
Use FARGATE (on-demand) for API and redirect services.

**Rationale:**
- Worker tasks take 100-500ms per message batch.
- SQS visibility timeout = 60s. Spot interruption = 2min warning.
- Worker can finish current batch and drain gracefully.
- Cost saving: ~70% on worker compute.

**Consequences:**
- Must handle SIGTERM gracefully — stop polling SQS, finish batch.
- Capacity provider strategy: base=1 FARGATE, weight=4 SPOT.
- Monitor spot reclamation rate — if > 20%, rebalance weights.
```

## PART 7 — Well-Architected Review

```
# Run Well-Architected Tool review
aws wellarchitected create-workload \
  --workload-name "url-shortener-prod" \
  --description "URL Shortener production workload" \
  --environment PRODUCTION \
  --aws-regions ap-south-1 eu-west-1 \
  --pillar-priorities \
    operationalExcellence reliability security \
    performanceEfficiency costOptimization sustainability \
  --review-owner "chetan@company.com" \
  --lenses wellarchitected serverless

WORKLOAD_ID=$(aws wellarchitected list-workloads \
  --workload-name-prefix url-shortener \
  --query 'WorkloadSummaries[0].WorkloadId' \
  --output text)

# List all questions in a pillar
aws wellarchitected list-answers \
  --workload-id $WORKLOAD_ID \
  --lens-alias wellarchitected \
  --pillar-id reliability \
  --query 'AnswerSummaries[].[QuestionId,QuestionTitle,Risk]' \
  --output table

# Answer a question
aws wellarchitected update-answer \
  --workload-id $WORKLOAD_ID \
  --lens-alias wellarchitected \
  --question-id bp_rel_withstand_component_failures \
  --selected-choices \
    rel_withstand_component_failures_fault_isolation_boundaries \
    rel_withstand_component_failures_redundant_resources \
  --notes "Multi-AZ ECS, Aurora Multi-AZ, DynamoDB Global Tables, PDB on all services"

# Generate improvement plan report
aws wellarchitected get-lens-review-report \
  --workload-id $WORKLOAD_ID \
  --lens-alias wellarchitected \
  --query 'LensReviewReport.Base64String' \
  --output text | base64 --decode > well-architected-report.pdf
```

## PART 8 — Cost Model for the Full Architecture

```
URL SHORTENER — MONTHLY COST ESTIMATE
(Assuming: 100M redirects/day, 1M URLs created/month)

COMPUTE:
├── ECS Fargate (redirect): 10 tasks × $0.04048/vCPU-hr × 730hr = $296
├── ECS Fargate (api): 3 tasks × $0.04048/vCPU-hr × 730hr = $89
├── ECS Fargate Spot (worker): 70% savings = ~$30
└── Lambda: 1B req/month × $0.20/million = $200

NETWORKING:
├── ALB: $0.008/LCU + $16.43/mo = ~$60
├── CloudFront: 1TB/mo = $85
├── NAT Gateway: 100GB/mo = $14
└── Data Transfer: cross-AZ = ~$20

DATABASE:
├── Aurora Global (r6g.xlarge multi-AZ): $350
├── DynamoDB (PAY_PER_REQUEST): ~$150
├── DAX (3 nodes r5.large): $486
└── ElastiCache (3×2 cache.r6g.large): $380

SECURITY & OBSERVABILITY:
├── GuardDuty: ~$40
├── Security Hub: ~$10
├── CloudTrail: ~$10
├── CloudWatch: ~$50
├── X-Ray: ~$20
└── WAF: $5/rule + $0.60/million req = ~$30

STORAGE:
├── S3 (1TB): $23
├── RDS storage (500GB gp3): $57
└── EBS (cluster nodes): $40

TOTAL ESTIMATED: ~$2,400/month
Per 100M redirects: $0.024/thousand = extremely cost-efficient

SAVINGS OPPORTUNITIES:
├── 1-year Reserved (Aurora + ElastiCache): save ~$350/mo
├── Savings Plans (compute): save ~$120/mo
├── Remove DAX if p99 < 5ms without it: save $486/mo
└── Total potential savings: ~$950/mo (40% reduction)
```

## Phase 5 — Architect Interview Master Sheet

<img width="950" height="481" alt="image" src="https://github.com/user-attachments/assets/d4fbff6e-60d2-4eab-8cae-2a578657e826" />

<img width="940" height="570" alt="image" src="https://github.com/user-attachments/assets/e60e22f5-12bc-453a-b6a3-e0813953d389" />

## Your Complete Learning Journey

```
Phase 1 (Foundations):    IAM, EC2, VPC, S3, RDS, CloudWatch
Phase 2 (DevOps Core):    ECR, ECS, EKS, CodePipeline, CloudFormation, Route53
Phase 3 (Security):       Organizations, SCPs, GuardDuty, WAF, TGW, KMS
Phase 4 (Serverless):     Lambda, API Gateway, SQS, SNS, EventBridge, Step Functions
Phase 5 (Architect):      Well-Architected, DR, Aurora Global, Migration 6Rs, Capstone

Certifications you're now ready for:
├── AWS Solutions Architect Associate (SAA-C03)  → Phases 1-3
├── AWS DevOps Engineer Professional             → Phases 1-4
├── AWS Solutions Architect Professional        → All 5 Phases
└── AWS Security Specialty                       → Phase 3 deep + Phase 1-2

European Job Targets armed with this knowledge:
├── Booking.com (Amsterdam)     → DevOps/SRE, EKS heavy
├── Revolut (Dublin/Lisbon)     → Platform Engineering
├── Grafana Labs (Stockholm)    → Remote, cloud-native
├── Adyen (Amsterdam)           → Payments, high reliability
├── Stripe (Dublin)             → Infrastructure Engineering
└── Any German Mittelstand      → Cloud Architect, AWS migration
```
















