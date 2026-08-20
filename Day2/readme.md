# 🚀 AWS Phase 2 — DevOps Core (Production Grade)

## PART 1 — ECR (Elastic Container Registry)

Before ECS or EKS can run your containers, they need to pull images from somewhere. ECR is AWS's private, highly available, IAM-integrated container registry.

**ECR Core Concepts**

**Registry:** Every AWS account gets one private registry per region: <account-id>.dkr.ecr.<region>.amazonaws.com

**Repository:** One per image (like a Docker Hub repo). Holds all versions/tags of one image.

**Image Scanning:** ECR can scan images for OS and package vulnerabilities using Clair (Basic) or Amazon Inspector (Enhanced — CVE database, deeper analysis).

**Lifecycle Policies:** Auto-delete old/untagged images to control storage costs.

**Replication:** Cross-region and cross-account replication for DR and global deployments.

**Pull-through Cache:** ECR can proxy Docker Hub, Quay, GitHub Container Registry — caches images locally, avoids Docker Hub rate limits.

**Hands-On: ECR Production Setup**

```
export AWS_REGION=ap-south-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# Create a repository with immutable tags (production best practice)
# Immutable = once a tag is pushed, it cannot be overwritten
aws ecr create-repository \
  --repository-name url-shortener/api \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS \
  --region $AWS_REGION

# Create repo for a sidecar (e.g. nginx proxy)
aws ecr create-repository \
  --repository-name url-shortener/nginx \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION

# Set lifecycle policy — keep last 10 tagged, delete untagged after 1 day
aws ecr put-lifecycle-policy \
  --repository-name url-shortener/api \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Delete untagged images after 1 day",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 1
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "Keep only last 10 tagged images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {"type": "expire"}
      }
    ]
  }'

# Set repository policy — allow only specific roles to pull
aws ecr set-repository-policy \
  --repository-name url-shortener/api \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowECSTaskPull",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::'$ACCOUNT_ID':role/ecs-task-execution-role",
            "arn:aws:iam::'$ACCOUNT_ID':role/eks-node-role"
          ]
        },
        "Action": [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability"
        ]
      }
    ]
  }'

# Authenticate Docker to ECR (token valid 12 hours)
aws ecr get-login-password --region $AWS_REGION \
  | docker login --username AWS --password-stdin $ECR_REGISTRY

# Build your FastAPI URL shortener image
# (Assumes you have your Dockerfile from the 35-day course)
cd ~/url-shortener

# Multi-stage production Dockerfile
cat > Dockerfile << 'EOF'
# ── Stage 1: Builder ──────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
  && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --upgrade pip \
  && pip wheel --no-cache-dir --no-deps --wheel-dir /wheels -r requirements.txt

# ── Stage 2: Runtime ──────────────────────────────────────
FROM python:3.12-slim AS runtime

# Security: non-root user
RUN groupadd --gid 1001 appgroup \
  && useradd --uid 1001 --gid appgroup --shell /bin/bash --create-home appuser

WORKDIR /app

# Install runtime OS deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
  && rm -rf /var/lib/apt/lists/*

# Copy wheels from builder and install
COPY --from=builder /wheels /wheels
RUN pip install --no-cache /wheels/*

# Copy application code
COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "2", "--log-level", "info"]
EOF

# Build with build args and labels for traceability
GIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "local")
BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
VERSION="v1.0.0"

docker build \
  --build-arg BUILD_DATE=$BUILD_DATE \
  --build-arg GIT_SHA=$GIT_SHA \
  --label "org.opencontainers.image.created=$BUILD_DATE" \
  --label "org.opencontainers.image.revision=$GIT_SHA" \
  --label "org.opencontainers.image.version=$VERSION" \
  --tag url-shortener/api:$VERSION \
  --tag url-shortener/api:$GIT_SHA \
  .

# Tag for ECR
docker tag url-shortener/api:$VERSION \
  $ECR_REGISTRY/url-shortener/api:$VERSION

docker tag url-shortener/api:$VERSION \
  $ECR_REGISTRY/url-shortener/api:$GIT_SHA

# Push both tags
docker push $ECR_REGISTRY/url-shortener/api:$VERSION
docker push $ECR_REGISTRY/url-shortener/api:$GIT_SHA

# Describe the pushed image (get digest — immutable identifier)
aws ecr describe-images \
  --repository-name url-shortener/api \
  --query 'imageDetails[].[imageTags,imageSizeInBytes,imagePushedAt,imageScanStatus.status]' \
  --output table

# Check vulnerability scan results
aws ecr describe-image-scan-findings \
  --repository-name url-shortener/api \
  --image-id imageTag=$VERSION \
  --query 'imageScanFindings.findingSeverityCounts'
```

**Pull-Through Cache Setup (avoid Docker Hub rate limits)**

```
# Configure pull-through cache for Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "dockerhub" \
  --upstream-registry-url "registry-1.docker.io" \
  --region $AWS_REGION

# Now instead of: docker pull redis:7-alpine
# Use:            docker pull $ECR_REGISTRY/dockerhub/redis:7-alpine
# ECR caches it and you never hit Docker Hub limits
```

## PART 2 — ECS (Elastic Container Service)

ECS is AWS's native container orchestrator. Two launch types: **EC2** (you manage nodes) and **Fargate** (serverless — AWS manages nodes). In production, Fargate for most workloads, EC2 for GPU or cost-sensitive high-scale.

**ECS Architecture — Deep Model**

```
ECS Cluster
├── Service A (url-shortener-api)
│   ├── Task Definition (blueprint — like a Pod spec)
│   │   ├── Container: api (FastAPI) — 512 CPU, 1024 MB
│   │   ├── Container: nginx (sidecar proxy) — 256 CPU, 256 MB
│   │   └── Container: datadog-agent (sidecar) — 128 CPU, 256 MB
│   ├── Tasks (running instances of task definition)
│   │   ├── Task 1 (AZ-a, IP: 10.0.2.15) — RUNNING
│   │   ├── Task 2 (AZ-b, IP: 10.0.4.22) — RUNNING
│   │   └── Task 3 (AZ-a, IP: 10.0.2.31) — RUNNING
│   ├── Load Balancer: ALB → Target Group
│   └── Auto Scaling: Target tracking on CPU/memory/ALB requests
│
└── Service B (url-shortener-worker)
    └── Task Definition (SQS consumer, no ALB needed)
```

**Task Definition — The Most Important Concept in ECS**

A Task Definition is like a Kubernetes Pod spec. It defines everything about how your container runs.

```
# Create IAM roles first

# Task Execution Role — used by ECS AGENT to pull images, write logs
cat > ecs-task-execution-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ecs-task-execution-role \
  --assume-role-policy-document file://ecs-task-execution-trust.json

aws iam attach-role-policy \
  --role-name ecs-task-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Also allow reading secrets (for DB passwords, API keys)
cat > ecs-secrets-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue",
      "ssm:GetParameters",
      "kms:Decrypt"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy \
  --role-name ecs-task-execution-role \
  --policy-name SecretsAccess \
  --policy-document file://ecs-secrets-policy.json

# Task Role — used by YOUR APPLICATION CODE inside the container
cat > ecs-task-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "ArnLike": {
        "aws:SourceArn": "arn:aws:ecs:ap-south-1:'$ACCOUNT_ID':*"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name ecs-url-shortener-task-role \
  --assume-role-policy-document file://ecs-task-trust.json

# App needs S3 + DynamoDB + SQS access
cat > ecs-app-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::url-shortener-*/*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem","dynamodb:GetItem","dynamodb:UpdateItem","dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:ap-south-1:*:table/url-shortener*"
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage","sqs:ReceiveMessage","sqs:DeleteMessage"],
      "Resource": "arn:aws:sqs:ap-south-1:*:url-shortener-*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ecs-url-shortener-task-role \
  --policy-name AppPermissions \
  --policy-document file://ecs-app-policy.json
```

```
# Create a secret in Secrets Manager for DB password
aws secretsmanager create-secret \
  --name /url-shortener/prod/db-password \
  --secret-string "YourSuperSecureDBPass123!" \
  --region $AWS_REGION

aws secretsmanager create-secret \
  --name /url-shortener/prod/redis-url \
  --secret-string "redis://your-elasticache-endpoint:6379" \
  --region $AWS_REGION

# Create CloudWatch Log Group
aws logs create-log-group \
  --log-group-name /ecs/url-shortener
aws logs put-retention-policy \
  --log-group-name /ecs/url-shortener \
  --retention-in-days 30

# Register the Task Definition (production-grade)
cat > task-definition.json << EOF
{
  "family": "url-shortener",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecs-task-execution-role",
  "taskRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecs-url-shortener-task-role",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "${ECR_REGISTRY}/url-shortener/api:v1.0.0",
      "portMappings": [{"containerPort": 8000, "protocol": "tcp"}],
      "essential": true,
      "cpu": 512,
      "memory": 1024,
      "memoryReservation": 512,
      "environment": [
        {"name": "ENVIRONMENT", "value": "production"},
        {"name": "AWS_REGION",  "value": "${AWS_REGION}"},
        {"name": "LOG_LEVEL",   "value": "INFO"}
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:${AWS_REGION}:${ACCOUNT_ID}:secret:/url-shortener/prod/db-password"
        },
        {
          "name": "REDIS_URL",
          "valueFrom": "arn:aws:secretsmanager:${AWS_REGION}:${ACCOUNT_ID}:secret:/url-shortener/prod/redis-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/url-shortener",
          "awslogs-region": "${AWS_REGION}",
          "awslogs-stream-prefix": "api"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60
      },
      "readonlyRootFilesystem": true,
      "linuxParameters": {
        "initProcessEnabled": true,
        "capabilities": {
          "drop": ["ALL"]
        }
      },
      "ulimits": [
        {"name": "nofile", "softLimit": 65536, "hardLimit": 65536}
      ]
    },
    {
      "name": "log-router",
      "image": "public.ecr.aws/aws-observability/aws-for-fluent-bit:stable",
      "essential": false,
      "cpu": 64,
      "memory": 128,
      "firelensConfiguration": {
        "type": "fluentbit",
        "options": {"enable-ecs-log-metadata": "true"}
      }
    }
  ]
}
EOF

aws ecs register-task-definition \
  --cli-input-json file://task-definition.json
```

```
# Create ECS Cluster
aws ecs create-cluster \
  --cluster-name url-shortener-prod \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=3 \
  --settings name=containerInsights,value=enabled \
  --tags key=Environment,value=production

# Create ALB for ECS
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name url-shortener-alb \
  --subnets $PUB_SUBNET_A $PUB_SUBNET_B \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# Create target group (for ECS Fargate — IP mode required)
TG_ARN=$(aws elbv2 create-target-group \
  --name url-shortener-tg \
  --protocol HTTP \
  --port 8000 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --health-check-timeout-seconds 10 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# Create HTTPS listener (with redirect from HTTP)
CERT_ARN="arn:aws:acm:ap-south-1:${ACCOUNT_ID}:certificate/your-cert-id"

aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=$CERT_ARN \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Create ECS Service
cat > service-definition.json << EOF
{
  "cluster": "url-shortener-prod",
  "serviceName": "url-shortener-api",
  "taskDefinition": "url-shortener",
  "desiredCount": 3,
  "launchType": "FARGATE",
  "platformVersion": "LATEST",
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": ["${PRIV_SUBNET_A}", "${PRIV_SUBNET_B}"],
      "securityGroups": ["${ECS_SG}"],
      "assignPublicIp": "DISABLED"
    }
  },
  "loadBalancers": [{
    "targetGroupArn": "${TG_ARN}",
    "containerName": "api",
    "containerPort": 8000
  }],
  "deploymentConfiguration": {
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    },
    "maximumPercent": 200,
    "minimumHealthyPercent": 100
  },
  "deploymentController": {"type": "ECS"},
  "enableExecuteCommand": true,
  "propagateTags": "SERVICE",
  "tags": [
    {"key": "Environment", "value": "production"},
    {"key": "Service",     "value": "url-shortener"}
  ]
}
EOF

aws ecs create-service --cli-input-json file://service-definition.json

# Watch deployment progress
aws ecs describe-services \
  --cluster url-shortener-prod \
  --services url-shortener-api \
  --query 'services[0].deployments'
```

**ECS Auto Scaling (Application Auto Scaling)**

```
# Register the ECS service as a scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/url-shortener-prod/url-shortener-api \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Scale on CPU — target tracking (maintains ~60% CPU)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/url-shortener-prod/url-shortener-api \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'

# Scale on ALB requests per target — for traffic-based scaling
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/url-shortener-prod/url-shortener-api \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name alb-request-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000.0,
    "CustomizedMetricSpecification": {
      "MetricName": "RequestCountPerTarget",
      "Namespace": "AWS/ApplicationELB",
      "Dimensions": [
        {"Name": "TargetGroup", "Value": "targetgroup/url-shortener-tg/abc123"},
        {"Name": "LoadBalancer", "Value": "app/url-shortener-alb/xyz789"}
      ],
      "Statistic": "Sum",
      "Unit": "Count"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

**ECS Exec — Production Debugging**

```
# SSH into a running Fargate container (no bastion needed!)
TASK_ARN=$(aws ecs list-tasks \
  --cluster url-shortener-prod \
  --service-name url-shortener-api \
  --query 'taskArns[0]' \
  --output text)

aws ecs execute-command \
  --cluster url-shortener-prod \
  --task $TASK_ARN \
  --container api \
  --interactive \
  --command "/bin/bash"

# Inside the container:
# env | grep -v PASSWORD    — check env vars (filter secrets)
# curl localhost:8000/health — check app
# cat /proc/1/net/tcp       — check connections
# df -h                     — check disk
```

**Blue/Green Deployment with CodeDeploy**

```
# For zero-downtime deployments with instant rollback capability
# First create a second target group (green)
GREEN_TG_ARN=$(aws elbv2 create-target-group \
  --name url-shortener-tg-green \
  --protocol HTTP \
  --port 8000 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# Update ECS service to use CODE_DEPLOY controller
aws ecs update-service \
  --cluster url-shortener-prod \
  --service url-shortener-api \
  --deployment-controller type=CODE_DEPLOY

# Create CodeDeploy app and deployment group
aws deploy create-application \
  --application-name url-shortener \
  --compute-platform ECS

aws deploy create-deployment-group \
  --application-name url-shortener \
  --deployment-group-name prod \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --ecs-services clusterName=url-shortener-prod,serviceName=url-shortener-api \
  --load-balancer-info '{
    "targetGroupPairInfoList": [{
      "targetGroups": [
        {"name": "url-shortener-tg"},
        {"name": "url-shortener-tg-green"}
      ],
      "prodTrafficRoute": {
        "listenerArns": ["'$HTTPS_LISTENER_ARN'"]
      }
    }]
  }' \
  --blue-green-deployment-configuration '{
    "terminateBlueInstancesOnDeploymentSuccess": {
      "action": "TERMINATE",
      "terminationWaitTimeInMinutes": 5
    },
    "deploymentReadyOption": {
      "actionOnTimeout": "CONTINUE_DEPLOYMENT"
    }
  }' \
  --service-role-arn arn:aws:iam::${ACCOUNT_ID}:role/CodeDeployECSRole
```

## PART 3 — EKS (Elastic Kubernetes Service)

EKS is managed Kubernetes on AWS. AWS manages the control plane (etcd, API server, scheduler, controller manager). You manage worker nodes and workloads.

**EKS Architecture — Production Grade**

```
AWS Cloud
├── EKS Control Plane (AWS managed, Multi-AZ)
│   ├── API Server (kube-apiserver) — accessible via endpoint
│   ├── etcd cluster (3 nodes across AZs, encrypted at rest)
│   ├── Scheduler (kube-scheduler)
│   └── Controller Manager (kube-controller-manager)
│
└── Your AWS Account
    ├── Node Group A — On-Demand (system workloads, m5.xlarge)
    │   ├── Node 1 (AZ-a) — kube-system, monitoring
    │   └── Node 2 (AZ-b) — kube-system, monitoring
    │
    ├── Node Group B — On-Demand (app workloads, m5.2xlarge)
    │   ├── Node 3 (AZ-a)
    │   ├── Node 4 (AZ-b)
    │   └── Node 5 (AZ-a)
    │
    ├── Node Group C — Spot (batch/worker, m5.xlarge+m4.xlarge)
    │   └── Nodes 6-10 (mixed AZs)
    │
    ├── Karpenter (auto-provisions exactly right node for pending pod)
    │
    ├── Add-ons:
    │   ├── CoreDNS
    │   ├── kube-proxy
    │   ├── VPC CNI (aws-node) — pods get VPC IPs directly
    │   ├── EBS CSI Driver — PersistentVolumes on EBS
    │   ├── EFS CSI Driver — ReadWriteMany volumes
    │   └── AWS Load Balancer Controller — ALB/NLB from Ingress/Service
    │
    └── Cluster Add-ons (you install):
        ├── Cluster Autoscaler or Karpenter
        ├── External DNS
        ├── Cert Manager
        ├── metrics-server
        └── Prometheus + Grafana
```

**Step 1: Production EKS Cluster with eksctl**

```
# Install eksctl
curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sL \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

```
# cluster-config.yaml — production EKS cluster definition
cat > cluster-config.yaml << 'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: url-shortener-prod
  region: ap-south-1
  version: "1.30"
  tags:
    Environment: production
    Project: url-shortener

# Use existing VPC from Phase 1
vpc:
  id: "vpc-xxxxxxxxxxxxxxxxx"   # your VPC_ID
  subnets:
    private:
      ap-south-1a: {id: "subnet-xxxxxxxxx"}  # PRIV_SUBNET_A
      ap-south-1b: {id: "subnet-xxxxxxxxx"}  # PRIV_SUBNET_B
    public:
      ap-south-1a: {id: "subnet-xxxxxxxxx"}  # PUB_SUBNET_A
      ap-south-1b: {id: "subnet-xxxxxxxxx"}  # PUB_SUBNET_B
  clusterEndpoints:
    privateAccess: true    # nodes talk to API server privately
    publicAccess: true     # you can reach API server from laptop
  publicAccessCIDRs:
    - "YOUR_IP/32"         # restrict public API access to your IP only

# Secrets encryption at rest using KMS
secretsEncryption:
  keyARN: "arn:aws:kms:ap-south-1:ACCOUNT_ID:key/your-kms-key-id"

# CloudWatch logging — all control plane logs
cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler

# IRSA — IAM Roles for Service Accounts (pods get fine-grained IAM)
iam:
  withOIDC: true    # enables IRSA
  serviceAccounts:
    - metadata:
        name: url-shortener-sa
        namespace: url-shortener
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
      roleName: eks-url-shortener-role
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
    - metadata:
        name: external-dns
        namespace: kube-system
      wellKnownPolicies:
        externalDNS: true
    - metadata:
        name: ebs-csi-controller-sa
        namespace: kube-system
      wellKnownPolicies:
        ebsCSIController: true
    - metadata:
        name: cluster-autoscaler
        namespace: kube-system
      wellKnownPolicies:
        autoScaler: true

addons:
  - name: vpc-cni
    version: latest
    attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    configurationValues: |
      {
        "env": {
          "ENABLE_PREFIX_DELEGATION": "true",
          "WARM_PREFIX_TARGET": "1"
        }
      }
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true

# System node group — dedicated for kube-system workloads
nodeGroups: []

managedNodeGroups:
  # System nodes — stable, on-demand, tainted so only system pods land here
  - name: system-ng
    instanceType: m5.large
    minSize: 2
    maxSize: 4
    desiredCapacity: 2
    privateNetworking: true
    availabilityZones: ["ap-south-1a", "ap-south-1b"]
    amiFamily: AmazonLinux2023
    labels:
      role: system
      node-type: on-demand
    taints:
      - key: CriticalAddonsOnly
        value: "true"
        effect: NoSchedule
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/url-shortener-prod: "owned"

  # Application nodes — on-demand
  - name: app-ng-ondemand
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true
    availabilityZones: ["ap-south-1a", "ap-south-1b"]
    amiFamily: AmazonLinux2023
    labels:
      role: app
      node-type: on-demand
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/url-shortener-prod: "owned"

  # Spot nodes — for workers and batch
  - name: app-ng-spot
    instanceTypes:
      - m5.xlarge
      - m5a.xlarge
      - m4.xlarge
      - m5d.xlarge
    spot: true
    minSize: 0
    maxSize: 20
    desiredCapacity: 2
    privateNetworking: true
    availabilityZones: ["ap-south-1a", "ap-south-1b"]
    amiFamily: AmazonLinux2023
    labels:
      role: spot-worker
      node-type: spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/url-shortener-prod: "owned"
EOF

# Create the cluster (takes ~15-20 minutes)
eksctl create cluster -f cluster-config.yaml

# Verify
kubectl get nodes -o wide
kubectl get pods -A
```

**Step 2: Install Core Add-ons**

```
# ── AWS Load Balancer Controller ──────────────────────────
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=url-shortener-prod \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_REGION \
  --set vpcId=$VPC_ID \
  --set image.repository=602401143452.dkr.ecr.ap-south-1.amazonaws.com/amazon/aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller

# ── Cluster Autoscaler ────────────────────────────────────
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=url-shortener-prod \
  --set awsRegion=$AWS_REGION \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.scale-down-delay-after-add=5m \
  --set extraArgs.scale-down-unneeded-time=5m

# ── Metrics Server ────────────────────────────────────────
kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl top nodes
kubectl top pods -A

# ── External DNS (Route53 integration) ───────────────────
helm repo add external-dns \
  https://kubernetes-sigs.github.io/external-dns/

helm install external-dns external-dns/external-dns \
  -n kube-system \
  --set provider=aws \
  --set aws.region=$AWS_REGION \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns \
  --set policy=upsert-only \
  --set registry=txt \
  --set txtOwnerId=url-shortener-prod

# ── Cert Manager (TLS certificates) ──────────────────────
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  -n cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=\
"arn:aws:iam::${ACCOUNT_ID}:role/cert-manager-role"
```

**Step 3: Deploy URL Shortener on EKS**

```
# Create namespace with labels for network policies
kubectl create namespace url-shortener
kubectl label namespace url-shortener \
  environment=production \
  app.kubernetes.io/name=url-shortener
```

```
# Full production manifest — save as k8s/url-shortener.yaml
cat > k8s/url-shortener.yaml << 'MANIFEST'
---
# ServiceAccount with IRSA annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: url-shortener-sa
  namespace: url-shortener
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/eks-url-shortener-role

---
# ConfigMap for non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: url-shortener-config
  namespace: url-shortener
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  AWS_REGION: "ap-south-1"
  DYNAMODB_TABLE: "url-shortener"

---
# ExternalSecret (using AWS Secrets Manager via External Secrets Operator)
# OR use native Kubernetes secret (base64 encoded — not production without Sealed Secrets)
apiVersion: v1
kind: Secret
metadata:
  name: url-shortener-secrets
  namespace: url-shortener
type: Opaque
stringData:
  DATABASE_URL: "postgresql://devopsadmin:pass@rds-endpoint:5432/urlshortener"
  REDIS_URL: "redis://elasticache-endpoint:6379"

---
# PodDisruptionBudget — ensures min 2 pods always up during node drains
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: url-shortener-pdb
  namespace: url-shortener
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: url-shortener

---
# Deployment — production grade
apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-shortener
  namespace: url-shortener
  labels:
    app: url-shortener
    version: v1.0.0
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: url-shortener
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0      # zero downtime — always maintain full capacity
  template:
    metadata:
      labels:
        app: url-shortener
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: url-shortener-sa

      # Spread pods across AZs and nodes
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: url-shortener
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: url-shortener

      # Anti-affinity — don't put two pods on same node
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: [url-shortener]
              topologyKey: kubernetes.io/hostname

      # Only schedule on on-demand nodes (not spot)
      nodeSelector:
        node-type: on-demand

      # Graceful shutdown
      terminationGracePeriodSeconds: 60

      # Init container — wait for DB to be ready
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z $DB_HOST 5432; do
                echo "Waiting for PostgreSQL..."
                sleep 2
              done
              echo "PostgreSQL is ready!"
          env:
            - name: DB_HOST
              value: "your-rds-endpoint.ap-south-1.rds.amazonaws.com"
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 50m
              memory: 32Mi

      containers:
        - name: api
          image: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/url-shortener/api:v1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
              name: http
              protocol: TCP

          envFrom:
            - configMapRef:
                name: url-shortener-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: url-shortener-secrets
                  key: DATABASE_URL
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: url-shortener-secrets
                  key: REDIS_URL
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName

          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 3

          startupProbe:
            httpGet:
              path: /health/live
              port: 8000
            failureThreshold: 30
            periodSeconds: 5

          # Security context — hardened container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1001
            runAsGroup: 1001
            capabilities:
              drop: ["ALL"]
            seccompProfile:
              type: RuntimeDefault

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-cache
              mountPath: /app/.cache

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]  # drain in-flight requests

      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-cache
          emptyDir: {}

      # Security context at pod level
      securityContext:
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: url-shortener-hpa
  namespace: url-shortener
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: url-shortener
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
  behavior:
    scaleOut:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleIn:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: url-shortener
  namespace: url-shortener
  labels:
    app: url-shortener
spec:
  selector:
    app: url-shortener
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
  type: ClusterIP

---
# Ingress with ALB (uses AWS Load Balancer Controller)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: url-shortener
  namespace: url-shortener
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:ap-south-1:ACCOUNT_ID:certificate/xxx"
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/success-codes: "200"
    alb.ingress.kubernetes.io/wafv2-acl-arn: "arn:aws:wafv2:ap-south-1:ACCOUNT_ID:regional/webacl/xxx"
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      idle_timeout.timeout_seconds=60,
      routing.http2.enabled=true,
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=url-shortener-alb-logs
spec:
  rules:
    - host: api.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: url-shortener
                port:
                  number: 80

---
# Gateway API (modern replacement for Ingress)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: url-shortener-route
  namespace: url-shortener
spec:
  parentRefs:
    - name: prod-gateway
      namespace: kube-system
  hostnames:
    - "api.yourdomain.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: url-shortener
          port: 80

---
# NetworkPolicy — zero-trust networking
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: url-shortener-netpol
  namespace: url-shortener
spec:
  podSelector:
    matchLabels:
      app: url-shortener
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: TCP
          port: 8000
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8   # VPC internal
      ports:
        - protocol: TCP
          port: 5432            # PostgreSQL
        - protocol: TCP
          port: 6379            # Redis
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 169.254.169.254/32
      ports:
        - protocol: TCP
          port: 443             # AWS APIs (Secrets Manager, DynamoDB, S3)
MANIFEST

kubectl apply -f k8s/url-shortener.yaml

# Verify everything
kubectl get all -n url-shortener
kubectl describe ingress url-shortener -n url-shortener
kubectl get hpa -n url-shortener
kubectl top pods -n url-shortener
```

**Step 4: StatefulSet for Redis on EKS**

```
cat > k8s/redis-statefulset.yaml << 'EOF'
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: url-shortener
data:
  redis.conf: |
    maxmemory 512mb
    maxmemory-policy allkeys-lru
    appendonly yes
    appendfilename appendonly.aof
    save 900 1
    save 300 10
    save 60 10000
    requirepass ""
    protected-mode no

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: url-shortener
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command: ["redis-server", "/config/redis.conf"]
          ports:
            - containerPort: 6379
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 768Mi
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /config
          livenessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 5
            periodSeconds: 5
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            runAsUser: 999
      volumes:
        - name: config
          configMap:
            name: redis-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3   # custom StorageClass (below)
        resources:
          requests:
            storage: 10Gi

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:ap-south-1:ACCOUNT_ID:key/your-key"
volumeBindingMode: WaitForFirstConsumer   # provision in same AZ as pod
reclaimPolicy: Retain                     # don't delete EBS on PVC delete

---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: url-shortener
spec:
  clusterIP: None    # headless service for StatefulSet
  selector:
    app: redis
  ports:
    - port: 6379
EOF

kubectl apply -f k8s/redis-statefulset.yaml
```

## PART 4 — EKS UPGRADES (Production Runbook)

This is the most critical operational skill for a senior DevOps/SRE role. EKS upgrades done wrong cause outages.

**Understanding the EKS Upgrade Sequence**

```
RULE: Control plane version must be upgraded before node groups.
RULE: Nodes can be at most 2 minor versions behind control plane.
RULE: Add-ons must be upgraded after control plane, before or after nodes.

Correct order:
1. Review release notes + deprecation warnings
2. Backup etcd (covered below)
3. Upgrade control plane  (1.29 → 1.30)
4. Upgrade add-ons        (vpc-cni, coredns, kube-proxy)
5. Upgrade node groups    (one at a time, rolling)
6. Upgrade kubectl + eksctl locally
7. Validate workloads
```

**Pre-Upgrade Checklist**

```
# 1. Check current versions
kubectl version
aws eks describe-cluster \
  --name url-shortener-prod \
  --query 'cluster.version' \
  --output text

eksctl get cluster --name url-shortener-prod

# 2. Check deprecation warnings — critical before ANY upgrade
# Install kubent (Kubernetes No-Trouble)
sh -c "$(curl -sSfL https://git.io/install-kubent)"

kubent --target-version 1.30

# 3. Check PodDisruptionBudgets that might block node drain
kubectl get pdb -A

# 4. Check all add-on versions
aws eks describe-addon-versions --kubernetes-version 1.30 \
  --query 'addons[].{Name:addonName,Latest:addonVersions[0].addonVersion}' \
  --output table

aws eks list-addons --cluster-name url-shortener-prod
aws eks describe-addon \
  --cluster-name url-shortener-prod \
  --addon-name vpc-cni \
  --query 'addon.{Name:addonName,Version:addonVersion,Status:status}'

# 5. Verify cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get componentstatuses

# 6. Check node group state
aws eks describe-nodegroup \
  --cluster-name url-shortener-prod \
  --nodegroup-name app-ng-ondemand \
  --query 'nodegroup.{Status:status,Health:health}'
```

**etcd Backup — Before Every Major Operation**

In EKS, AWS manages etcd but you can't access it directly. Here's the complete backup strategy:

```
# ── Method 1: Velero — Full cluster backup (resources + PVs) ─────────────

# Install Velero CLI
VELERO_VERSION=v1.13.0
curl -fsSL \
  "https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/

# Create S3 bucket for backups
BACKUP_BUCKET="eks-velero-backups-${ACCOUNT_ID}-${AWS_REGION}"
aws s3api create-bucket \
  --bucket $BACKUP_BUCKET \
  --region $AWS_REGION \
  --create-bucket-configuration LocationConstraint=$AWS_REGION

aws s3api put-bucket-versioning \
  --bucket $BACKUP_BUCKET \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket $BACKUP_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms"
      }
    }]
  }'

# Create IAM policy for Velero
cat > velero-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes", "ec2:DescribeSnapshots",
        "ec2:CreateSnapshot", "ec2:DeleteSnapshot",
        "ec2:DescribeTags", "ec2:CreateTags"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject", "s3:DeleteObject", "s3:PutObject",
        "s3:AbortMultipartUpload", "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::${BACKUP_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::${BACKUP_BUCKET}"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name VeleroBackupPolicy \
  --policy-document file://velero-policy.json

# Install Velero on the cluster with IRSA
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket $BACKUP_BUCKET \
  --backup-location-config region=$AWS_REGION \
  --snapshot-location-config region=$AWS_REGION \
  --service-account-annotations \
    "eks.amazonaws.com/role-arn=arn:aws:iam::${ACCOUNT_ID}:role/velero-role" \
  --use-node-agent \
  --features=EnableCSI

# Verify Velero is healthy
velero backup-location get

# ─── PRE-UPGRADE BACKUP ──────────────────────────────────
# Full cluster backup before upgrade
velero backup create pre-upgrade-$(date +%Y%m%d-%H%M) \
  --include-namespaces='*' \
  --include-cluster-resources=true \
  --storage-location default \
  --snapshot-volumes=true \
  --wait

# Verify backup succeeded
velero backup describe pre-upgrade-$(date +%Y%m%d) --details
velero backup logs pre-upgrade-$(date +%Y%m%d)

# Namespace-specific backup (faster, targeted)
velero backup create url-shortener-pre-upgrade \
  --include-namespaces url-shortener \
  --snapshot-volumes=true \
  --wait

# Schedule automated daily backups
velero schedule create daily-cluster-backup \
  --schedule="0 2 * * *" \
  --include-namespaces='*' \
  --include-cluster-resources=true \
  --ttl 720h   # keep 30 days

velero schedule get
```

```
# ── Method 2: etcdctl direct backup (self-managed clusters / kops) ────────
# For EKS this runs on control plane nodes — AWS doesn't expose these.
# But you MUST know this for CKA + interviews.

# On a self-managed cluster (e.g. kubeadm), SSH to control plane:
export ETCDCTL_API=3

ETCD_CERT=/etc/kubernetes/pki/etcd/server.crt
ETCD_KEY=/etc/kubernetes/pki/etcd/server.key
ETCD_CA=/etc/kubernetes/pki/etcd/ca.crt
ETCD_ENDPOINT=https://127.0.0.1:2379

# Check etcd cluster health
etcdctl \
  --endpoints=$ETCD_ENDPOINT \
  --cacert=$ETCD_CA \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  endpoint health

etcdctl \
  --endpoints=$ETCD_ENDPOINT \
  --cacert=$ETCD_CA \
  --cert=$ETCD_CA \
  --key=$ETCD_KEY \
  endpoint status --write-out=table

# Take a snapshot backup
etcdctl \
  --endpoints=$ETCD_ENDPOINT \
  --cacert=$ETCD_CA \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Verify snapshot integrity
etcdctl snapshot status \
  /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --write-out=table

# Upload to S3
aws s3 cp /backup/etcd-snapshot-$(date +%Y%m%d).db \
  s3://$BACKUP_BUCKET/etcd-snapshots/

# ── etcd RESTORE (disaster recovery) ─────────────────────
# Stop kube-apiserver first
sudo systemctl stop kube-apiserver

# Restore from snapshot
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore \
  --name=master-1 \
  --initial-cluster="master-1=https://10.0.1.5:2380" \
  --initial-cluster-token=etcd-cluster-restore \
  --initial-advertise-peer-urls=https://10.0.1.5:2380 \
  --cacert=$ETCD_CA \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Update etcd manifest to use restored data dir
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restore|g' \
  /etc/kubernetes/manifests/etcd.yaml

# Restart
sudo systemctl restart kubelet
sudo systemctl start kube-apiserver
kubectl get nodes
```

**Execute the EKS Control Plane Upgrade**

```
# ── UPGRADE CONTROL PLANE ────────────────────────────────
# EKS upgrades are in-place — API server, etcd, scheduler all updated by AWS
# Typical duration: 10-20 minutes

aws eks update-cluster-version \
  --name url-shortener-prod \
  --kubernetes-version 1.30

# Watch upgrade status
watch -n 30 'aws eks describe-cluster \
  --name url-shortener-prod \
  --query "cluster.{Version:version,Status:status,PlatformVersion:platformVersion}" \
  --output table'

# Or wait for completion
aws eks wait cluster-active --name url-shortener-prod

# Verify control plane
aws eks describe-cluster \
  --name url-shortener-prod \
  --query 'cluster.version'

# ── UPGRADE ADD-ONS ───────────────────────────────────────
# Must upgrade after control plane, in this order:
# vpc-cni → kube-proxy → coredns → ebs-csi-driver

for ADDON in vpc-cni kube-proxy coredns aws-ebs-csi-driver; do
  echo "Upgrading addon: $ADDON"

  # Get latest compatible version
  LATEST_VERSION=$(aws eks describe-addon-versions \
    --kubernetes-version 1.30 \
    --addon-name $ADDON \
    --query 'addons[0].addonVersions[0].addonVersion' \
    --output text)

  echo "Latest version for $ADDON: $LATEST_VERSION"

  aws eks update-addon \
    --cluster-name url-shortener-prod \
    --addon-name $ADDON \
    --addon-version $LATEST_VERSION \
    --resolve-conflicts OVERWRITE

  # Wait for addon to be Active
  aws eks wait addon-active \
    --cluster-name url-shortener-prod \
    --addon-name $ADDON

  echo "$ADDON upgraded successfully"
done

# Verify all add-ons
aws eks list-addons --cluster-name url-shortener-prod \
  --query 'addons' --output table
```

**Node Group Rolling Upgrade**

```
# ── UPGRADE NODE GROUPS — One group at a time ─────────────

# Strategy: upgrade system nodes first, validate, then app nodes

# Step 1: Upgrade system node group
aws eks update-nodegroup-version \
  --cluster-name url-shortener-prod \
  --nodegroup-name system-ng \
  --force   # force update even if PDB would block

# Watch the node group update
watch -n 30 'aws eks describe-nodegroup \
  --cluster-name url-shortener-prod \
  --nodegroup-name system-ng \
  --query "nodegroup.{Status:status,UpdateID:updateConfig}" \
  --output table'

aws eks wait nodegroup-active \
  --cluster-name url-shortener-prod \
  --nodegroup-name system-ng

# Validate after system node upgrade
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get pods -A | grep -v Running | grep -v Completed

# Step 2: Upgrade app node group (production workloads)
# First — scale up so we have buffer capacity
aws eks update-nodegroup-config \
  --cluster-name url-shortener-prod \
  --nodegroup-name app-ng-ondemand \
  --scaling-config minSize=4,maxSize=12,desiredSize=6

sleep 60   # let new nodes join

# Upgrade the node group (rolling by default — new nodes, drain old ones)
aws eks update-nodegroup-version \
  --cluster-name url-shortener-prod \
  --nodegroup-name app-ng-ondemand

aws eks wait nodegroup-active \
  --cluster-name url-shortener-prod \
  --nodegroup-name app-ng-ondemand

# Scale back to normal
aws eks update-nodegroup-config \
  --cluster-name url-shortener-prod \
  --nodegroup-name app-ng-ondemand \
  --scaling-config minSize=2,maxSize=10,desiredSize=3

# Manual node drain (when you need fine control)
# Example: drain a specific node for maintenance
NODE_NAME="ip-10-0-2-100.ap-south-1.compute.internal"

# Cordon first — prevent new pods from scheduling
kubectl cordon $NODE_NAME

# Drain — evict all pods (respects PDBs, waits for them)
kubectl drain $NODE_NAME \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s

# After maintenance, uncordon
kubectl uncordon $NODE_NAME
```

**Post-Upgrade Validation**

```
# Full cluster health check after upgrade
echo "=== Node Status ==="
kubectl get nodes -o wide

echo "=== System Pods ==="
kubectl get pods -n kube-system -o wide

echo "=== Application Pods ==="
kubectl get pods -n url-shortener -o wide

echo "=== HPA Status ==="
kubectl get hpa -A

echo "=== PDB Status ==="
kubectl get pdb -A

echo "=== Events (last 10 minutes) ==="
kubectl get events -A \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -20

# Check application is serving traffic
ALB_DNS=$(kubectl get ingress url-shortener \
  -n url-shortener \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl -s https://$ALB_DNS/health | jq .

# Run smoke tests
for i in {1..10}; do
  STATUS=$(curl -so /dev/null -w "%{http_code}" https://$ALB_DNS/health)
  echo "Request $i: HTTP $STATUS"
  sleep 1
done

# Verify versions are all consistent
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

## PART 5 — EKS Karpenter (Modern Node Autoscaling)

Cluster Autoscaler is the old way. Karpenter is the new standard — it provisions exactly the right node for pending pods in under 60 seconds.

```
# Install Karpenter
export KARPENTER_VERSION=v0.37.0

helm repo add karpenter https://charts.karpenter.sh/
helm repo update

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "kube-system" \
  --create-namespace \
  --set "settings.clusterName=url-shortener-prod" \
  --set "settings.clusterEndpoint=$(aws eks describe-cluster \
    --name url-shortener-prod \
    --query 'cluster.endpoint' --output text)" \
  --set "settings.interruptionQueue=url-shortener-prod-karpenter" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi

# NodePool — defines what kinds of nodes Karpenter can provision
cat > k8s/karpenter-nodepool.yaml << 'EOF'
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [m, c, r]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
  limits:
    cpu: "200"
    memory: 400Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 720h    # recycle nodes every 30 days
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: eks-node-role
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: url-shortener-prod
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: url-shortener-prod
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        encrypted: true
EOF

kubectl apply -f k8s/karpenter-nodepool.yaml
```

## PART 6 — Production Observability Stack

```
# ── Prometheus + Grafana on EKS ───────────────────────────
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

cat > prometheus-values.yaml << 'EOF'
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 50Gi
    additionalScrapeConfigs:
      - job_name: url-shortener
        static_configs:
          - targets: ['url-shortener.url-shortener.svc.cluster.local:8000']

grafana:
  adminPassword: "your-secure-password"
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internet-facing
    hosts:
      - grafana.yourdomain.com

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi
EOF

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  -f prometheus-values.yaml

# Check stack
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

## Phase 2 — Architecture: What You've Built

```
Internet
    │
    ▼
Route 53 (api.yourdomain.com)
    │
    ▼
AWS WAF → ALB (AWS Load Balancer Controller)
    │
    ▼
EKS Cluster (url-shortener-prod, v1.30)
├── Namespace: url-shortener
│   ├── Deployment (3→20 replicas, HPA)
│   │   └── Pods (IRSA → DynamoDB/S3/SQS, no static keys)
│   ├── StatefulSet: Redis (EBS gp3, 10Gi)
│   ├── NetworkPolicy (zero-trust)
│   └── PodDisruptionBudget (minAvailable: 2)
│
├── Namespace: monitoring
│   └── Prometheus + Grafana
│
└── Karpenter (right-sized nodes in <60s)

ECR: url-shortener/api (immutable, KMS-encrypted, scanned)
RDS: PostgreSQL Multi-AZ (private subnet)
Secrets Manager: DB pass, Redis URL
Velero: Daily backups → S3 (encrypted, versioned)
CloudWatch: Control plane logs, Container Insights
```

**Phase 2 Interview Cheat Sheet**

<img width="913" height="387" alt="image" src="https://github.com/user-attachments/assets/acf6a208-e6a6-49be-95f4-434d04293408" />

<img width="890" height="277" alt="image" src="https://github.com/user-attachments/assets/5799f3bb-b158-4fb9-bb82-373a47852f34" />









































