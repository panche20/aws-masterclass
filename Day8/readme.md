# 🏗️ Project Design — CloudForge IDP

### Internal Developer Platform on AWS + EKS

## What Is an IDP and Why Does It Matter

```
An Internal Developer Platform (IDP) is the infrastructure
platform that a company builds so their application developers
can deploy, monitor, and operate services WITHOUT needing to
understand the underlying AWS/Kubernetes complexity.

Famous IDPs:
├── Spotify    → Backstage (open-sourced it)
├── Netflix    → Metaflow + internal tooling
├── Uber       → uDeploy
└── Booking    → Their internal Kubernetes platform

What you're building:
A production-grade IDP that a real company could use.
Not a toy. Not a tutorial. A real platform.
```

**Project Name: CloudForge**

```
CloudForge — A Production Internal Developer Platform

Tagline: "From git push to production in under 5 minutes,
          with full observability and zero manual steps."

What it does:
├── Developer pushes code to GitHub
├── Pipeline automatically builds, tests, scans, deploys
├── Service runs on EKS with autoscaling
├── Full observability (metrics, logs, traces, dashboards)
├── Self-service: developer requests infra via PR (GitOps)
├── Security: zero-trust, secrets management, policy enforcement
└── Multi-environment: dev → staging → production promotion
```

**Full Architecture**

```
═══════════════════════════════════════════════════════════
                    CLOUDFORGE IDP
═══════════════════════════════════════════════════════════

LAYER 1 — INFRASTRUCTURE (Terraform)
├── AWS Organizations (3 accounts: tooling, staging, prod)
├── EKS Cluster per environment (1.30, Karpenter)
├── VPC per account (Transit Gateway connected)
├── ECR (image registry, cross-account)
├── RDS Aurora PostgreSQL (per env)
├── ElastiCache Redis (per env)
├── S3 (artifacts, state, logs)
└── KMS (per env encryption keys)

LAYER 2 — PLATFORM SERVICES (Kubernetes)
├── ArgoCD          — GitOps continuous delivery
├── Backstage       — Developer portal (service catalog)
├── Crossplane      — Self-service infra via Kubernetes CRDs
├── External Secrets Operator — Secrets Manager → K8s secrets
├── Cert Manager    — TLS certificates (Let's Encrypt / ACM)
├── External DNS    — Route53 sync from Kubernetes
├── AWS LB Controller — ALB/NLB from Ingress/Service
└── Karpenter       — Intelligent node autoscaling

LAYER 3 — CI/CD PIPELINE (GitHub Actions + ArgoCD)
├── GitHub Actions  — Build, test, scan, push to ECR
├── ArgoCD          — Pull-based GitOps deployment
├── ArgoCD Image Updater — Auto-update image tags in git
├── ArgoCD ApplicationSets — Multi-env/multi-cluster
└── Argo Rollouts   — Canary + Blue/Green deployments

LAYER 4 — OBSERVABILITY STACK
├── Prometheus      — Metrics collection
├── Grafana         — Dashboards + alerting
├── Loki            — Log aggregation
├── Tempo           — Distributed tracing
├── OpenTelemetry   — Instrumentation (OTEL collector)
├── Alertmanager    — Alert routing → Slack/PagerDuty
└── Grafana OnCall  — On-call scheduling

LAYER 5 — SECURITY
├── Falco           — Runtime threat detection
├── OPA Gatekeeper  — Policy enforcement (no root, resource limits)
├── Trivy Operator  — Continuous vulnerability scanning
├── kube-bench      — CIS benchmark compliance
├── AWS GuardDuty   — Threat detection
└── IRSA            — Zero static credentials

LAYER 6 — SAMPLE MICROSERVICES (to demo the platform)
├── api-service     (FastAPI — your URL shortener evolved)
├── worker-service  (async click processor)
├── analytics-service (metrics aggregator)
└── frontend        (React dashboard)
```

**Repository Structure**

```
cloudforge/
├── README.md                    ← stunning README (recruiters read this first)
├── docs/
│   ├── architecture.md
│   ├── getting-started.md
│   ├── runbooks/
│   └── adr/                     ← Architecture Decision Records
│
├── terraform/                   ← ALL infrastructure as code
│   ├── modules/
│   │   ├── eks/
│   │   ├── vpc/
│   │   ├── rds/
│   │   ├── ecr/
│   │   └── kms/
│   ├── environments/
│   │   ├── tooling/
│   │   ├── staging/
│   │   └── production/
│   └── global/                  ← Organizations, IAM, ECR
│
├── kubernetes/                  ← Platform Kubernetes manifests
│   ├── platform/
│   │   ├── argocd/
│   │   ├── monitoring/
│   │   ├── logging/
│   │   ├── tracing/
│   │   ├── security/
│   │   └── ingress/
│   └── applications/            ← ArgoCD ApplicationSets
│
├── helm/                        ← Custom Helm charts
│   ├── cloudforge-app/          ← Generic app chart (all services use this)
│   └── cloudforge-platform/     ← Platform umbrella chart
│
├── services/                    ← Sample microservices
│   ├── api-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   ├── buildspec.yml
│   │   └── helm/
│   ├── worker-service/
│   └── analytics-service/
│
├── .github/
│   └── workflows/
│       ├── ci-api-service.yml
│       ├── ci-worker-service.yml
│       ├── terraform-plan.yml
│       ├── terraform-apply.yml
│       └── security-scan.yml
│
└── scripts/
    ├── bootstrap.sh             ← One-command cluster setup
    ├── dr-failover.sh
    └── load-test.sh
```

**Build Plan — 8 Weeks**

```
Week 1: Foundation (Terraform + EKS)
Week 2: GitOps Layer (ArgoCD + Image Updater)
Week 3: CI Pipeline (GitHub Actions, full security gates)
Week 4: Observability Stack (Prometheus, Grafana, Loki, Tempo)
Week 5: Security Layer (Falco, OPA, Trivy, IRSA)
Week 6: Self-Service (Crossplane, Backstage, External Secrets)
Week 7: Advanced Deployments (Argo Rollouts canary/blue-green)
Week 8: Polish (README, docs, ADRs, load tests, demo video)
```

**What Recruiters Will See on GitHub**

```
cloudforge/
  ⭐ Production-grade Terraform modules (not scripts — modules)
  ⭐ EKS with Karpenter (not Cluster Autoscaler — shows currency)
  ⭐ GitOps with ArgoCD (industry standard, every EU company uses it)
  ⭐ Full LGTM observability stack (Loki+Grafana+Tempo+Mimir)
  ⭐ OPA policies (shows security-first mindset)
  ⭐ Argo Rollouts (progressive delivery — advanced)
  ⭐ Crossplane (infrastructure as Kubernetes CRDs — cutting edge)
  ⭐ OpenTelemetry (vendor-neutral observability — industry direction)
  ⭐ GitHub Actions with security gates (real-world CI)
  ⭐ Architecture Decision Records (shows architect thinking)
  ⭐ Stunning README with architecture diagram
```

# 🏗️ CloudForge — Week 1: Terraform + EKS Foundation

**What We're Building This Week**

```
Week 1 Deliverables:
├── GitHub repo (professional structure, branch protection)
├── Terraform backend (S3 + DynamoDB — remote state)
├── Terraform modules (VPC, EKS, ECR, KMS, IAM)
├── 3-environment setup (tooling, staging, production)
├── EKS cluster (1.30, Karpenter, production-grade)
├── Kubeconfig + cluster validation
└── GitHub Actions (Terraform plan on PR, apply on merge)

By end of week:
kubectl get nodes   ← working
terraform plan      ← clean
GitHub PR           ← triggers automated plan
```

## PART 1 — GitHub Repository Setup

**Create the Repository**

```
# Install GitHub CLI
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
  https://cli.github.com/packages stable main" \
  | sudo tee /etc/apt/sources.list.d/github-cli.list
sudo apt update && sudo apt install gh -y

# Authenticate
gh auth login

# Create repository
gh repo create cloudforge \
  --public \
  --description "Production-grade Internal Developer Platform on AWS EKS — GitOps, Observability, Security, Self-Service" \
  --clone

cd cloudforge
```

**Professional README First**

Recruiters hit your README before anything else. Write it now, update it as you build.

```
cat > README.md << 'EOF'
# ☁️ CloudForge — Internal Developer Platform

> Production-grade Internal Developer Platform (IDP) built on AWS EKS.
> From `git push` to production in under 5 minutes — with full
> observability, zero-trust security, and self-service infrastructure.

![Architecture](docs/images/architecture.png)

## 🏗️ Architecture Overview

CloudForge is a fully production-grade IDP that demonstrates:

- **GitOps** — ArgoCD-driven deployments, all state in git
- **Progressive Delivery** — Canary and Blue/Green via Argo Rollouts
- **Full Observability** — Prometheus, Grafana, Loki, Tempo (LGTM stack)
- **Zero-Trust Security** — OPA Gatekeeper, Falco, Trivy, IRSA
- **Self-Service Infra** — Crossplane CRDs, Backstage developer portal
- **Infrastructure as Code** — Terraform modules for all AWS resources
- **Multi-Environment** — Tooling, Staging, Production accounts

## 🗂️ Repository Structure
```

```
cloudforge/
├── terraform/ # All AWS infrastructure (modular)
├── kubernetes/ # Platform manifests (ArgoCD, monitoring, security)
├── helm/ # Custom Helm charts
├── services/ # Sample microservices
├── .github/workflows/ # CI/CD pipelines
├── docs/ # Architecture docs, runbooks, ADRs
└── scripts/ # Operational scripts
```

```
## 🚀 Quick Start

bash
# Prerequisites: AWS CLI, Terraform, kubectl, Helm, ArgoCD CLI
./scripts/bootstrap.sh --env staging

See [Getting Started](docs/getting-started.md) for full setup guide.

## 📐 Technology Stack

| Layer | Technology |
|---|---|
| Cloud | AWS (EKS, RDS Aurora, ElastiCache, ECR, S3, KMS) |
| Container Orchestration | Kubernetes 1.30 |
| Node Autoscaling | Karpenter |
| GitOps | ArgoCD + ArgoCD Image Updater |
| Progressive Delivery | Argo Rollouts |
| CI | GitHub Actions |
| IaC | Terraform (modular) |
| Observability | Prometheus, Grafana, Loki, Tempo, OpenTelemetry |
| Security | OPA Gatekeeper, Falco, Trivy Operator, GuardDuty |
| Self-Service | Crossplane, External Secrets Operator, Backstage |
| Service Mesh | (Week 6) |

## 📊 Platform Capabilities

- ✅ Zero-downtime deployments (Argo Rollouts canary)
- ✅ Automatic TLS (cert-manager + ACM)
- ✅ Automatic DNS (external-dns + Route53)
- ✅ Secret rotation (External Secrets + Secrets Manager)
- ✅ Runtime security (Falco threat detection)
- ✅ Policy enforcement (OPA — no root containers, resource limits required)
- ✅ Continuous vulnerability scanning (Trivy Operator)
- ✅ Multi-environment promotion (dev → staging → prod via ArgoCD)
- ✅ Self-service databases (Crossplane RDS CRDs)

## 🌍 Designed For

European cloud-native engineering teams needing a production IDP
that eliminates platform toil and accelerates developer velocity.

## 📝 Architecture Decision Records

- [ADR-001: GitOps over push-based CI/CD](docs/adr/001-gitops.md)
- [ADR-002: Karpenter over Cluster Autoscaler](docs/adr/002-karpenter.md)
- [ADR-003: LGTM stack over ELK](docs/adr/003-lgtm-stack.md)
- [ADR-004: Crossplane for self-service infra](docs/adr/004-crossplane.md)

---

*Built by Chetan — DevOps Engineer | [LinkedIn](https://linkedin.com/in/yourprofile)*
EOF
```

**Directory Structure**

```
# Create full directory tree
mkdir -p \
  terraform/modules/{eks,vpc,rds,ecr,kms,iam,elasticache} \
  terraform/environments/{tooling,staging,production} \
  terraform/global \
  kubernetes/platform/{argocd,monitoring,logging,tracing,security,ingress,autoscaling} \
  kubernetes/applications \
  helm/{cloudforge-app,cloudforge-platform} \
  services/{api-service,worker-service,analytics-service} \
  .github/workflows \
  docs/{adr,runbooks,images} \
  scripts

# Root .gitignore
cat > .gitignore << 'EOF'
# Terraform
**/.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
.terraform.lock.hcl
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Secrets — NEVER commit these
*.pem
*.key
*.p12
kubeconfig
.env
.env.*
secrets.yaml
*-secrets.yaml

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp

# Python
__pycache__/
*.pyc
.pytest_cache/
venv/
.coverage

# Node
node_modules/
EOF

# Pre-commit config (enforces quality before every commit)
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.92.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_trivy
        args:
          - --args=--severity=CRITICAL

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: detect-private-key
      - id: no-commit-to-branch
        args: [--branch, main]

  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint-docker

  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
EOF

# Install pre-commit
pip install pre-commit --break-system-packages
pre-commit install

# Initial commit
git add .
git commit -m "feat: initial repository structure

- Professional README with full tech stack
- Complete directory structure
- .gitignore for Terraform, secrets, OS files
- pre-commit hooks (terraform fmt/validate, secret detection)
- Enforces no direct commits to main branch"

git push origin main
```

**Branch Protection**

```
# Protect main branch — PRs required, no force push
gh api repos/:owner/cloudforge/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["terraform-plan","security-scan"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true}' \
  --field restrictions=null \
  --field allow_force_pushes=false \
  --field allow_deletions=false

# Create development branch for your work
git checkout -b feat/week1-foundation
```

**PART 2 — Terraform Backend Setup**

Before writing any infrastructure code, set up remote state. This is non-negotiable for production.

```
export AWS_REGION=ap-south-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export PROJECT="cloudforge"

# Bootstrap script — creates backend resources manually
# (Can't use Terraform to create Terraform's own backend)
cat > scripts/bootstrap-backend.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

PROJECT=${1:-cloudforge}
REGION=${2:-ap-south-1}
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "Bootstrapping Terraform backend for project: $PROJECT"
echo "Account: $ACCOUNT_ID | Region: $REGION"

# KMS key for state encryption
KMS_KEY_ID=$(aws kms create-key \
  --description "${PROJECT} Terraform state encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --query 'KeyMetadata.KeyId' \
  --output text)

aws kms create-alias \
  --alias-name "alias/${PROJECT}-terraform-state" \
  --target-key-id $KMS_KEY_ID

aws kms enable-key-rotation --key-id $KMS_KEY_ID

echo "KMS Key: $KMS_KEY_ID"

# S3 bucket for Terraform state
STATE_BUCKET="${PROJECT}-terraform-state-${ACCOUNT_ID}-${REGION}"

aws s3api create-bucket \
  --bucket $STATE_BUCKET \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION

# Enable versioning (never lose state)
aws s3api put-bucket-versioning \
  --bucket $STATE_BUCKET \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket $STATE_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "'$KMS_KEY_ID'"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Block all public access
aws s3api put-public-access-block \
  --bucket $STATE_BUCKET \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,\
BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Lifecycle — delete old state versions after 90 days
aws s3api put-bucket-lifecycle-configuration \
  --bucket $STATE_BUCKET \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-old-versions",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 90}
    }]
  }'

# Bucket policy — deny non-TLS and non-HTTPS access
aws s3api put-bucket-policy \
  --bucket $STATE_BUCKET \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyNonTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::'$STATE_BUCKET'",
        "arn:aws:s3:::'$STATE_BUCKET'/*"
      ],
      "Condition": {
        "Bool": {"aws:SecureTransport": "false"}
      }
    }]
  }'

echo "State bucket: $STATE_BUCKET"

# DynamoDB table for state locking
LOCK_TABLE="${PROJECT}-terraform-locks"

aws dynamodb create-table \
  --table-name $LOCK_TABLE \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --sse-specification Enabled=true,SSEType=KMS \
  --tags Key=Project,Value=$PROJECT Key=Purpose,Value=terraform-locking \
  --region $REGION

aws dynamodb wait table-exists \
  --table-name $LOCK_TABLE \
  --region $REGION

echo "Lock table: $LOCK_TABLE"

# Output backend config
cat > terraform/backend.hcl << EOF
bucket         = "$STATE_BUCKET"
region         = "$REGION"
encrypt        = true
kms_key_id     = "alias/${PROJECT}-terraform-state"
dynamodb_table = "$LOCK_TABLE"
EOF

echo ""
echo "✅ Backend bootstrap complete!"
echo "State bucket:  $STATE_BUCKET"
echo "Lock table:    $LOCK_TABLE"
echo "KMS alias:     alias/${PROJECT}-terraform-state"
echo ""
echo "Backend config written to: terraform/backend.hcl"
SCRIPT

chmod +x scripts/bootstrap-backend.sh
./scripts/bootstrap-backend.sh cloudforge ap-south-1
```

**PART 3 — Terraform Modules**

**Module 1 — VPC**

```
cat > terraform/modules/vpc/main.tf << 'EOF'
# ═══════════════════════════════════════════════════════
# CloudForge VPC Module
# Production-grade VPC with public/private subnets,
# NAT Gateways, VPC Flow Logs, and EKS-ready tagging
# ═══════════════════════════════════════════════════════

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)

  # EKS requires specific tags on subnets for ALB controller
  # and Karpenter to work correctly
  public_subnet_tags = merge(var.tags, {
    "kubernetes.io/role/elb"                    = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  })

  private_subnet_tags = merge(var.tags, {
    "kubernetes.io/role/internal-elb"           = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "karpenter.sh/discovery"                    = var.cluster_name
  })
}

data "aws_availability_zones" "available" {
  state = "available"
}

# ── VPC ──────────────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-vpc"
  })
}

# ── Internet Gateway ──────────────────────────────────────
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-igw"
  })
}

# ── Public Subnets ────────────────────────────────────────
resource "aws_subnet" "public" {
  count = length(local.azs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.public_subnet_tags, {
    Name = "${var.project}-${var.environment}-public-${local.azs[count.index]}"
    Tier = "public"
  })
}

# ── Private Subnets ───────────────────────────────────────
resource "aws_subnet" "private" {
  count = length(local.azs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + length(local.azs))
  availability_zone = local.azs[count.index]

  tags = merge(local.private_subnet_tags, {
    Name = "${var.project}-${var.environment}-private-${local.azs[count.index]}"
    Tier = "private"
  })
}

# ── Database Subnets (isolated, no route to internet) ─────
resource "aws_subnet" "database" {
  count = length(local.azs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + (length(local.azs) * 2))
  availability_zone = local.azs[count.index]

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-db-${local.azs[count.index]}"
    Tier = "database"
  })
}

# ── Elastic IPs for NAT Gateways ─────────────────────────
resource "aws_eip" "nat" {
  count  = var.single_nat_gateway ? 1 : length(local.azs)
  domain = "vpc"

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-nat-eip-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

# ── NAT Gateways ──────────────────────────────────────────
# Production: one per AZ (HA)
# Dev/Staging: single NAT (cost saving)
resource "aws_nat_gateway" "main" {
  count = var.single_nat_gateway ? 1 : length(local.azs)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-nat-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

# ── Route Tables ──────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-public-rt"
  })
}

resource "aws_route_table" "private" {
  count  = var.single_nat_gateway ? 1 : length(local.azs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[var.single_nat_gateway ? 0 : count.index].id
  }

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-private-rt-${count.index + 1}"
  })
}

resource "aws_route_table" "database" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-db-rt"
  })
}

# ── Route Table Associations ──────────────────────────────
resource "aws_route_table_association" "public" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[
    var.single_nat_gateway ? 0 : count.index
  ].id
}

resource "aws_route_table_association" "database" {
  count          = length(local.azs)
  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.database.id
}

# ── VPC Flow Logs ─────────────────────────────────────────
resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/${var.project}-${var.environment}-flow-logs"
  retention_in_days = var.flow_log_retention_days
  kms_key_id        = var.kms_key_arn

  tags = var.tags
}

resource "aws_iam_role" "flow_logs" {
  name = "${var.project}-${var.environment}-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "flow_logs" {
  name = "flow-logs-policy"
  role = aws_iam_role.flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Resource = "*"
    }]
  })
}

resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-flow-logs"
  })
}

# ── S3 VPC Endpoint (free — eliminates NAT cost for S3) ───
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = concat(
    aws_route_table.private[*].id,
    [aws_route_table.database.id]
  )

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-s3-endpoint"
  })
}

# ── DynamoDB VPC Endpoint (free) ──────────────────────────
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-dynamodb-endpoint"
  })
}

# ── ECR Interface Endpoints (private ECR pulls from EKS) ──
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "${var.project}-${var.environment}-vpce-"
  description = "Security group for VPC Interface Endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-vpce-sg"
  })
}

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-ecr-api-endpoint"
  })
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-ecr-dkr-endpoint"
  })
}
EOF
```

```
cat > terraform/modules/vpc/variables.tf << 'EOF'
variable "project"     { type = string }
variable "environment" { type = string }
variable "region"      { type = string }
variable "vpc_cidr"    { type = string }
variable "cluster_name" { type = string }
variable "kms_key_arn"  { type = string }
variable "tags"         { type = map(string) default = {} }

variable "single_nat_gateway" {
  type        = bool
  default     = false
  description = "Use single NAT Gateway (cost saving for dev/staging)"
}

variable "flow_log_retention_days" {
  type    = number
  default = 30
}
EOF

cat > terraform/modules/vpc/outputs.tf << 'EOF'
output "vpc_id"              { value = aws_vpc.main.id }
output "vpc_cidr"            { value = aws_vpc.main.cidr_block }
output "public_subnet_ids"   { value = aws_subnet.public[*].id }
output "private_subnet_ids"  { value = aws_subnet.private[*].id }
output "database_subnet_ids" { value = aws_subnet.database[*].id }
output "nat_gateway_ids"     { value = aws_nat_gateway.main[*].id }
output "internet_gateway_id" { value = aws_internet_gateway.main.id }
EOF
```

**Module 2 — KMS**

```
cat > terraform/modules/kms/main.tf << 'EOF'
# ═══════════════════════════════════════════════════════
# CloudForge KMS Module
# Customer-managed keys per environment
# ═══════════════════════════════════════════════════════

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

resource "aws_kms_key" "main" {
  description             = "${var.project} ${var.environment} encryption key"
  deletion_window_in_days = var.deletion_window_in_days
  enable_key_rotation     = true
  multi_region            = var.multi_region

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowEKSServiceAccount"
        Effect = "Allow"
        Principal = {
          Service = [
            "eks.amazonaws.com",
            "ec2.amazonaws.com",
            "rds.amazonaws.com",
            "secretsmanager.amazonaws.com",
            "logs.amazonaws.com",
            "cloudtrail.amazonaws.com"
          ]
        }
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey",
          "kms:Encrypt",
          "kms:GenerateDataKey*",
          "kms:ReEncrypt*"
        ]
        Resource = "*"
      },
      {
        Sid    = "AllowKeyAdmins"
        Effect = "Allow"
        Principal = {
          AWS = var.key_admin_arns
        }
        Action = [
          "kms:Create*", "kms:Describe*", "kms:Enable*",
          "kms:List*",   "kms:Put*",      "kms:Update*",
          "kms:Revoke*", "kms:Disable*",  "kms:Get*",
          "kms:Delete*", "kms:ScheduleKeyDeletion",
          "kms:CancelKeyDeletion"
        ]
        Resource = "*"
      }
    ]
  })

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-key"
  })
}

resource "aws_kms_alias" "main" {
  name          = "alias/${var.project}-${var.environment}"
  target_key_id = aws_kms_key.main.key_id
}
EOF

cat > terraform/modules/kms/variables.tf << 'EOF'
variable "project"     { type = string }
variable "environment" { type = string }
variable "tags"        { type = map(string) default = {} }
variable "key_admin_arns" {
  type    = list(string)
  default = []
}
variable "deletion_window_in_days" {
  type    = number
  default = 30
}
variable "multi_region" {
  type    = bool
  default = false
}
EOF

cat > terraform/modules/kms/outputs.tf << 'EOF'
output "key_id"   { value = aws_kms_key.main.key_id }
output "key_arn"  { value = aws_kms_key.main.arn }
output "alias"    { value = aws_kms_alias.main.name }
EOF
```

**Module 3 — EKS (The Core Module)**

```
cat > terraform/modules/eks/main.tf << 'EOF'
# ═══════════════════════════════════════════════════════
# CloudForge EKS Module
# Production-grade EKS cluster with:
# - KMS encryption for secrets at rest
# - Private endpoint + public (restricted)
# - IRSA enabled (OIDC)
# - Karpenter-ready node groups
# - CloudWatch control plane logging
# ═══════════════════════════════════════════════════════

data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

locals {
  cluster_oidc_issuer_url = aws_eks_cluster.main.identity[0].oidc[0].issuer
  oidc_provider           = replace(local.cluster_oidc_issuer_url, "https://", "")
}

# ── EKS Cluster IAM Role ──────────────────────────────────
resource "aws_iam_role" "cluster" {
  name = "${var.project}-${var.environment}-eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.cluster.name
}

# ── Security Group for Cluster ────────────────────────────
resource "aws_security_group" "cluster" {
  name_prefix = "${var.project}-${var.environment}-eks-cluster-"
  description = "EKS cluster security group"
  vpc_id      = var.vpc_id

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-eks-cluster-sg"
  })

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group_rule" "cluster_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.cluster.id
  description       = "Allow all egress"
}

# ── EKS Cluster ───────────────────────────────────────────
resource "aws_eks_cluster" "main" {
  name     = "${var.project}-${var.environment}"
  version  = var.kubernetes_version
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = var.public_access_cidrs
    security_group_ids      = [aws_security_group.cluster.id]
  }

  # Encrypt Kubernetes secrets at rest with KMS
  encryption_config {
    resources = ["secrets"]
    provider {
      key_arn = var.kms_key_arn
    }
  }

  # Enable all control plane logging
  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  kubernetes_network_config {
    service_ipv4_cidr = var.service_cidr
    ip_family         = "ipv4"
  }

  access_config {
    authentication_mode                         = "API_AND_CONFIG_MAP"
    bootstrap_cluster_creator_admin_permissions = true
  }

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-eks"
  })

  depends_on = [
    aws_iam_role_policy_attachment.cluster_policy,
  ]
}

# ── OIDC Provider (enables IRSA) ──────────────────────────
data "tls_certificate" "cluster" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "main" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.cluster.certificates[0].sha1_fingerprint]
  url             = local.cluster_oidc_issuer_url

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-oidc"
  })
}

# ── Node IAM Role ─────────────────────────────────────────
resource "aws_iam_role" "node" {
  name = "${var.project}-${var.environment}-eks-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "node_policies" {
  for_each = toset([
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
    "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy",
  ])

  policy_arn = each.value
  role       = aws_iam_role.node.name
}

# ── System Node Group (always on-demand, dedicated for platform) ──
resource "aws_eks_node_group" "system" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "system"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids
  instance_types  = ["m5.large"]
  capacity_type   = "ON_DEMAND"

  scaling_config {
    desired_size = 2
    min_size     = 2
    max_size     = 4
  }

  update_config {
    max_unavailable_percentage = 25
  }

  launch_template {
    id      = aws_launch_template.system.id
    version = aws_launch_template.system.latest_version
  }

  labels = {
    role      = "system"
    node-type = "on-demand"
  }

  taint {
    key    = "CriticalAddonsOnly"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-system-node"
    "k8s.io/cluster-autoscaler/enabled"                         = "true"
    "k8s.io/cluster-autoscaler/${var.project}-${var.environment}" = "owned"
  })

  lifecycle {
    ignore_changes = [scaling_config[0].desired_size]
  }

  depends_on = [aws_iam_role_policy_attachment.node_policies]
}

resource "aws_launch_template" "system" {
  name_prefix   = "${var.project}-${var.environment}-system-"
  instance_type = "m5.large"

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      iops                  = 3000
      throughput            = 125
      encrypted             = true
      kms_key_id            = var.kms_key_arn
      delete_on_termination = true
    }
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"   # IMDSv2 required
    http_put_response_hop_limit = 1
    instance_metadata_tags      = "enabled"
  }

  monitoring {
    enabled = true
  }

  tag_specifications {
    resource_type = "instance"
    tags = merge(var.tags, {
      Name = "${var.project}-${var.environment}-system-node"
    })
  }

  tags = var.tags
}

# ── EKS Add-ons ───────────────────────────────────────────
resource "aws_eks_addon" "vpc_cni" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "vpc-cni"
  addon_version            = var.addon_versions.vpc_cni
  resolve_conflicts_on_update = "OVERWRITE"

  configuration_values = jsonencode({
    env = {
      ENABLE_PREFIX_DELEGATION = "true"
      WARM_PREFIX_TARGET       = "1"
    }
  })

  tags = var.tags
}

resource "aws_eks_addon" "coredns" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "coredns"
  addon_version            = var.addon_versions.coredns
  resolve_conflicts_on_update = "OVERWRITE"

  tags = var.tags

  depends_on = [aws_eks_node_group.system]
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "kube-proxy"
  addon_version            = var.addon_versions.kube_proxy
  resolve_conflicts_on_update = "OVERWRITE"

  tags = var.tags
}

resource "aws_eks_addon" "ebs_csi" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "aws-ebs-csi-driver"
  addon_version            = var.addon_versions.ebs_csi
  service_account_role_arn = aws_iam_role.ebs_csi.arn
  resolve_conflicts_on_update = "OVERWRITE"

  tags = var.tags
}

# ── IRSA Role for EBS CSI Driver ─────────────────────────
resource "aws_iam_role" "ebs_csi" {
  name = "${var.project}-${var.environment}-ebs-csi-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.main.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.oidc_provider}:aud" = "sts.amazonaws.com"
          "${local.oidc_provider}:sub" = "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "ebs_csi" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  role       = aws_iam_role.ebs_csi.name
}

# ── Karpenter IAM Role ────────────────────────────────────
resource "aws_iam_role" "karpenter" {
  name = "${var.project}-${var.environment}-karpenter-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.main.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.oidc_provider}:aud" = "sts.amazonaws.com"
          "${local.oidc_provider}:sub" = "system:serviceaccount:karpenter:karpenter"
        }
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_policy" "karpenter" {
  name = "${var.project}-${var.environment}-karpenter-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:CreateLaunchTemplate",
          "ec2:CreateFleet",
          "ec2:RunInstances",
          "ec2:CreateTags",
          "ec2:TerminateInstances",
          "ec2:DeleteLaunchTemplate",
          "ec2:DescribeLaunchTemplates",
          "ec2:DescribeInstances",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeSubnets",
          "ec2:DescribeImages",
          "ec2:DescribeInstanceTypes",
          "ec2:DescribeInstanceTypeOfferings",
          "ec2:DescribeAvailabilityZones",
          "ec2:DescribeSpotPriceHistory",
          "pricing:GetProducts",
          "ssm:GetParameter"
        ]
        Resource = "*"
      },
      {
        Effect   = "Allow"
        Action   = ["iam:PassRole"]
        Resource = aws_iam_role.node.arn
      },
      {
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:GetQueueUrl",
          "sqs:ReceiveMessage"
        ]
        Resource = "*"
      }
    ]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "karpenter" {
  policy_arn = aws_iam_policy.karpenter.arn
  role       = aws_iam_role.karpenter.name
}

# ── Karpenter SQS Queue (Spot interruption handling) ──────
resource "aws_sqs_queue" "karpenter" {
  name                      = "${var.project}-${var.environment}-karpenter"
  message_retention_seconds = 300
  sqs_managed_sse_enabled   = true

  tags = var.tags
}

resource "aws_sqs_queue_policy" "karpenter" {
  queue_url = aws_sqs_queue.karpenter.url

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = { Service = ["events.amazonaws.com", "sqs.amazonaws.com"] }
        Action    = "sqs:SendMessage"
        Resource  = aws_sqs_queue.karpenter.arn
      }
    ]
  })
}

# EventBridge rules for Spot interruption + instance rebalance
resource "aws_cloudwatch_event_rule" "karpenter_interruption" {
  for_each = {
    spot_interruption = {
      name   = "spot-interruption"
      source = "aws.ec2"
      detail = { "instance-action" = ["terminate"] }
    }
    rebalance = {
      name   = "instance-rebalance"
      source = "aws.ec2"
      detail = {}
    }
  }

  name        = "${var.project}-${var.environment}-karpenter-${each.value.name}"
  description = "Karpenter ${each.value.name} handler"

  event_pattern = jsonencode({
    source      = [each.value.source]
    detail-type = [
      each.key == "spot_interruption"
        ? "EC2 Spot Instance Interruption Warning"
        : "EC2 Instance Rebalance Recommendation"
    ]
  })

  tags = var.tags
}

resource "aws_cloudwatch_event_target" "karpenter_sqs" {
  for_each = aws_cloudwatch_event_rule.karpenter_interruption

  rule = each.value.name
  arn  = aws_sqs_queue.karpenter.arn
}
EOF
```

**PART 4 — Environment Configuration**

```
cat > terraform/environments/staging/main.tf << 'EOF'
# ═══════════════════════════════════════════════════════
# CloudForge — Staging Environment
# ═══════════════════════════════════════════════════════

terraform {
  required_version = ">= 1.9.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }

  backend "s3" {
    key = "staging/terraform.tfstate"
    # bucket, region, kms_key_id, dynamodb_table
    # come from backend.hcl (never hardcode here)
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = local.common_tags
  }
}

locals {
  project     = "cloudforge"
  environment = "staging"
  region      = var.region

  common_tags = {
    Project     = local.project
    Environment = local.environment
    ManagedBy   = "terraform"
    Repository  = "github.com/yourname/cloudforge"
    Owner       = "platform-team"
  }
}

# ── KMS ──────────────────────────────────────────────────
module "kms" {
  source = "../../modules/kms"

  project     = local.project
  environment = local.environment
  tags        = local.common_tags
}

# ── VPC ──────────────────────────────────────────────────
module "vpc" {
  source = "../../modules/vpc"

  project      = local.project
  environment  = local.environment
  region       = local.region
  vpc_cidr     = "10.1.0.0/16"
  cluster_name = "${local.project}-${local.environment}"
  kms_key_arn  = module.kms.key_arn
  tags         = local.common_tags

  # Single NAT for staging (cost saving — not production)
  single_nat_gateway      = true
  flow_log_retention_days = 14
}

# ── EKS ──────────────────────────────────────────────────
module "eks" {
  source = "../../modules/eks"

  project             = local.project
  environment         = local.environment
  vpc_id              = module.vpc.vpc_id
  private_subnet_ids  = module.vpc.private_subnet_ids
  kms_key_arn         = module.kms.key_arn
  kubernetes_version  = "1.30"
  tags                = local.common_tags

  # Staging: allow your IP only
  public_access_cidrs = var.allowed_cidr_blocks
}
EOF

cat > terraform/environments/staging/variables.tf << 'EOF'
variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "allowed_cidr_blocks" {
  type        = list(string)
  description = "CIDRs allowed to reach EKS API server"
}
EOF

cat > terraform/environments/staging/terraform.tfvars << 'EOF'
region = "ap-south-1"
# Replace with your actual IP:
allowed_cidr_blocks = ["YOUR_IP/32"]
EOF
```

**PART 5 — GitHub Actions Workflows**

**Terraform Plan on PR**

```
cat > .github/workflows/terraform-plan.yml << 'EOF'
name: Terraform Plan

on:
  pull_request:
    paths:
      - 'terraform/**'
    branches:
      - main

permissions:
  id-token: write    # OIDC — no stored AWS keys
  contents: read
  pull-requests: write

env:
  TF_VERSION: "1.9.5"
  AWS_REGION: "ap-south-1"

jobs:
  plan:
    name: Terraform Plan — ${{ matrix.environment }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [staging]
      fail-fast: false

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: GitHubActions-TerraformPlan

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        id: fmt
        run: terraform fmt -check -recursive
        working-directory: terraform/

      - name: Terraform Init
        id: init
        run: |
          terraform init \
            -backend-config=../../backend.hcl \
            -input=false
        working-directory: terraform/environments/${{ matrix.environment }}

      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color
        working-directory: terraform/environments/${{ matrix.environment }}

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -input=false \
            -no-color \
            -out=tfplan \
            2>&1 | tee plan_output.txt
          echo "exitcode=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT
        working-directory: terraform/environments/${{ matrix.environment }}
        continue-on-error: true

      - name: Security Scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: terraform/
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Comment PR with Plan
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync(
              'terraform/environments/${{ matrix.environment }}/plan_output.txt',
              'utf8'
            );
            const maxLen = 65000;
            const truncated = plan.length > maxLen
              ? plan.substring(0, maxLen) + '\n...[truncated]'
              : plan;

            const output = `## Terraform Plan — \`${{ matrix.environment }}\`

            #### Format \`${{ steps.fmt.outcome }}\`
            #### Init \`${{ steps.init.outcome }}\`
            #### Validate \`${{ steps.validate.outcome }}\`
            #### Plan \`${{ steps.plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`terraform
            ${truncated}
            \`\`\`

            </details>

            *Pushed by @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Plan Status
        if: steps.plan.outputs.exitcode == '1'
        run: exit 1
EOF
```

**Terraform Apply on Merge**

```
cat > .github/workflows/terraform-apply.yml << 'EOF'
name: Terraform Apply

on:
  push:
    branches:
      - main
    paths:
      - 'terraform/**'

permissions:
  id-token: write
  contents: read

env:
  TF_VERSION: "1.9.5"
  AWS_REGION: "ap-south-1"

jobs:
  apply:
    name: Terraform Apply — ${{ matrix.environment }}
    runs-on: ubuntu-latest
    environment: ${{ matrix.environment }}   # GitHub environment protection rules
    strategy:
      matrix:
        environment: [staging]
      max-parallel: 1   # never apply envs in parallel

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: |
          terraform init \
            -backend-config=../../backend.hcl \
            -input=false
        working-directory: terraform/environments/${{ matrix.environment }}

      - name: Terraform Apply
        run: |
          terraform apply \
            -auto-approve \
            -input=false
        working-directory: terraform/environments/${{ matrix.environment }}

      - name: Notify Slack on Success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Terraform applied successfully to *${{ matrix.environment }}*",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "✅ *Terraform Apply* — `${{ matrix.environment }}`\nBy: ${{ github.actor }}\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
                }
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚨 Terraform FAILED on *${{ matrix.environment }}*"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
EOF
```

**GitHub OIDC — No Stored AWS Keys**

```
# GitHub Actions uses OIDC to assume an IAM role
# No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY ever stored

cat > terraform/global/github-oidc.tf << 'EOF'
# ── GitHub OIDC Provider ──────────────────────────────────
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]

  tags = {
    Name    = "github-actions-oidc"
    Project = "cloudforge"
  }
}

# ── IAM Role for GitHub Actions Terraform ─────────────────
resource "aws_iam_role" "github_terraform" {
  name = "cloudforge-github-terraform-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # Only your repository can assume this role
          "token.actions.githubusercontent.com:sub" = "repo:YOUR_GITHUB_USERNAME/cloudforge:*"
        }
      }
    }]
  })

  tags = {
    Project = "cloudforge"
    Purpose = "github-actions-terraform"
  }
}

resource "aws_iam_role_policy_attachment" "github_terraform" {
  # Scope this down in production — PowerUserAccess for dev speed
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
  role       = aws_iam_role.github_terraform.name
}

output "github_terraform_role_arn" {
  value = aws_iam_role.github_terraform.arn
}
EOF

# Apply the global OIDC config first
cd terraform/global
terraform init
terraform apply -auto-approve

# Copy the role ARN output and add to GitHub secrets
# Settings → Secrets → Actions → New repository secret
# Name: AWS_TERRAFORM_ROLE_ARN
# Value: arn:aws:iam::ACCOUNT_ID:role/cloudforge-github-terraform-role
```

**PART 6 — Deploy the Cluster**

```
# Initialize and apply staging environment
cd terraform/environments/staging

terraform init \
  -backend-config=../../backend.hcl \
  -input=false

terraform validate

terraform plan \
  -var-file=terraform.tfvars \
  -out=tfplan

# Review the plan output carefully before applying
# You should see: VPC, subnets, IGW, NAT, EKS cluster, node groups

terraform apply tfplan

# Takes ~15-20 minutes for EKS cluster creation
```

**Configure kubectl**

```
# Update kubeconfig
aws eks update-kubeconfig \
  --name cloudforge-staging \
  --region ap-south-1 \
  --alias cloudforge-staging

# Verify cluster is accessible
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A

# Expected output:
# NAME                             STATUS   ROLES    AGE   VERSION
# ip-10-1-x-x.ap-south-1.compute  Ready    <none>   2m    v1.30.x
# ip-10-1-x-x.ap-south-1.compute  Ready    <none>   2m    v1.30.x

# Check system pods are running
kubectl get pods -n kube-system

# Check EBS CSI driver
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

**Install Karpenter**

```
# Get cluster values from Terraform output
CLUSTER_NAME=$(terraform output -raw cluster_name)
KARPENTER_ROLE_ARN=$(terraform output -raw karpenter_role_arn)
KARPENTER_QUEUE=$(terraform output -raw karpenter_queue_name)
CLUSTER_ENDPOINT=$(terraform output -raw cluster_endpoint)

export KARPENTER_VERSION=v0.37.0

helm repo add karpenter oci://public.ecr.aws/karpenter/karpenter
helm repo update

helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace karpenter \
  --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.clusterEndpoint=${CLUSTER_ENDPOINT}" \
  --set "settings.interruptionQueue=${KARPENTER_QUEUE}" \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=${KARPENTER_ROLE_ARN}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait

kubectl get pods -n karpenter
```

**Karpenter NodePool and NodeClass**

```
cat > kubernetes/platform/autoscaling/karpenter-nodepool.yaml << 'EOF'
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: cloudforge-staging-eks-node-role
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: cloudforge-staging
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: cloudforge-staging
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        encrypted: true
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 1
    httpTokens: required     # IMDSv2 only
  tags:
    Project: cloudforge
    ManagedBy: karpenter

---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        node-type: karpenter
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
          values: [amd64, arm64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
        - key: karpenter.k8s.aws/instance-size
          operator: NotIn
          values: [nano, micro, small]
      taints: []

  limits:
    cpu: "100"
    memory: 200Gi

  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 720h

---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    metadata:
      labels:
        node-type: spot-worker
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [m, c]
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
      taints:
        - key: spot
          value: "true"
          effect: NoSchedule

  limits:
    cpu: "200"
    memory: 400Gi

  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 60s
    expireAfter: 168h    # Recycle spot nodes weekly
EOF

kubectl apply -f kubernetes/platform/autoscaling/karpenter-nodepool.yaml

# Verify Karpenter is watching
kubectl get nodepools
kubectl get ec2nodeclasses
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=50
```

**StorageClass Setup**

```
cat > kubernetes/platform/storage/storageclasses.yaml << 'EOF'
---
# Remove the default annotation from gp2
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/aws-ebs

---
# gp3 — our new default (faster, cheaper than gp2)
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
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain       # never auto-delete in production
allowVolumeExpansion: true

---
# High IOPS for databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
EOF

kubectl apply -f kubernetes/platform/storage/storageclasses.yaml
kubectl get storageclasses
```

**PART 7 — Week 1 Validation**

```
#!/bin/bash
# scripts/validate-week1.sh
# Run this to verify everything is working

set -euo pipefail

echo "═══════════════════════════════════════"
echo "  CloudForge Week 1 Validation"
echo "═══════════════════════════════════════"

CLUSTER="cloudforge-staging"
PASS=0
FAIL=0

check() {
  local name=$1
  local cmd=$2
  if eval "$cmd" &>/dev/null; then
    echo "  ✅ $name"
    ((PASS++))
  else
    echo "  ❌ $name"
    ((FAIL++))
  fi
}

echo ""
echo "── Cluster Access ──────────────────────"
check "kubectl cluster-info" "kubectl cluster-info"
check "Nodes ready (>=2)" \
  "[ $(kubectl get nodes --no-headers | grep -c Ready) -ge 2 ]"

echo ""
echo "── System Pods ─────────────────────────"
check "CoreDNS running" \
  "kubectl get pods -n kube-system -l k8s-app=kube-dns --no-headers | grep -q Running"
check "EBS CSI running" \
  "kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver --no-headers | grep -q Running"
check "kube-proxy running" \
  "kubectl get pods -n kube-system -l k8s-app=kube-proxy --no-headers | grep -q Running"

echo ""
echo "── Karpenter ───────────────────────────"
check "Karpenter pods running" \
  "kubectl get pods -n karpenter --no-headers | grep -q Running"
check "NodePool exists" \
  "kubectl get nodepools default &>/dev/null"
check "EC2NodeClass exists" \
  "kubectl get ec2nodeclasses default &>/dev/null"

echo ""
echo "── Storage ─────────────────────────────"
check "gp3 is default StorageClass" \
  "kubectl get sc gp3 -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}' | grep -q true"

echo ""
echo "── AWS Resources ───────────────────────"
check "EKS cluster active" \
  "aws eks describe-cluster --name $CLUSTER --query 'cluster.status' --output text | grep -q ACTIVE"
check "S3 state bucket exists" \
  "aws s3 ls | grep -q cloudforge-terraform-state"
check "DynamoDB lock table exists" \
  "aws dynamodb describe-table --table-name cloudforge-terraform-locks &>/dev/null"

echo ""
echo "── Terraform ───────────────────────────"
check "State is not empty" \
  "terraform show -json 2>/dev/null | python3 -c 'import sys,json; d=json.load(sys.stdin); exit(0 if d[\"values\"] else 1)'"

echo ""
echo "═══════════════════════════════════════"
echo "  Results: ✅ $PASS passed | ❌ $FAIL failed"
echo "═══════════════════════════════════════"

[ $FAIL -eq 0 ] && echo "  🎉 Week 1 Complete!" || echo "  ⚠️  Fix failures before Week 2"
```

```
chmod +x scripts/validate-week1.sh
./scripts/validate-week1.sh
```

**PART 8 — ADRs (Architecture Decision Records)**

These go in your README and show recruiters you think like an architect.

```
cat > docs/adr/001-gitops.md << 'EOF'
# ADR-001: GitOps over Push-Based CI/CD

**Status:** Accepted
**Date:** 2024-01-15

## Context
Need a deployment model for Kubernetes workloads across
staging and production environments.

## Decision
Use ArgoCD (pull-based GitOps) over push-based deployments
from CI pipelines directly to Kubernetes.

## Rationale
- Git is the single source of truth — any drift is detectable
- ArgoCD continuously reconciles — accidental manual changes auto-revert
- Audit trail: every deployment is a git commit with author
- Separation of concerns: CI builds artifacts, ArgoCD deploys
- Easier multi-cluster: ArgoCD manages multiple clusters from one place

## Consequences
- ArgoCD must be running and healthy for deployments to work
- Developers need to understand GitOps flow (PR → merge → deploy)
- Image tag updates need ArgoCD Image Updater (automated)
EOF

cat > docs/adr/002-karpenter.md << 'EOF'
# ADR-002: Karpenter over Cluster Autoscaler

**Status:** Accepted

## Context
EKS cluster needs automatic node provisioning for variable workloads.

## Decision
Use Karpenter instead of Kubernetes Cluster Autoscaler (CAS).

## Rationale
- Karpenter provisions exactly the right instance for pending pods (<60s)
- CAS scales existing node groups — limited to pre-defined instance types
- Karpenter considers CPU, memory, GPU, architecture, spot availability
- Consolidation: Karpenter actively removes underutilized nodes
- Cost: 40-60% lower compute cost through right-sizing

## Consequences
- Karpenter is AWS-specific (not portable to GKE/AKS)
- Requires EC2NodeClass and NodePool CRDs (learning curve)
- Need to handle Spot interruptions (built into Karpenter via SQS)
EOF

cat > docs/adr/003-lgtm-stack.md << 'EOF'
# ADR-003: LGTM Stack over ELK

**Status:** Accepted

## Context
Need a full observability stack: metrics, logs, traces.

## Decision
Use Grafana LGTM stack (Loki + Grafana + Tempo + Mimir)
over ELK stack (Elasticsearch + Logstash + Kibana).

## Rationale
- Single UI (Grafana) for metrics, logs, and traces — correlated views
- Loki: label-based log indexing — 10x cheaper than Elasticsearch at scale
- Tempo: trace storage integrates natively with Grafana
- OpenTelemetry: vendor-neutral instrumentation (future-proof)
- Grafana Cloud: managed option available (exit strategy)
- ELK: expensive at scale, Elasticsearch memory-heavy

## Consequences
- Loki query language (LogQL) is different from Elasticsearch DSL
- Less full-text search capability than Elasticsearch
- Team needs to learn Grafana + LogQL + PromQL + TraceQL
EOF
```

**Week 1 Commit and Push**

```
cd ~/cloudforge

git add .
git commit -m "feat: Week 1 — Terraform foundation + EKS cluster

Infrastructure:
- Terraform modules: VPC, KMS, EKS (production-grade)
- S3 backend + DynamoDB locking (encrypted, versioned)
- EKS 1.30 cluster (KMS secrets encryption, private endpoint)
- System node group (on-demand, tainted for platform workloads)
- Karpenter (node autoscaling, Spot interruption handling)
- VPC endpoints for S3, DynamoDB, ECR (no NAT for AWS APIs)
- VPC Flow Logs (CloudWatch, encrypted)
- gp3 default StorageClass

CI/CD:
- GitHub Actions: terraform plan on PR (with PR comment)
- GitHub Actions: terraform apply on merge to main
- GitHub OIDC: no stored AWS credentials
- Trivy security scan on terraform configs

Documentation:
- ADR-001: GitOps decision
- ADR-002: Karpenter decision
- ADR-003: LGTM stack decision
- Validation script

Validated:
- kubectl get nodes ✅
- Karpenter NodePool ✅
- gp3 default StorageClass ✅
- GitHub Actions pipeline ✅"

git push origin feat/week1-foundation

# Open PR
gh pr create \
  --title "feat: Week 1 — Terraform Foundation + EKS Cluster" \
  --body "## Week 1: CloudForge Foundation

### What's included
- Production-grade Terraform modules (VPC, EKS, KMS)
- EKS 1.30 with Karpenter node autoscaling
- GitHub OIDC (zero stored credentials)
- Automated terraform plan/apply via GitHub Actions
- Architecture Decision Records

### Validation
\`\`\`
✅ 12/12 checks passed
\`\`\`

### Next Week
Week 2: ArgoCD GitOps layer + ArgoCD Image Updater"
```

## Week 1 Summary

```
What you built:
├── ✅ GitHub repo (branch protection, pre-commit, OIDC)
├── ✅ Terraform backend (S3 + DynamoDB, KMS encrypted)
├── ✅ VPC module (3-AZ, public/private/db subnets, flow logs, endpoints)
├── ✅ KMS module (CMK, rotation, service access)
├── ✅ EKS module (1.30, KMS encryption, IRSA, Karpenter IAM)
├── ✅ EKS cluster (system nodes, add-ons, IMDSv2 enforced)
├── ✅ Karpenter (NodePool, EC2NodeClass, Spot interruption SQS)
├── ✅ StorageClass (gp3 default, io2 for databases)
├── ✅ GitHub Actions (plan on PR, apply on merge, OIDC, Trivy)
└── ✅ ADRs (GitOps, Karpenter, LGTM decisions documented)

What recruiters see:
├── Karpenter (not CA — shows you're current)
├── IMDSv2 enforced (security-first)
├── KMS encrypted secrets at rest
├── OIDC auth (no stored credentials)
├── Trivy in CI (security gate)
└── ADRs (architect-level thinking)
```




