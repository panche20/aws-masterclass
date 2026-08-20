# 🏗️ AWS Mastery Curriculum — Main Roadmap

## Duration: ~8 Weeks | Goal: DevOps Engineer → Cloud Architect

**Phase 1 — Foundations (Week 1–2)**

Core AWS building blocks. No fluff, real hands-on from Day 1.

- IAM (users, roles, policies, least privilege)
- EC2 (instances, security groups, key pairs, user data)
- VPC (subnets, route tables, IGW, NAT Gateway)
- S3 (buckets, versioning, lifecycle, static hosting)
- RDS & DynamoDB basics
- CloudWatch basics (logs, alarms, metrics)

**Phase 2 — DevOps Core (Week 3–4)**

What interviewers actually test for DevOps roles.

- ECS & ECR (containers on AWS, Fargate)
- EKS (Kubernetes on AWS — leverages your CKA/CKAD knowledge)
- CodePipeline, CodeBuild, CodeDeploy (CI/CD)
- Elastic Load Balancing + Auto Scaling Groups
- CloudFormation (IaC — compare with your Terraform knowledge)
- Systems Manager (SSM), Secrets Manager, Parameter Store
- Route 53 (DNS, health checks, routing policies)

**Phase 3 — Intermediate / Production Patterns (Week 5–6)**

Real-world architecture patterns used in production.

- Multi-account strategy (AWS Organizations, SCPs)
- Advanced VPC (VPC Peering, Transit Gateway, PrivateLink)
- Security deep-dive (GuardDuty, Security Hub, WAF, Shield)
- Observability stack (CloudWatch Container Insights, X-Ray, OpenTelemetry)
- Cost optimization strategies (Savings Plans, Spot Instances, Trusted Advisor)
- Advanced S3 (replication, encryption, presigned URLs, S3 Select)

**Phase 4 — Advanced & Serverless (Week 7)**

Cutting-edge patterns + serverless architectures.

- Lambda (functions, layers, event sources, cold starts)
- API Gateway (REST + HTTP APIs)
- SQS, SNS, EventBridge (event-driven patterns)
- Step Functions (orchestration)
- DynamoDB advanced (GSI, LSI, streams, DAX)
- ElastiCache (Redis on AWS)

**Phase 5 — Cloud Architect Capstone (Week 8)**

Architect-level thinking: design, trade-offs, cost, resilience.

- Well-Architected Framework (all 6 pillars)
- Designing highly available, fault-tolerant systems
- Disaster Recovery strategies (RPO/RTO, pilot light, warm standby, multi-region active-active)
- Migration strategies (6 R's: Rehost, Replatform, Refactor…)
- Capstone Project: Design and deploy a production-grade 3-tier architecture on AWS from scratch

**Certifications This Prepares You For**

- ✅ AWS Solutions Architect Associate (SAA-C03) — after Phase 3
- ✅ AWS DevOps Engineer Professional — after Phase 4
- ✅ AWS Solutions Architect Professional — after Phase 5


# 🏗️ AWS Phase 1 — Complete Foundations

## PART 1 — IAM (Identity & Access Management)

IAM is the security backbone of AWS. Every API call, every service interaction goes through IAM.

**Core Concepts**

**The IAM hierarchy:**

```
AWS Account (root)
    └── Users (human identities)
    └── Groups (collection of users)
    └── Roles (assumed by services/applications)
    └── Policies (JSON documents defining permissions)
```

**The golden rule**: Least Privilege — grant only what is needed, nothing more.

**IAM Policy Structure**

Every IAM policy is a JSON document with this anatomy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "eu-west-1"
        }
      }
    }
  ]
}
```

- **Effect** — Allow or Deny. Explicit Deny always wins.
- **Action** — what API calls are permitted (e.g. s3:GetObject, ec2:*)
- **Resource** — which specific ARN(s) this applies to. * means all.
- **Condition** — optional constraints (IP, region, MFA, time, tags)

**Types of Policies**

<img width="933" height="297" alt="image" src="https://github.com/user-attachments/assets/cb092c9a-a246-41b6-a0fc-da051c1808dc" />

**IAM Roles — The Most Important Concept**

Roles are not users. They are assumed temporarily by:

- EC2 instances (Instance Profiles)
- Lambda functions
- ECS tasks
- Cross-account access
- Federated users (SSO, SAML)

When EC2 assumes a role, it gets **temporary credentials** via the metadata service — no hardcoded keys ever.

**Hands-On: IAM via AWS CLI**

First, configure your CLI on your EC2 instance:

```
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version

# Configure (use your IAM user keys)
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region name: ap-south-1   (Mumbai - closest to India)
# Default output format: json
```

```
# List all IAM users
aws iam list-users

# Create a new IAM user
aws iam create-user --user-name devops-chetan

# Create a group
aws iam create-group --group-name DevOps-Engineers

# Add user to group
aws iam add-user-to-group \
  --user-name devops-chetan \
  --group-name DevOps-Engineers

# Attach AWS managed policy to group
aws iam attach-group-policy \
  --group-name DevOps-Engineers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# Create a custom policy file
cat > s3-readonly-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::*"]
    }
  ]
}
EOF

# Create the policy in AWS
aws iam create-policy \
  --policy-name S3ReadOnlyCustom \
  --policy-document file://s3-readonly-policy.json

# Create a role for EC2 (trust policy first)
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name EC2-S3-Access-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach S3 policy to the role
aws iam attach-role-policy \
  --role-name EC2-S3-Access-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create instance profile and attach role (needed for EC2)
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-Profile

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-Profile \
  --role-name EC2-S3-Access-Role
```

### IAM Interview Questions

**Q: What's the difference between an IAM role and an IAM user?**
A: Users have permanent credentials (access keys/passwords). Roles have no credentials — they issue temporary tokens via STS when assumed. Services like EC2 and Lambda always use roles, never users.

**Q: If an explicit Allow and explicit Deny both exist for the same action, what happens?**
A: Deny wins. Always. The order of evaluation: Explicit Deny → Explicit Allow → Implicit Deny (default).

**Q: How does an EC2 instance access S3 without hardcoding keys?**
A: Via an IAM Instance Profile. The role is attached to EC2, and the app retrieves temporary credentials from http://169.254.169.254/latest/meta-data/iam/security-credentials/.

**PART 2 — EC2 (Elastic Compute Cloud)**

EC2 is virtual machines on AWS. Understanding EC2 deeply means understanding compute, networking, storage, and security together.

**Instance Types — The Naming Convention**

```
m5.xlarge
│ │  └── Size: nano, micro, small, medium, large, xlarge, 2xlarge...
│ └──── Generation: 5 (higher = newer, better price/perf)
└────── Family:
        m = general purpose
        c = compute optimized
        r = memory optimized
        t = burstable (t3, t3a — great for dev/test)
        p = GPU (ML workloads)
        i = storage optimized (NVMe SSDs)
        g = GPU graphics
```

For your EC2 Ubuntu instance: you're likely on t3.micro or t3.small (free tier eligible).

**Key EC2 Concepts**

**AMI (Amazon Machine Image):** Snapshot of an OS + software that becomes your instance. You can create custom AMIs from a running instance.

**Security Groups:** Stateful firewalls at the instance level. Inbound rules control what comes in. Outbound is allow-all by default. Stateful means return traffic is automatically allowed.

**Key Pairs:** SSH access. AWS stores the public key, you keep the private .pem. Never lose it.

**User Data:** Shell script that runs once at first boot. Used for bootstrapping (installing packages, starting services).

**EBS (Elastic Block Store):** Persistent block storage attached to EC2. Like a hard drive. Survives instance stop/start (but not termination unless you uncheck "delete on termination").

**Hands-On: Launch and Manage EC2**

```
# Describe your running instances
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,InstanceType,PublicIpAddress]' \
  --output table

# List available AMIs (Amazon Linux 2023 in ap-south-1)
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'Images[0].ImageId' \
  --output text

# Create a security group
aws ec2 create-security-group \
  --group-name devops-sg \
  --description "DevOps practice security group"

# Save the GroupId from output, then add rules
aws ec2 authorize-security-group-ingress \
  --group-name devops-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-name devops-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-name devops-sg \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Launch an instance with user data bootstrap script
cat > userdata.sh << 'EOF'
#!/bin/bash
yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx
echo "<h1>Hello from $(hostname)</h1>" > /usr/share/nginx/html/index.html
EOF

aws ec2 run-instances \
  --image-id ami-0f58b397bc5c1f2e8 \
  --instance-type t3.micro \
  --key-name your-key-pair-name \
  --security-groups devops-sg \
  --user-data file://userdata.sh \
  --iam-instance-profile Name=EC2-S3-Profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-practice},{Key=Environment,Value=dev}]' \
  --count 1

# Stop and start an instance (preserves EBS)
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 start-instances --instance-ids i-0123456789abcdef0

# Create an AMI from running instance
aws ec2 create-image \
  --instance-id i-0123456789abcdef0 \
  --name "devops-practice-ami-$(date +%Y%m%d)" \
  --description "Custom AMI for DevOps practice"

# Describe EBS volumes
aws ec2 describe-volumes \
  --query 'Volumes[].[VolumeId,Size,State,AvailabilityZone]' \
  --output table

# Create and attach an additional EBS volume
aws ec2 create-volume \
  --size 20 \
  --availability-zone ap-south-1a \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=data-volume}]'

# After volume is created (copy VolumeId from output):
aws ec2 attach-volume \
  --volume-id vol-0123456789abcdef0 \
  --instance-id i-0123456789abcdef0 \
  --device /dev/sdf
```

```
# On the instance itself — format and mount the new volume
lsblk                          # see all block devices
sudo mkfs.ext4 /dev/xvdf       # format (only first time!)
sudo mkdir -p /data
sudo mount /dev/xvdf /data

# Make mount persistent across reboots
echo '/dev/xvdf /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

**EC2 Pricing Models — Must Know for Interviews**

<img width="921" height="297" alt="image" src="https://github.com/user-attachments/assets/dbd1bd4e-dced-4d10-9f07-9168ab47ec11" />

**Spot interruption:** AWS can reclaim Spot with 2-minute warning. Design stateless workloads for Spot.

**EC2 Interview Questions**

**Q: What's the difference between stopping and terminating an EC2 instance?**
A: Stop preserves the EBS root volume — data survives. Terminate deletes the root EBS volume by default (unless you uncheck it). Stopped instances don't incur compute costs but do incur EBS storage costs.

**Q: How would you make EC2 highly available?**
A: Launch instances across multiple Availability Zones, put them behind an Application Load Balancer, and use an Auto Scaling Group to maintain desired capacity automatically.

**Q: What is EC2 instance metadata and how do you access it?**
A: It's instance-specific data available at 169.254.169.254. You access it with:

```
# IMDSv2 (secure, token-based — always use this)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**PART 3 — VPC (Virtual Private Cloud)**

VPC is your private network on AWS. Everything runs inside a VPC. This is where most DevOps and architect interviews go deep.

**VPC Architecture — The Big Picture**

```
AWS Region (ap-south-1)
└── VPC (10.0.0.0/16)
    ├── Availability Zone A (ap-south-1a)
    │   ├── Public Subnet  (10.0.1.0/24)  ← has route to IGW
    │   └── Private Subnet (10.0.2.0/24)  ← no direct internet
    ├── Availability Zone B (ap-south-1b)
    │   ├── Public Subnet  (10.0.3.0/24)
    │   └── Private Subnet (10.0.4.0/24)
    ├── Internet Gateway   ← allows internet in/out for public subnets
    ├── NAT Gateway        ← allows private subnets to call out, not in
    ├── Route Tables       ← rules: where does traffic go?
    └── Security Groups / NACLs ← firewalls
```

**Key Components**

**CIDR Block:** The IP range of your VPC. /16 gives 65,536 IPs. Subnets carve this up.

**Internet Gateway (IGW):** Attached to the VPC. Enables two-way internet for public subnets.

**NAT Gateway:** Lives in public subnet. Lets private subnet resources (like RDS, app servers) make outbound calls to the internet (for updates, API calls) without being reachable inbound.

**Route Table:** Every subnet has one. Rules say: "traffic to 10.0.0.0/16 stays local; traffic to 0.0.0.0/0 goes to IGW (public) or NAT (private)."

**Security Groups vs NACLs:**

<img width="907" height="252" alt="image" src="https://github.com/user-attachments/assets/fe5d705d-fe72-40e9-95f2-ef41886c1544" />

**Hands-On: Build a Production VPC from Scratch**

```
# Create the VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' \
  --output text)

aws ec2 create-tags \
  --resources $VPC_ID \
  --tags Key=Name,Value=devops-vpc

# Enable DNS hostnames (needed for RDS, EKS)
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames

echo "VPC created: $VPC_ID"

# Create public subnets in two AZs
PUB_SUBNET_A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --query 'Subnet.SubnetId' \
  --output text)

PUB_SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-1b \
  --query 'Subnet.SubnetId' \
  --output text)

# Create private subnets
PRIV_SUBNET_A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-1a \
  --query 'Subnet.SubnetId' \
  --output text)

PRIV_SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.4.0/24 \
  --availability-zone ap-south-1b \
  --query 'Subnet.SubnetId' \
  --output text)

# Tag all subnets
aws ec2 create-tags --resources $PUB_SUBNET_A \
  --tags Key=Name,Value=public-subnet-a Key=Type,Value=public
aws ec2 create-tags --resources $PUB_SUBNET_B \
  --tags Key=Name,Value=public-subnet-b Key=Type,Value=public
aws ec2 create-tags --resources $PRIV_SUBNET_A \
  --tags Key=Name,Value=private-subnet-a Key=Type,Value=private
aws ec2 create-tags --resources $PRIV_SUBNET_B \
  --tags Key=Name,Value=private-subnet-b Key=Type,Value=private

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_A \
  --map-public-ip-on-launch
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_B \
  --map-public-ip-on-launch

# Create and attach Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

aws ec2 create-tags --resources $IGW_ID \
  --tags Key=Name,Value=devops-igw

# Create public route table
PUB_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Add default route to IGW
aws ec2 create-route \
  --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate public subnets with public route table
aws ec2 associate-route-table \
  --subnet-id $PUB_SUBNET_A \
  --route-table-id $PUB_RT

aws ec2 associate-route-table \
  --subnet-id $PUB_SUBNET_B \
  --route-table-id $PUB_RT

aws ec2 create-tags --resources $PUB_RT \
  --tags Key=Name,Value=public-route-table

# Allocate Elastic IP for NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

# Create NAT Gateway in public subnet A
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET_A \
  --allocation-id $EIP_ALLOC \
  --query 'NatGateway.NatGatewayId' \
  --output text)

echo "NAT Gateway: $NAT_GW (takes ~2 minutes to become available)"

# Wait for NAT gateway to be available
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# Create private route table with NAT as default route
PRIV_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)

aws ec2 create-route \
  --route-table-id $PRIV_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW

# Associate private subnets
aws ec2 associate-route-table \
  --subnet-id $PRIV_SUBNET_A \
  --route-table-id $PRIV_RT

aws ec2 associate-route-table \
  --subnet-id $PRIV_SUBNET_B \
  --route-table-id $PRIV_RT

aws ec2 create-tags --resources $PRIV_RT \
  --tags Key=Name,Value=private-route-table

echo "VPC setup complete!"
echo "VPC: $VPC_ID"
echo "Public Subnets: $PUB_SUBNET_A, $PUB_SUBNET_B"
echo "Private Subnets: $PRIV_SUBNET_A, $PRIV_SUBNET_B"
```

*⚠️ **Cost note**: NAT Gateway charges ~$0.045/hour + data. Delete it after practice with aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW. Also release the EIP.*

**VPC Endpoints — Connecting to AWS Services Privately**

Without VPC Endpoints, traffic to S3/DynamoDB from your private subnet goes through the internet (via NAT). Endpoints keep traffic inside AWS backbone.

```
# Create S3 Gateway Endpoint (free — no hourly charge)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.s3 \
  --route-table-ids $PRIV_RT \
  --vpc-endpoint-type Gateway
```

**VPC Interview Questions**

**Q: What is the difference between a Security Group and a NACL?**
A: Security Groups are stateful instance-level firewalls — allow-only, return traffic automatic. NACLs are stateless subnet-level firewalls — support both allow and deny, and you must explicitly allow both inbound and outbound for bidirectional traffic. NACLs evaluate rules in number order; first match wins.

**Q: Can two VPCs communicate? How?**
A: Yes, via VPC Peering (direct 1-to-1, non-transitive) or Transit Gateway (hub-and-spoke, transitive, scales to thousands of VPCs). CIDR ranges must not overlap.

**Q: Why would you use a NAT Gateway instead of a NAT Instance?**
A: NAT Gateway is managed by AWS — no patching, auto-scales, highly available within an AZ. NAT Instance is an EC2 you manage yourself, cheaper but more operational burden. For production, always NAT Gateway.

**PART 4 — S3 (Simple Storage Service)**

S3 is object storage. Not a file system, not a block device — objects stored in buckets, retrieved by key (path).

**Core Concepts**

- **Bucket:** Global namespace container. Name must be globally unique across all AWS accounts.
- **Object:** File + metadata. Max size 5TB. Upload >5GB must use multipart upload.
- **Key:** The "path" to your object (e.g. logs/2024/01/app.log).
- **Region:** Buckets live in one region. Data doesn't leave unless you configure replication.

**Storage Classes — Cost vs Access Trade-off**

<img width="930" height="392" alt="image" src="https://github.com/user-attachments/assets/8a55ce07-784a-4f4a-8a77-c5053ae2a43c" />

**Hands-On: S3 Operations**

```
# Create a bucket (name must be globally unique)
BUCKET_NAME="devops-chetan-$(date +%s)"
aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

echo "Bucket: $BUCKET_NAME"

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Block all public access (security best practice)
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Enable server-side encryption (SSE-S3 by default)
aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Upload files
echo "Hello from Chetan" > test.txt
aws s3 cp test.txt s3://$BUCKET_NAME/test.txt
aws s3 cp test.txt s3://$BUCKET_NAME/logs/2024/test.txt

# List objects
aws s3 ls s3://$BUCKET_NAME --recursive

# Sync a local directory to S3
mkdir -p my-app-files
echo "file1" > my-app-files/file1.txt
echo "file2" > my-app-files/file2.txt
aws s3 sync ./my-app-files s3://$BUCKET_NAME/app/

# Download from S3
aws s3 cp s3://$BUCKET_NAME/test.txt downloaded-test.txt

# Generate a presigned URL (time-limited access — no auth needed)
aws s3 presign s3://$BUCKET_NAME/test.txt \
  --expires-in 3600

# List object versions (versioning is on)
aws s3api list-object-versions \
  --bucket $BUCKET_NAME \
  --prefix test.txt

# Set a lifecycle policy (move to IA after 30 days, delete after 365)
aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET_NAME \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "transition-and-expire",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Transitions": [
          {
            "Days": 30,
            "StorageClass": "STANDARD_IA"
          },
          {
            "Days": 90,
            "StorageClass": "GLACIER"
          }
        ],
        "Expiration": {
          "Days": 365
        },
        "NoncurrentVersionExpiration": {
          "NoncurrentDays": 30
        }
      }
    ]
  }'

# Enable access logging
aws s3api put-bucket-logging \
  --bucket $BUCKET_NAME \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "'$BUCKET_NAME'",
      "TargetPrefix": "access-logs/"
    }
  }'
```

**S3 Bucket Policy (Resource-Based)**

```
# Allow only your specific EC2 role to access the bucket
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2RoleOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${ACCOUNT_ID}:role/EC2-S3-Access-Role"
      },
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}",
        "arn:aws:s3:::${BUCKET_NAME}/*"
      ]
    },
    {
      "Sid": "DenyHTTP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file://bucket-policy.json
```

**Static Website Hosting**

```
# Create a separate public bucket for static site
SITE_BUCKET="devops-chetan-site-$(date +%s)"

aws s3api create-bucket \
  --bucket $SITE_BUCKET \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Remove public access block (needed for public website)
aws s3api delete-public-access-block --bucket $SITE_BUCKET

# Enable static website hosting
aws s3 website s3://$SITE_BUCKET \
  --index-document index.html \
  --error-document error.html

# Create index page
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<body>
  <h1>Chetan's DevOps Portfolio</h1>
  <p>Hosted on S3!</p>
</body>
</html>
EOF

# Upload with public read
aws s3 cp index.html s3://$SITE_BUCKET/index.html \
  --acl public-read

echo "Site URL: http://$SITE_BUCKET.s3-website.ap-south-1.amazonaws.com"
```

**S3 Interview Questions**

**Q: S3 is object storage — what does that mean practically?**
A: You read and write whole objects via HTTP (GET/PUT). You can't partially update a file like you would on a filesystem — you must rewrite the whole object. There are no directories, only key prefixes that look like paths.

**Q: How would you secure sensitive data in S3?**
A: Block all public access. Use bucket policies with least-privilege. Enable SSE-KMS encryption with a customer-managed key. Enable versioning and MFA Delete. Enable CloudTrail for audit logging. Use VPC Endpoints so traffic never leaves AWS network.

**Q: What is S3 consistency model?**
A: Since Dec 2020, S3 provides strong read-after-write consistency for all operations. After a PUT or DELETE, any subsequent GET or LIST will reflect the change immediately.

**PART 5 — RDS & DynamoDB**

**RDS (Relational Database Service)**

RDS is managed relational databases. AWS handles patching, backups, failover. You handle schema and queries.

**Supported engines**: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Aurora (AWS-native).

**Aurora deserves special mention:** AWS's cloud-native engine, compatible with MySQL and PostgreSQL, but 5x faster than MySQL, auto-scales storage to 128TB, and has up to 15 read replicas.

**RDS Key Concepts**

**Multi-AZ:** Synchronous replication to a standby in another AZ. Automatic failover in ~60-120 seconds. Used for high availability, not for read scaling.

**Read Replicas:** Asynchronous replication. Used for read scaling — offload SELECT queries. Can be cross-region. Can be promoted to standalone (for DR).

**Automated Backups:** Daily snapshots + transaction logs. Point-in-time restore to any second within retention period (1-35 days).

**Hands-On: Launch RDS**

```
# Create a DB subnet group (RDS needs subnets in 2+ AZs)
aws rds create-db-subnet-group \
  --db-subnet-group-name devops-db-subnets \
  --db-subnet-group-description "Subnet group for RDS" \
  --subnet-ids $PRIV_SUBNET_A $PRIV_SUBNET_B

# Create a security group for RDS
RDS_SG=$(aws ec2 create-security-group \
  --group-name rds-sg \
  --description "RDS Security Group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# Allow PostgreSQL from EC2 security group only
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp \
  --port 5432 \
  --source-group devops-sg

# Launch RDS PostgreSQL (db.t3.micro is free tier)
aws rds create-db-instance \
  --db-instance-identifier devops-postgres \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username devopsadmin \
  --master-user-password "SecurePass123!" \
  --allocated-storage 20 \
  --storage-type gp2 \
  --db-subnet-group-name devops-db-subnets \
  --vpc-security-group-ids $RDS_SG \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --multi-az \
  --storage-encrypted \
  --tags Key=Name,Value=devops-postgres Key=Environment,Value=dev

# Wait for it to be available (takes ~10 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier devops-postgres

# Get the endpoint
aws rds describe-db-instances \
  --db-instance-identifier devops-postgres \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text

# Create a read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier devops-postgres-replica \
  --source-db-instance-identifier devops-postgres \
  --db-instance-class db.t3.micro

# Describe automated backups
aws rds describe-db-instance-automated-backups \
  --db-instance-identifier devops-postgres
```

**DynamoDB**

DynamoDB is a fully managed, serverless NoSQL database. Single-digit millisecond latency at any scale.

**Key concepts:**

- **Table:** Collection of items (like a table in SQL)
- **Item:** A record (like a row)
- **Attribute:** A field (like a column, but flexible — each item can have different attributes)
- **Partition Key:** Hash key that determines which partition holds the data
- **Sort Key (optional):** Range key within a partition — enables range queries
- **GSI (Global Secondary Index):** Query on non-key attributes

**Hands-On: DynamoDB**

```
# Create a table for a URL shortener (from your FastAPI project!)
aws dynamodb create-table \
  --table-name url-shortener \
  --attribute-definitions \
    AttributeName=short_code,AttributeType=S \
    AttributeName=created_at,AttributeType=S \
  --key-schema \
    AttributeName=short_code,KeyType=HASH \
    AttributeName=created_at,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=Project,Value=url-shortener

# Wait for table to be active
aws dynamodb wait table-exists --table-name url-shortener

# Put an item
aws dynamodb put-item \
  --table-name url-shortener \
  --item '{
    "short_code": {"S": "abc123"},
    "created_at": {"S": "2024-01-15T10:00:00Z"},
    "original_url": {"S": "https://google.com"},
    "clicks": {"N": "0"},
    "owner": {"S": "chetan"}
  }'

# Get an item by primary key
aws dynamodb get-item \
  --table-name url-shortener \
  --key '{
    "short_code": {"S": "abc123"},
    "created_at": {"S": "2024-01-15T10:00:00Z"}
  }'

# Update an attribute (increment click counter atomically!)
aws dynamodb update-item \
  --table-name url-shortener \
  --key '{
    "short_code": {"S": "abc123"},
    "created_at": {"S": "2024-01-15T10:00:00Z"}
  }' \
  --update-expression "SET clicks = clicks + :inc" \
  --expression-attribute-values '{":inc": {"N": "1"}}'

# Add a GSI for querying by owner
aws dynamodb update-table \
  --table-name url-shortener \
  --attribute-definitions \
    AttributeName=short_code,AttributeType=S \
    AttributeName=created_at,AttributeType=S \
    AttributeName=owner,AttributeType=S \
  --global-secondary-index-updates \
    '[{
      "Create": {
        "IndexName": "owner-index",
        "KeySchema": [
          {"AttributeName": "owner", "KeyType": "HASH"},
          {"AttributeName": "created_at", "KeyType": "RANGE"}
        ],
        "Projection": {"ProjectionType": "ALL"}
      }
    }]'

# Query using GSI
aws dynamodb query \
  --table-name url-shortener \
  --index-name owner-index \
  --key-condition-expression "#o = :owner" \
  --expression-attribute-names '{"#o": "owner"}' \
  --expression-attribute-values '{":owner": {"S": "chetan"}}'

# Enable TTL (auto-delete expired items)
aws dynamodb update-time-to-live \
  --table-name url-shortener \
  --time-to-live-specification "Enabled=true, AttributeName=expires_at"
```

**RDS vs DynamoDB — When to Use What**

<img width="972" height="350" alt="image" src="https://github.com/user-attachments/assets/effe4207-2187-453d-9707-5c8e8104de9b" />

**PART 6 — CloudWatch (Monitoring & Observability)**

CloudWatch is AWS's native observability service. As a DevOps engineer, you live in CloudWatch.

**Core Components**

**Metrics:** Numeric time-series data. EC2 sends CPU, network, disk metrics every 5 minutes (1 min with detailed monitoring enabled).

**Logs:** Log groups → log streams → log events. Your app pushes logs here via CloudWatch Agent or SDK.

**Alarms:** Trigger on metric thresholds. Actions: SNS notification, Auto Scaling, EC2 stop/reboot/recover.

**Dashboards:** Custom visualizations of metrics.

**Events / EventBridge:** React to events in your AWS account (EC2 state change, S3 upload, CodePipeline failure) and route to targets (Lambda, SQS, SNS).

**Log Insights:** SQL-like query language for log analysis.

**Hands-On: CloudWatch**

```
# Install CloudWatch Agent on your EC2 Ubuntu instance
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb

# Create the agent configuration
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc/
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "/ec2/devops-practice/syslog",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "/ec2/devops-practice/nginx-access",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "DevOpsPractice",
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_user", "cpu_usage_system"],
        "metrics_collection_interval": 60,
        "totalcpu": true
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["disk_used_percent"],
        "metrics_collection_interval": 60,
        "resources": ["/"]
      }
    }
  }
}
EOF

# Start the agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -m ec2 -a status
```

```
# Create a CloudWatch Alarm (CPU > 80% for 2 consecutive 5-min periods)
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

aws cloudwatch put-metric-alarm \
  --alarm-name "High-CPU-devops-instance" \
  --alarm-description "Alert when CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --alarm-actions arn:aws:sns:ap-south-1:ACCOUNT_ID:devops-alerts \
  --ok-actions arn:aws:sns:ap-south-1:ACCOUNT_ID:devops-alerts \
  --treat-missing-data breaching

# Create an SNS topic for alerts
aws sns create-topic --name devops-alerts

# Subscribe your email
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:ACCOUNT_ID:devops-alerts \
  --protocol email \
  --notification-endpoint your.email@gmail.com

# List alarms
aws cloudwatch describe-alarms \
  --query 'MetricAlarms[].[AlarmName,StateValue,MetricName]' \
  --output table

# Put a custom metric (useful for application-level metrics)
aws cloudwatch put-metric-data \
  --namespace "DevOpsPractice/App" \
  --metric-name "URLShortenerRequests" \
  --value 42 \
  --unit Count \
  --dimensions Service=url-shortener,Environment=dev

# Create a Log Group and set retention
aws logs create-log-group \
  --log-group-name /application/url-shortener

aws logs put-retention-policy \
  --log-group-name /application/url-shortener \
  --retention-in-days 30

# Query logs with Insights (powerful — run in console or CLI)
aws logs start-query \
  --log-group-name /ec2/devops-practice/nginx-access \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
```

**CloudWatch Interview Questions**

**Q: What is the difference between CloudWatch Metrics and CloudWatch Logs?**
A: Metrics are numeric time-series data (CPU%, request count) that you alarm on and graph. Logs are raw text events from applications and services that you search and analyze. Metrics Filters can extract metric values from log patterns, bridging both.

**Q: How would you monitor a containerized application on ECS/EKS?**
A: Enable Container Insights — it automatically collects CPU, memory, network, and disk metrics at the container, task, and service level. Push application logs to CloudWatch Logs via the awslogs log driver. Set alarms on CPU and memory reservation. Use X-Ray for distributed tracing across microservices.

**Q: EC2 doesn't show memory metrics by default. Why, and how do you fix it?**
A: AWS can't see inside the OS — memory is OS-level. You install the CloudWatch Agent on the instance, which reads /proc/meminfo and pushes it as a custom metric. Then you can alarm on mem_used_percent.

## Phase 1 — Architecture Summary

Here's how everything in Phase 1 connects in a real production environment:

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
Application Load Balancer (public subnet)
    │
    ├── EC2 Auto Scaling Group (public or private subnet)
    │       │ assumes IAM Role → access S3, Secrets Manager
    │       │ sends logs → CloudWatch Logs
    │       │ monitored → CloudWatch Metrics + Alarms → SNS → Email
    │
    └── RDS PostgreSQL (private subnet, Multi-AZ)
            │
            └── Read Replica (private subnet)

S3 Buckets:
    ├── App assets (static files, versioned)
    ├── Logs archive (lifecycle → Glacier)
    └── Terraform state (with DynamoDB locking — from your prior work!)

DynamoDB:
    └── Session store / URL shortener (PAY_PER_REQUEST, TTL enabled)

VPC:
    ├── Public subnets: ALB, NAT Gateway, Bastion host
    └── Private subnets: EC2 app servers, RDS, ElastiCache
```

**Phase 1 — Interview Cheat Sheet**

<img width="902" height="476" alt="image" src="https://github.com/user-attachments/assets/2cf6e66d-e7c2-48a6-a30f-e82ab8d5468d" />

### What's Next — Phase 2

Once you're done with Phase 1 hands-on this weekend, Phase 2 (DevOps Core) will cover:

- **ECS + ECR** — running your FastAPI Docker image on AWS
- **EKS** — your Kubernetes knowledge, now cloud-native
- **CodePipeline + CodeBuild** — CI/CD fully on AWS (complement to your GitHub Actions work)
- **CloudFormation** — IaC side-by-side with your Terraform knowledge
- **Route 53** — DNS, health checks, weighted and failover routing

































