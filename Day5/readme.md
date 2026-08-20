# ⚡ AWS Phase 4 — Advanced, Serverless & Event-Driven Architecture

## PART 1 — AWS Lambda (Deep Dive)

Lambda is compute without servers. You write a function, AWS runs it in response to events. You pay per 100ms of execution — nothing when idle.

**Lambda Execution Model — What Actually Happens**

```
First invocation (cold start):
  Download code → Init runtime → Init function → Execute handler
  Total: 200ms–2s depending on runtime and package size

Subsequent invocations (warm start):
  Execute handler (execution environment is reused)
  Total: your actual function time

Concurrent invocations:
  Request 1 → Execution Env 1 (warm)
  Request 2 → Execution Env 2 (warm)
  Request 3 → Execution Env 3 (COLD START — new env)
  Request 4 → Execution Env 1 (warm — request 1 finished)

Provisioned Concurrency:
  Pre-warms N execution environments — zero cold starts
  Use for: latency-sensitive APIs, not batch jobs
```

**Lambda Internals Every Senior Engineer Must Know**

```
Execution Environment:
├── /tmp — 512MB–10GB ephemeral storage (persists between warm invocations)
├── /var/task — your code (read-only)
├── /var/runtime — runtime (Python, Node, etc.)
└── Environment variables — plaintext or from Secrets Manager

Memory: 128MB–10GB (CPU scales linearly with memory)
Timeout: max 15 minutes
Payload: 6MB sync, 256KB async (use S3 for larger)
Concurrency: 1000 default per region (soft limit — request increase)

Reserved Concurrency: guaranteed slots for this function (throttles others)
Provisioned Concurrency: pre-initialized envs (eliminates cold starts)
```

**Production Lambda — URL Shortener Functions**

```
# lambda/create_short_url/handler.py
import json
import os
import string
import random
import time
import boto3
from botocore.exceptions import ClientError
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext
from aws_lambda_powertools.utilities.data_classes import APIGatewayProxyEvent
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.validation import validate, SchemaValidationError

# Powertools — AWS's opinionated Lambda toolkit
logger = Logger(service="url-shortener")
tracer = Tracer(service="url-shortener")
metrics = Metrics(namespace="URLShortener", service="create-url")

app = APIGatewayRestResolver()

# Reuse clients across warm invocations (outside handler)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])
sqs = boto3.client('sqs')

# Input schema
CREATE_URL_SCHEMA = {
    "$schema": "http://json-schema.org/draft-07/schema",
    "type": "object",
    "properties": {
        "url": {
            "type": "string",
            "format": "uri",
            "maxLength": 2048
        },
        "custom_code": {
            "type": "string",
            "pattern": "^[a-zA-Z0-9_-]{3,20}$"
        },
        "ttl_days": {
            "type": "integer",
            "minimum": 1,
            "maximum": 365
        }
    },
    "required": ["url"]
}

def generate_short_code(length: int = 7) -> str:
    chars = string.ascii_letters + string.digits
    return ''.join(random.choices(chars, k=length))

@tracer.capture_method
def save_to_dynamodb(short_code: str, original_url: str,
                     owner: str, ttl_days: int) -> dict:
    """Save URL mapping with conditional write (prevent duplicates)."""
    now = int(time.time())
    expires_at = now + (ttl_days * 86400)

    item = {
        'short_code': short_code,
        'created_at': str(now),
        'original_url': original_url,
        'owner': owner,
        'clicks': 0,
        'created_timestamp': now,
        'expires_at': expires_at,
    }

    try:
        # Conditional write — fail if short_code already exists
        table.put_item(
            Item=item,
            ConditionExpression='attribute_not_exists(short_code)'
        )
        return item
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            raise ValueError(f"Short code '{short_code}' already exists")
        raise

@tracer.capture_method
def publish_analytics_event(short_code: str, owner: str, url: str):
    """Async publish to SQS for analytics processing."""
    sqs.send_message(
        QueueUrl=os.environ['ANALYTICS_QUEUE_URL'],
        MessageBody=json.dumps({
            'event': 'url_created',
            'short_code': short_code,
            'owner': owner,
            'url': url,
            'timestamp': int(time.time())
        }),
        MessageGroupId=owner,        # FIFO queue — group per owner
        MessageDeduplicationId=short_code
    )

@app.post("/urls")
@tracer.capture_method
@metrics.log_metrics(capture_cold_start_metric=True)
def create_url():
    """Create a short URL."""
    body = app.current_event.json_body

    try:
        validate(event=body, schema=CREATE_URL_SCHEMA)
    except SchemaValidationError as e:
        return {"statusCode": 400,
                "body": json.dumps({"error": str(e)})}

    original_url = body['url']
    custom_code = body.get('custom_code')
    ttl_days = body.get('ttl_days', 30)

    # Extract owner from JWT authorizer context
    owner = app.current_event.request_context.authorizer.get(
        'claims', {}
    ).get('sub', 'anonymous')

    # Generate or validate short code
    if custom_code:
        short_code = custom_code
    else:
        # Retry up to 3 times for collision avoidance
        for attempt in range(3):
            short_code = generate_short_code()
            try:
                item = save_to_dynamodb(
                    short_code, original_url, owner, ttl_days
                )
                break
            except ValueError:
                if attempt == 2:
                    raise
                continue
    else:
        item = save_to_dynamodb(short_code, original_url, owner, ttl_days)

    # Async analytics (fire-and-forget)
    publish_analytics_event(short_code, owner, original_url)

    # Emit business metric
    metrics.add_metric(name="UrlsCreated", unit=MetricUnit.Count, value=1)

    base_url = os.environ['BASE_URL']
    return {
        "statusCode": 201,
        "body": json.dumps({
            "short_code": short_code,
            "short_url": f"{base_url}/{short_code}",
            "original_url": original_url,
            "expires_at": item['expires_at']
        })
    }

@logger.inject_lambda_context(log_event=True)
@tracer.capture_lambda_handler
def handler(event: dict, context: LambdaContext) -> dict:
    return app.resolve(event, context)
```

```
# lambda/redirect/handler.py — the hot path (must be fast)
import os
import json
import time
import boto3
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit

logger = Logger(service="url-shortener")
tracer = Tracer(service="url-shortener")
metrics = Metrics(namespace="URLShortener", service="redirect")

# Reuse clients — critical for performance
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])
sqs = boto3.client('sqs')

# Lambda cache for hot URLs (simple dict — persists across warm invocations)
# In production use ElastiCache or DAX, but this avoids DDB cost for top URLs
_url_cache = {}
_cache_ttl = {}
CACHE_DURATION = 300  # 5 minutes

@tracer.capture_method
def get_url_cached(short_code: str) -> dict | None:
    now = time.time()

    # Check in-memory cache first
    if short_code in _url_cache:
        if now < _cache_ttl.get(short_code, 0):
            metrics.add_metric(name="CacheHits", unit=MetricUnit.Count, value=1)
            return _url_cache[short_code]

    # Cache miss — query DynamoDB
    metrics.add_metric(name="CacheMisses", unit=MetricUnit.Count, value=1)

    response = table.query(
        KeyConditionExpression='short_code = :sc',
        ExpressionAttributeValues={':sc': short_code},
        Limit=1,
        ScanIndexForward=False
    )

    if not response['Items']:
        return None

    item = response['Items'][0]

    # Check TTL manually (DDB TTL deletion isn't instant)
    if item.get('expires_at', 9999999999) < now:
        return None

    # Store in cache
    _url_cache[short_code] = item
    _cache_ttl[short_code] = now + CACHE_DURATION

    return item

@logger.inject_lambda_context
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event: dict, context) -> dict:
    short_code = event['pathParameters']['code']

    item = get_url_cached(short_code)

    if not item:
        return {
            'statusCode': 404,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Short URL not found'})
        }

    # Async click tracking — never block the redirect
    sqs.send_message(
        QueueUrl=os.environ['CLICKS_QUEUE_URL'],
        MessageBody=json.dumps({
            'event': 'url_clicked',
            'short_code': short_code,
            'timestamp': int(time.time()),
            'user_agent': event['headers'].get('user-agent', ''),
            'referer': event['headers'].get('referer', ''),
            'ip': event['requestContext']['identity']['sourceIp']
        })
    )

    metrics.add_metric(name="Redirects", unit=MetricUnit.Count, value=1)

    return {
        'statusCode': 301,
        'headers': {
            'Location': item['original_url'],
            'Cache-Control': 'public, max-age=300',
            'X-Short-Code': short_code
        },
        'body': ''
    }
```

**Deploy Lambda with Infrastructure as Code**

```
# requirements.txt for Lambda layer
cat > requirements.txt << 'EOF'
aws-lambda-powertools[all]==2.40.0
boto3==1.34.0
EOF

# Build Lambda layer (shared dependencies)
mkdir -p layer/python
pip install -r requirements.txt \
  --target layer/python \
  --platform mlinux_2_x86_64 \
  --implementation cp \
  --python-version 3.12 \
  --only-binary=:all: \
  --upgrade

cd layer
zip -r ../powertools-layer.zip python/
cd ..

# Publish the layer
LAYER_ARN=$(aws lambda publish-layer-version \
  --layer-name url-shortener-powertools \
  --description "AWS Lambda Powertools + dependencies" \
  --zip-file fileb://powertools-layer.zip \
  --compatible-runtimes python3.12 \
  --compatible-architectures x86_64 arm64 \
  --query 'LayerVersionArn' --output text)

echo "Layer ARN: $LAYER_ARN"

# Package the function
cd lambda/create_short_url
zip -r ../../create-short-url.zip handler.py
cd ../..

# Create Lambda execution role
cat > lambda-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name url-shortener-lambda-role \
  --assume-role-policy-document file://lambda-trust.json

cat > lambda-policy.json << EOF
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
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query",
        "dynamodb:DeleteItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:${AWS_REGION}:${ACCOUNT_ID}:table/url-shortener*",
        "arn:aws:dynamodb:${AWS_REGION}:${ACCOUNT_ID}:table/url-shortener*/index/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage"],
      "Resource": "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:url-shortener-*"
    },
    {
      "Effect": "Allow",
      "Action": ["xray:PutTraceSegments","xray:PutTelemetryRecords"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name url-shortener-lambda-role \
  --policy-name LambdaPermissions \
  --policy-document file://lambda-policy.json

# Deploy create-url function
aws lambda create-function \
  --function-name url-shortener-create \
  --runtime python3.12 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/url-shortener-lambda-role \
  --handler handler.handler \
  --zip-file fileb://create-short-url.zip \
  --timeout 30 \
  --memory-size 512 \
  --architectures arm64 \
  --layers $LAYER_ARN \
  --environment Variables='{
    "DYNAMODB_TABLE":"url-shortener",
    "ANALYTICS_QUEUE_URL":"https://sqs.ap-south-1.amazonaws.com/'$ACCOUNT_ID'/url-shortener-analytics.fifo",
    "BASE_URL":"https://api.yourdomain.com",
    "POWERTOOLS_SERVICE_NAME":"url-shortener",
    "POWERTOOLS_LOG_LEVEL":"INFO",
    "AWS_LAMBDA_LOG_FORMAT":"JSON"
  }' \
  --tracing-config Mode=Active \
  --vpc-config SubnetIds=$PRIV_SUBNET_A,$PRIV_SUBNET_B,SecurityGroupIds=$LAMBDA_SG \
  --ephemeral-storage '{"Size": 1024}' \
  --dead-letter-config TargetArn=arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:url-shortener-dlq \
  --tags Project=url-shortener,Environment=production

# Deploy redirect function (hot path — arm64 + more memory = faster)
aws lambda create-function \
  --function-name url-shortener-redirect \
  --runtime python3.12 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/url-shortener-lambda-role \
  --handler handler.handler \
  --zip-file fileb://redirect.zip \
  --timeout 5 \
  --memory-size 1024 \
  --architectures arm64 \
  --layers $LAYER_ARN \
  --environment Variables='{
    "DYNAMODB_TABLE":"url-shortener",
    "CLICKS_QUEUE_URL":"https://sqs.ap-south-1.amazonaws.com/'$ACCOUNT_ID'/url-shortener-clicks"
  }' \
  --tracing-config Mode=Active

# Enable Provisioned Concurrency for redirect (zero cold starts)
aws lambda publish-version \
  --function-name url-shortener-redirect \
  --description "v1 - production"

aws lambda put-provisioned-concurrency-config \
  --function-name url-shortener-redirect \
  --qualifier 1 \
  --provisioned-concurrent-executions 10

# Reserved Concurrency — cap max (protect DynamoDB)
aws lambda put-function-concurrency \
  --function-name url-shortener-create \
  --reserved-concurrent-executions 100

# Function URL (direct HTTPS invoke without API Gateway — for webhooks)
aws lambda create-function-url-config \
  --function-name url-shortener-create \
  --auth-type AWS_IAM \
  --invoke-mode RESPONSE_STREAM   # streaming response

# Lambda aliases (for traffic shifting — canary)
aws lambda create-alias \
  --function-name url-shortener-redirect \
  --name production \
  --function-version 1 \
  --routing-config AdditionalVersionWeights={"2"=0.1}
  # 90% → v1, 10% → v2 (canary)
```

**Lambda Extensions — Advanced Observability**

```
# Lambda extensions run as sidecar processes in the execution environment
# Useful for: flushing logs, collecting metrics, secret caching

# Install Datadog Lambda extension
aws lambda update-function-configuration \
  --function-name url-shortener-redirect \
  --layers \
    $LAYER_ARN \
    "arn:aws:lambda:ap-south-1:464622532012:layer:Datadog-Extension:56"

# AWS Parameters and Secrets Lambda Extension
# (caches SSM + Secrets Manager — avoids API call per invocation)
aws lambda update-function-configuration \
  --function-name url-shortener-create \
  --layers \
    $LAYER_ARN \
    "arn:aws:lambda:ap-south-1:590474943231:layer:AWS-Parameters-and-Secrets-Lambda-Extension:11"

# In your code — access secrets via localhost:
# GET http://localhost:2773/secretsmanager/get?secretId=/url-shortener/prod/api-key
```

## PART 2 — API Gateway (REST + HTTP API)

API Gateway sits in front of Lambda and provides auth, throttling, caching, request transformation, and CORS.

**REST API vs HTTP API**

<img width="942" height="347" alt="image" src="https://github.com/user-attachments/assets/77b8bb0e-809a-44f7-b4cf-24337ca1d87b" />

```
# Create REST API with full features
REST_API_ID=$(aws apigateway create-rest-api \
  --name url-shortener-api \
  --description "URL Shortener REST API" \
  --endpoint-configuration types=REGIONAL \
  --minimum-compression-size 1024 \
  --query 'id' --output text)

# Get root resource ID
ROOT_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $REST_API_ID \
  --query 'items[?path==`/`].id' --output text)

# Create /urls resource
URLS_RESOURCE=$(aws apigateway create-resource \
  --rest-api-id $REST_API_ID \
  --parent-id $ROOT_RESOURCE_ID \
  --path-part urls \
  --query 'id' --output text)

# Create POST /urls method with request validation
aws apigateway put-method \
  --rest-api-id $REST_API_ID \
  --resource-id $URLS_RESOURCE \
  --http-method POST \
  --authorization-type COGNITO_USER_POOLS \
  --authorizer-id $COGNITO_AUTHORIZER_ID \
  --request-validator-id $VALIDATOR_ID \
  --request-models '{"application/json": "CreateURLModel"}'

# Lambda integration
aws apigateway put-integration \
  --rest-api-id $REST_API_ID \
  --resource-id $URLS_RESOURCE \
  --http-method POST \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:${AWS_REGION}:lambda:path/2015-03-31/functions/arn:aws:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:url-shortener-create:production/invocations"

# Create /{code} resource for redirects
CODE_RESOURCE=$(aws apigateway create-resource \
  --rest-api-id $REST_API_ID \
  --parent-id $ROOT_RESOURCE_ID \
  --path-part '{code}' \
  --query 'id' --output text)

aws apigateway put-method \
  --rest-api-id $REST_API_ID \
  --resource-id $CODE_RESOURCE \
  --http-method GET \
  --authorization-type NONE   # public — anyone can redirect

aws apigateway put-integration \
  --rest-api-id $REST_API_ID \
  --resource-id $CODE_RESOURCE \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:${AWS_REGION}:lambda:path/2015-03-31/functions/arn:aws:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:url-shortener-redirect:production/invocations"

# Enable caching (per route, keyed by path param)
aws apigateway create-deployment \
  --rest-api-id $REST_API_ID \
  --stage-name prod \
  --stage-description "Production stage"

aws apigateway update-stage \
  --rest-api-id $REST_API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/cacheClusterEnabled,value=true \
    op=replace,path=/cacheClusterSize,value=0.5 \
    op=replace,path=/*/*/caching/enabled,value=false \
    op=replace,path=/~1{code}/GET/caching/enabled,value=true \
    op=replace,path=/~1{code}/GET/caching/ttlInSeconds,value=300 \
    op=replace,path=/~1{code}/GET/caching/cacheKeyParameters/0/name,value=method.request.path.code \
    op=replace,path=/*/*/throttling/burstLimit,value=5000 \
    op=replace,path=/*/*/throttling/rateLimit,value=2000 \
    op=replace,path=/loggingLevel,value=INFO \
    op=replace,path=/dataTraceEnabled,value=false \
    op=replace,path=/tracingEnabled,value=true

# Usage plans (API keys for external consumers)
USAGE_PLAN=$(aws apigateway create-usage-plan \
  --name url-shortener-standard \
  --description "Standard tier - 10k req/day" \
  --throttle burstLimit=100,rateLimit=50 \
  --quota limit=10000,offset=0,period=DAY \
  --api-stages apiId=$REST_API_ID,stage=prod \
  --query 'id' --output text)

API_KEY=$(aws apigateway create-api-key \
  --name chetan-api-key \
  --enabled \
  --query 'id' --output text)

aws apigateway create-usage-plan-key \
  --usage-plan-id $USAGE_PLAN \
  --key-id $API_KEY \
  --key-type API_KEY

# Grant API Gateway permission to invoke Lambda
aws lambda add-permission \
  --function-name url-shortener-create \
  --qualifier production \
  --statement-id apigateway-prod-post \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${AWS_REGION}:${ACCOUNT_ID}:${REST_API_ID}/prod/POST/urls"
```

## PART 3 — SQS (Simple Queue Service)

SQS decouples producers from consumers. If your consumer dies, messages wait. If traffic spikes, messages buffer.

**SQS Queue Types**

```
STANDARD QUEUE:
  ├── At-least-once delivery (may deliver duplicate messages)
  ├── Best-effort ordering (not guaranteed)
  ├── Virtually unlimited throughput
  └── Use for: async tasks where duplicates are tolerable

FIFO QUEUE (.fifo suffix):
  ├── Exactly-once processing (deduplication)
  ├── Strict ordering within a MessageGroupId
  ├── 3000 msg/sec with batching, 300 without
  └── Use for: financial transactions, ordered workflows
```

**Hands-On: SQS Full Setup**

```
# Create Standard Queue for click tracking
CLICKS_QUEUE=$(aws sqs create-queue \
  --queue-name url-shortener-clicks \
  --attributes '{
    "VisibilityTimeout": "60",
    "MessageRetentionPeriod": "86400",
    "ReceiveMessageWaitTimeSeconds": "20",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:'$AWS_REGION':'$ACCOUNT_ID':url-shortener-clicks-dlq\",\"maxReceiveCount\":\"3\"}",
    "KmsMasterKeyId": "alias/url-shortener-prod",
    "SqsManagedSseEnabled": "false"
  }' \
  --query 'QueueUrl' --output text)

# Dead Letter Queue (DLQ) — receives messages that failed 3+ times
DLQ=$(aws sqs create-queue \
  --queue-name url-shortener-clicks-dlq \
  --attributes '{
    "MessageRetentionPeriod": "1209600"
  }' \
  --query 'QueueUrl' --output text)

DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url $DLQ \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Update clicks queue with DLQ reference
aws sqs set-queue-attributes \
  --queue-url $CLICKS_QUEUE \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"'$DLQ_ARN'\",\"maxReceiveCount\":\"3\"}"
  }'

# FIFO Queue for analytics (ordered, deduplicated)
ANALYTICS_QUEUE=$(aws sqs create-queue \
  --queue-name url-shortener-analytics.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "false",
    "VisibilityTimeout": "300",
    "MessageRetentionPeriod": "86400",
    "ReceiveMessageWaitTimeSeconds": "20",
    "DeduplicationScope": "messageGroup",
    "FifoThroughputLimit": "perMessageGroupId"
  }' \
  --query 'QueueUrl' --output text)

# Queue Policy — allow specific services/accounts to send
aws sqs set-queue-attributes \
  --queue-url $CLICKS_QUEUE \
  --attributes '{
    "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"lambda.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"*\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"arn:aws:lambda:'$AWS_REGION':'$ACCOUNT_ID':function:url-shortener-redirect\"}}},{\"Effect\":\"Deny\",\"Principal\":\"*\",\"Action\":\"sqs:*\",\"Resource\":\"*\",\"Condition\":{\"Bool\":{\"aws:SecureTransport\":\"false\"}}}]}"
  }'

# Lambda consumes from SQS (Event Source Mapping)
aws lambda create-event-source-mapping \
  --function-name url-shortener-click-processor \
  --event-source-arn $(aws sqs get-queue-attributes \
    --queue-url $CLICKS_QUEUE \
    --attribute-names QueueArn \
    --query 'Attributes.QueueArn' --output text) \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 5 \
  --function-response-types ReportBatchItemFailures \
  --scaling-config MaximumConcurrency=10 \
  --filter-criteria '{
    "Filters": [{
      "Pattern": "{\"body\": {\"event\": [\"url_clicked\"]}}"
    }]
  }'
```

**Click Processor Lambda — SQS Consumer Pattern**

```
# lambda/click_processor/handler.py
import json
import boto3
import os
from collections import defaultdict
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.utilities.batch import (
    BatchProcessor, EventType, process_partial_response
)
from aws_lambda_powertools.utilities.data_classes.sqs_event import SQSRecord

logger = Logger(service="url-shortener")
tracer = Tracer(service="url-shortener")
metrics = Metrics(namespace="URLShortener", service="click-processor")

processor = BatchProcessor(event_type=EventType.SQS)

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])
kinesis = boto3.client('kinesis')

@tracer.capture_method
def record_handler(record: SQSRecord):
    """Process a single click event from SQS."""
    payload = json.loads(record.body)

    short_code = payload['short_code']
    timestamp = payload['timestamp']

    # Atomic increment of click counter
    table.update_item(
        Key={
            'short_code': short_code,
            'created_at': get_created_at(short_code)
        },
        UpdateExpression='ADD clicks :inc SET last_clicked = :ts',
        ExpressionAttributeValues={
            ':inc': 1,
            ':ts': timestamp
        }
    )

    # Forward to Kinesis for real-time analytics
    kinesis.put_record(
        StreamName=os.environ['KINESIS_STREAM'],
        Data=json.dumps(payload),
        PartitionKey=short_code
    )

    logger.info("Click recorded", short_code=short_code)

def get_created_at(short_code: str) -> str:
    """Get sort key from DDB — cached in practice."""
    response = table.query(
        KeyConditionExpression='short_code = :sc',
        ExpressionAttributeValues={':sc': short_code},
        Limit=1,
        ProjectionExpression='created_at'
    )
    return response['Items'][0]['created_at'] if response['Items'] else ''

@logger.inject_lambda_context
@tracer.capture_lambda_handler
@metrics.log_metrics
def handler(event, context):
    # process_partial_response handles partial batch failures
    # Failed records go back to queue (up to maxReceiveCount times)
    # Successful records are deleted automatically
    return process_partial_response(
        event=event,
        record_handler=record_handler,
        processor=processor,
        context=context
    )
```

## PART 4 — SNS (Simple Notification Service)

SNS is a pub/sub messaging service. One message published → multiple subscribers receive it.

```
# SNS Topic with encryption
EVENTS_TOPIC=$(aws sns create-topic \
  --name url-shortener-events \
  --attributes '{
    "KmsMasterKeyId": "alias/url-shortener-prod",
    "FifoTopic": "false"
  }' \
  --tags Key=Project,Value=url-shortener \
  --query 'TopicArn' --output text)

# Subscribe SQS to SNS (fanout pattern)
# One SNS message → multiple SQS queues → multiple consumers
aws sns subscribe \
  --topic-arn $EVENTS_TOPIC \
  --protocol sqs \
  --notification-endpoint $(aws sqs get-queue-attributes \
    --queue-url $CLICKS_QUEUE \
    --attribute-names QueueArn \
    --query 'Attributes.QueueArn' --output text) \
  --attributes '{
    "FilterPolicy": "{\"event_type\": [\"url_clicked\"]}",
    "FilterPolicyScope": "MessageBody",
    "RawMessageDelivery": "true"
  }'

# Subscribe Lambda directly to SNS
aws sns subscribe \
  --topic-arn $EVENTS_TOPIC \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:url-shortener-notifier

# Subscribe email for ops alerts
aws sns subscribe \
  --topic-arn $EVENTS_TOPIC \
  --protocol email \
  --notification-endpoint chetan@company.com \
  --attributes '{"FilterPolicy": "{\"event_type\": [\"abuse_detected\"]}"}'

# Allow SQS queue to receive from SNS (queue policy update)
aws sqs set-queue-attributes \
  --queue-url $CLICKS_QUEUE \
  --attributes '{
    "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"*\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"'$EVENTS_TOPIC'\"}}}]}"
  }'

# SNS + SQS Fanout pattern (message resilience)
# All 3 queues receive EVERY message:
#
#              SNS Topic
#             /    |    \
#           SQS   SQS   Lambda
#         (clicks)(analytics)(notifier)
#
# If Lambda crashes → SNS retries
# If SQS consumer crashes → message stays in queue
```

## PART 5 — EventBridge (Event-Driven Architecture)

EventBridge is the backbone of event-driven AWS architectures. It routes events from AWS services, custom apps, and SaaS to targets.

```
# Create custom Event Bus for url-shortener domain
EVENT_BUS_ARN=$(aws events create-event-bus \
  --name url-shortener-bus \
  --query 'EventBusArn' --output text)

# Schema registry — auto-discover event schemas
aws schemas create-registry \
  --registry-name url-shortener-schemas \
  --description "URL Shortener domain event schemas"

aws schemas create-discoverer \
  --source-arn $EVENT_BUS_ARN \
  --description "Auto-discover schemas from url-shortener-bus"

# Publish events from Lambda to EventBridge
# (In your Lambda code:)
cat > event_publisher.py << 'EOF'
import boto3
import json
from datetime import datetime, timezone

events = boto3.client('events')

def publish_event(event_type: str, detail: dict,
                  source: str = "com.urlshortener"):
    """Publish a domain event to EventBridge."""
    events.put_events(
        Entries=[{
            'Source': source,
            'DetailType': event_type,
            'Detail': json.dumps({
                **detail,
                'timestamp': datetime.now(timezone.utc).isoformat(),
                'version': '1.0'
            }),
            'EventBusName': 'url-shortener-bus',
        }]
    )

# Usage:
# publish_event('URLCreated', {'short_code': 'abc123', 'owner': 'chetan'})
# publish_event('URLClicked', {'short_code': 'abc123', 'ip': '1.2.3.4'})
# publish_event('AbuseDetected', {'short_code': 'xyz', 'reason': 'phishing'})
EOF

# ── EventBridge Rules — route events to targets ────────────

# Rule 1: URL abuse detected → SNS alert + disable URL
aws events put-rule \
  --name url-abuse-detected \
  --event-bus-name url-shortener-bus \
  --event-pattern '{
    "source": ["com.urlshortener"],
    "detail-type": ["AbuseDetected"],
    "detail": {
      "severity": ["HIGH", "CRITICAL"]
    }
  }' \
  --state ENABLED \
  --description "Alert and disable abusive URLs"

aws events put-targets \
  --rule url-abuse-detected \
  --event-bus-name url-shortener-bus \
  --targets \
    Id=sns-alert,Arn=$EVENTS_TOPIC \
    Id=disable-url-lambda,Arn=arn:aws:lambda:${AWS_REGION}:${ACCOUNT_ID}:function:url-shortener-disable

# Rule 2: Daily stats aggregation (scheduled)
aws events put-rule \
  --name daily-stats-aggregation \
  --event-bus-name url-shortener-bus \
  --schedule-expression "cron(0 1 * * ? *)" \
  --state ENABLED \
  --description "Aggregate daily click stats at 1 AM UTC"

aws events put-targets \
  --rule daily-stats-aggregation \
  --event-bus-name url-shortener-bus \
  --targets '[{
    "Id": "stats-aggregator",
    "Arn": "arn:aws:lambda:'$AWS_REGION':'$ACCOUNT_ID':function:url-shortener-stats-aggregator",
    "Input": "{\"task\": \"daily_aggregation\", \"retention_days\": 90}"
  }]'

# Rule 3: Forward all events to CloudWatch for debugging
aws events put-rule \
  --name all-events-to-cloudwatch \
  --event-bus-name url-shortener-bus \
  --event-pattern '{"source": ["com.urlshortener"]}' \
  --state ENABLED

aws events put-targets \
  --rule all-events-to-cloudwatch \
  --event-bus-name url-shortener-bus \
  --targets '[{
    "Id": "cloudwatch-logs",
    "Arn": "arn:aws:logs:'$AWS_REGION':'$ACCOUNT_ID':log-group:/events/url-shortener"
  }]'

# Rule 4: Cross-account event sharing
# (receive events from other accounts in your org)
aws events put-permission \
  --event-bus-name url-shortener-bus \
  --action events:PutEvents \
  --principal '*' \
  --statement-id org-wide \
  --condition '{"Type": "StringEquals", "Key": "aws:PrincipalOrgID", "Value": "o-xxxxxxxxxxxx"}'

# Archive all events (replay capability)
aws events create-archive \
  --archive-name url-shortener-archive \
  --event-source-arn $EVENT_BUS_ARN \
  --description "Archive all domain events for 90 days" \
  --retention-days 90

# Replay archived events (e.g. after a bug fix)
aws events start-replay \
  --replay-name replay-2024-01-15-clicks \
  --event-source-arn "arn:aws:events:${AWS_REGION}:${ACCOUNT_ID}:archive/url-shortener-archive" \
  --event-start-time "2024-01-15T00:00:00Z" \
  --event-end-time "2024-01-15T23:59:59Z" \
  --destination '{
    "Arn": "'$EVENT_BUS_ARN'",
    "FilterArns": []
  }'
```

## PART 6 — Step Functions (Workflow Orchestration)

Step Functions orchestrates multi-step workflows with retries, error handling, parallel execution, and wait states. Think of it as a state machine as a service.

**When to Use Step Functions vs Lambda Chaining**

```
Lambda chaining (BAD for long workflows):
  Lambda A → calls Lambda B → calls Lambda C
  Problems:
  ├── Timeout chain (max 15min each, but you're blocking)
  ├── No retry visibility
  ├── One failure = you don't know where it stopped
  └── Hard to add steps later

Step Functions (RIGHT approach):
  State Machine defines flow visually and in JSON
  ├── Each step has configurable retries + catch
  ├── Audit trail of every execution step
  ├── Wait states for human approval or external events
  ├── Parallel execution branches
  └── Max duration: 1 year (Standard) or 5 min (Express)
```

**URL Shortener — Abuse Detection Workflow**

```
{
  "Comment": "URL abuse detection and remediation workflow",
  "StartAt": "AnalyzeURL",
  "States": {
    "AnalyzeURL": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:url-shortener-analyze",
      "Parameters": {
        "short_code.$": "$.short_code",
        "original_url.$": "$.original_url"
      },
      "ResultPath": "$.analysis",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "AnalysisFailed",
          "ResultPath": "$.error"
        }
      ],
      "Next": "EvaluateRisk"
    },

    "EvaluateRisk": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.analysis.risk_score",
          "NumericGreaterThanOrEqualTo": 90,
          "Next": "HighRiskPath"
        },
        {
          "Variable": "$.analysis.risk_score",
          "NumericGreaterThanOrEqualTo": 60,
          "Next": "MediumRiskPath"
        }
      ],
      "Default": "URLApproved"
    },

    "HighRiskPath": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "DisableURL",
          "States": {
            "DisableURL": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:url-shortener-disable",
              "Parameters": {
                "short_code.$": "$.short_code",
                "reason": "High risk score — auto-disabled"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "NotifySecurityTeam",
          "States": {
            "NotifySecurityTeam": {
              "Type": "Task",
              "Resource": "arn:aws:states:::sns:publish",
              "Parameters": {
                "TopicArn": "arn:aws:sns:ap-south-1:ACCOUNT_ID:security-alerts",
                "Message.$": "States.Format('HIGH RISK URL detected: {} Score: {}', $.short_code, $.analysis.risk_score)",
                "Subject": "🚨 URL Abuse Alert"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "BanOwner",
          "States": {
            "BanOwner": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:url-shortener-ban-owner",
              "Parameters": {
                "owner.$": "$.owner"
              },
              "End": true
            }
          }
        }
      ],
      "Next": "HighRiskComplete"
    },

    "MediumRiskPath": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "https://sqs.ap-south-1.amazonaws.com/ACCOUNT_ID/url-shortener-review",
        "MessageBody": {
          "short_code.$": "$.short_code",
          "risk_score.$": "$.analysis.risk_score",
          "task_token.$": "$$.Task.Token",
          "review_url.$": "States.Format('https://admin.yourdomain.com/review/{}', $.short_code)"
        }
      },
      "TimeoutSeconds": 86400,
      "HeartbeatSeconds": 3600,
      "Catch": [
        {
          "ErrorEquals": ["States.HeartbeatTimeout", "States.Timeout"],
          "Next": "ReviewTimeout"
        }
      ],
      "Next": "EvaluateReviewDecision"
    },

    "EvaluateReviewDecision": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.review_decision",
          "StringEquals": "APPROVE",
          "Next": "URLApproved"
        },
        {
          "Variable": "$.review_decision",
          "StringEquals": "REJECT",
          "Next": "DisableURL"
        }
      ],
      "Default": "URLApproved"
    },

    "URLApproved": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:updateItem",
      "Parameters": {
        "TableName": "url-shortener",
        "Key": {
          "short_code": {"S.$": "$.short_code"}
        },
        "UpdateExpression": "SET #s = :approved",
        "ExpressionAttributeNames": {"#s": "status"},
        "ExpressionAttributeValues": {":approved": {"S": "active"}}
      },
      "End": true
    },

    "ReviewTimeout": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:ap-south-1:ACCOUNT_ID:security-alerts",
        "Message.$": "States.Format('Review timed out for {}. URL disabled by default.', $.short_code)"
      },
      "Next": "DisableURL"
    },

    "DisableURL": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:url-shortener-disable",
      "Parameters": {
        "short_code.$": "$.short_code"
      },
      "End": true
    },

    "HighRiskComplete": {
      "Type": "Succeed"
    },

    "AnalysisFailed": {
      "Type": "Fail",
      "Error": "AnalysisError",
      "Cause": "URL analysis Lambda failed after retries"
    }
  }
}
```

```
# Deploy the state machine
aws stepfunctions create-state-machine \
  --name url-shortener-abuse-detection \
  --definition file://abuse-detection-sfn.json \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/stepfunctions-role \
  --type STANDARD \
  --logging-configuration '{
    "level": "ALL",
    "includeExecutionData": true,
    "destinations": [{
      "cloudWatchLogsLogGroup": {
        "logGroupArn": "arn:aws:logs:'$AWS_REGION':'$ACCOUNT_ID':log-group:/stepfunctions/url-shortener"
      }
    }]
  }' \
  --tracing-configuration enabled=true

# Start an execution
EXECUTION_ARN=$(aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:${AWS_REGION}:${ACCOUNT_ID}:stateMachine:url-shortener-abuse-detection \
  --name "check-abc123-$(date +%s)" \
  --input '{
    "short_code": "abc123",
    "original_url": "https://suspicious-site.com",
    "owner": "user-xyz"
  }' \
  --query 'executionArn' --output text)

# Watch execution
aws stepfunctions describe-execution \
  --execution-arn $EXECUTION_ARN \
  --query '{Status:status,Start:startDate,Stop:stopDate}'

# Get full execution history (every state transition)
aws stepfunctions get-execution-history \
  --execution-arn $EXECUTION_ARN \
  --query 'events[].[timestamp,type,stateEnteredEventDetails.name]' \
  --output table

# Send task success for WaitForTaskToken (manual review approved)
aws stepfunctions send-task-success \
  --task-token "AAAAKgAAAAIAAAAAAAAAAXxxxxxxxxxx" \
  --task-output '{"review_decision": "APPROVE"}'

# List failed executions
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:${AWS_REGION}:${ACCOUNT_ID}:stateMachine:url-shortener-abuse-detection \
  --status-filter FAILED \
  --query 'executions[].[name,status,startDate]' \
  --output table
```

## PART 7 — DynamoDB Advanced (Production Patterns)

You've used DynamoDB. Now go deep — the patterns that matter at scale.

**Single-Table Design**

In DynamoDB, you should model ALL your access patterns in ONE table. Multiple tables is the SQL mindset. Single table is the DynamoDB mindset.

```
URL Shortener — Single Table Design

PK (partition key)   SK (sort key)         Attributes
────────────────────────────────────────────────────────────────────
URL#abc123          METADATA              original_url, owner, clicks
URL#abc123          CLICK#2024-01-15T10   ip, user_agent, referer
URL#abc123          CLICK#2024-01-15T11   ip, user_agent, referer
USER#chetan         PROFILE               email, plan, created_at
USER#chetan         URL#abc123            created_at (ownership index)
USER#chetan         URL#xyz789            created_at
PLAN#standard       LIMIT                 max_urls=100, max_clicks=10000
ABUSE#abc123        FLAG#2024-01-15       reason, reporter

Access patterns covered:
├── Get URL metadata → Query PK=URL#abc123, SK=METADATA
├── Get all clicks for URL → Query PK=URL#abc123, SK begins_with CLICK#
├── Get all URLs for user → Query PK=USER#chetan, SK begins_with URL#
├── Get user profile → GetItem PK=USER#chetan, SK=PROFILE
└── Get abuse flags → Query PK=ABUSE#abc123
```

```
# Create production DynamoDB table with single-table design
aws dynamodb create-table \
  --table-name url-shortener-v2 \
  --billing-mode PAY_PER_REQUEST \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=GSI1PK,AttributeType=S \
    AttributeName=GSI1SK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[
    {
      "IndexName": "GSI1",
      "KeySchema": [
        {"AttributeName": "GSI1PK", "KeyType": "HASH"},
        {"AttributeName": "GSI1SK", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ]' \
  --sse-specification Enabled=true,SSEType=KMS \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true \
  --tags Key=Project,Value=url-shortener

# Enable TTL
aws dynamodb update-time-to-live \
  --table-name url-shortener-v2 \
  --time-to-live-specification Enabled=true,AttributeName=expires_at

# Write items using single-table pattern
aws dynamodb transact-write-items \
  --transact-items '[
    {
      "Put": {
        "TableName": "url-shortener-v2",
        "Item": {
          "PK": {"S": "URL#abc123"},
          "SK": {"S": "METADATA"},
          "GSI1PK": {"S": "USER#chetan"},
          "GSI1SK": {"S": "URL#abc123"},
          "original_url": {"S": "https://google.com"},
          "owner": {"S": "chetan"},
          "clicks": {"N": "0"},
          "created_at": {"S": "2024-01-15T10:00:00Z"},
          "expires_at": {"N": "1736899200"}
        },
        "ConditionExpression": "attribute_not_exists(PK)"
      }
    },
    {
      "Update": {
        "TableName": "url-shortener-v2",
        "Key": {
          "PK": {"S": "USER#chetan"},
          "SK": {"S": "PROFILE"}
        },
        "UpdateExpression": "ADD url_count :inc",
        "ExpressionAttributeValues": {":inc": {"N": "1"}}
      }
    }
  ]'

# DynamoDB Streams → Lambda for real-time processing
aws lambda create-event-source-mapping \
  --function-name url-shortener-stream-processor \
  --event-source-arn $(aws dynamodb describe-table \
    --table-name url-shortener-v2 \
    --query 'Table.LatestStreamArn' --output text) \
  --starting-position TRIM_HORIZON \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 5 \
  --bisect-batch-on-function-error \
  --destination-config '{
    "OnFailure": {
      "Destination": "arn:aws:sqs:'$AWS_REGION':'$ACCOUNT_ID':url-shortener-stream-dlq"
    }
  }' \
  --filter-criteria '{
    "Filters": [{
      "Pattern": "{\"eventName\": [\"INSERT\", \"MODIFY\"]}"
    }]
  }'

# DAX — DynamoDB Accelerator (microsecond reads for hot data)
DAX_CLUSTER=$(aws dax create-cluster \
  --cluster-name url-shortener-dax \
  --node-type dax.r5.large \
  --replication-factor 3 \
  --iam-role-arn arn:aws:iam::${ACCOUNT_ID}:role/dax-role \
  --subnet-group-name url-shortener-dax-subnets \
  --security-group-ids $DAX_SG \
  --sse-specification Enabled=true \
  --query 'Cluster.ClusterArn' --output text)

# DAX endpoint (drop-in replacement for DynamoDB client in SDK)
# In Python:
# import amazondax
# dax = amazondax.AmazonDaxClient(endpoints=['dax-cluster.xxx.dax-clusters.us-east-1.amazonaws.com:8111'])
# Same API as boto3 dynamodb — just reads from cache automatically
```

## PART 8 — Kinesis (Real-Time Streaming)

Kinesis is for real-time data streams. Unlike SQS (pull, delete-after-consume), Kinesis retains data and multiple consumers can read the same stream independently.

```
# Create Kinesis Data Stream
STREAM_ARN=$(aws kinesis create-stream \
  --stream-name url-shortener-clickstream \
  --stream-mode-details StreamMode=ON_DEMAND \
  --query 'StreamARN' --output text 2>/dev/null || \
  aws kinesis describe-stream-summary \
    --stream-name url-shortener-clickstream \
    --query 'StreamDescriptionSummary.StreamARN' --output text)

# Kinesis Data Firehose — stream to S3 with transformation
aws firehose create-delivery-stream \
  --delivery-stream-name url-shortener-clicks-to-s3 \
  --delivery-stream-type KinesisStreamAsSource \
  --kinesis-stream-source-configuration \
    KinesisStreamARN=$STREAM_ARN,\
    RoleARN=arn:aws:iam::${ACCOUNT_ID}:role/firehose-role \
  --extended-s3-destination-configuration '{
    "RoleARN": "arn:aws:iam::'$ACCOUNT_ID':role/firehose-role",
    "BucketARN": "arn:aws:s3:::'$ARTIFACT_BUCKET'",
    "Prefix": "clickstream/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    "ErrorOutputPrefix": "clickstream-errors/",
    "BufferingHints": {
      "SizeInMBs": 128,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "SNAPPY",
    "DataFormatConversionConfiguration": {
      "Enabled": true,
      "OutputFormatConfiguration": {
        "Serializer": {"ParquetSerDe": {}}
      },
      "SchemaConfiguration": {
        "DatabaseName": "url_shortener",
        "TableName": "clicks",
        "RoleARN": "arn:aws:iam::'$ACCOUNT_ID':role/firehose-role"
      }
    },
    "ProcessingConfiguration": {
      "Enabled": true,
      "Processors": [{
        "Type": "Lambda",
        "Parameters": [{
          "ParameterName": "LambdaArn",
          "ParameterValue": "arn:aws:lambda:'$AWS_REGION':'$ACCOUNT_ID':function:url-shortener-stream-enricher"
        }]
      }]
    },
    "S3BackupMode": "FailedDataOnly"
  }'

# Lambda consuming from Kinesis (enhanced fan-out)
aws kinesis register-stream-consumer \
  --stream-arn $STREAM_ARN \
  --consumer-name url-shortener-realtime-consumer

aws lambda create-event-source-mapping \
  --function-name url-shortener-realtime-processor \
  --event-source-arn $STREAM_ARN \
  --starting-position LATEST \
  --batch-size 1000 \
  --maximum-batching-window-in-seconds 5 \
  --parallelization-factor 10 \
  --bisect-batch-on-function-error \
  --function-response-types ReportBatchItemFailures
```

## PART 9 — Complete Serverless Architecture (Production)

```
User Request
    │
    ▼
CloudFront (cache static assets + redirect responses)
    │
    ▼
API Gateway (REST API, usage plans, request validation)
    ├── POST /urls → Lambda:create (512MB, arm64, 30s timeout, DLQ)
    │     └── DynamoDB transact-write → SQS Analytics.fifo
    │                                   EventBridge: URLCreated
    │                                   Step Functions: AbuseDetection
    │
    └── GET /{code} → Lambda:redirect (1024MB, arm64, Provisioned=10)
          ├── DAX (microsecond cache)
          │     └── DynamoDB fallback
          └── SQS:clicks (async, fire-and-forget)

SQS:clicks → Lambda:click-processor (batch=100, window=5s)
    └── DynamoDB: atomic increment
    └── Kinesis: clickstream

Kinesis → Lambda:realtime-processor (parallelization=10)
    └── Real-time dashboards

Kinesis → Firehose → S3 (parquet) → Athena → QuickSight

EventBridge (url-shortener-bus)
    ├── URLCreated → Lambda:notifier (welcome email)
    ├── AbuseDetected → SNS → security team
    └── Scheduled: 01:00 UTC → Lambda:stats-aggregator

Step Functions: AbuseDetection workflow
    ├── Parallel: disable + notify + ban
    └── WaitForTaskToken: human review with 24h timeout

DynamoDB Streams → Lambda:stream-processor
    └── Sync to ElasticSearch for full-text search
```

## Phase 4 Interview Cheat Sheet

<img width="915" height="452" alt="image" src="https://github.com/user-attachments/assets/007be617-8e12-4aed-875c-53d64d434846" />

<img width="930" height="512" alt="image" src="https://github.com/user-attachments/assets/76b3e46a-06fe-4f35-9ae3-67dbdfbcf885" />



















