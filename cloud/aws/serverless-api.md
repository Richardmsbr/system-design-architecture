# Serverless API Architecture

Event-driven, auto-scaling API architecture using AWS Lambda and API Gateway.

---

## Architecture Overview

```
                                    INTERNET
                                        │
                                        ▼
                              ┌─────────────────┐
                              │    Route 53     │
                              │  api.domain.com │
                              └────────┬────────┘
                                       │
                              ┌────────┴────────┐
                              │   CloudFront    │
                              │  (Edge Caching) │
                              └────────┬────────┘
                                       │
                              ┌────────┴────────┐
                              │   API Gateway   │
                              │   (REST/HTTP)   │
                              │                 │
                              │ - Rate limiting │
                              │ - Auth (Cognito)│
                              │ - Validation    │
                              └────────┬────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│     Lambda      │          │     Lambda      │          │     Lambda      │
│   (Users API)   │          │  (Orders API)   │          │ (Products API)  │
│                 │          │                 │          │                 │
│ Runtime: Node20 │          │ Runtime: Python │          │ Runtime: Node20 │
│ Memory: 256MB   │          │ Memory: 512MB   │          │ Memory: 256MB   │
│ Timeout: 10s    │          │ Timeout: 30s    │          │ Timeout: 10s    │
└────────┬────────┘          └────────┬────────┘          └────────┬────────┘
         │                             │                             │
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│    DynamoDB     │          │    DynamoDB     │          │    DynamoDB     │
│   (Users Table) │          │  (Orders Table) │          │(Products Table) │
└─────────────────┘          └────────┬────────┘          └─────────────────┘
                                      │
                                      │ Stream
                                      ▼
                             ┌─────────────────┐
                             │     Lambda      │
                             │  (Event Proc)   │
                             └────────┬────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
           ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
           │     SQS     │   │     SNS     │   │   Kinesis   │
           │   (Queue)   │   │   (Notify)  │   │ (Analytics) │
           └─────────────┘   └─────────────┘   └─────────────┘
```

---

## Components

### API Gateway Configuration

```yaml
# OpenAPI 3.0 Specification
openapi: "3.0.1"
info:
  title: "Serverless API"
  version: "1.0.0"

paths:
  /users:
    get:
      x-amazon-apigateway-integration:
        type: aws_proxy
        httpMethod: POST
        uri: arn:aws:lambda:region:account:function:users-get
      security:
        - CognitoAuth: []

    post:
      x-amazon-apigateway-integration:
        type: aws_proxy
        httpMethod: POST
        uri: arn:aws:lambda:region:account:function:users-create
      x-amazon-apigateway-request-validator: all

  /users/{id}:
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      x-amazon-apigateway-integration:
        type: aws_proxy
        httpMethod: POST
        uri: arn:aws:lambda:region:account:function:users-get-by-id

components:
  securitySchemes:
    CognitoAuth:
      type: apiKey
      name: Authorization
      in: header
      x-amazon-apigateway-authtype: cognito_user_pools
      x-amazon-apigateway-authorizer:
        type: cognito_user_pools
        providerARNs:
          - arn:aws:cognito-idp:region:account:userpool/pool-id
```

### Lambda Function Structure

```
project/
├── src/
│   ├── handlers/
│   │   ├── users.ts
│   │   ├── orders.ts
│   │   └── products.ts
│   ├── services/
│   │   ├── dynamodb.ts
│   │   └── validation.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── logging.ts
│   │   └── error-handler.ts
│   └── utils/
│       └── response.ts
├── tests/
├── package.json
└── serverless.yml
```

### Lambda Handler Example

```typescript
import { APIGatewayProxyHandler } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler: APIGatewayProxyHandler = async (event) => {
  const userId = event.pathParameters?.id;

  if (!userId) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: 'User ID required' }),
    };
  }

  try {
    const result = await docClient.send(
      new GetCommand({
        TableName: process.env.USERS_TABLE,
        Key: { id: userId },
      })
    );

    if (!result.Item) {
      return {
        statusCode: 404,
        body: JSON.stringify({ error: 'User not found' }),
      };
    }

    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': 'max-age=60',
      },
      body: JSON.stringify(result.Item),
    };
  } catch (error) {
    console.error('Error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal server error' }),
    };
  }
};
```

---

## DynamoDB Design

### Single-Table Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SINGLE TABLE                                  │
├─────────────┬─────────────┬─────────────────────────────────────────────┤
│     PK      │     SK      │              Attributes                     │
├─────────────┼─────────────┼─────────────────────────────────────────────┤
│ USER#123    │ PROFILE     │ name, email, created_at                     │
│ USER#123    │ ORDER#001   │ total, status, items                        │
│ USER#123    │ ORDER#002   │ total, status, items                        │
│ PRODUCT#456 │ DETAILS     │ name, price, inventory                      │
│ ORDER#001   │ ITEM#1      │ product_id, quantity, price                 │
│ ORDER#001   │ ITEM#2      │ product_id, quantity, price                 │
├─────────────┴─────────────┴─────────────────────────────────────────────┤
│                                                                         │
│  GSI1: GSI1PK / GSI1SK                                                 │
│  - ORDER#001 / USER#123  (Query orders by order ID)                    │
│  - PRODUCT#456 / ORDER#001  (Query products in orders)                 │
│                                                                         │
│  GSI2: status / created_at                                             │
│  - PENDING / 2024-01-15  (Query orders by status)                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Access Patterns

| Access Pattern | Key Condition | Index |
|----------------|---------------|-------|
| Get user profile | PK = USER#123, SK = PROFILE | Table |
| Get user orders | PK = USER#123, SK begins_with ORDER# | Table |
| Get order details | GSI1PK = ORDER#001 | GSI1 |
| List pending orders | status = PENDING | GSI2 |

---

## Authentication

### Cognito Configuration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         COGNITO USER POOL                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  User Attributes:                                                       │
│  - email (required, verified)                                          │
│  - name                                                                 │
│  - custom:role                                                         │
│                                                                         │
│  Password Policy:                                                       │
│  - Minimum 12 characters                                               │
│  - Require uppercase, lowercase, numbers, symbols                      │
│                                                                         │
│  MFA:                                                                   │
│  - Optional (TOTP)                                                     │
│                                                                         │
│  App Client:                                                            │
│  - OAuth 2.0 flows: Authorization Code, Implicit                       │
│  - Token validity: Access 1h, Refresh 30d                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### JWT Validation in Lambda

```typescript
import { CognitoJwtVerifier } from 'aws-jwt-verify';

const verifier = CognitoJwtVerifier.create({
  userPoolId: process.env.USER_POOL_ID!,
  tokenUse: 'access',
  clientId: process.env.CLIENT_ID!,
});

export const authenticate = async (token: string) => {
  try {
    const payload = await verifier.verify(token);
    return {
      userId: payload.sub,
      email: payload.email,
      groups: payload['cognito:groups'] || [],
    };
  } catch (error) {
    throw new Error('Invalid token');
  }
};
```

---

## Event Processing

### DynamoDB Streams + Lambda

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  DynamoDB   │────>│   Stream    │────>│   Lambda    │
│   Write     │     │   (24h)     │     │  Processor  │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                         ┌─────────────────────┼─────────────────────┐
                         │                     │                     │
                         ▼                     ▼                     ▼
                ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
                │  Send Email │       │   Update    │       │    Sync     │
                │  (via SES)  │       │   Search    │       │  Analytics  │
                └─────────────┘       │(Elasticsearch)      │  (Kinesis)  │
                                      └─────────────┘       └─────────────┘
```

### Stream Processor

```typescript
import { DynamoDBStreamHandler } from 'aws-lambda';
import { unmarshall } from '@aws-sdk/util-dynamodb';

export const handler: DynamoDBStreamHandler = async (event) => {
  for (const record of event.Records) {
    const eventName = record.eventName;
    const newImage = record.dynamodb?.NewImage;
    const oldImage = record.dynamodb?.OldImage;

    if (eventName === 'INSERT' && newImage) {
      const item = unmarshall(newImage as any);

      if (item.PK.startsWith('ORDER#')) {
        await sendOrderConfirmation(item);
        await updateInventory(item);
        await publishToAnalytics(item);
      }
    }

    if (eventName === 'MODIFY' && newImage && oldImage) {
      const newItem = unmarshall(newImage as any);
      const oldItem = unmarshall(oldImage as any);

      if (newItem.status !== oldItem.status) {
        await sendStatusUpdate(newItem);
      }
    }
  }
};
```

---

## Performance Optimization

### Lambda Best Practices

```
1. Minimize cold starts
   - Provisioned concurrency for critical paths
   - Keep deployment packages small
   - Use Lambda Layers for dependencies

2. Connection reuse
   - Initialize clients outside handler
   - Use keep-alive for HTTP connections

3. Memory optimization
   - Right-size memory (CPU scales with memory)
   - Profile with Lambda Power Tuning

4. Response optimization
   - Enable API Gateway caching
   - Use compression
   - Paginate responses
```

### Provisioned Concurrency

```hcl
resource "aws_lambda_provisioned_concurrency_config" "api" {
  function_name                     = aws_lambda_function.api.function_name
  provisioned_concurrent_executions = 5
  qualifier                         = aws_lambda_alias.live.name
}

# Auto-scale provisioned concurrency
resource "aws_appautoscaling_target" "lambda" {
  max_capacity       = 50
  min_capacity       = 5
  resource_id        = "function:${aws_lambda_function.api.function_name}:${aws_lambda_alias.live.name}"
  scalable_dimension = "lambda:function:ProvisionedConcurrency"
  service_namespace  = "lambda"
}
```

---

## Cost Estimation

### Per-Million Requests

| Component | Free Tier | Cost per 1M |
|-----------|-----------|-------------|
| API Gateway (REST) | 1M/month | $3.50 |
| API Gateway (HTTP) | 1M/month | $1.00 |
| Lambda (256MB, 100ms) | 1M/month | $0.21 |
| DynamoDB (On-demand) | 25 WCU/RCU | $1.25 reads, $6.25 writes |
| CloudWatch Logs | 5GB/month | $0.50/GB |

### Example: 10M Requests/Month

```
API Gateway HTTP:     10M * $1.00/M = $10.00
Lambda:               10M * $0.21/M = $2.10
DynamoDB:             ~$50 (depending on pattern)
CloudWatch:           ~$5
Total:                ~$70/month
```

---

## Monitoring

### Key Metrics

```
API Gateway:
- Count, Latency, 4XXError, 5XXError
- CacheHitCount, CacheMissCount

Lambda:
- Invocations, Duration, Errors, Throttles
- ConcurrentExecutions, IteratorAge (for streams)

DynamoDB:
- ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits
- ThrottledRequests, SystemErrors
```

### X-Ray Tracing

```typescript
import { Tracer } from '@aws-lambda-powertools/tracer';

const tracer = new Tracer({ serviceName: 'users-api' });

export const handler = async (event: any) => {
  const segment = tracer.getSegment();
  const subsegment = segment?.addNewSubsegment('database-query');

  try {
    const result = await queryDatabase();
    subsegment?.close();
    return result;
  } catch (error) {
    subsegment?.addError(error as Error);
    subsegment?.close();
    throw error;
  }
};
```

---

## Security

### API Gateway Throttling

```hcl
resource "aws_api_gateway_usage_plan" "api" {
  name = "standard"

  throttle_settings {
    burst_limit = 100
    rate_limit  = 50
  }

  quota_settings {
    limit  = 10000
    period = "DAY"
  }
}
```

### Lambda VPC Configuration

```
For accessing private resources:
- Place Lambda in VPC subnets
- Use VPC endpoints for AWS services
- NAT Gateway for internet access (adds cost and latency)

Best practice:
- Keep Lambda outside VPC unless needed
- Use VPC endpoints for DynamoDB, S3, Secrets Manager
```

---

## Terraform

See [terraform/aws/serverless/](../../../terraform/aws/serverless/) for complete implementation.
