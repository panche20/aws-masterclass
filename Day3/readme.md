# 🚀 AWS Phase 2 Continued — Full DevOps Platform

## PART 1 — AWS CodeBuild (Managed Build Service)

CodeBuild is a fully managed CI service. No Jenkins servers to maintain. It spins up a fresh build environment per job, runs your buildspec, and tears it down. You pay per build minute.

**How CodeBuild Works**

```
Source (CodeCommit / GitHub / S3 / Bitbucket)
    │
    ▼
CodeBuild Project
    ├── Environment (Docker image, compute size, env vars)
    ├── buildspec.yml (your build instructions)
    └── Artifacts (S3, ECR, nothing)
```

**buildspec.yml — The Heart of CodeBuild**

```
# buildspec.yml — production grade for your FastAPI URL shortener
version: 0.2

env:
  variables:
    AWS_REGION: "ap-south-1"
    ECR_REPO: "url-shortener/api"
    APP_NAME: "url-shortener"
  parameter-store:
    SONAR_TOKEN: "/url-shortener/sonar-token"
  secrets-manager:
    SNYK_TOKEN: "/url-shortener/prod/snyk-token"

phases:
  install:
    runtime-versions:
      python: 3.12
      docker: 20
    commands:
      - echo "Installing dependencies..."
      - pip install --upgrade pip
      - pip install pytest pytest-cov bandit safety flake8
      # Install Trivy for container scanning
      - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
      # Install hadolint for Dockerfile linting
      - wget -O /usr/local/bin/hadolint https://github.com/hadolint/hadolint/releases/latest/download/hadolint-Linux-x86_64
      - chmod +x /usr/local/bin/hadolint

  pre_build:
    commands:
      - echo "Pre-build phase starting..."
      - ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
      - GIT_SHA=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-8)
      - BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
      - IMAGE_TAG="${GIT_SHA}-${CODEBUILD_BUILD_NUMBER}"
      - FULL_IMAGE="${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"

      # Authenticate to ECR
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

      # Lint Dockerfile
      - echo "Linting Dockerfile..."
      - hadolint Dockerfile

      # Static analysis
      - echo "Running bandit security scan..."
      - bandit -r app/ -ll -f json -o bandit-report.json || true

      # Dependency vulnerability scan
      - echo "Scanning dependencies..."
      - safety check --json > safety-report.json || true

      # Unit tests with coverage
      - echo "Running unit tests..."
      - pytest tests/ -v --cov=app --cov-report=xml --cov-report=term-missing --junitxml=test-results.xml

      # Coverage gate — fail if below 80%
      - COVERAGE=$(python -c "import xml.etree.ElementTree as ET; t=ET.parse('coverage.xml').getroot(); print(round(float(t.get('line-rate'))*100))")
      - echo "Coverage is ${COVERAGE}%"
      - "[ $COVERAGE -ge 80 ] || (echo 'Coverage below 80% - failing build' && exit 1)"

  build:
    commands:
      - echo "Building Docker image..."
      - |
        docker build \
          --build-arg BUILD_DATE=$BUILD_DATE \
          --build-arg GIT_SHA=$GIT_SHA \
          --build-arg VERSION=$IMAGE_TAG \
          --label "org.opencontainers.image.created=$BUILD_DATE" \
          --label "org.opencontainers.image.revision=$GIT_SHA" \
          --cache-from $ECR_REGISTRY/$ECR_REPO:latest \
          --tag $FULL_IMAGE \
          --tag $ECR_REGISTRY/$ECR_REPO:latest \
          .

      # Container vulnerability scan — fail on CRITICAL
      - echo "Scanning container image with Trivy..."
      - |
        trivy image \
          --exit-code 1 \
          --severity CRITICAL \
          --no-progress \
          --format json \
          --output trivy-report.json \
          $FULL_IMAGE
      - trivy image --severity HIGH,CRITICAL --no-progress $FULL_IMAGE

      # Push images
      - echo "Pushing image to ECR..."
      - docker push $FULL_IMAGE
      - docker push $ECR_REGISTRY/$ECR_REPO:latest

      # Write image definition for CodeDeploy/ECS
      - |
        printf '[{"name":"api","imageUri":"%s"}]' $FULL_IMAGE \
          > imagedefinitions.json

      # Write imageDetail for EKS/Helm
      - |
        cat > imageDetail.json << EOF
        {
          "imageTag": "$IMAGE_TAG",
          "imageUri": "$FULL_IMAGE",
          "gitSha": "$GIT_SHA",
          "buildDate": "$BUILD_DATE",
          "buildNumber": "$CODEBUILD_BUILD_NUMBER"
        }
        EOF

  post_build:
    commands:
      - echo "Build completed: $IMAGE_TAG"

      # Update SSM Parameter with latest image tag (for downstream pipelines)
      - |
        aws ssm put-parameter \
          --name "/url-shortener/prod/latest-image-tag" \
          --value "$IMAGE_TAG" \
          --type String \
          --overwrite

      # Publish test results to CodeBuild Test Reports
      - echo "Post-build complete"

reports:
  unit-test-report:
    files:
      - test-results.xml
    file-format: JUNITXML
  coverage-report:
    files:
      - coverage.xml
    file-format: COBERTURAXML

artifacts:
  files:
    - imagedefinitions.json
    - imageDetail.json
    - trivy-report.json
    - bandit-report.json
    - safety-report.json
    - k8s/**/*
    - helm/**/*
  secondary-artifacts:
    trivy-report:
      files:
        - trivy-report.json

cache:
  paths:
    - '/root/.cache/pip/**/*'
    - '/root/.docker/**/*'
```

**Create CodeBuild Project via CLI**

```
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=ap-south-1

# Create CodeBuild service role
cat > codebuild-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "codebuild.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name CodeBuildURLShortenerRole \
  --assume-role-policy-document file://codebuild-trust.json

cat > codebuild-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeImages"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetBucketAcl",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::url-shortener-artifacts-${ACCOUNT_ID}",
        "arn:aws:s3:::url-shortener-artifacts-${ACCOUNT_ID}/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameters",
        "ssm:PutParameter",
        "secretsmanager:GetSecretValue",
        "kms:Decrypt"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["sts:GetCallerIdentity"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeDhcpOptions",
        "ec2:DescribeVpcs",
        "ec2:CreateNetworkInterfacePermission"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name CodeBuildURLShortenerRole \
  --policy-name CodeBuildPermissions \
  --policy-document file://codebuild-policy.json

# Create S3 bucket for artifacts
ARTIFACT_BUCKET="url-shortener-artifacts-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket $ARTIFACT_BUCKET \
  --region $AWS_REGION \
  --create-bucket-configuration LocationConstraint=$AWS_REGION

aws s3api put-bucket-versioning \
  --bucket $ARTIFACT_BUCKET \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket $ARTIFACT_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Lifecycle — delete old artifacts after 90 days
aws s3api put-bucket-lifecycle-configuration \
  --bucket $ARTIFACT_BUCKET \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-old-artifacts",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Expiration": {"Days": 90},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
    }]
  }'

# Create the CodeBuild project
cat > codebuild-project.json << EOF
{
  "name": "url-shortener-build",
  "description": "Build, test and push url-shortener API",
  "source": {
    "type": "GITHUB",
    "location": "https://github.com/your-username/url-shortener",
    "buildspec": "buildspec.yml",
    "gitCloneDepth": 1,
    "reportBuildStatus": true
  },
  "sourceVersion": "main",
  "artifacts": {
    "type": "S3",
    "location": "${ARTIFACT_BUCKET}",
    "name": "build-output",
    "namespaceType": "BUILD_ID",
    "packaging": "ZIP",
    "overrideArtifactName": false
  },
  "environment": {
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/standard:7.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true,
    "environmentVariables": [
      {"name": "AWS_REGION",    "value": "${AWS_REGION}", "type": "PLAINTEXT"},
      {"name": "ECR_REPO",      "value": "url-shortener/api", "type": "PLAINTEXT"}
    ]
  },
  "serviceRole": "arn:aws:iam::${ACCOUNT_ID}:role/CodeBuildURLShortenerRole",
  "vpcConfig": {
    "vpcId": "${VPC_ID}",
    "subnets": ["${PRIV_SUBNET_A}", "${PRIV_SUBNET_B}"],
    "securityGroupIds": ["${CODEBUILD_SG}"]
  },
  "cache": {
    "type": "S3",
    "location": "${ARTIFACT_BUCKET}/cache"
  },
  "logsConfig": {
    "cloudWatchLogs": {
      "status": "ENABLED",
      "groupName": "/codebuild/url-shortener",
      "streamName": "build"
    },
    "s3Logs": {
      "status": "ENABLED",
      "location": "${ARTIFACT_BUCKET}/build-logs"
    }
  },
  "timeoutInMinutes": 30,
  "queuedTimeoutInMinutes": 60,
  "tags": [
    {"key": "Project",     "value": "url-shortener"},
    {"key": "Environment", "value": "production"}
  ]
}
EOF

aws codebuild create-project \
  --cli-input-json file://codebuild-project.json

# Connect GitHub via webhook (triggers build on every push)
aws codebuild create-webhook \
  --project-name url-shortener-build \
  --filter-groups \
    '[[ {"type": "EVENT", "pattern": "PUSH,PULL_REQUEST_MERGED"},
        {"type": "HEAD_REF", "pattern": "^refs/heads/main$"},
        {"type": "FILE_PATH", "pattern": "^(app|tests|Dockerfile|buildspec)"}
    ]]'

# Trigger a manual build
aws codebuild start-build \
  --project-name url-shortener-build \
  --source-version main

# Watch build logs in real time
BUILD_ID=$(aws codebuild list-builds-for-project \
  --project-name url-shortener-build \
  --sort-order DESCENDING \
  --query 'ids[0]' --output text)

aws codebuild batch-get-builds \
  --ids $BUILD_ID \
  --query 'builds[0].{Status:buildStatus,Phase:currentPhase,Duration:buildComplete}' \
  --output table
```

## PART 2 — AWS CodeDeploy

CodeDeploy automates deployments to EC2, ECS, Lambda, and on-premises. Three deployment strategies: In-Place, Rolling, Blue/Green.

**Deployment Strategies Explained**

```
IN-PLACE (EC2 only):
  Stop app → Deploy new → Start app
  Downtime during deployment. Never for production.

ROLLING:
  [v1][v1][v1][v1]  →  [v2][v2][v1][v1]  →  [v2][v2][v2][v2]
  Partial downtime risk. Good for non-critical.

BLUE/GREEN:
  Blue (v1) ────────────────── still live
  Green (v2) ─── deploy ─── test ─── shift traffic ─── terminate blue
  Zero downtime. Instant rollback (flip back to blue).
  Used for: ECS, Lambda, and API Gateway.

CANARY (Lambda/ECS):
  5% traffic → green (v2)     ← bake for 10 minutes
  95% traffic → blue (v1)     ← monitor alarms
  If no alarms → shift 100% to green
  If alarm fires → rollback instantly
```

**AppSpec File — CodeDeploy's buildspec equivalent**

```
# appspec.yml — for ECS Blue/Green deployment
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "api"
          ContainerPort: 8000
        PlatformVersion: "LATEST"
        NetworkConfiguration:
          AwsvpcConfiguration:
            Subnets:
              - subnet-xxxxxxxxx
              - subnet-yyyyyyyyy
            SecurityGroups:
              - sg-xxxxxxxxx
            AssignPublicIp: DISABLED

Hooks:
  # Run validation after green is deployed but before traffic shifts
  - BeforeAllowTraffic: "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:validate-green-deployment"
  # Run smoke tests after 100% traffic shifted
  - AfterAllowTraffic: "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:post-deployment-smoke-test"
```

```
# Create the Lambda validation hooks
cat > validate-green.py << 'EOF'
import boto3
import urllib.request
import json

def handler(event, context):
    deployment_id = event['DeploymentId']
    lifecycle_hook = event['LifecycleEventHookExecutionId']
    codedeploy = boto3.client('codedeploy')

    try:
        # Hit the green target group health endpoint
        # In real scenario, get the green ALB endpoint from deployment info
        response = urllib.request.urlopen(
            'http://internal-alb-dns/health', timeout=10
        )
        data = json.loads(response.read())

        if data.get('status') == 'healthy':
            print("Green deployment is healthy, allowing traffic shift")
            codedeploy.put_lifecycle_event_hook_execution_status(
                deploymentId=deployment_id,
                lifecycleEventHookExecutionId=lifecycle_hook,
                status='Succeeded'
            )
        else:
            raise Exception(f"Unhealthy response: {data}")

    except Exception as e:
        print(f"Validation failed: {e}")
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=lifecycle_hook,
            status='Failed'
        )
EOF

zip validate-green.zip validate-green.py

aws lambda create-function \
  --function-name validate-green-deployment \
  --runtime python3.12 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/lambda-codedeploy-hook-role \
  --handler validate-green.handler \
  --zip-file fileb://validate-green.zip \
  --timeout 60

# Create CodeDeploy Application and Deployment Group
aws deploy create-application \
  --application-name url-shortener \
  --compute-platform ECS

aws deploy create-deployment-group \
  --application-name url-shortener \
  --deployment-group-name prod-blue-green \
  --deployment-config-name CodeDeployDefault.ECSCanary10Percent5Minutes \
  --service-role-arn arn:aws:iam::${ACCOUNT_ID}:role/CodeDeployECSRole \
  --ecs-services clusterName=url-shortener-prod,serviceName=url-shortener-api \
  --load-balancer-info '{
    "targetGroupPairInfoList": [{
      "targetGroups": [
        {"name": "url-shortener-tg"},
        {"name": "url-shortener-tg-green"}
      ],
      "prodTrafficRoute": {
        "listenerArns": ["arn:aws:elasticloadbalancing:ap-south-1:'$ACCOUNT_ID':listener/app/url-shortener-alb/xxx/yyy"]
      },
      "testTrafficRoute": {
        "listenerArns": ["arn:aws:elasticloadbalancing:ap-south-1:'$ACCOUNT_ID':listener/app/url-shortener-alb/xxx/zzz"]
      }
    }]
  }' \
  --blue-green-deployment-configuration '{
    "terminateBlueInstancesOnDeploymentSuccess": {
      "action": "TERMINATE",
      "terminationWaitTimeInMinutes": 10
    },
    "deploymentReadyOption": {
      "actionOnTimeout": "STOP_DEPLOYMENT",
      "waitTimeInMinutes": 60
    }
  }' \
  --auto-rollback-configuration '{
    "enabled": true,
    "events": [
      "DEPLOYMENT_FAILURE",
      "DEPLOYMENT_STOP_ON_ALARM",
      "DEPLOYMENT_STOP_ON_REQUEST"
    ]
  }' \
  --alarm-configuration '{
    "enabled": true,
    "alarms": [
      {"name": "url-shortener-5xx-rate"},
      {"name": "url-shortener-high-latency"},
      {"name": "url-shortener-target-unhealthy"}
    ]
  }'

# Available deployment configs — know these for interviews
aws deploy list-deployment-configs --query 'deploymentConfigsList' --output table
# ECSLinear10PercentEvery1Minute  — gradually 10% per minute
# ECSLinear10PercentEvery3Minutes — gradually 10% per 3 min
# ECSCanary10Percent5Minutes      — 10% canary, wait 5 min, then 100%
# ECSCanary10Percent15Minutes     — 10% canary, wait 15 min, then 100%
# ECSAllAtOnce                    — flip all at once (not recommended prod)
```

## PART 3 — AWS CodePipeline (Full CI/CD Orchestration)

CodePipeline ties CodeBuild + CodeDeploy together into an automated delivery pipeline. Every commit flows automatically from source to production.

**Pipeline Architecture**

```
GitHub (push to main)
    │
    ▼
Stage 1: SOURCE
    └── Pulls code, stores in S3 artifact bucket
    │
    ▼
Stage 2: BUILD
    ├── Action 1: CodeBuild (test + build + push to ECR)
    └── Output: imagedefinitions.json + k8s manifests
    │
    ▼
Stage 3: VALIDATE
    ├── Action 1: CodeBuild (run integration tests against staging)
    └── Action 2: Manual approval gate (optional for prod)
    │
    ▼
Stage 4: DEPLOY-STAGING
    └── CodeDeploy → ECS staging cluster
    │
    ▼
Stage 5: INTEGRATION-TEST
    └── CodeBuild (Newman/Postman API tests against staging)
    │
    ▼
Stage 6: APPROVAL
    └── Manual approval (SNS notification → team lead approves)
    │
    ▼
Stage 7: DEPLOY-PRODUCTION
    └── CodeDeploy → ECS prod (Canary 10% → wait → 100%)
```

**Build the Full Pipeline**

```
# Create CodePipeline service role
cat > codepipeline-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "codepipeline.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name CodePipelineURLShortenerRole \
  --assume-role-policy-document file://codepipeline-trust.json

cat > codepipeline-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::${ARTIFACT_BUCKET}",
        "arn:aws:s3:::${ARTIFACT_BUCKET}/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild",
        "codebuild:StopBuild"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:RegisterApplicationRevision",
        "codedeploy:GetApplicationRevision"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["iam:PassRole"],
      "Resource": [
        "arn:aws:iam::${ACCOUNT_ID}:role/ecs-task-execution-role",
        "arn:aws:iam::${ACCOUNT_ID}:role/ecs-url-shortener-task-role"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish",
        "codestar-connections:UseConnection"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:GetMetricData"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name CodePipelineURLShortenerRole \
  --policy-name CodePipelinePermissions \
  --policy-document file://codepipeline-policy.json

# Create GitHub connection (CodeStar connection)
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name url-shortener-github

# IMPORTANT: After creation, go to AWS Console →
# Developer Tools → Connections → complete the OAuth handshake manually
# Then get the connection ARN:
GITHUB_CONNECTION_ARN=$(aws codestar-connections list-connections \
  --query 'Connections[?ConnectionName==`url-shortener-github`].ConnectionArn' \
  --output text)

# Create SNS topic for approvals
APPROVAL_TOPIC=$(aws sns create-topic \
  --name url-shortener-pipeline-approval \
  --query 'TopicArn' --output text)

aws sns subscribe \
  --topic-arn $APPROVAL_TOPIC \
  --protocol email \
  --notification-endpoint your.email@gmail.com

# Create CodeBuild project for integration tests
cat > integration-test-buildspec.yml << 'EOF'
version: 0.2
phases:
  install:
    commands:
      - npm install -g newman
  build:
    commands:
      - STAGING_URL=$(aws ssm get-parameter --name /url-shortener/staging/alb-url --query Parameter.Value --output text)
      - echo "Running integration tests against $STAGING_URL"
      - newman run tests/postman-collection.json
          --environment tests/staging-env.json
          --global-var "base_url=$STAGING_URL"
          --reporters cli,json
          --reporter-json-export newman-results.json
          --bail
artifacts:
  files:
    - newman-results.json
EOF

aws codebuild create-project \
  --name url-shortener-integration-tests \
  --source '{"type":"CODEPIPELINE","buildspec":"integration-test-buildspec.yml"}' \
  --artifacts '{"type":"CODEPIPELINE"}' \
  --environment '{"type":"LINUX_CONTAINER","image":"aws/codebuild/standard:7.0","computeType":"BUILD_GENERAL1_SMALL"}' \
  --service-role "arn:aws:iam::${ACCOUNT_ID}:role/CodeBuildURLShortenerRole"

# Create the full pipeline
cat > pipeline.json << EOF
{
  "name": "url-shortener-pipeline",
  "roleArn": "arn:aws:iam::${ACCOUNT_ID}:role/CodePipelineURLShortenerRole",
  "artifactStore": {
    "type": "S3",
    "location": "${ARTIFACT_BUCKET}",
    "encryptionKey": {
      "id": "arn:aws:kms:${AWS_REGION}:${ACCOUNT_ID}:alias/codepipeline-key",
      "type": "KMS"
    }
  },
  "stages": [
    {
      "name": "Source",
      "actions": [{
        "name": "GitHub-Source",
        "actionTypeId": {
          "category": "Source",
          "owner": "AWS",
          "provider": "CodeStarSourceConnection",
          "version": "1"
        },
        "configuration": {
          "ConnectionArn": "${GITHUB_CONNECTION_ARN}",
          "FullRepositoryId": "your-username/url-shortener",
          "BranchName": "main",
          "OutputArtifactFormat": "CODE_ZIP",
          "DetectChanges": "true"
        },
        "outputArtifacts": [{"name": "SourceOutput"}],
        "runOrder": 1
      }]
    },
    {
      "name": "Build",
      "actions": [{
        "name": "Build-and-Push",
        "actionTypeId": {
          "category": "Build",
          "owner": "AWS",
          "provider": "CodeBuild",
          "version": "1"
        },
        "configuration": {
          "ProjectName": "url-shortener-build",
          "EnvironmentVariables": "[{\"name\":\"ENVIRONMENT\",\"value\":\"production\",\"type\":\"PLAINTEXT\"}]"
        },
        "inputArtifacts":  [{"name": "SourceOutput"}],
        "outputArtifacts": [{"name": "BuildOutput"}],
        "runOrder": 1
      }]
    },
    {
      "name": "Deploy-Staging",
      "actions": [{
        "name": "ECS-Deploy-Staging",
        "actionTypeId": {
          "category": "Deploy",
          "owner": "AWS",
          "provider": "CodeDeployToECS",
          "version": "1"
        },
        "configuration": {
          "ApplicationName": "url-shortener-staging",
          "DeploymentGroupName": "staging-blue-green",
          "TaskDefinitionTemplateArtifact": "BuildOutput",
          "TaskDefinitionTemplatePath": "taskdef.json",
          "AppSpecTemplateArtifact": "BuildOutput",
          "AppSpecTemplatePath": "appspec.yml",
          "Image1ArtifactName": "BuildOutput",
          "Image1ContainerName": "IMAGE1_NAME"
        },
        "inputArtifacts": [{"name": "BuildOutput"}],
        "runOrder": 1
      }]
    },
    {
      "name": "Integration-Tests",
      "actions": [{
        "name": "Run-Integration-Tests",
        "actionTypeId": {
          "category": "Build",
          "owner": "AWS",
          "provider": "CodeBuild",
          "version": "1"
        },
        "configuration": {
          "ProjectName": "url-shortener-integration-tests"
        },
        "inputArtifacts":  [{"name": "BuildOutput"}],
        "outputArtifacts": [{"name": "TestOutput"}],
        "runOrder": 1
      }]
    },
    {
      "name": "Approval",
      "actions": [{
        "name": "Manual-Approval",
        "actionTypeId": {
          "category": "Approval",
          "owner": "AWS",
          "provider": "Manual",
          "version": "1"
        },
        "configuration": {
          "NotificationArn": "${APPROVAL_TOPIC}",
          "CustomData": "Please review integration test results before approving production deployment.",
          "ExternalEntityLink": "https://console.aws.amazon.com/codesuite/codepipeline/pipelines/url-shortener-pipeline"
        },
        "runOrder": 1
      }]
    },
    {
      "name": "Deploy-Production",
      "actions": [{
        "name": "ECS-Deploy-Production",
        "actionTypeId": {
          "category": "Deploy",
          "owner": "AWS",
          "provider": "CodeDeployToECS",
          "version": "1"
        },
        "configuration": {
          "ApplicationName": "url-shortener",
          "DeploymentGroupName": "prod-blue-green",
          "TaskDefinitionTemplateArtifact": "BuildOutput",
          "TaskDefinitionTemplatePath": "taskdef.json",
          "AppSpecTemplateArtifact": "BuildOutput",
          "AppSpecTemplatePath": "appspec.yml",
          "Image1ArtifactName": "BuildOutput",
          "Image1ContainerName": "IMAGE1_NAME"
        },
        "inputArtifacts": [{"name": "BuildOutput"}],
        "runOrder": 1
      }]
    }
  ]
}
EOF

aws codepipeline create-pipeline \
  --cli-input-json file://pipeline.json

# Add pipeline notifications (all state changes)
aws codestar-notifications create-notification-rule \
  --name url-shortener-pipeline-notifications \
  --resource "arn:aws:codepipeline:${AWS_REGION}:${ACCOUNT_ID}:url-shortener-pipeline" \
  --event-type-ids \
    codepipeline-pipeline-pipeline-execution-failed \
    codepipeline-pipeline-pipeline-execution-succeeded \
    codepipeline-pipeline-stage-execution-failed \
    codepipeline-pipeline-action-execution-failed \
  --targets "Type=SNS,Address=${APPROVAL_TOPIC}" \
  --detail-type FULL

# View pipeline state
aws codepipeline get-pipeline-state \
  --name url-shortener-pipeline \
  --query 'stageStates[].{Stage:stageName,Status:latestExecution.status}' \
  --output table
```

## PART 4 — AWS CloudFormation (IaC Deep Dive)

You know Terraform from your 35-day course. CloudFormation is AWS-native IaC. In interviews you'll be asked to compare them. In enterprises you'll often see both.

**CloudFormation vs Terraform — Key Differences**

<img width="935" height="436" alt="image" src="https://github.com/user-attachments/assets/0c13c8bc-55dd-4528-a702-2eacfb8d7b1d" />

**CloudFormation Template Anatomy**

```
AWSTemplateFormatVersion: '2010-09-09'
Description: "URL Shortener — Complete infrastructure stack"

# ── Metadata: organize parameters in console ──────────────
Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label: {default: "Network"}
        Parameters: [VpcId, PrivateSubnet1, PrivateSubnet2]
      - Label: {default: "Application"}
        Parameters: [Environment, AppVersion, DesiredCount]
      - Label: {default: "Database"}
        Parameters: [DBInstanceClass, DBPassword]

# ── Parameters: inputs at deploy time ─────────────────────
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, production]
    Default: production
    Description: Deployment environment

  AppVersion:
    Type: String
    Description: Docker image tag to deploy

  VpcId:
    Type: AWS::EC2::VPC::Id

  PrivateSubnet1:
    Type: AWS::EC2::Subnet::Id

  PrivateSubnet2:
    Type: AWS::EC2::Subnet::Id

  DBInstanceClass:
    Type: String
    Default: db.t3.medium
    AllowedValues:
      - db.t3.micro
      - db.t3.medium
      - db.r5.large

  DBPassword:
    Type: String
    NoEcho: true      # hides value in console and events
    MinLength: 12
    Description: RDS master password

  DesiredCount:
    Type: Number
    Default: 3
    MinValue: 1
    MaxValue: 20

# ── Mappings: lookup tables based on param values ─────────
Mappings:
  EnvironmentConfig:
    dev:
      MultiAZ: false
      DeletionPolicy: Delete
      LogRetentionDays: 7
    staging:
      MultiAZ: false
      DeletionPolicy: Snapshot
      LogRetentionDays: 14
    production:
      MultiAZ: true
      DeletionPolicy: Snapshot
      LogRetentionDays: 90

  RegionAMI:
    ap-south-1:
      AmazonLinux2023: ami-0f58b397bc5c1f2e8
    eu-west-1:
      AmazonLinux2023: ami-0694d931cee176e7d

# ── Conditions: create resources conditionally ────────────
Conditions:
  IsProduction: !Equals [!Ref Environment, production]
  IsNotDev: !Not [!Equals [!Ref Environment, dev]]
  CreateReadReplica: !And
    - !Condition IsProduction
    - !Equals [!Ref DBInstanceClass, db.r5.large]

# ── Resources: the actual AWS resources ───────────────────
Resources:

  # ── CloudWatch Log Group ────────────────────────────────
  AppLogGroup:
    Type: AWS::Logs::LogGroup
    DeletionPolicy: !FindInMap [EnvironmentConfig, !Ref Environment, DeletionPolicy]
    Properties:
      LogGroupName: !Sub "/ecs/${AWS::StackName}"
      RetentionInDays: !FindInMap [EnvironmentConfig, !Ref Environment, LogRetentionDays]
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ── ECS Task Execution Role ─────────────────────────────
  TaskExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "${AWS::StackName}-task-execution-role"
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: {Service: ecs-tasks.amazonaws.com}
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
      Policies:
        - PolicyName: SecretsAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - secretsmanager:GetSecretValue
                  - kms:Decrypt
                Resource: !Sub "arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:/url-shortener/*"

  # ── ECS Task Role ───────────────────────────────────────
  TaskRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: {Service: ecs-tasks.amazonaws.com}
            Action: sts:AssumeRole
      Policies:
        - PolicyName: AppPermissions
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: [dynamodb:PutItem, dynamodb:GetItem, dynamodb:UpdateItem, dynamodb:Query]
                Resource: !GetAtt UrlShortenerTable.Arn
              - Effect: Allow
                Action: [s3:GetObject, s3:PutObject]
                Resource: !Sub "${AssetsBucket.Arn}/*"

  # ── DynamoDB Table ──────────────────────────────────────
  UrlShortenerTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: !If [IsProduction, Retain, Delete]
    Properties:
      TableName: !Sub "url-shortener-${Environment}"
      BillingMode: PAY_PER_REQUEST
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: !If [IsProduction, true, false]
      SSESpecification:
        SSEEnabled: true
        SSEType: KMS
      AttributeDefinitions:
        - AttributeName: short_code
          AttributeType: S
        - AttributeName: created_at
          AttributeType: S
        - AttributeName: owner
          AttributeType: S
      KeySchema:
        - AttributeName: short_code
          KeyType: HASH
        - AttributeName: created_at
          KeyType: RANGE
      GlobalSecondaryIndexes:
        - IndexName: owner-index
          KeySchema:
            - AttributeName: owner
              KeyType: HASH
            - AttributeName: created_at
              KeyType: RANGE
          Projection:
            ProjectionType: ALL
      TimeToLiveSpecification:
        AttributeName: expires_at
        Enabled: true
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ── RDS PostgreSQL ──────────────────────────────────────
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: !Sub "${AWS::StackName} DB subnets"
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2

  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: !FindInMap [EnvironmentConfig, !Ref Environment, DeletionPolicy]
    Properties:
      DBInstanceIdentifier: !Sub "${AWS::StackName}-postgres"
      DBInstanceClass: !Ref DBInstanceClass
      Engine: postgres
      EngineVersion: "15.4"
      MasterUsername: devopsadmin
      MasterUserPassword: !Ref DBPassword
      DBName: urlshortener
      AllocatedStorage: 20
      StorageType: gp3
      StorageEncrypted: true
      MultiAZ: !FindInMap [EnvironmentConfig, !Ref Environment, MultiAZ]
      BackupRetentionPeriod: !If [IsProduction, 30, 1]
      DeletionProtection: !If [IsProduction, true, false]
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref DBSecurityGroup
      EnablePerformanceInsights: !If [IsProduction, true, false]
      MonitoringInterval: !If [IsProduction, 60, 0]
      MonitoringRoleArn: !If
        - IsProduction
        - !GetAtt RDSMonitoringRole.Arn
        - !Ref AWS::NoValue
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ── ECS Cluster ─────────────────────────────────────────
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub "${AWS::StackName}-cluster"
      CapacityProviders: [FARGATE, FARGATE_SPOT]
      DefaultCapacityProviderStrategy:
        - CapacityProvider: FARGATE
          Weight: 1
          Base: 1
        - CapacityProvider: FARGATE_SPOT
          Weight: 3
      ClusterSettings:
        - Name: containerInsights
          Value: enabled

  # ── ECS Task Definition ─────────────────────────────────
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub "${AWS::StackName}"
      NetworkMode: awsvpc
      RequiresCompatibilities: [FARGATE]
      Cpu: '1024'
      Memory: '2048'
      ExecutionRoleArn: !GetAtt TaskExecutionRole.Arn
      TaskRoleArn: !GetAtt TaskRole.Arn
      ContainerDefinitions:
        - Name: api
          Image: !Sub "${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/url-shortener/api:${AppVersion}"
          PortMappings:
            - ContainerPort: 8000
          Essential: true
          Environment:
            - Name: ENVIRONMENT
              Value: !Ref Environment
            - Name: DYNAMODB_TABLE
              Value: !Ref UrlShortenerTable
          Secrets:
            - Name: DATABASE_URL
              ValueFrom: !Sub "arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:/url-shortener/prod/db-password"
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref AppLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: api
          HealthCheck:
            Command: ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
            Interval: 30
            Timeout: 10
            Retries: 3
            StartPeriod: 60

  # ── ECS Service ─────────────────────────────────────────
  ECSService:
    Type: AWS::ECS::Service
    DependsOn: ALBHTTPSListener
    Properties:
      ServiceName: !Sub "${AWS::StackName}-api"
      Cluster: !Ref ECSCluster
      TaskDefinition: !Ref TaskDefinition
      DesiredCount: !Ref DesiredCount
      LaunchType: FARGATE
      PlatformVersion: LATEST
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets:
            - !Ref PrivateSubnet1
            - !Ref PrivateSubnet2
          SecurityGroups:
            - !Ref ECSSecurityGroup
          AssignPublicIp: DISABLED
      LoadBalancers:
        - TargetGroupArn: !Ref ALBTargetGroup
          ContainerName: api
          ContainerPort: 8000
      DeploymentConfiguration:
        DeploymentCircuitBreaker:
          Enable: true
          Rollback: true
        MaximumPercent: 200
        MinimumHealthyPercent: 100
      EnableExecuteCommand: true
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ── Auto Scaling ─────────────────────────────────────────
  ScalableTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      ServiceNamespace: ecs
      ResourceId: !Sub "service/${ECSCluster}/${ECSService.Name}"
      ScalableDimension: ecs:service:DesiredCount
      MinCapacity: !If [IsProduction, 3, 1]
      MaxCapacity: !If [IsProduction, 20, 5]

  CPUScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: cpu-target-tracking
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref ScalableTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 60.0
        PredefinedMetricSpecification:
          PredefinedMetricType: ECSServiceAverageCPUUtilization
        ScaleInCooldown: 300
        ScaleOutCooldown: 60

  # ── S3 Assets Bucket ─────────────────────────────────────
  AssetsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: !If [IsProduction, Retain, Delete]
    Properties:
      BucketName: !Sub "${AWS::StackName}-assets-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
            BucketKeyEnabled: true
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      LifecycleConfiguration:
        Rules:
          - Id: expire-old-versions
            Status: Enabled
            NoncurrentVersionExpiration:
              NoncurrentDays: 90

# ── Outputs: values used by other stacks or pipelines ─────
Outputs:
  ALBDNSName:
    Description: Application Load Balancer DNS
    Value: !GetAtt ApplicationLoadBalancer.DNSName
    Export:
      Name: !Sub "${AWS::StackName}-ALBDNSName"

  ECSClusterName:
    Description: ECS Cluster Name
    Value: !Ref ECSCluster
    Export:
      Name: !Sub "${AWS::StackName}-ClusterName"

  DynamoDBTableName:
    Description: DynamoDB Table Name
    Value: !Ref UrlShortenerTable
    Export:
      Name: !Sub "${AWS::StackName}-DynamoDBTable"

  DBEndpoint:
    Description: RDS Endpoint
    Value: !GetAtt DBInstance.Endpoint.Address
    Export:
      Name: !Sub "${AWS::StackName}-DBEndpoint"
```

**Deploy and Manage CloudFormation Stacks**

```
# Validate template before deploying
aws cloudformation validate-template \
  --template-body file://url-shortener-stack.yaml

# Deploy with changeset (safe — preview before apply, like terraform plan)
aws cloudformation create-change-set \
  --stack-name url-shortener-prod \
  --template-body file://url-shortener-stack.yaml \
  --change-set-name initial-deploy \
  --parameters \
    ParameterKey=Environment,ParameterValue=production \
    ParameterKey=AppVersion,ParameterValue=v1.0.0 \
    ParameterKey=VpcId,ParameterValue=$VPC_ID \
    ParameterKey=PrivateSubnet1,ParameterValue=$PRIV_SUBNET_A \
    ParameterKey=PrivateSubnet2,ParameterValue=$PRIV_SUBNET_B \
    ParameterKey=DBPassword,ParameterValue=SecurePass123! \
  --capabilities CAPABILITY_NAMED_IAM \
  --change-set-type CREATE

# Wait for changeset to finish computing
aws cloudformation wait change-set-create-complete \
  --stack-name url-shortener-prod \
  --change-set-name initial-deploy

# Review the changeset (like terraform plan output)
aws cloudformation describe-change-set \
  --stack-name url-shortener-prod \
  --change-set-name initial-deploy \
  --query 'Changes[].{Action:ResourceChange.Action,Resource:ResourceChange.ResourceType,LogicalId:ResourceChange.LogicalResourceId,Replacement:ResourceChange.Replacement}' \
  --output table

# Execute the changeset
aws cloudformation execute-change-set \
  --stack-name url-shortener-prod \
  --change-set-name initial-deploy

# Watch the rollout
aws cloudformation wait stack-create-complete \
  --stack-name url-shortener-prod

# Get stack events during deploy
aws cloudformation describe-stack-events \
  --stack-name url-shortener-prod \
  --query 'StackEvents[].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId,ResourceStatusReason]' \
  --output table | head -40

# Update the stack with a new version (rolling changeset update)
aws cloudformation create-change-set \
  --stack-name url-shortener-prod \
  --template-body file://url-shortener-stack.yaml \
  --change-set-name update-v1.1.0 \
  --parameters \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=AppVersion,ParameterValue=v1.1.0 \
    ParameterKey=VpcId,UsePreviousValue=true \
    ParameterKey=PrivateSubnet1,UsePreviousValue=true \
    ParameterKey=PrivateSubnet2,UsePreviousValue=true \
    ParameterKey=DBPassword,UsePreviousValue=true \
  --capabilities CAPABILITY_NAMED_IAM \
  --change-set-type UPDATE

aws cloudformation execute-change-set \
  --stack-name url-shortener-prod \
  --change-set-name update-v1.1.0

# Detect drift — has anyone manually changed resources?
aws cloudformation detect-stack-drift \
  --stack-name url-shortener-prod

DRIFT_ID=$(aws cloudformation detect-stack-drift \
  --stack-name url-shortener-prod \
  --query 'StackDriftDetectionId' --output text)

aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id $DRIFT_ID

aws cloudformation describe-stack-resource-drifts \
  --stack-name url-shortener-prod \
  --stack-resource-drift-status-filters MODIFIED DELETED

# StackSets — deploy same template across multiple accounts/regions
aws cloudformation create-stack-set \
  --stack-set-name url-shortener-global \
  --template-body file://url-shortener-stack.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --permission-model SERVICE_MANAGED

aws cloudformation create-stack-instances \
  --stack-set-name url-shortener-global \
  --regions ap-south-1 eu-west-1 us-east-1 \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-yyyyyyy \
  --parameter-overrides \
    ParameterKey=Environment,ParameterValue=production
```

**CloudFormation Custom Resource (Lambda-backed)**

When CloudFormation doesn't support a resource natively, you write a Lambda to handle it.

```
# custom_resource_handler.py
import json
import boto3
import cfnresponse

def handler(event, context):
    """
    Custom resource: create a DNS-validated ACM certificate
    and wait for validation (CloudFormation native resource
    doesn't wait for DNS validation automatically)
    """
    props = event['ResourceProperties']
    domain = props['DomainName']
    hosted_zone_id = props['HostedZoneId']

    acm = boto3.client('acm', region_name='us-east-1')
    route53 = boto3.client('route53')

    try:
        if event['RequestType'] in ['Create', 'Update']:
            # Request certificate
            cert_arn = acm.request_certificate(
                DomainName=domain,
                ValidationMethod='DNS',
                SubjectAlternativeNames=[f'*.{domain}'],
                IdempotencyToken=event['LogicalResourceId']
            )['CertificateArn']

            # Get validation CNAME record
            import time
            for _ in range(30):
                cert = acm.describe_certificate(CertificateArn=cert_arn)
                options = cert['Certificate'].get('DomainValidationOptions', [])
                if options and 'ResourceRecord' in options[0]:
                    record = options[0]['ResourceRecord']
                    break
                time.sleep(10)

            # Add CNAME to Route53
            route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch={
                    'Changes': [{
                        'Action': 'UPSERT',
                        'ResourceRecordSet': {
                            'Name': record['Name'],
                            'Type': 'CNAME',
                            'TTL': 60,
                            'ResourceRecords': [{'Value': record['Value']}]
                        }
                    }]
                }
            )

            # Wait for validation
            waiter = acm.get_waiter('certificate_validated')
            waiter.wait(CertificateArn=cert_arn,
                       WaiterConfig={'Delay': 30, 'MaxAttempts': 40})

            cfnresponse.send(event, context, cfnresponse.SUCCESS,
                           {'CertificateArn': cert_arn},
                           physicalResourceId=cert_arn)

        elif event['RequestType'] == 'Delete':
            acm.delete_certificate(
                CertificateArn=event['PhysicalResourceId']
            )
            cfnresponse.send(event, context, cfnresponse.SUCCESS, {})

    except Exception as e:
        print(f"Error: {e}")
        cfnresponse.send(event, context, cfnresponse.FAILED,
                        {'Error': str(e)})
```

## PART 5 — Route 53 (DNS + Traffic Management)

Route 53 is AWS's DNS service but also a global traffic management layer. This is where you control how the world reaches your application.

**Routing Policies — The Core of Route 53**

```
SIMPLE:         one record → one IP. No health checks. Basic.

WEIGHTED:       record A → 80% traffic (v1)
                record B → 20% traffic (v2)
                Use for: canary releases, A/B testing, gradual migration

LATENCY:        Route to region with lowest latency for the user.
                ap-south-1 → serves India users
                eu-west-1  → serves Europe users
                Use for: multi-region active-active

FAILOVER:       Primary: ap-south-1 (health checked)
                Secondary: eu-west-1 (promoted if primary fails)
                Use for: active-passive DR

GEOLOCATION:    Users from IN → ap-south-1
                Users from DE → eu-central-1
                Users from * → us-east-1 (default)
                Use for: compliance, GDPR, content localization

GEOPROXIMITY:   Like Geolocation but with bias.
                You can expand or shrink a region's coverage area.
                Use for: fine-grained global load distribution

MULTIVALUE:     Up to 8 records, all health checked.
                Like a simple load balancer at DNS level.
                NOT a replacement for ALB — no TLS termination.

IP-BASED:       Route based on CIDR blocks.
                Use for: corporate network routing, ISP differentiation
```

**Hands-On: Complete Route 53 Setup**

```
# Create hosted zone
HOSTED_ZONE_ID=$(aws route53 create-hosted-zone \
  --name yourdomain.com \
  --caller-reference $(date +%s) \
  --query 'HostedZone.Id' \
  --output text | cut -d'/' -f3)

echo "Hosted Zone ID: $HOSTED_ZONE_ID"

# Get name servers to configure at your registrar
aws route53 get-hosted-zone \
  --id $HOSTED_ZONE_ID \
  --query 'DelegationSet.NameServers'

# ── Health Checks ───────────────────────────────────────────
# CloudWatch-integrated health check on the API
HEALTH_CHECK_ID=$(aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "api.yourdomain.com",
    "Port": 443,
    "RequestInterval": 30,
    "FailureThreshold": 3,
    "EnableSNI": true,
    "Regions": ["ap-southeast-1", "eu-west-1", "us-east-1"]
  }' \
  --query 'HealthCheck.Id' --output text)

# Tag the health check
aws route53 change-tags-for-resource \
  --resource-type healthcheck \
  --resource-id $HEALTH_CHECK_ID \
  --add-tags Key=Name,Value=url-shortener-api-health

# CloudWatch alarm on health check (alerts when unhealthy)
aws cloudwatch put-metric-alarm \
  --alarm-name url-shortener-health-check \
  --metric-name HealthCheckStatus \
  --namespace AWS/Route53 \
  --dimensions Name=HealthCheckId,Value=$HEALTH_CHECK_ID \
  --statistic Minimum \
  --period 60 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:${ACCOUNT_ID}:url-shortener-alerts
  # NOTE: Route53 health check alarms must be in us-east-1

# ── Weighted Routing (Canary release) ─────────────────────
# 90% to v1 (ap-south-1), 10% to v2 (eu-west-1)
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
          "Weight": 90,
          "AliasTarget": {
            "HostedZoneId": "ZP97RAFLXTNZK",
            "DNSName": "your-alb-ap-south-1.ap-south-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "canary-eu-west-1",
          "Weight": 10,
          "AliasTarget": {
            "HostedZoneId": "Z32O12XQLNTSW2",
            "DNSName": "your-alb-eu-west-1.eu-west-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'

# ── Latency-Based Routing (Multi-region active-active) ─────
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "latency-ap-south-1",
          "Region": "ap-south-1",
          "HealthCheckId": "'$HEALTH_CHECK_ID'",
          "AliasTarget": {
            "HostedZoneId": "ZP97RAFLXTNZK",
            "DNSName": "your-alb-india.ap-south-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "latency-eu-west-1",
          "Region": "eu-west-1",
          "HealthCheckId": "health-check-eu-id",
          "AliasTarget": {
            "HostedZoneId": "Z32O12XQLNTSW2",
            "DNSName": "your-alb-europe.eu-west-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'

# ── Failover Routing ────────────────────────────────────────
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "primary",
          "Failover": "PRIMARY",
          "HealthCheckId": "'$HEALTH_CHECK_ID'",
          "AliasTarget": {
            "HostedZoneId": "ZP97RAFLXTNZK",
            "DNSName": "primary-alb.ap-south-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "secondary",
          "Failover": "SECONDARY",
          "AliasTarget": {
            "HostedZoneId": "Z32O12XQLNTSW2",
            "DNSName": "dr-alb.eu-west-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'

# ── Geolocation Routing ─────────────────────────────────────
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "india-users",
          "GeoLocation": {"CountryCode": "IN"},
          "AliasTarget": {
            "HostedZoneId": "ZP97RAFLXTNZK",
            "DNSName": "india-alb.ap-south-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "europe-users",
          "GeoLocation": {"ContinentCode": "EU"},
          "AliasTarget": {
            "HostedZoneId": "Z32O12XQLNTSW2",
            "DNSName": "europe-alb.eu-west-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.yourdomain.com",
          "Type": "A",
          "SetIdentifier": "global-default",
          "GeoLocation": {"CountryCode": "*"},
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "global-alb.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'

# ── Private Hosted Zone (internal DNS) ─────────────────────
# For internal service discovery — db.internal, redis.internal
PRIVATE_ZONE_ID=$(aws route53 create-hosted-zone \
  --name internal.url-shortener \
  --caller-reference $(date +%s)-private \
  --vpc VPCRegion=$AWS_REGION,VPCId=$VPC_ID \
  --hosted-zone-config Comment="Private zone",PrivateZone=true \
  --query 'HostedZone.Id' --output text | cut -d'/' -f3)

# Add internal CNAME for RDS
aws route53 change-resource-record-sets \
  --hosted-zone-id $PRIVATE_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db.internal.url-shortener",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{
          "Value": "devops-postgres.xxxxxxxxx.ap-south-1.rds.amazonaws.com"
        }]
      }
    }]
  }'

# Now your app uses db.internal.url-shortener — no hardcoded RDS endpoints
# If you change databases, update one DNS record, not every service config

# ── DNS Query Logging ───────────────────────────────────────
aws route53 create-query-logging-config \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --cloud-watch-logs-log-group-arn \
    "arn:aws:logs:us-east-1:${ACCOUNT_ID}:log-group:/route53/yourdomain.com"

# Test DNS resolution from CLI
dig api.yourdomain.com
nslookup api.yourdomain.com
aws route53 test-dns-answer \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --record-name api.yourdomain.com \
  --record-type A
```

## PART 6 — Elastic Load Balancing (ALB + NLB Deep Dive)

You've been using ALBs throughout. Now let's understand them deeply — because every interview asks about this.

**ALB vs NLB vs CLB**

<img width="947" height="438" alt="image" src="https://github.com/user-attachments/assets/ecd0ae89-1602-4913-82ce-61e508feba94" />

**ALB Advanced Features**

```
# ── ALB with Advanced Routing Rules ────────────────────────

ALB_ARN=$(aws elbv2 create-load-balancer \
  --name url-shortener-alb \
  --subnets $PUB_SUBNET_A $PUB_SUBNET_B \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# Enable access logs
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn $ALB_ARN \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=$ARTIFACT_BUCKET \
    Key=access_logs.s3.prefix,Value=alb-logs \
    Key=idle_timeout.timeout_seconds,Value=60 \
    Key=routing.http2.enabled,Value=true \
    Key=routing.http.desync_mitigation_mode,Value=defensive \
    Key=waf.fail_open.enabled,Value=false

# Create multiple target groups for different services
API_TG=$(aws elbv2 create-target-group \
  --name url-shortener-api-tg \
  --protocol HTTP --port 8000 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

ADMIN_TG=$(aws elbv2 create-target-group \
  --name url-shortener-admin-tg \
  --protocol HTTP --port 8001 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path /admin/health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

METRICS_TG=$(aws elbv2 create-target-group \
  --name url-shortener-metrics-tg \
  --protocol HTTP --port 9090 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path /metrics \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# HTTPS Listener with advanced rules
LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=$CERT_ARN \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=$API_TG \
  --query 'Listeners[0].ListenerArn' --output text)

# Rule 1: /admin/* → admin target group (priority 10)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 10 \
  --conditions \
    Field=path-pattern,Values='/admin/*' \
  --actions Type=forward,TargetGroupArn=$ADMIN_TG

# Rule 2: X-Internal-Request: true header → metrics (priority 20)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 20 \
  --conditions '[{
    "Field": "http-header",
    "HttpHeaderConfig": {
      "HttpHeaderName": "X-Internal-Request",
      "Values": ["true"]
    }
  }]' \
  --actions Type=forward,TargetGroupArn=$METRICS_TG

# Rule 3: Block requests from specific IPs (WAF is better, but this works)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 5 \
  --conditions '[{
    "Field": "source-ip",
    "SourceIpConfig": {
      "Values": ["1.2.3.4/32", "5.6.7.8/32"]
    }
  }]' \
  --actions '[{
    "Type": "fixed-response",
    "FixedResponseConfig": {
      "StatusCode": "403",
      "ContentType": "application/json",
      "MessageBody": "{\"error\": \"Forbidden\"}"
    }
  }]'

# Rule 4: Redirect old URL format to new (301 permanent)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 30 \
  --conditions Field=path-pattern,Values='/go/*' \
  --actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Host": "api.yourdomain.com",
      "Path": "/r/#{path}",
      "StatusCode": "HTTP_301"
    }
  }]'

# NLB for internal TCP services (gRPC, database proxies)
NLB_ARN=$(aws elbv2 create-load-balancer \
  --name url-shortener-nlb-internal \
  --subnets $PRIV_SUBNET_A $PRIV_SUBNET_B \
  --scheme internal \
  --type network \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)
```

**Auto Scaling Groups (ASG) Deep Dive**

```
# Create Launch Template (modern replacement for Launch Configuration)
LT_ID=$(aws ec2 create-launch-template \
  --launch-template-name url-shortener-lt \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-0f58b397bc5c1f2e8",
    "InstanceType": "m5.large",
    "KeyName": "your-key-pair",
    "IamInstanceProfile": {
      "Name": "EC2-S3-Profile"
    },
    "SecurityGroupIds": ["'$ECS_SG'"],
    "BlockDeviceMappings": [{
      "DeviceName": "/dev/xvda",
      "Ebs": {
        "VolumeSize": 30,
        "VolumeType": "gp3",
        "Iops": 3000,
        "Throughput": 125,
        "Encrypted": true,
        "DeleteOnTermination": true
      }
    }],
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpEndpoint": "enabled",
      "HttpPutResponseHopLimit": 1
    },
    "UserData": "'$(base64 -w0 userdata.sh)'",
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [
        {"Key": "Name", "Value": "url-shortener-asg"},
        {"Key": "Environment", "Value": "production"}
      ]
    }]
  }' \
  --query 'LaunchTemplate.LaunchTemplateId' \
  --output text)

# Create ASG with mixed instances (On-Demand + Spot)
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name url-shortener-asg \
  --min-size 2 \
  --max-size 20 \
  --desired-capacity 3 \
  --vpc-zone-identifier "${PRIV_SUBNET_A},${PRIV_SUBNET_B}" \
  --target-group-arns $API_TG \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateId": "'$LT_ID'",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5a.large"},
        {"InstanceType": "m4.large"},
        {"InstanceType": "m5d.large"}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 25,
      "SpotAllocationStrategy": "price-capacity-optimized"
    }
  }' \
  --termination-policies OldestLaunchTemplate ClosestToNextInstanceHour \
  --tags \
    Key=Name,Value=url-shortener-asg,PropagateAtLaunch=true \
    Key=Environment,Value=production,PropagateAtLaunch=true

# Predictive Scaling (ML-based — forecasts load 48 hours ahead)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name url-shortener-asg \
  --policy-name predictive-scaling \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [{
      "TargetValue": 60,
      "PredefinedMetricPairSpecification": {
        "PredefinedMetricType": "ASGCPUUtilization"
      }
    }],
    "Mode": "ForecastAndScale",
    "SchedulingBufferTime": 300
  }'
```

## Phase 2 Complete — Full Architecture

```
GitHub (push to main branch)
    │
    ▼
CodePipeline
├── Stage 1: Source (CodeStar → GitHub)
├── Stage 2: Build (CodeBuild → test → scan → push ECR)
├── Stage 3: Deploy Staging (CodeDeploy → ECS Blue/Green)
├── Stage 4: Integration Tests (Newman/Postman via CodeBuild)
├── Stage 5: Manual Approval (SNS → email)
└── Stage 6: Deploy Production (CodeDeploy → ECS Canary 10%)

Route 53 (api.yourdomain.com)
├── Health Check → CloudWatch Alarm → SNS
├── Latency routing: ap-south-1 (India) ↔ eu-west-1 (Europe)
└── Failover: primary → secondary if health check fails

ALB (HTTPS, TLS 1.3, WAF attached)
├── /admin/* → Admin Target Group
├── /r/*     → API Target Group (URL redirect)
└── /*       → API Target Group (ECS Fargate tasks)

ECS Fargate Cluster (url-shortener-prod)
├── Service: url-shortener-api (3→20 tasks, Fargate + Spot)
│   └── Tasks: IRSA → DynamoDB, S3, Secrets Manager
└── Container Insights → CloudWatch → Grafana

CloudFormation Stack
└── Manages: ECS Cluster, Task Def, RDS, DynamoDB,
             S3, IAM Roles, ALB, ASG — everything IaC

RDS PostgreSQL Multi-AZ (private subnet)
DynamoDB (PAY_PER_REQUEST, TTL, PITR, KMS)
ElastiCache Redis (private subnet)
ECR (immutable tags, lifecycle policy, KMS, image scanning)
```

### Phase 2 Complete — Interview Cheat Sheet

<img width="907" height="417" alt="image" src="https://github.com/user-attachments/assets/05409666-2a78-41de-9428-966279646f79" />

<img width="912" height="305" alt="image" src="https://github.com/user-attachments/assets/13372b2b-6063-440d-810c-85884de3cfbc" />

























