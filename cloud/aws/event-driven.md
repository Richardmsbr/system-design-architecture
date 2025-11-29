# Event-Driven Microservices Architecture

Loosely coupled, scalable microservices using event-driven patterns on AWS.

---

## Architecture Overview

```
                                    CLIENTS
                                        │
                                        ▼
                              ┌─────────────────┐
                              │   API Gateway   │
                              └────────┬────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│  Order Service  │          │  User Service   │          │Payment Service  │
│                 │          │                 │          │                 │
│   ┌─────────┐   │          │   ┌─────────┐   │          │   ┌─────────┐   │
│   │   ECS   │   │          │   │   ECS   │   │          │   │   ECS   │   │
│   └────┬────┘   │          │   └────┬────┘   │          │   └────┬────┘   │
│        │        │          │        │        │          │        │        │
│   ┌────┴────┐   │          │   ┌────┴────┐   │          │   ┌────┴────┐   │
│   │   RDS   │   │          │   │ DynamoDB│   │          │   │   RDS   │   │
│   └─────────┘   │          │   └─────────┘   │          │   └─────────┘   │
└────────┬────────┘          └────────┬────────┘          └────────┬────────┘
         │                             │                             │
         │      Publish Events         │                             │
         └─────────────────────────────┼─────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AMAZON EVENTBRIDGE                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                           EVENT BUS                                  │   │
│  │                                                                      │   │
│  │  Events:                                                             │   │
│  │  - order.created                                                     │   │
│  │  - order.paid                                                        │   │
│  │  - user.registered                                                   │   │
│  │  - payment.processed                                                 │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                       │                                     │
│                    ┌──────────────────┼──────────────────┐                 │
│                    │                  │                  │                 │
│                    ▼                  ▼                  ▼                 │
│              ┌──────────┐       ┌──────────┐       ┌──────────┐           │
│              │  Rule 1  │       │  Rule 2  │       │  Rule 3  │           │
│              │order.*   │       │payment.* │       │  *.failed│           │
│              └──────────┘       └──────────┘       └──────────┘           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│ Inventory Svc   │          │ Notification    │          │  Analytics      │
│                 │          │    Service      │          │   Service       │
│   ┌─────────┐   │          │                 │          │                 │
│   │ Lambda  │   │          │   ┌─────────┐   │          │   ┌─────────┐   │
│   └────┬────┘   │          │   │   SQS   │   │          │   │ Kinesis │   │
│        │        │          │   └────┬────┘   │          │   └────┬────┘   │
│   ┌────┴────┐   │          │        │        │          │        │        │
│   │ DynamoDB│   │          │   ┌────┴────┐   │          │   ┌────┴────┐   │
│   └─────────┘   │          │   │ Lambda  │   │          │   │Firehose │   │
└─────────────────┘          │   └────┬────┘   │          │   └────┬────┘   │
                             │        │        │          │        │        │
                             │   ┌────┴────┐   │          │   ┌────┴────┐   │
                             │   │   SES   │   │          │   │   S3    │   │
                             │   │   SNS   │   │          │   │  Athena │   │
                             │   └─────────┘   │          │   └─────────┘   │
                             └─────────────────┘          └─────────────────┘
```

---

## Event Design

### Event Schema

```json
{
  "version": "1.0",
  "id": "evt_12345",
  "source": "order-service",
  "type": "order.created",
  "time": "2024-01-15T10:30:00Z",
  "data": {
    "orderId": "ord_67890",
    "customerId": "cust_111",
    "items": [
      {"productId": "prod_222", "quantity": 2, "price": 29.99}
    ],
    "total": 59.98
  },
  "metadata": {
    "correlationId": "corr_abc123",
    "causationId": "evt_00000",
    "userId": "user_456"
  }
}
```

### Event Catalog

| Event | Source | Consumers | Purpose |
|-------|--------|-----------|---------|
| order.created | Order Service | Inventory, Notification, Analytics | New order placed |
| order.paid | Payment Service | Order, Notification, Fulfillment | Payment confirmed |
| order.shipped | Fulfillment | Order, Notification | Order shipped |
| order.cancelled | Order Service | Payment, Inventory, Notification | Order cancelled |
| user.registered | User Service | Notification, Analytics | New user signup |
| inventory.low | Inventory Service | Notification, Procurement | Stock below threshold |

---

## EventBridge Configuration

### Event Bus

```hcl
resource "aws_cloudwatch_event_bus" "main" {
  name = "ecommerce-events"
}

resource "aws_schemas_registry" "main" {
  name = "ecommerce-events"
}

resource "aws_schemas_schema" "order_created" {
  name          = "order.created"
  registry_name = aws_schemas_registry.main.name
  type          = "JSONSchemaDraft4"
  content = jsonencode({
    "$schema" = "http://json-schema.org/draft-04/schema#"
    title     = "OrderCreated"
    type      = "object"
    properties = {
      orderId = { type = "string" }
      customerId = { type = "string" }
      total = { type = "number" }
    }
    required = ["orderId", "customerId", "total"]
  })
}
```

### Event Rules

```hcl
# Route order events to inventory service
resource "aws_cloudwatch_event_rule" "order_to_inventory" {
  name           = "order-to-inventory"
  event_bus_name = aws_cloudwatch_event_bus.main.name

  event_pattern = jsonencode({
    source      = ["order-service"]
    detail-type = ["order.created", "order.cancelled"]
  })
}

resource "aws_cloudwatch_event_target" "inventory_lambda" {
  rule           = aws_cloudwatch_event_rule.order_to_inventory.name
  event_bus_name = aws_cloudwatch_event_bus.main.name
  target_id      = "inventory-handler"
  arn            = aws_lambda_function.inventory_handler.arn

  retry_policy {
    maximum_event_age_in_seconds = 3600
    maximum_retry_attempts       = 3
  }

  dead_letter_config {
    arn = aws_sqs_queue.dlq.arn
  }
}
```

---

## Saga Pattern Implementation

### Order Saga (Choreography)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ORDER SAGA (Choreography)                          │
└─────────────────────────────────────────────────────────────────────────────┘

Happy Path:
───────────

 ┌──────────┐    order.created    ┌──────────┐   inventory.reserved   ┌──────────┐
 │  Order   │ ──────────────────> │Inventory │ ─────────────────────> │ Payment  │
 │ Service  │                     │ Service  │                        │ Service  │
 └──────────┘                     └──────────┘                        └──────────┘
                                                                            │
                                                                            │ payment.processed
                                                                            ▼
 ┌──────────┐   order.confirmed   ┌──────────┐    order.fulfilled     ┌──────────┐
 │  Order   │ <────────────────── │Shipping  │ <───────────────────── │ Payment  │
 │ Service  │                     │ Service  │                        │ Service  │
 └──────────┘                     └──────────┘                        └──────────┘


Compensation (Payment Failed):
──────────────────────────────

 ┌──────────┐   payment.failed    ┌──────────┐  inventory.released   ┌──────────┐
 │  Order   │ <────────────────── │ Payment  │ ─────────────────────> │Inventory │
 │ Service  │                     │ Service  │                        │ Service  │
 └──────────┘                     └──────────┘                        └──────────┘
      │
      │ order.cancelled
      ▼
 ┌──────────┐
 │  Notify  │
 │ Customer │
 └──────────┘
```

### Saga Orchestrator (Step Functions)

```json
{
  "Comment": "Order Processing Saga",
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:reserve-inventory",
      "Catch": [{
        "ErrorEquals": ["InsufficientInventory"],
        "Next": "NotifyOutOfStock"
      }],
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:process-payment",
      "Catch": [{
        "ErrorEquals": ["PaymentFailed"],
        "Next": "ReleaseInventory"
      }],
      "Next": "ConfirmOrder"
    },
    "ConfirmOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:confirm-order",
      "Next": "SendConfirmation"
    },
    "SendConfirmation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:send-confirmation",
      "End": true
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:release-inventory",
      "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:cancel-order",
      "Next": "NotifyPaymentFailed"
    },
    "NotifyPaymentFailed": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:notify-failure",
      "End": true
    },
    "NotifyOutOfStock": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:notify-out-of-stock",
      "End": true
    }
  }
}
```

---

## Message Processing Patterns

### SQS with Lambda

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ EventBridge │────>│     SQS     │────>│   Lambda    │
│    Rule     │     │   Queue     │     │  Consumer   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           │ Failed messages
                           ▼
                    ┌─────────────┐     ┌─────────────┐
                    │     DLQ     │────>│   Lambda    │
                    │             │     │  Processor  │
                    └─────────────┘     └─────────────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │   Alert     │
                                        │  CloudWatch │
                                        └─────────────┘
```

### Queue Configuration

```hcl
resource "aws_sqs_queue" "notifications" {
  name                       = "notifications-queue"
  visibility_timeout_seconds = 300  # 6x Lambda timeout
  message_retention_seconds  = 1209600  # 14 days
  receive_wait_time_seconds  = 20  # Long polling

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.notifications_dlq.arn
    maxReceiveCount     = 3
  })
}

resource "aws_sqs_queue" "notifications_dlq" {
  name                       = "notifications-dlq"
  message_retention_seconds  = 1209600
}

resource "aws_lambda_event_source_mapping" "notifications" {
  event_source_arn                   = aws_sqs_queue.notifications.arn
  function_name                      = aws_lambda_function.notification_handler.arn
  batch_size                         = 10
  maximum_batching_window_in_seconds = 5

  function_response_types = ["ReportBatchItemFailures"]
}
```

### Batch Processing with Partial Failures

```typescript
import { SQSHandler, SQSBatchResponse } from 'aws-lambda';

export const handler: SQSHandler = async (event): Promise<SQSBatchResponse> => {
  const batchItemFailures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.body);
      await processMessage(message);
    } catch (error) {
      console.error(`Failed to process message ${record.messageId}:`, error);
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures };
};
```

---

## Service Communication

### Synchronous (API Gateway)

```
For real-time responses:
- User-facing APIs
- Simple queries
- Low-latency requirements
```

### Asynchronous (Events)

```
For eventual consistency:
- Cross-service updates
- Long-running processes
- Fire-and-forget notifications
```

### Request-Response over Events

```typescript
// Publisher
const correlationId = uuid();
await eventBridge.putEvents({
  Entries: [{
    Source: 'order-service',
    DetailType: 'inventory.check.requested',
    Detail: JSON.stringify({
      correlationId,
      productId: 'prod_123',
      quantity: 5,
      replyTo: 'order-service-responses'
    })
  }]
});

// Wait for response (with timeout)
const response = await waitForEvent(correlationId, 5000);
```

---

## Observability

### Distributed Tracing

```typescript
import { Tracer } from '@aws-lambda-powertools/tracer';
import { Logger } from '@aws-lambda-powertools/logger';

const tracer = new Tracer({ serviceName: 'order-service' });
const logger = new Logger({ serviceName: 'order-service' });

export const handler = async (event: any) => {
  const correlationId = event.detail?.metadata?.correlationId;

  logger.appendKeys({ correlationId });
  tracer.putAnnotation('correlationId', correlationId);

  // Processing with tracing
  const segment = tracer.getSegment();
  const subsegment = segment?.addNewSubsegment('process-order');

  try {
    const result = await processOrder(event.detail);
    subsegment?.close();
    return result;
  } catch (error) {
    subsegment?.addError(error as Error);
    subsegment?.close();
    throw error;
  }
};
```

### Event Lineage

```
Event: order.created (evt_001)
  └── Caused by: API call (req_123)
      │
      ├── Triggered: inventory.reserved (evt_002)
      │   └── Correlation: corr_abc
      │
      ├── Triggered: payment.processed (evt_003)
      │   └── Correlation: corr_abc
      │
      └── Triggered: notification.sent (evt_004)
          └── Correlation: corr_abc
```

---

## Error Handling

### Retry Strategy

```hcl
# EventBridge to Lambda
retry_policy {
  maximum_event_age_in_seconds = 3600  # 1 hour
  maximum_retry_attempts       = 3
}

# SQS to Lambda
resource "aws_sqs_queue" "orders" {
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 5
  })
}
```

### Dead Letter Queue Processing

```typescript
// DLQ processor - runs on schedule or alert
export const processDLQ = async () => {
  const messages = await sqs.receiveMessage({
    QueueUrl: DLQ_URL,
    MaxNumberOfMessages: 10,
  });

  for (const message of messages.Messages || []) {
    const originalEvent = JSON.parse(message.Body!);

    // Log for investigation
    console.error('Failed event:', {
      messageId: message.MessageId,
      event: originalEvent,
      attributes: message.Attributes,
    });

    // Option 1: Retry with fixes
    // Option 2: Manual intervention required
    // Option 3: Discard after logging

    await sqs.deleteMessage({
      QueueUrl: DLQ_URL,
      ReceiptHandle: message.ReceiptHandle!,
    });
  }
};
```

---

## Terraform

See [terraform/aws/event-driven/](../../../terraform/aws/event-driven/) for complete implementation.
