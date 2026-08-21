# 🎯 AWS SAA-C03 — Complete Exam Mastery

## Exam Overview — Know What You're Fighting

```
Exam name:     AWS Certified Solutions Architect — Associate
Code:          SAA-C03
Questions:     65 (scored) + ~15 unscored (you won't know which)
Duration:      130 minutes (~2 min per question)
Pass score:    720/1000
Format:        Multiple choice (1 answer) + Multiple response (2-3 answers)
Cost:          $150 USD (Pearson VUE or testing center)
Validity:      3 years

Question distribution (approximate):
├── Design Resilient Architectures      26%
├── Design High-Performing Architectures 24%
├── Design Secure Architectures         30%
└── Design Cost-Optimized Architectures 20%
```

## Domain 1 — Design Resilient Architectures (26%)

This is your Phase 1-3 and Phase 5 knowledge compressed into exam-ready answers.

**High Availability vs Fault Tolerance**

```
HIGH AVAILABILITY (HA):
  System remains operational despite component failures.
  May have brief interruption (seconds to minutes).
  Achieved with: Multi-AZ, Auto Scaling, health checks.
  Example: RDS Multi-AZ — automatic failover in ~60s.

FAULT TOLERANCE (FT):
  System continues operating with ZERO interruption.
  No degradation visible to users.
  Achieved with: Active-active, redundant paths, no SPOF.
  Example: S3 — 11 nines durability, multiple AZ storage.

KEY EXAM TRICK:
  "Highly available" = tolerate failure WITH brief interruption.
  "Fault tolerant" = tolerate failure with ZERO interruption.
  When exam says "company cannot tolerate any downtime" → FT, not HA.
```

**Storage Selection — The SAA-C03 Matrix**

```
SCENARIO                          → RIGHT ANSWER
────────────────────────────────────────────────────────────
Shared file system, Linux EC2     → EFS (NFS, ReadWriteMany)
Shared file system, Windows EC2   → FSx for Windows File Server
High-perf shared storage, HPC     → FSx for Lustre
Block storage for EC2             → EBS
Object storage, any size          → S3
Backup archive, cheapest possible → S3 Glacier Deep Archive
Frequently changed data, EC2      → EBS gp3
Database-level IOPS (> 64k)       → EBS io2 Block Express
Temp storage, fastest possible    → EC2 Instance Store (NVMe)
Content delivery, global          → S3 + CloudFront
On-prem to S3, 10TB one-time      → AWS Snowball Edge
On-prem to S3, ongoing hybrid     → AWS Storage Gateway
NFS share for on-prem + cloud     → AWS Storage Gateway (File)
```

**EBS Volume Types — Must Memorize**

```
gp2 (General Purpose SSD):
  ├── Baseline: 3 IOPS/GB
  ├── Burst: up to 3000 IOPS (< 1TB)
  ├── Max: 16,000 IOPS (5.3TB+)
  └── OLD — use gp3 instead for new workloads

gp3 (General Purpose SSD) ← DEFAULT CHOICE:
  ├── Baseline: 3000 IOPS (regardless of size)
  ├── Provisionable: up to 16,000 IOPS separately
  ├── Throughput: up to 1000 MB/s
  └── 20% cheaper than gp2

io2 Block Express (Provisioned IOPS):
  ├── Up to 256,000 IOPS
  ├── Up to 4000 MB/s throughput
  ├── Multi-attach (multiple EC2 can mount same volume)
  └── Use for: SAP HANA, large databases, mission-critical

st1 (Throughput Optimized HDD):
  ├── Max 500 IOPS, 500 MB/s
  ├── CANNOT be boot volume
  └── Use for: Big data, data warehouses, log processing

sc1 (Cold HDD):
  ├── Max 250 IOPS, 250 MB/s
  ├── Cheapest EBS option
  └── Use for: Infrequently accessed data

KEY EXAM TRICKS:
  "Needs highest IOPS" → io2 Block Express
  "Boot volume needed" → CANNOT use st1 or sc1
  "Multi-attach" → ONLY io1 or io2
  "Cost effective general purpose" → gp3
  "Throughput heavy, sequential" → st1
```

**EC2 Instance Types for Exam**

```
Family  Purpose              Memory ratio  Key use cases
──────────────────────────────────────────────────────────────
t3/t4g  Burstable general    1:4          Dev/test, low-traffic web
m5/m6i  General purpose      1:4          Most production workloads
c5/c6i  Compute optimized    1:2          ML inference, video encoding
r5/r6i  Memory optimized     1:8+         In-memory DB, real-time analytics
x2       Memory extreme       1:32         SAP HANA, in-memory databases
p4/p3   GPU (ML training)   Varies       Deep learning, HPC
g4       GPU (inference)     Varies       ML inference, video transcoding
i3/i4i  Storage optimized   Varies       NoSQL DB, data warehousing
d2/d3   Dense storage HDD   Varies       Hadoop, data lakes

GRAVITON (arm64): g suffix → m6g, c6g, r6g
  ├── 40% better price/performance
  └── Exam: "most cost-effective for this workload" → Graviton
```

**Auto Scaling Deep Dive**

```
SCALING POLICIES:

Simple Scaling:
  CloudWatch Alarm fires → add/remove fixed N instances.
  Has cooldown period (default 300s).
  PROBLEM: Can't react to rapid changes quickly.
  OUTDATED — don't use in new architectures.

Step Scaling:
  CPU 60-70% → add 1 instance
  CPU 70-80% → add 2 instances
  CPU > 80%  → add 4 instances
  No cooldown — reacts faster than simple.

Target Tracking (PREFERRED):
  "Keep CPU at 60%"
  ASG automatically adds/removes to maintain target.
  Like a thermostat — always trying to hit the target.
  EXAM DEFAULT ANSWER for "automatic scaling" questions.

Scheduled Scaling:
  "Every Monday 8 AM → scale to 10 instances"
  Use when load is predictable.

Predictive Scaling:
  ML analyzes historical patterns, pre-scales 48h ahead.
  Best for recurring predictable patterns.

LIFECYCLE HOOKS:
  Pending:Wait → run script (install software) → continue
  Terminating:Wait → drain connections, snapshot → continue
  Max wait: 3600 seconds

EXAM TRICKS:
  "Scale based on custom metric (SQS queue depth)" 
    → Target Tracking with custom metric
  "Scale before traffic arrives"
    → Scheduled or Predictive Scaling
  "Minimum cost, handles variable load"
    → Target Tracking + Spot instances in mixed fleet
```

**RDS Key Exam Topics**

```
MULTI-AZ vs READ REPLICAS — most common exam confusion:

Multi-AZ:
  ├── Purpose: HIGH AVAILABILITY (not performance)
  ├── Sync replication to standby
  ├── Automatic failover (~60s)
  ├── Standby cannot serve reads (not accessible)
  ├── Same region only (different AZ)
  └── Use for: Production databases needing HA

Read Replicas:
  ├── Purpose: READ SCALING (not HA)
  ├── Async replication
  ├── Can be in different regions (cross-region)
  ├── Can be promoted to primary (breaks replication)
  ├── Up to 15 replicas (Aurora), 5 (RDS)
  └── Use for: Read-heavy workloads, reporting

CAN YOU COMBINE BOTH? YES:
  Primary (Multi-AZ) → Read Replicas
  HA + read scaling together.

AURORA SPECIFIC (heavy exam topic):
  ├── 6 copies of data across 3 AZs (auto)
  ├── Self-healing (repairs bad blocks automatically)
  ├── Up to 15 read replicas (< 10ms replica lag)
  ├── Aurora Serverless v2: auto-scales in < 1s
  ├── Aurora Global: 1 primary + 5 secondary regions
  │   ├── < 1s cross-region replication
  │   └── Promote secondary in < 1 minute (RPO < 1s, RTO < 60s)
  ├── Backtrack: rewind database to any point (no restore needed)
  └── 5x MySQL performance, 3x PostgreSQL

RDS ENCRYPTION:
  ├── Must enable at creation time
  ├── Cannot encrypt existing unencrypted instance directly
  ├── Workaround: snapshot → copy with encryption → restore
  ├── Read replicas of encrypted DB must be encrypted
  └── KMS CMK or AWS-managed key

RDS PROXY:
  ├── Pools database connections
  ├── Reduces connection storms (Lambda → RDS problem)
  ├── Automatic failover for Multi-AZ (< 66% faster)
  └── IAM authentication for Lambda → RDS
  EXAM: "Lambda function causing too many DB connections" → RDS Proxy
```

## Domain 2 — High-Performing Architectures (24%)

**Caching Strategy — Know Every Layer**

```
LAYER           SERVICE          WHAT IT CACHES           TTL
──────────────────────────────────────────────────────────────────
DNS             Route53          DNS records               60-300s
CDN             CloudFront       HTTP responses, files     Configurable
API             API Gateway      API responses             Max 3600s
Application     ElastiCache      Session, computed data    App-defined
Database        DAX              DynamoDB reads            5min default
DB query        RDS              Buffer pool               Internal
Object          S3               Static assets             Forever (versioned)

ELASTICACHE ENGINES:
Redis:
  ├── Persistence (RDB snapshots, AOF logs)
  ├── Multi-AZ with automatic failover
  ├── Pub/Sub messaging
  ├── Sorted sets (leaderboards)
  ├── Cluster mode (horizontal sharding)
  └── Use for: Sessions, leaderboards, queues, pub/sub

Memcached:
  ├── No persistence (data lost on restart)
  ├── Multi-threaded (uses multiple CPUs)
  ├── Simple key-value only
  ├── Horizontal scaling via node addition
  └── Use for: Pure caching, simple objects

EXAM TRICK:
  "Needs persistence" → Redis
  "Multi-threaded performance" → Memcached
  "Leaderboard / sorted" → Redis
  "Session store" → Redis
  "Simplest caching, just objects" → Memcached
```

**CloudFront Deep Dive**

```
ORIGINS:
  ├── S3 bucket (with Origin Access Control — OAC)
  ├── ALB (HTTP/HTTPS)
  ├── EC2 (must be public)
  ├── API Gateway
  ├── Custom HTTP origin (any HTTP server)
  └── Multiple origins with origin groups (failover)

BEHAVIORS:
  ├── Path pattern: /api/* → ALB origin
  ├── Path pattern: /static/* → S3 origin
  └── Default (*) → primary origin

CACHE INVALIDATION:
  aws cloudfront create-invalidation \
    --distribution-id EDFDVBD6EXAMPLE \
    --paths "/*"
  Cost: first 1000/month free, then $0.005/path

SIGNED URLS vs SIGNED COOKIES:
  Signed URL:    access to ONE specific file
                 use for: individual paid downloads
  Signed Cookie: access to MULTIPLE files
                 use for: premium content sections

FIELD-LEVEL ENCRYPTION:
  Encrypts specific POST fields (credit card, SSN) at edge.
  Only decrypted by your application with private key.
  Even CloudFront and ALB can't read the plaintext.

ORIGIN ACCESS CONTROL (OAC) — replaces OAI:
  ├── CloudFront → S3: bucket policy allows only CloudFront
  ├── S3 bucket stays private (no public access)
  └── OAC supports SSE-KMS (OAI didn't)

PRICE CLASSES:
  All edges:       US, EU, Asia, SA, AU, ME, Africa
  Price Class 200: US, EU, Asia (excludes SA, AU)
  Price Class 100: US, EU only (cheapest)

EXAM TRICKS:
  "S3 content, users only access via CloudFront" → OAC
  "Different content per geography" → CloudFront + Lambda@Edge
  "Restrict premium content to paid users" → Signed Cookies
  "Single file download link" → Signed URL
  "Add custom headers to requests" → CloudFront origin request policy
```

**SQS, SNS, Kinesis — Exam Differentiator**

```
SQS:
  ├── Pull model (consumers poll)
  ├── Message deleted after successful consume
  ├── Retention: 1 min to 14 days (default 4 days)
  ├── Max message size: 256KB (use S3 pointer for larger)
  ├── Standard: at-least-once, best-effort ordering
  ├── FIFO: exactly-once, strict ordering, 3000 msg/s
  ├── Visibility timeout: 0s to 12 hours
  ├── Long polling: reduces empty responses (WaitTimeSeconds=20)
  └── DLQ: messages that failed N times

SNS:
  ├── Push model (AWS pushes to subscribers)
  ├── Fan-out (one publish → many subscribers)
  ├── No persistence (message lost if subscriber unavailable)
  ├── Subscribers: SQS, Lambda, HTTP/S, email, SMS, mobile push
  ├── Message filtering (subscribers get only relevant messages)
  └── FIFO SNS → FIFO SQS only

Kinesis Data Streams:
  ├── Real-time streaming (millisecond latency)
  ├── Retention: 1-365 days (consumers can re-read)
  ├── Multiple consumers read SAME stream
  ├── Ordered per shard
  ├── Shard: 1 MB/s in, 2 MB/s out
  └── Enhanced fan-out: 2 MB/s per consumer per shard

Kinesis Data Firehose:
  ├── Fully managed delivery to S3, Redshift, OpenSearch, Splunk
  ├── No consumers to manage — fire and forget
  ├── Transform via Lambda
  ├── NOT real-time — buffer (60s or 1MB minimum)
  └── Use for: log ingestion, ETL to data lake

EXAM SCENARIOS:
  "Decouple producer from consumer, messages processed once"
    → SQS Standard

  "Financial transactions, strict ordering, no duplicates"
    → SQS FIFO

  "One event → notify multiple different services"
    → SNS fanout (SNS → multiple SQS queues)

  "Real-time analytics, multiple consumers"
    → Kinesis Data Streams

  "Ingest logs to S3/Redshift, don't need real-time"
    → Kinesis Firehose

  "Mobile push notifications"
    → SNS (supports APNs, FCM)

  "SQS queue getting too large, auto-scale consumers"
    → ASG + CloudWatch Alarm on ApproximateNumberOfMessages
```

## Domain 3 — Design Secure Architectures (30%)

This is the highest-weighted domain. Every question touches security.

**IAM Policy Evaluation Logic — Most Tested**

```
EVALUATION ORDER (memorize this):

1. EXPLICIT DENY (SCPs, resource policies, IAM) → DENY immediately
2. EXPLICIT ALLOW                               → ALLOW
3. IMPLICIT DENY (default)                      → DENY

CROSS-ACCOUNT:
  Both the IAM policy in Account A AND the resource policy 
  in Account B must ALLOW the action.
  One Allow is not enough — need BOTH.

CONDITION KEYS TO KNOW:
  aws:RequestedRegion     — restrict to specific regions
  aws:PrincipalOrgID      — restrict to org members only
  aws:SourceIp            — restrict by IP (not in VPC)
  aws:VpcSourceIp         — restrict by IP inside VPC
  aws:MultiFactorAuthPresent — require MFA
  s3:prefix               — restrict S3 path
  iam:PassedToService     — restrict which services get the role
  
RESOURCE-BASED vs IDENTITY-BASED:
  Identity-based: attached to user/role (what can THIS identity do?)
  Resource-based: attached to resource (who can access THIS resource?)
  Examples of resource-based: S3 bucket policy, SQS policy,
    Lambda resource policy, KMS key policy, SNS topic policy

EXAM TRICK: "Allow cross-account Lambda invocation"
  → Need: Lambda resource policy + caller's IAM role permission
  → Both sides must allow
```

**VPC Security Exam Topics**

```
SECURITY GROUP vs NACL:

Security Group:
  ├── Instance-level (ENI)
  ├── STATEFUL (return traffic auto-allowed)
  ├── Allow rules ONLY (no deny)
  ├── All rules evaluated (no order)
  └── Changes take effect immediately

NACL:
  ├── Subnet-level
  ├── STATELESS (must explicitly allow inbound AND outbound)
  ├── Allow AND Deny rules
  ├── Rules evaluated in order (lowest number first)
  ├── Rule 100 evaluated before Rule 200
  └── Explicit DENY: put lower number than the allow

EXAM SCENARIO: "Block a specific IP address from reaching EC2"
  → Use NACL (Security Groups can't deny, only allow)
  → Add DENY rule for that IP with lower priority number

BASTION HOST / JUMP BOX:
  ├── EC2 in PUBLIC subnet
  ├── SSH access from your IP only
  ├── From bastion, SSH to private EC2
  └── Modern alternative: Systems Manager Session Manager
      (no bastion, no open SSH port, full audit trail)

VPC ENDPOINTS:
  Gateway Endpoint (FREE):
    ├── S3 and DynamoDB only
    └── Route table entry: traffic to S3/DDB stays in AWS

  Interface Endpoint (costs $0.01/hr):
    ├── All other AWS services
    ├── Creates ENI in your subnet
    └── Private DNS resolves service to private IP

EXAM TRICK: "EC2 in private subnet needs to access S3 without NAT"
  → S3 Gateway Endpoint (free, no NAT cost)

VPC PEERING:
  ├── Non-transitive (A↔B, B↔C does NOT mean A↔C)
  ├── No overlapping CIDRs
  ├── Must update BOTH route tables
  └── Can peer cross-account, cross-region

TRANSIT GATEWAY:
  ├── Transitive routing
  ├── Hub-and-spoke topology
  ├── Scales to thousands of VPCs
  └── Cross-account, cross-region
```

**Encryption Exam Patterns**

```
S3 ENCRYPTION OPTIONS:

SSE-S3 (Server-Side Encryption with S3-managed keys):
  ├── AWS manages everything
  ├── AES-256
  ├── Header: x-amz-server-side-encryption: AES256
  └── Default since Jan 2023 — always on

SSE-KMS (Server-Side with KMS):
  ├── You control the key policy
  ├── Audit trail in CloudTrail (every decrypt logged)
  ├── Header: x-amz-server-side-encryption: aws:kms
  ├── KMS API calls count toward quotas
  └── Use for: compliance, audit requirements

SSE-C (Server-Side with Customer-provided key):
  ├── YOU provide the key with every request
  ├── AWS encrypts, discards your key
  ├── You must manage key rotation
  └── Use for: must own key material (regulatory)

CSE (Client-Side Encryption):
  ├── You encrypt BEFORE uploading
  ├── AWS never sees plaintext
  └── Use for: extreme sensitivity

ENFORCE ENCRYPTION in transit:
  Bucket policy: deny if aws:SecureTransport = false
  (We wrote this in Phase 1!)

KMS EXAM TOPICS:
  ├── CMK: customer managed (you control policy)
  ├── AWS-managed key: AWS managed (free, no control)
  ├── Data key: symmetric key generated by KMS for envelope encryption
  ├── Envelope encryption: KMS encrypts data key, data key encrypts data
  ├── Key rotation: CMK auto-rotation = yearly
  └── Cross-region: must copy CMK to target region (or use multi-region key)

EXAM TRICK: "Encrypt data but AWS must not manage the key material"
  → CloudHSM (dedicated hardware, you own key material)
  → SSE-C or CSE (you provide/manage keys)
```

**Cognito — Auth Service**

```
COGNITO USER POOLS:
  ├── Authentication (sign-up, sign-in, MFA)
  ├── Returns JWT (ID token, access token, refresh token)
  ├── Integrates with API Gateway as authorizer
  ├── Social IdP (Google, Facebook, Apple)
  └── SAML + OIDC federation (enterprise SSO)

COGNITO IDENTITY POOLS (Federated Identities):
  ├── Authorization (exchange token for AWS credentials)
  ├── Grants temporary IAM role to authenticated users
  ├── Supports: User Pools, Google, Facebook, SAML, guest
  └── Use for: users need to call AWS APIs directly (S3, DynamoDB)

EXAM SCENARIO MAPPING:
  "Users log in to web app, access S3 bucket"
    → Cognito User Pool (auth) + Identity Pool (temp AWS creds)

  "API Gateway, only authenticated users"
    → Cognito User Pool Authorizer on API Gateway

  "Users from corporate SAML IdP need AWS access"
    → Cognito User Pool (SAML federation) + Identity Pool

  "Mobile app users upload directly to S3"
    → Identity Pool → temp credentials → S3 PutObject
```

## Domain 4 — Cost-Optimized Architectures (20%)

**Pricing Model Selection — High Frequency Exam Topic**

```
ON-DEMAND:
  ├── No commitment, pay per second/hour
  ├── Most expensive per unit
  └── Use for: unpredictable workloads, short-term

RESERVED INSTANCES (1 or 3 year):
  ├── Standard RI: up to 72% off, specific instance type
  ├── Convertible RI: up to 54% off, can change instance type
  ├── Scheduled RI: specific time windows (deprecated — use Savings Plans)
  └── Payment: All Upfront (most discount) / Partial / No Upfront

SAVINGS PLANS:
  ├── Compute Savings Plans: up to 66% off, any instance type/region/OS
  ├── EC2 Instance Savings Plans: up to 72% off, specific family+region
  ├── SageMaker Savings Plans: for SageMaker
  └── More flexible than RIs — PREFER Savings Plans for new workloads

SPOT INSTANCES:
  ├── Up to 90% off
  ├── Can be interrupted with 2-minute notice
  ├── Use for: batch, big data, ML training, CI/CD
  └── NEVER for: databases, stateful apps, critical single-instance

DEDICATED HOSTS:
  ├── Physical server dedicated to you
  ├── Use for: bring-your-own-license (BYOL) — Oracle, Windows Server
  └── Most expensive option

DEDICATED INSTANCES:
  ├── Hardware not shared with other customers
  ├── May share hardware with YOUR other instances
  └── Use for: compliance requirements (not BYOL)

EXAM SCENARIO MAPPING:
  "Reduce cost for steady-state production DB"
    → Reserved Instance or Savings Plans (1-year)

  "Batch processing, can handle interruption"
    → Spot Instances

  "Oracle DB, need to use existing license"
    → Dedicated Hosts (BYOL)

  "Variable compute, want flexibility to change instance types"
    → Compute Savings Plans

  "Regulatory requirement: no shared hardware"
    → Dedicated Instances (if BYOL not needed) or Dedicated Hosts
```

**S3 Storage Class Selection**

```
S3 Standard:
  ├── 11 nines durability, 4 nines availability
  ├── No minimum storage duration
  └── Use for: frequently accessed data

S3 Intelligent-Tiering:
  ├── Auto-moves data between access tiers
  ├── Monitoring fee: $0.0025/1000 objects
  ├── No retrieval fee
  └── Use for: unknown or changing access patterns

S3 Standard-IA (Infrequent Access):
  ├── Same durability, lower availability (99.9%)
  ├── Minimum storage: 30 days
  ├── Retrieval fee per GB
  └── Use for: DR copies, backups accessed < 1/month

S3 One Zone-IA:
  ├── Single AZ (99.5% availability)
  ├── 20% cheaper than Standard-IA
  ├── Data lost if AZ fails
  └── Use for: reproducible data, secondary backup copies

S3 Glacier Instant Retrieval:
  ├── Millisecond retrieval
  ├── Minimum: 90 days
  └── Use for: archive accessed quarterly

S3 Glacier Flexible Retrieval:
  ├── Retrieval: Expedited (1-5 min), Standard (3-5 hr), Bulk (5-12 hr)
  ├── Minimum: 90 days
  └── Use for: archive, compliance, accessed 1-2/year

S3 Glacier Deep Archive:
  ├── Retrieval: Standard (12 hr), Bulk (48 hr)
  ├── Minimum: 180 days
  ├── Cheapest S3 storage ($0.00099/GB/month)
  └── Use for: 7-10 year compliance archives

LIFECYCLE POLICY EXAM PATTERNS:
  "Logs kept 1 year, rarely accessed after 30 days"
    → Standard → Standard-IA (30d) → Glacier (90d) → expire (365d)

  "Compliance requires 7-year retention, never accessed"
    → Standard → Glacier Deep Archive (30d)

  "Don't know access patterns"
    → Intelligent-Tiering
```

## Exam Practice — 50 High-Value Questions

Work through every one of these. These are the patterns that repeat most frequently in the real exam.

**Q1. A company runs a three-tier web app. The web tier uses EC2 instances in an ASG behind an ALB. The application tier runs EC2 in private subnets. The database is RDS MySQL. Users report slowness during business hours. Analysis shows 90% of database queries are reads. What should a solutions architect recommend?**

- A) Enable RDS Multi-AZ
- B) Add RDS Read Replicas and redirect read traffic to them
- C) Enable ElastiCache in front of RDS
- D) Upgrade to a larger RDS instance class

**Answer: B**

Multi-AZ (A) is for HA, not read scaling. ElastiCache (C) helps but requires application code changes and doesn't address DB read load directly. Upgrade (D) is reactive and expensive. Read Replicas (B) directly addresses 90% read traffic at RDS level — the cleanest architectural solution.

**Q2. A solutions architect needs to design a solution where messages from a producer application must be processed by FIVE different consumer applications. Each consumer must process EVERY message. What is the most operationally efficient solution?**

- A) Create five SQS queues and have the producer send to all five
- B) Create one SQS queue and have five consumers poll it
- C) Create an SNS topic, subscribe five SQS queues, producer publishes to SNS
- D) Create a Kinesis stream with five shards

**Answer: C**

SNS → SQS fanout. Each SQS queue gets its own copy of every message. Each consumer independently processes from its queue. B would have consumers competing for the same messages (each message processed only once). This is the classic fanout pattern.

**Q3. A company stores sensitive financial data in S3. A compliance officer requires that all access to the data must be logged, and the encryption keys must be auditable. What should the solutions architect configure?**

- A) SSE-S3 with S3 access logging enabled
- B) SSE-KMS with a customer-managed key, CloudTrail enabled
- C) SSE-C with client-managed keys
- D) Client-side encryption with AWS Encryption SDK

**Answer: B**

SSE-KMS with CMK: every encrypt/decrypt call logs to CloudTrail. CMK key policy gives you full audit and control. SSE-S3 (A) doesn't log individual key usage. SSE-C (C) is operationally complex. CSE (D) means AWS never sees ciphertext — auditing becomes your problem entirely.

**Q4. An application runs on EC2 in a private subnet and needs to download software updates from the internet. The company wants minimum cost. What should the solutions architect recommend?**

- A) Attach an Elastic IP to the EC2 instance
- B) Add an Internet Gateway and update route tables
- C) Deploy a NAT Gateway in a public subnet
- D) Deploy a NAT Instance in a public subnet

**Answer: C**

Private subnet EC2 can't have public IPs. IGW alone (B) doesn't help private subnets. NAT Gateway (C) is AWS-managed, highly available. NAT Instance (D) is valid but "minimum operational overhead" points to managed NAT Gateway over self-managed NAT Instance.

**Q5. A company needs to migrate 50TB of data from on-premises to S3. The internet connection is 100Mbps and is shared with business operations. The migration must complete within two weeks and must not impact business traffic. What should they use?**

- A) AWS Direct Connect
- B) S3 Transfer Acceleration
- C) AWS Snowball Edge
- D) AWS DataSync over VPN

**Answer: C**

50TB over 100Mbps shared = 50TB/(100Mbps × 0.5 usable) = ~90 days. Way more than 2 weeks. Snowball Edge ships physically — 50TB fits in one device, delivers in ~1 week. Direct Connect (A) takes weeks to provision. S3 Transfer Acceleration (B) still uses internet — won't fit in 2 weeks.

**Q6. A web application serves users globally. Static assets (images, CSS, JS) are stored in S3. Users in Asia report slow load times. What is the most cost-effective solution?**

- A) Create S3 buckets in multiple regions
- B) Enable S3 Transfer Acceleration
- C) Deploy CloudFront in front of the S3 bucket
- D) Use Route53 geolocation routing to different S3 buckets

**Answer: C**

CloudFront caches content at 400+ edge locations globally. Subsequent requests from Asia hit the nearest edge, not origin S3. Most cost-effective for static assets. S3 Transfer Acceleration (B) speeds uploads not downloads from S3 to users.

**Q7. A Lambda function is timing out when connecting to RDS MySQL in a VPC. CloudWatch shows Lambda is creating thousands of database connections per minute and the DB is refusing connections. What is the most appropriate fix?**

- A) Increase Lambda memory
- B) Add RDS Read Replicas
- C) Enable RDS Multi-AZ
- D) Configure RDS Proxy between Lambda and RDS

**Answer: D**

Lambda creates a new connection per invocation. At scale = connection storm. RDS Proxy pools and multiplexes connections — Lambda connects to proxy, proxy maintains small pool to RDS. Classic "Lambda → RDS connection exhaustion" → RDS Proxy answer.

**Q8. A company wants to allow users to upload files directly to S3 from a web browser without routing through application servers. The solution must be secure. What should the architect configure?**

- A) Make the S3 bucket public
- B) Generate pre-signed URLs on the server and return them to the browser
- C) Use S3 Transfer Acceleration
- D) Configure S3 as a static website

**Answer: B**

Pre-signed URLs: server generates a time-limited signed URL (using IAM credentials), browser uploads directly to S3 using that URL. Bucket stays private. User can't do anything beyond what the URL allows. This is the standard secure direct-upload pattern.

**Q9. An application must maintain user session state across multiple EC2 instances behind an ALB. Currently sessions break when requests hit different instances. What is the LEAST expensive fix?**

- A) Enable ALB sticky sessions
- B) Store sessions in ElastiCache Redis
- C) Use EFS to share sessions across instances
- D) Add more EC2 instances

**Answer: A or B depending on context.**

If the question says "least expensive" → A (sticky sessions — free, just an ALB setting)
If the question says "most scalable" or "instances can be terminated" → B (ElastiCache)

**EXAM TRICK:** Sticky sessions break if instance terminates. ElastiCache persists sessions across instance failures. Read the question carefully.

**Q10. A solutions architect needs to design a data lake on AWS. The company ingests 1TB/day of logs. Data must be queryable without loading into a database. Cost must be minimized. What should they use?**

- A) RDS MySQL with large storage
- B) Redshift with compressed storage
- C) S3 + Athena with Parquet format
- D) DynamoDB with on-demand capacity

**Answer: C**

S3 + Athena: store data in S3 (cheapest storage), query with Athena (pay per query, per TB scanned). Parquet is columnar — Athena scans only needed columns (85% cost reduction). No database to manage. Classic serverless data lake answer.

**Q11. A company runs a critical application on EC2. Recovery Point Objective (RPO) is 1 hour and Recovery Time Objective (RTO) is 15 minutes. What backup and recovery strategy should be used?**

- A) Daily AMI backups, manually launch new instance on failure
- B) Hourly EBS snapshots, Auto Scaling Group with Launch Template using latest snapshot
- C) Enable EC2 hibernate
- D) Use RDS automated backups

**Answer: B**

Hourly EBS snapshots satisfies RPO=1hr. ASG with Launch Template auto-launches replacement from snapshot — can meet RTO=15min if AMI pre-built. RDS (D) is not applicable for EC2 application tier. Daily AMI (A) violates 1hr RPO.

**Q12. A company needs to share a file system across 100 Linux EC2 instances simultaneously. The file system must be scalable and fully managed. What should the architect recommend?**

- A) EBS volume with Multi-Attach
- B) S3 mounted as a filesystem
- C) Amazon EFS
- D) FSx for Windows

**Answer: C**

EFS: fully managed NFS, ReadWriteMany, scales automatically, supports thousands of concurrent connections. EBS Multi-Attach (A) is only for io1/io2 and cluster-aware applications. FSx for Windows (D) is for Windows workloads. S3 (B) is object storage, not a true filesystem.

**Q13. A mobile application stores user data in DynamoDB. The app has read-heavy traffic with bursts of 10x normal load. Response times must stay under 1ms. What should the architect add?**

- A) DynamoDB Auto Scaling
- B) DAX (DynamoDB Accelerator)
- C) ElastiCache Redis in front of DynamoDB
- D) DynamoDB Global Tables

**Answer: B**

DAX: in-memory cache for DynamoDB, microsecond reads, fully compatible (drop-in SDK replacement). 1ms requirement = DynamoDB alone (~1-3ms) may not consistently hit it. DAX hits it every time. Global Tables (D) is for multi-region, not performance. ElastiCache (C) requires application code changes; DAX doesn't.

**Q14. A company uses SQS to decouple an order processing system. Orders are being processed multiple times, causing duplicate shipments. What should the architect do?**

- A) Enable long polling on the SQS queue
- B) Reduce the visibility timeout
- C) Migrate to an SQS FIFO queue
- D) Increase the message retention period

**Answer: C**

FIFO queues provide exactly-once processing with deduplication. Standard SQS is at-least-once — duplicates are expected. Long polling (A) reduces empty receives, not duplicates. Reducing visibility timeout (B) makes duplicates WORSE (message becomes visible again while still processing).

**Q15. A web application needs to authenticate users and allow them to access their own folder in S3 (e.g., s3://mybucket/users/{userId}/). What AWS services should be used?**

- A) IAM users with individual S3 bucket policies
- B) Cognito User Pool + Identity Pool with IAM role policy using ${cognito-identity.amazonaws.com:sub}
- C) CloudFront signed URLs for each user
- D) API Gateway with Lambda to proxy all S3 access

**Answer: B**

Cognito Identity Pool issues temporary IAM credentials. The IAM role policy uses policy variable ${cognito-identity.amazonaws.com:sub} to restrict each user to their own prefix. Scales to millions of users without creating IAM users. Classic mobile/web app pattern.

**Q16. A company has a VPC with two public subnets and two private subnets in two AZs. The architecture runs a web tier in public subnets and app tier in private subnets. Which component allows the app tier to make outbound internet calls for software updates?**

- A) Internet Gateway
- B) NAT Gateway in private subnet
- C) NAT Gateway in public subnet
- D) VPC Peering connection

**Answer: C**

NAT Gateway must be in a PUBLIC subnet (it needs internet access via the IGW). Private subnet route table has 0.0.0.0/0 → NAT Gateway. Outbound works. Inbound from internet → blocked (NAT only does outbound). Common exam trap: NAT Gateway in PRIVATE subnet = wrong.

**Q17. A company wants to implement a disaster recovery solution with an RTO of under 1 minute and RPO of near-zero. Cost is not the primary concern. What should the architect recommend?**

- A) Backup and Restore
- B) Pilot Light
- C) Warm Standby
- D) Multi-Site Active-Active

**Answer: D**

RTO < 1 minute and RPO near-zero = only Active-Active achieves this. Traffic already running in both regions — failover is just DNS/routing change. Warm Standby takes minutes to scale up. Pilot Light takes 10-30 minutes.

**Q18. A company is running a batch workload processing genomics data. The jobs run for 6 hours and can be restarted if interrupted. Cost optimization is the top priority. What EC2 purchasing option should be used?**

- A) On-Demand Instances
- B) Reserved Instances
- C) Spot Instances
- D) Dedicated Hosts

**Answer: C**

"Can be restarted if interrupted" + "cost optimization" = Spot. Up to 90% discount. 6-hour jobs that can restart tolerate the 2-minute interruption notice. This is the textbook Spot use case.

**Q19. An application writes data to DynamoDB. Occasionally the writes fail with ProvisionedThroughputExceededException. The traffic is unpredictable. What should the architect recommend?**

- A) Increase provisioned write capacity
- B) Switch to DynamoDB on-demand capacity mode
- C) Enable DynamoDB Auto Scaling
- D) Use DynamoDB Accelerator (DAX)

**Answer: B**

On-demand mode handles any traffic level automatically — no capacity planning, no throttling. Auto Scaling (C) reacts to throttling but has a lag — doesn't prevent the initial burst from throttling. On-demand eliminates throttling entirely for unpredictable workloads.

**Q20. A company needs to process records from a Kinesis Data Stream. Multiple applications need to read the same stream simultaneously and independently. What should the architect configure?**

- A) Create multiple Kinesis streams, one per application
- B) Use Enhanced Fan-Out consumers on the Kinesis stream
- C) Use SQS instead of Kinesis
- D) Add more shards to the stream

**Answer: B**

Enhanced Fan-Out: each consumer gets dedicated 2 MB/s throughput per shard. Without it, all consumers SHARE 2 MB/s per shard. Enhanced Fan-Out allows true parallel independent consumption. This is exactly the "multiple consumers read same stream" pattern.

**Q21. A company's EC2-based application communicates with an S3 bucket. The security team requires that traffic must not traverse the public internet. What should the architect implement?**

- A) S3 Transfer Acceleration
- B) VPC Gateway Endpoint for S3
- C) NAT Gateway
- D) AWS PrivateLink for S3

**Answer: B**

S3 Gateway Endpoint: routes S3 traffic through AWS backbone, not public internet. Free. Added to route table. Classic "private EC2 → S3 without internet" = Gateway Endpoint. PrivateLink (D) is Interface Endpoint for S3 — valid but costs money. Gateway Endpoint is free and preferred.

Q22. A financial services company needs to block a specific IP address from accessing their web application on an ALB. The IP is attacking the application. What is the quickest way to block it?

A) Update the Security Group to deny the IP
B) Add a NACL deny rule for the IP
C) Configure WAF with an IP-based rule
D) Add a deny rule in the ALB listener

Answer: C or B

Security Groups (A) — CANNOT deny. Only allow rules.
NACL (B) — can deny, but applies to entire subnet. Works but coarser.
WAF (C) — IP set match rule. Best answer when ALB is the target.
ALB listener (D) — can't deny by IP.

EXAM TRICK: If question mentions "web application" and "ALB" → WAF. If question is about VPC-level blocking → NACL.

Q23. A company is moving from a monolithic application to microservices. They want to route requests to different backend services based on URL path. What should they use?

A) Network Load Balancer
B) Application Load Balancer with path-based routing
C) Classic Load Balancer
D) Route53 with weighted routing

Answer: B
ALB supports content-based routing: /api/ → Service A, /images/* → Service B, /auth/* → Service C. NLB (A) is Layer 4 — no path awareness. Classic LB (C) is legacy. Route53 (D) is DNS-level — not path-based.*

Q24. A solutions architect is designing a solution where an AWS Lambda function needs to access resources in a VPC (like RDS in a private subnet). What must be configured?

A) Attach an Elastic IP to the Lambda function
B) Configure Lambda with VPC settings (VPC, subnets, security group)
C) Deploy Lambda in a public subnet
D) Enable VPC peering between Lambda and the VPC

Answer: B
Lambda VPC config: specify VPC ID, private subnets, and security group. Lambda creates ENIs in your subnets to communicate with VPC resources. No Elastic IP needed. Lambda doesn't "deploy" into public/private subnets — it uses ENIs.

Q25. A company wants to ensure their S3 bucket contents are replicated to another region for disaster recovery. Existing objects must also be replicated. What must the architect enable?

A) S3 Transfer Acceleration
B) Cross-Region Replication with S3 Batch Operations for existing objects
C) S3 Intelligent-Tiering
D) S3 Object Lock

Answer: B
CRR only replicates NEW objects going forward. For existing objects, you need S3 Batch Operations (batch replication job). Both components needed. Common trap: assuming CRR handles existing objects.

Q26. A company needs to run a containerized application that requires no server management, automatically scales to zero when idle, and charges only when running. What should the architect use?

A) ECS with EC2 launch type
B) ECS with Fargate launch type
C) EC2 Auto Scaling Group
D) AWS Lambda

Answer: B or D

Lambda (D): true scale-to-zero, per-execution billing, but 15-minute max runtime, 10GB memory limit.
Fargate (B): no server management, scales automatically, but min billing is per task duration.

EXAM TRICK: "containerized" → Fargate. "function/handler" → Lambda. Both are serverless but Fargate runs containers, Lambda runs functions.

Q27. A web app is hosted on EC2 behind an ALB. The app stores session data locally on each EC2 instance. When the ASG scales in, users lose their sessions. What should the architect do?

A) Use ALB sticky sessions
B) Store session data in ElastiCache Redis
C) Increase minimum capacity in ASG
D) Enable EC2 termination protection

Answer: B
Sticky sessions (A) breaks when instance terminates — scale-in event terminates instances. ElastiCache Redis is external — sessions survive instance termination. This is the "scale-in causes session loss" → externalize session state pattern.

Q28. A company has an on-premises data center connected to AWS via AWS Direct Connect. They want to ensure connectivity even if the Direct Connect link fails. What should be the backup solution?

A) Add a second Direct Connect connection
B) Configure a Site-to-Site VPN as a backup
C) Use the public internet directly
D) Deploy AWS Snowball Edge

Answer: B
VPN over internet as Direct Connect backup is the standard HA pattern. If both connections needed for SLA → A (second DX). But "backup" = VPN (B). Cost-effective resilience.

Q29. A company runs a stateless web application on EC2. They want the most cost-effective architecture that maintains availability during AZ failure. What should they configure?

A) Single EC2 in one AZ with daily snapshots
B) EC2 in multiple AZs with an ALB and Auto Scaling Group
C) EC2 in one AZ with Reserved Instance
D) EC2 Spot Fleet across multiple AZs

Answer: B
Multi-AZ + ALB + ASG: stateless app scales horizontally. AZ failure → ASG launches in other AZ, ALB routes away from failed AZ. Most cost-effective HA architecture. Spot Fleet (D) valid for cost but Spot interruption could violate availability requirement.

Q30. A company stores 100TB of video files in S3. The files are created once and never modified. They need to ensure data is protected against accidental deletion. What should they implement?

A) Enable S3 versioning only
B) Enable S3 Object Lock in Compliance mode with versioning
C) Replicate to another S3 bucket
D) Store in S3 Glacier

Answer: B
Object Lock + Compliance mode: no one (including root) can delete or overwrite objects until retention period expires. Versioning alone (A) — objects can still be deleted (delete marker). Compliance mode = regulatory-grade WORM storage.

Q31. An application needs to send email notifications to users. The company wants a scalable managed solution without managing email servers. What should they use?

A) EC2 running an SMTP server
B) Amazon SES (Simple Email Service)
C) SNS with email protocol
D) Lambda with third-party SMTP

Answer: B
SES: managed email sending service. High deliverability, scales automatically. SNS email (C) is for operational notifications to admins, not transactional email to users. SES handles unsubscribes, bounces, and complaints.

Q32. A company's DynamoDB table is experiencing hot partition issues. One partition key value (a popular product ID) receives 80% of all traffic. What should the architect recommend?

A) Increase provisioned throughput
B) Enable DynamoDB Auto Scaling
C) Use a composite key with a random suffix added to the partition key
D) Switch to on-demand capacity

Answer: C
Write sharding: add random suffix (1-N) to the partition key. product#123_1, product#123_2... Distribute writes across N partitions. Reads: fan-out query to all N partitions and aggregate. On-demand (D) doesn't solve hot partition — you'll still throttle on the hot partition even with infinite capacity.

Q33. A solutions architect needs to migrate a MySQL database to AWS with minimal downtime. The application must remain available during migration. What should they use?

A) mysqldump and import to RDS
B) AWS DMS with full-load-and-CDC mode
C) AWS Snowball for database migration
D) RDS snapshot restore

Answer: B
DMS full-load-and-CDC: loads existing data while capturing change data (CDC = change data capture). After initial sync, CDC keeps target in sync with source in near-real-time. Cutover when lag is near zero. Minimal downtime. mysqldump (A) requires downtime for final export.

Q34. A company needs to audit all API calls made in their AWS account. They need to store logs for 7 years and be able to query them. What should they configure?

A) VPC Flow Logs to S3
B) CloudTrail with S3 storage and Athena for querying
C) CloudWatch Logs with extended retention
D) Config with the full history feature

Answer: B
CloudTrail captures ALL API calls. S3 for 7-year storage (lifecycle to Glacier after 90 days). Athena for SQL querying. VPC Flow Logs (A) captures network traffic, not API calls. Config (D) tracks resource configuration, not all API calls.

Q35. A company wants to implement least-privilege access for their application. The application runs on EC2 and needs to write to a specific S3 bucket. What is the BEST approach?

A) Store access keys in environment variables on EC2
B) Create an IAM user with S3 access and store keys in the application
C) Create an IAM role with S3 access and attach it to the EC2 instance profile
D) Use the root account credentials

Answer: C
IAM role attached to instance profile: no credentials stored, automatic rotation, works via IMDSv2. This eliminates credential management entirely. Stored keys (A, B) are security anti-patterns — can be exposed via instance compromise.

Q36. An application uses API Gateway and Lambda. During peak hours, Lambda functions are hitting concurrency limits and requests are being throttled. How should the architect resolve this WITHOUT increasing Lambda concurrency limits?

A) Enable API Gateway caching
B) Use SQS between API Gateway and Lambda
C) Add CloudFront in front of API Gateway
D) Use Step Functions instead of Lambda

Answer: B
SQS buffer: API Gateway → SQS → Lambda. SQS absorbs traffic spikes. Lambda processes at its own rate. No throttling because requests are queued, not rejected. API Gateway caching (A) helps GET requests but not writes. This is the "throttle protection" pattern.

Q37. A company hosts a static website on S3. They want to add HTTPS. S3 website hosting only supports HTTP. What is the solution?

A) Request an SSL cert and install on S3
B) Use CloudFront with ACM certificate in front of S3
C) Move the website to EC2 with Nginx
D) Use Route53 to redirect HTTP to HTTPS

Answer: B
CloudFront terminates HTTPS using ACM certificate. S3 origin stays HTTP (internal connection). Users see HTTPS. ACM certs for CloudFront must be in us-east-1 (global). This is the standard static site HTTPS pattern.

Q38. A company needs to process messages in EXACTLY the order they were received. Messages represent financial transactions and duplicates would cause double charges. What should they use?

A) SQS Standard queue
B) SQS FIFO queue
C) Kinesis Data Streams
D) SNS FIFO topic

Answer: B
SQS FIFO: exactly-once processing + strict ordering within a MessageGroupId. Standard SQS (A) is at-least-once (duplicates possible) and best-effort ordering. For financial transactions = FIFO.

Q39. A company is designing a serverless data processing pipeline. Files are uploaded to S3, then need to be processed by Lambda, results stored in DynamoDB, and finally a notification sent. What should connect these services?

A) AWS Glue
B) S3 Event Notifications → Lambda → DynamoDB → SNS
C) Step Functions orchestrating Lambda functions
D) Kinesis Data Streams

Answer: C
Step Functions: orchestrates multi-step workflow. Retry logic, error handling, visibility into each step. S3 event → Step Functions → Lambda → DynamoDB → SNS. Each step has retry configuration. Simple trigger (B) works but lacks retry/error handling. "Pipeline" → Step Functions.

Q40. A company wants to reduce data transfer costs. Their EC2 instances in one VPC frequently access S3 and DynamoDB in the same region. They are paying high NAT Gateway costs. What is the solution?

A) Move EC2 to public subnets
B) Add VPC Gateway Endpoints for S3 and DynamoDB
C) Use VPC Peering to S3
D) Deploy ElastiCache to reduce S3 calls

Answer: B
Gateway Endpoints for S3 and DynamoDB: FREE. Traffic routes through AWS backbone. Eliminates NAT Gateway processing for S3/DDB traffic. This is a common cost optimization — companies see immediate NAT cost reduction.

Q41. A company needs a fully managed message broker that supports JMS, AMQP, MQTT, and STOMP protocols. They're migrating from on-premises ActiveMQ. What should they use?

A) Amazon SQS
B) Amazon SNS
C) Amazon MQ
D) Amazon Kinesis

Answer: C
Amazon MQ: managed Apache ActiveMQ and RabbitMQ. Supports JMS, AMQP, MQTT, STOMP, OpenWire. Drop-in replacement for on-prem message brokers. SQS/SNS (A, B) are AWS-native — require code changes. "Migrating from ActiveMQ" → Amazon MQ.

Q42. An organization has multiple AWS accounts managed under AWS Organizations. They want to ensure all accounts in the production OU can only create resources in ap-south-1 and eu-west-1. What should the architect implement?

A) IAM policies in every account
B) Service Control Policy attached to the Production OU
C) AWS Config rules in every account
D) Permission boundaries on all IAM roles

Answer: B
SCP at OU level: apply once, enforces across ALL accounts in the OU automatically. New accounts added to OU immediately get the SCP. IAM policies (A) must be managed in every account separately — not scalable. SCPs are the right answer for org-wide guardrails.

Q43. A solutions architect needs to design an architecture that processes IoT sensor data from 10,000 devices. Data arrives continuously and must be analyzed in real-time. What should they use?

A) SQS → Lambda
B) Kinesis Data Streams → Kinesis Data Analytics → Lambda
C) DynamoDB Streams → Lambda
D) SNS → SQS → Lambda

Answer: B
Real-time streaming IoT data = Kinesis Data Streams. Kinesis Data Analytics: run SQL or Flink directly on the stream for real-time analysis. Lambda for downstream actions. SQS (A) is for discrete messages, not continuous streams. "Real-time analytics on streaming data" = Kinesis.

Q44. A company wants to share data with partner companies who have their own AWS accounts. The data is in an S3 bucket. The solution must not give partners direct S3 access and must log all access. What should the architect build?

A) Make S3 bucket public
B) Create IAM users for each partner
C) Deploy API Gateway + Lambda that reads from S3 and returns data
D) Use S3 bucket policies with partner account IDs

Answer: C
API Gateway + Lambda: controlled interface, full logging via CloudWatch/CloudTrail, can add rate limiting, authentication, and transformation. Partners never get S3 credentials. D is simpler but gives direct S3 access without request-level logging.

Q45. A web application stores transactional data in RDS PostgreSQL. They want to generate analytics reports without impacting production performance. What is the recommended approach?

A) Run reports on the primary RDS instance during off-peak hours
B) Create a Read Replica and run reports against it
C) Export to S3 daily and use Athena
D) Use DynamoDB for reporting

Answer: B
Read Replica for reporting: async replication, reports run on replica — zero impact on primary. If reports can tolerate slight data lag (seconds) → Read Replica. If reports need exact current data → run on primary (but impacts performance). Exam answer: Read Replica.

Q46. A company needs to store secrets like database passwords for an ECS application. Secrets must be encrypted and automatically rotated. What should they use?

A) Systems Manager Parameter Store (Standard)
B) Systems Manager Parameter Store (SecureString)
C) AWS Secrets Manager
D) Environment variables in the ECS Task Definition

Answer: C
Secrets Manager: automatic rotation via Lambda, encryption with KMS, native integration with RDS/ElastiCache/Redshift. Parameter Store SecureString (B) encrypts but no automatic rotation. Environment variables (D) are plaintext in task definition. "Automatic rotation" = Secrets Manager.

Q47. A company's application experiences latency spikes when Aurora RDS restarts during maintenance windows. How should the architect reduce this impact?

A) Disable automatic maintenance
B) Use Aurora Multi-AZ with read replicas configured as failover targets
C) Schedule maintenance during business hours
D) Use RDS proxy to pool connections

Answer: B or D
Aurora Read Replicas as failover: Aurora automatically promotes a read replica during failover — takes < 30 seconds vs 60-120s for RDS Multi-AZ. RDS Proxy (D) reduces connection storm after restart. The best answer depends on whether the question emphasizes "failover speed" (B) or "connection handling" (D).

Q48. A company needs to analyze 5 years of historical log data stored in S3 Glacier. Analysis runs monthly. What is the most cost-effective query solution?

A) Restore all logs to S3 Standard, then query with Athena
B) Use S3 Glacier Select to query logs in place
C) Load logs into Redshift for analysis
D) Restore to S3 Standard-IA, then query with Athena

Answer: B
S3 Glacier Select: run SQL queries directly on data in Glacier without full restore. Pay only for data scanned. Monthly analysis on Glacier = Glacier Select. Full restore (A, D) = expensive retrieval costs plus storage costs for duration.

Q49. A startup wants to run a web application. They want zero servers to manage, automatic scaling, and pay only for actual usage. What is the MOST serverless architecture?

A) EC2 Auto Scaling + RDS
B) ECS Fargate + Aurora Serverless
C) Lambda + API Gateway + DynamoDB + S3
D) Elastic Beanstalk + RDS

Answer: C
Lambda + API Gateway + DynamoDB + S3: fully serverless. Lambda scales to zero. DynamoDB on-demand. S3 scales infinitely. No infrastructure to manage. All pay-per-use. Fargate (B) is serverless compute but Aurora Serverless has a minimum. Lambda (C) is the most serverless.

Q50. A solutions architect needs to choose between SQS and Kinesis for an application that processes clickstream data from a website. Multiple analytics applications need to read the same data independently. Data must be retained for 7 days for replay. Which should they choose and why?

A) SQS Standard — easiest to set up, unlimited retention
B) SQS FIFO — exactly-once delivery, ordered processing
C) Kinesis Data Streams — multiple consumers, 7-day retention, replay
D) Kinesis Firehose — managed delivery to analytics destinations

Answer: C
Kinesis: retention up to 365 days (7 days = standard, configurable), multiple independent consumers via Enhanced Fan-Out, replay via sequence numbers. SQS (A, B) deletes messages after consume — only one consumer can process each message, no replay. Firehose (D) is delivery-only, no multiple consumers.

## Exam Day Strategy

```
BEFORE THE EXAM:
├── Schedule at a Pearson VUE center (more reliable than OnVUE home)
├── Arrive 30 minutes early — ID check, biometrics
├── No notes allowed — 6 pillars and pricing models must be memorized
└── Whiteboard/notepad provided — write your cheat sheet first

DURING THE EXAM:
├── Time budget: 130 min / 65 questions = 2 min/question
├── First pass: answer everything you know confidently (< 90s each)
├── Flag and skip: anything requiring >90s — come back
├── Second pass: work through flagged questions
├── Third pass: review if time permits
└── NEVER leave a question blank — guess if needed (no penalty)

QUESTION READING STRATEGY:
├── Read the LAST sentence first — that's the actual question
├── Look for: "most cost-effective", "least operational overhead",
│            "highest availability", "minimum downtime"
├── Eliminate: answers with anti-patterns (store creds in code, etc.)
├── Spot keywords:
│   ├── "shared file system Linux" → EFS
│   ├── "no server management" → Fargate/Lambda
│   ├── "existing license" → Dedicated Hosts
│   ├── "multiple consumers same data" → Kinesis
│   ├── "exactly-once ordered" → SQS FIFO
│   ├── "block IP address" → NACL or WAF
│   ├── "connect to DB from Lambda" → RDS Proxy
│   └── "real-time streaming" → Kinesis
└── When two answers seem right → pick the ONE that addresses
    ALL constraints in the question, not just some

COMMON TRAPS:
├── Multi-AZ = HA, Read Replica = performance (NOT interchangeable)
├── NAT Gateway in PUBLIC subnet (not private)
├── Security Groups CANNOT deny (only allow)
├── CRR does NOT replicate existing objects (need S3 Batch)
├── CloudFront ACM cert MUST be in us-east-1
├── Gateway Endpoints: only S3 and DynamoDB (everything else = Interface)
└── Spot Instances = can be interrupted (never for databases/critical single instance)
```

## 30-Day Study Plan

```
Week 1 — Foundation Review:
  Day 1-2:  IAM, Organizations, SCP (you know this — review exam angles)
  Day 3-4:  VPC, subnets, security groups, NACLs, endpoints
  Day 5-6:  EC2, ASG, ELB, pricing models
  Day 7:    Practice test 1 (Tutorials Dojo or ExamTopics — 65 questions)
             Review every wrong answer

Week 2 — Storage & Databases:
  Day 8-9:  S3 (storage classes, lifecycle, encryption, replication)
  Day 10-11: RDS, Aurora, ElastiCache, DynamoDB
  Day 12-13: EFS, FSx, Storage Gateway, Snowball
  Day 14:   Practice test 2 — track score and weak areas

Week 3 — Application Services:
  Day 15-16: Lambda, API Gateway, Step Functions
  Day 17-18: SQS, SNS, Kinesis, EventBridge
  Day 19-20: CloudFront, Route53, Global Accelerator
  Day 21:   Practice test 3 — must score 750+ before continuing

Week 4 — Architecture Patterns + Final Prep:
  Day 22-23: Well-Architected Framework (all 6 pillars)
  Day 24-25: DR strategies (backup/restore → active-active)
  Day 26-27: Migration strategies (6 R's), hybrid architectures
  Day 28:   Full timed mock exam (65Q in 130 min, strict)
  Day 29:   Review only weak areas — no new content
  Day 30:   Exam day — trust your preparation

RESOURCES:
├── Tutorials Dojo SAA-C03 practice exams — BEST practice tests
├── Adrian Cantrill SAA-C03 course — BEST video course
├── AWS Skill Builder — free official practice questions
├── AWS whitepapers: Well-Architected, Security Best Practices
└── Your Phase 1-5 notes from this curriculum (you built all of it!)

TARGET SCORE: 800+ (you want buffer above 720 pass mark)
```




























