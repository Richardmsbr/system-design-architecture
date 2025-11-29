# ML Inference Pipeline Architecture

Production-ready machine learning inference platform on AWS with auto-scaling, A/B testing, and model versioning.

---

## Architecture Overview

```
                                    INTERNET
                                        |
                                        v
                              +-------------------+
                              |     Route 53      |
                              |  ml.domain.com    |
                              +---------+---------+
                                        |
                              +---------+---------+
                              |    CloudFront     |
                              |   (Edge Cache)    |
                              +---------+---------+
                                        |
                              +---------+---------+
                              |   API Gateway     |
                              |                   |
                              | - Rate Limiting   |
                              | - API Keys        |
                              | - Request Valid.  |
                              +---------+---------+
                                        |
         +------------------------------+------------------------------+
         |                              |                              |
         v                              v                              v
+------------------+          +------------------+          +------------------+
|     Lambda       |          |     Lambda       |          |     Lambda       |
|   (Preprocess)   |          |   (Preprocess)   |          |   (Preprocess)   |
|                  |          |                  |          |                  |
| - Validate input |          | - Image resize   |          | - Text tokenize  |
| - Feature eng.   |          | - Normalize      |          | - Embedding      |
+--------+---------+          +--------+---------+          +--------+---------+
         |                              |                              |
         +------------------------------+------------------------------+
                                        |
                                        v
                              +-------------------+
                              |    SageMaker      |
                              |    Endpoints      |
                              |                   |
                              | +---------------+ |
                              | | Model A (80%) | |
                              | | Model B (20%) | |
                              | +---------------+ |
                              +---------+---------+
                                        |
         +------------------------------+------------------------------+
         |                              |                              |
         v                              v                              v
+------------------+          +------------------+          +------------------+
|   Real-time      |          |   Async Batch    |          |   Serverless     |
|   Inference      |          |   Transform      |          |   Inference      |
|                  |          |                  |          |                  |
| SageMaker        |          | SageMaker        |          | Lambda +         |
| Real-time        |          | Batch Transform  |          | SageMaker        |
| Endpoints        |          |                  |          | Serverless       |
+--------+---------+          +--------+---------+          +--------+---------+
         |                              |                              |
         +------------------------------+------------------------------+
                                        |
                                        v
                              +-------------------+
                              |   Results Store   |
                              |                   |
                              | - DynamoDB        |
                              | - ElastiCache     |
                              | - S3 (batch)      |
                              +-------------------+
```

---

## Model Registry and Versioning

```
+-----------------------------------------------------------------------------+
|                           MODEL LIFECYCLE                                    |
+-----------------------------------------------------------------------------+
|                                                                             |
|  +-------------+    +-------------+    +-------------+    +-------------+   |
|  |   Training  |--->|  Validation |--->|  Registry   |--->| Deployment  |   |
|  |   Pipeline  |    |   Tests     |    |   Store     |    |   Stage     |   |
|  +-------------+    +-------------+    +-------------+    +-------------+   |
|        |                  |                  |                  |           |
|        v                  v                  v                  v           |
|  +-------------+    +-------------+    +-------------+    +-------------+   |
|  |  SageMaker  |    |  Model      |    |  S3 Model   |    |  SageMaker  |   |
|  |  Training   |    |  Quality    |    |  Artifacts  |    |  Endpoint   |   |
|  |  Jobs       |    |  Metrics    |    |  + ECR      |    |  Config     |   |
|  +-------------+    +-------------+    +-------------+    +-------------+   |
|                                                                             |
+-----------------------------------------------------------------------------+

Model Versioning Strategy:
+---------+------------------+------------------+------------------+
| Version |     Status       |    Traffic %     |     Metrics      |
+---------+------------------+------------------+------------------+
| v1.0.0  | deprecated       |        0%        | accuracy: 0.85   |
| v1.1.0  | production       |       80%        | accuracy: 0.89   |
| v1.2.0  | canary           |       20%        | accuracy: 0.91   |
| v2.0.0  | staging          |        0%        | accuracy: 0.93   |
+---------+------------------+------------------+------------------+
```

---

## SageMaker Endpoint Configuration

### Multi-Model Endpoint

```
+-----------------------------------------------------------------------------+
|                        SAGEMAKER MULTI-MODEL ENDPOINT                        |
+-----------------------------------------------------------------------------+
|                                                                             |
|  Instance Fleet: ml.g4dn.xlarge (4 vCPU, 16GB RAM, 1 GPU)                  |
|                                                                             |
|  +-------------------------+  +-------------------------+                   |
|  |    Model Container 1    |  |    Model Container 2    |                   |
|  |                         |  |                         |                   |
|  |  +-------------------+  |  |  +-------------------+  |                   |
|  |  | Classification    |  |  |  | Object Detection |  |                   |
|  |  | Model (PyTorch)   |  |  |  | Model (TensorRT) |  |                   |
|  |  +-------------------+  |  |  +-------------------+  |                   |
|  |  | Memory: 4GB       |  |  |  | Memory: 8GB      |  |                   |
|  |  | GPU: 25%          |  |  |  | GPU: 50%         |  |                   |
|  |  +-------------------+  |  |  +-------------------+  |                   |
|  +-------------------------+  +-------------------------+                   |
|                                                                             |
|  Auto Scaling Policy:                                                       |
|  - Target: InvocationsPerInstance = 1000                                   |
|  - Min Instances: 2                                                         |
|  - Max Instances: 20                                                        |
|  - Scale-out cooldown: 60s                                                  |
|  - Scale-in cooldown: 300s                                                  |
|                                                                             |
+-----------------------------------------------------------------------------+
```

### Terraform Configuration

```hcl
# SageMaker Model
resource "aws_sagemaker_model" "inference" {
  name               = "ml-inference-model"
  execution_role_arn = aws_iam_role.sagemaker.arn

  primary_container {
    image          = "${aws_ecr_repository.model.repository_url}:latest"
    model_data_url = "s3://${aws_s3_bucket.models.bucket}/models/v1.2.0/model.tar.gz"

    environment = {
      SAGEMAKER_PROGRAM           = "inference.py"
      SAGEMAKER_SUBMIT_DIRECTORY  = "/opt/ml/model/code"
      SAGEMAKER_CONTAINER_LOG_LEVEL = "20"
    }
  }

  vpc_config {
    subnets            = var.private_subnet_ids
    security_group_ids = [aws_security_group.sagemaker.id]
  }

  tags = var.tags
}

# Endpoint Configuration with Production Variants
resource "aws_sagemaker_endpoint_configuration" "inference" {
  name = "ml-inference-config"

  # Production variant (80% traffic)
  production_variants {
    variant_name           = "production"
    model_name             = aws_sagemaker_model.inference.name
    initial_instance_count = 2
    instance_type          = "ml.g4dn.xlarge"
    initial_variant_weight = 80

    serverless_config {
      max_concurrency   = 50
      memory_size_in_mb = 4096
    }
  }

  # Canary variant (20% traffic)
  production_variants {
    variant_name           = "canary"
    model_name             = aws_sagemaker_model.inference_canary.name
    initial_instance_count = 1
    instance_type          = "ml.g4dn.xlarge"
    initial_variant_weight = 20
  }

  data_capture_config {
    enable_capture              = true
    initial_sampling_percentage = 100
    destination_s3_uri          = "s3://${aws_s3_bucket.capture.bucket}/capture"

    capture_options {
      capture_mode = "Input"
    }
    capture_options {
      capture_mode = "Output"
    }

    capture_content_type_header {
      json_content_types = ["application/json"]
    }
  }

  tags = var.tags
}

# Endpoint
resource "aws_sagemaker_endpoint" "inference" {
  name                 = "ml-inference-endpoint"
  endpoint_config_name = aws_sagemaker_endpoint_configuration.inference.name

  deployment_config {
    blue_green_update_policy {
      traffic_routing_configuration {
        type                     = "CANARY"
        canary_size {
          type  = "INSTANCE_COUNT"
          value = 1
        }
        wait_interval_in_seconds = 300
      }

      termination_wait_in_seconds = 120

      maximum_execution_timeout_in_seconds = 3600
    }

    auto_rollback_configuration {
      alarms = [aws_cloudwatch_metric_alarm.model_latency.alarm_name]
    }
  }

  tags = var.tags
}

# Auto Scaling
resource "aws_appautoscaling_target" "sagemaker" {
  max_capacity       = 20
  min_capacity       = 2
  resource_id        = "endpoint/${aws_sagemaker_endpoint.inference.name}/variant/production"
  scalable_dimension = "sagemaker:variant:DesiredInstanceCount"
  service_namespace  = "sagemaker"
}

resource "aws_appautoscaling_policy" "sagemaker" {
  name               = "sagemaker-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.sagemaker.resource_id
  scalable_dimension = aws_appautoscaling_target.sagemaker.scalable_dimension
  service_namespace  = aws_appautoscaling_target.sagemaker.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "SageMakerVariantInvocationsPerInstance"
    }
    target_value       = 1000
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## Inference Patterns

### Real-time Inference

```
Request Flow (p99 < 100ms):

Client --> API Gateway --> Lambda (preprocess) --> SageMaker Endpoint --> Response
                               |                          |
                               v                          v
                          ElastiCache              CloudWatch Metrics
                          (Feature Cache)          (Latency, Errors)


Preprocessing Lambda:

import json
import boto3
import numpy as np
from typing import Dict, Any

sagemaker = boto3.client('sagemaker-runtime')
cache = boto3.client('elasticache')

def handler(event: Dict[str, Any], context) -> Dict[str, Any]:
    body = json.loads(event['body'])

    # Feature engineering
    features = preprocess(body['data'])

    # Check cache for recent predictions
    cache_key = compute_hash(features)
    cached = get_from_cache(cache_key)
    if cached:
        return {
            'statusCode': 200,
            'body': json.dumps({'prediction': cached, 'cached': True})
        }

    # Invoke SageMaker endpoint
    response = sagemaker.invoke_endpoint(
        EndpointName='ml-inference-endpoint',
        ContentType='application/json',
        Body=json.dumps({'features': features.tolist()}),
        TargetVariant='production'  # or let it route based on weights
    )

    prediction = json.loads(response['Body'].read())

    # Cache result
    set_cache(cache_key, prediction, ttl=300)

    return {
        'statusCode': 200,
        'headers': {
            'X-Model-Version': response['InvokedProductionVariant'],
            'X-Latency-Ms': str(response['ResponseMetadata']['HTTPHeaders'].get('x-amzn-invoked-production-variant-latency'))
        },
        'body': json.dumps({'prediction': prediction, 'cached': False})
    }

def preprocess(data: Dict) -> np.ndarray:
    # Normalize, encode, transform
    features = np.array([
        data.get('feature1', 0),
        data.get('feature2', 0),
        # ... more features
    ])
    return (features - MEAN) / STD
```

### Batch Inference

```
Batch Processing Pipeline:

+-------------+     +-------------+     +-------------+     +-------------+
|   S3 Input  |---->|  SageMaker  |---->|  S3 Output  |---->|   Lambda    |
|   Bucket    |     |  Batch      |     |   Bucket    |     |  Notifier   |
|             |     |  Transform  |     |             |     |             |
+-------------+     +-------------+     +-------------+     +-------------+
      ^                                                            |
      |                                                            v
+-------------+                                           +-------------+
| Step        |<------------------------------------------|    SNS      |
| Functions   |                                           |  Topic      |
| Orchestrator|                                           +-------------+
+-------------+


Step Functions Definition:

{
  "StartAt": "ValidateInput",
  "States": {
    "ValidateInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:validate-input",
      "Next": "StartBatchTransform"
    },
    "StartBatchTransform": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sagemaker:createTransformJob.sync",
      "Parameters": {
        "TransformJobName.$": "States.Format('batch-{}', $$.Execution.Name)",
        "ModelName": "ml-inference-model",
        "TransformInput": {
          "DataSource": {
            "S3DataSource": {
              "S3DataType": "S3Prefix",
              "S3Uri.$": "$.inputS3Uri"
            }
          },
          "ContentType": "application/json",
          "SplitType": "Line"
        },
        "TransformOutput": {
          "S3OutputPath.$": "$.outputS3Uri",
          "AssembleWith": "Line"
        },
        "TransformResources": {
          "InstanceType": "ml.m5.xlarge",
          "InstanceCount": 5
        },
        "BatchStrategy": "MultiRecord",
        "MaxPayloadInMB": 6
      },
      "Next": "NotifyComplete"
    },
    "NotifyComplete": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:region:account:batch-complete",
        "Message.$": "States.Format('Batch job {} completed', $$.Execution.Name)"
      },
      "End": true
    }
  }
}
```

### Serverless Inference

```
For sporadic, unpredictable traffic:

+-------------+     +-------------------+     +-------------------+
|   API       |---->|    Lambda         |---->|    SageMaker      |
|   Gateway   |     |    (Orchestrator) |     |    Serverless     |
+-------------+     +-------------------+     |    Inference      |
                                              +-------------------+

Benefits:
- Zero cost when idle
- Automatic scaling to 50 concurrent
- Cold start: ~1-2s (first request)
- Best for: <1000 requests/hour

Configuration:

resource "aws_sagemaker_endpoint_configuration" "serverless" {
  name = "serverless-inference"

  production_variants {
    variant_name = "serverless"
    model_name   = aws_sagemaker_model.inference.name

    serverless_config {
      max_concurrency   = 50
      memory_size_in_mb = 4096  # 2048, 3072, 4096, 5120, 6144
    }
  }
}
```

---

## A/B Testing and Canary Deployments

```
Traffic Distribution:

                              +-------------------+
                              |   API Gateway     |
                              +---------+---------+
                                        |
                              +---------+---------+
                              |   SageMaker       |
                              |   Endpoint        |
                              +---------+---------+
                                        |
              +-------------------------+-------------------------+
              |                         |                         |
         [80% traffic]            [15% traffic]             [5% traffic]
              |                         |                         |
              v                         v                         v
     +----------------+        +----------------+        +----------------+
     | Production v1  |        |  Canary v2     |        |  Shadow v3     |
     | (stable)       |        |  (testing)     |        |  (no response) |
     +----------------+        +----------------+        +----------------+
              |                         |                         |
              v                         v                         v
     +----------------+        +----------------+        +----------------+
     | Metrics:       |        | Metrics:       |        | Metrics:       |
     | - latency p99  |        | - latency p99  |        | - latency p99  |
     | - error rate   |        | - error rate   |        | - error rate   |
     | - accuracy     |        | - accuracy     |        | - accuracy     |
     +----------------+        +----------------+        +----------------+


Model Quality Monitoring:

+-----------------------------------------------------------------------------+
|                        CLOUDWATCH DASHBOARD                                  |
+-----------------------------------------------------------------------------+
|                                                                             |
|  +---------------------------+  +---------------------------+               |
|  | Latency (p99)             |  | Error Rate                |               |
|  |                           |  |                           |               |
|  |   v1: 45ms [OK]           |  |   v1: 0.1% [OK]           |               |
|  |   v2: 52ms [OK]           |  |   v2: 0.3% [WARN]         |               |
|  |   v3: 48ms [OK]           |  |   v3: 0.1% [OK]           |               |
|  +---------------------------+  +---------------------------+               |
|                                                                             |
|  +---------------------------+  +---------------------------+               |
|  | Model Accuracy            |  | Data Drift Score          |               |
|  |                           |  |                           |               |
|  |   v1: 89.2% [OK]          |  |   Feature 1: 0.02 [OK]    |               |
|  |   v2: 91.5% [OK]          |  |   Feature 2: 0.15 [WARN]  |               |
|  |   v3: 92.1% [OK]          |  |   Feature 3: 0.05 [OK]    |               |
|  +---------------------------+  +---------------------------+               |
|                                                                             |
+-----------------------------------------------------------------------------+


Auto-Rollback Configuration:

resource "aws_cloudwatch_metric_alarm" "model_error_rate" {
  alarm_name          = "model-high-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ModelError"
  namespace           = "AWS/SageMaker"
  period              = 60
  statistic           = "Average"
  threshold           = 5  # 5% error rate
  alarm_description   = "Model error rate exceeded threshold"

  dimensions = {
    EndpointName = aws_sagemaker_endpoint.inference.name
    VariantName  = "canary"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

## Model Monitoring

### Data Capture and Analysis

```
Data Capture Pipeline:

SageMaker     -->    S3 Capture    -->    Lambda     -->    SageMaker
Endpoint             Bucket               Trigger           Model Monitor
   |                    |                    |                    |
   v                    v                    v                    v
Predictions         Raw Data            Process &            Detect Drift
(Input/Output)      (JSON Lines)        Transform            Generate Alerts


Model Monitor Types:

1. Data Quality Monitor
   - Input feature statistics
   - Missing values detection
   - Out-of-range values

2. Model Quality Monitor
   - Accuracy metrics (requires ground truth)
   - Precision, Recall, F1
   - Custom business metrics

3. Bias Drift Monitor
   - Fairness metrics across groups
   - Demographic parity
   - Equal opportunity

4. Feature Attribution Drift
   - SHAP values comparison
   - Feature importance changes


Baseline Statistics Example:

{
  "version": 0,
  "features": [
    {
      "name": "age",
      "inferred_type": "Fractional",
      "numerical_statistics": {
        "mean": 35.5,
        "std": 12.3,
        "min": 18,
        "max": 80,
        "distribution": {
          "buckets": [[18,30], [30,50], [50,80]],
          "counts": [3500, 4200, 2300]
        }
      }
    },
    {
      "name": "category",
      "inferred_type": "String",
      "string_statistics": {
        "unique_count": 5,
        "missing_count": 0,
        "value_distribution": {
          "A": 0.35,
          "B": 0.28,
          "C": 0.20,
          "D": 0.12,
          "E": 0.05
        }
      }
    }
  ]
}
```

### CloudWatch Metrics

```python
# Custom metrics for model monitoring
import boto3
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def publish_inference_metrics(
    model_version: str,
    latency_ms: float,
    prediction_class: str,
    confidence: float
):
    cloudwatch.put_metric_data(
        Namespace='MLInference',
        MetricData=[
            {
                'MetricName': 'InferenceLatency',
                'Dimensions': [
                    {'Name': 'ModelVersion', 'Value': model_version},
                ],
                'Value': latency_ms,
                'Unit': 'Milliseconds',
                'Timestamp': datetime.utcnow()
            },
            {
                'MetricName': 'PredictionConfidence',
                'Dimensions': [
                    {'Name': 'ModelVersion', 'Value': model_version},
                    {'Name': 'PredictionClass', 'Value': prediction_class},
                ],
                'Value': confidence,
                'Unit': 'None',
                'Timestamp': datetime.utcnow()
            }
        ]
    )

def publish_drift_metrics(feature_name: str, drift_score: float):
    cloudwatch.put_metric_data(
        Namespace='MLInference',
        MetricData=[
            {
                'MetricName': 'FeatureDrift',
                'Dimensions': [
                    {'Name': 'FeatureName', 'Value': feature_name},
                ],
                'Value': drift_score,
                'Unit': 'None',
                'Timestamp': datetime.utcnow()
            }
        ]
    )
```

---

## Cost Optimization

### Instance Selection Guide

```
+------------------+-------------+--------+--------+------------------+
| Instance Type    | vCPU        | Memory | GPU    | Use Case         |
+------------------+-------------+--------+--------+------------------+
| ml.t3.medium     | 2           | 4 GB   | -      | Dev/Test         |
| ml.m5.large      | 2           | 8 GB   | -      | Small CPU models |
| ml.m5.xlarge     | 4           | 16 GB  | -      | Medium CPU models|
| ml.c5.xlarge     | 4           | 8 GB   | -      | Compute-intensive|
| ml.g4dn.xlarge   | 4           | 16 GB  | T4     | GPU inference    |
| ml.g5.xlarge     | 4           | 16 GB  | A10G   | Large GPU models |
| ml.inf1.xlarge   | 4           | 8 GB   | Inf1   | AWS Inferentia   |
+------------------+-------------+--------+--------+------------------+

Cost Comparison (us-east-1, per hour):

ml.t3.medium:    $0.05   (testing)
ml.m5.large:     $0.12   (production CPU)
ml.g4dn.xlarge:  $0.74   (production GPU)
ml.inf1.xlarge:  $0.30   (optimized inference)
Serverless:      $0.00012 per second (when running)
```

### Cost Estimation

```
Scenario: 10M inferences/month, avg 100ms latency

Option 1: Real-time Endpoints (ml.g4dn.xlarge)
- 2 instances baseline: 2 * $0.74 * 730h = $1,081
- Auto-scaling (avg 3): 3 * $0.74 * 730h = $1,621
- Data transfer: ~$50
- Total: ~$1,700/month

Option 2: Serverless Inference
- Compute: 10M * 0.1s * $0.00012/s = $120
- Requests: 10M * $0.0000002 = $2
- Total: ~$125/month

Option 3: AWS Inferentia (ml.inf1)
- 2 instances: 2 * $0.30 * 730h = $438
- Better latency, lower cost
- Total: ~$500/month

Recommendation:
- Sporadic traffic (<1K/hour): Serverless
- Steady traffic (1K-10K/hour): Inferentia
- High traffic (>10K/hour): G4dn with auto-scaling
```

---

## Security

### IAM Roles

```hcl
# SageMaker execution role
resource "aws_iam_role" "sagemaker_execution" {
  name = "sagemaker-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "sagemaker.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "sagemaker_execution" {
  name = "sagemaker-execution-policy"
  role = aws_iam_role.sagemaker_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.models.arn,
          "${aws_s3_bucket.models.arn}/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "cloudwatch:PutMetricData",
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = aws_kms_key.sagemaker.arn
      }
    ]
  })
}
```

### VPC Configuration

```
+-----------------------------------------------------------------------------+
|                              VPC (10.0.0.0/16)                               |
+-----------------------------------------------------------------------------+
|                                                                             |
|  +---------------------------+     +---------------------------+            |
|  |    Private Subnet A       |     |    Private Subnet B       |            |
|  |    10.0.1.0/24            |     |    10.0.2.0/24            |            |
|  |                           |     |                           |            |
|  |  +---------------------+  |     |  +---------------------+  |            |
|  |  | SageMaker Endpoint  |  |     |  | SageMaker Endpoint  |  |            |
|  |  | (Primary)           |  |     |  | (Standby)           |  |            |
|  |  +---------------------+  |     |  +---------------------+  |            |
|  |                           |     |                           |            |
|  +---------------------------+     +---------------------------+            |
|                                                                             |
|  VPC Endpoints (Interface):                                                 |
|  - com.amazonaws.region.sagemaker.api                                       |
|  - com.amazonaws.region.sagemaker.runtime                                   |
|  - com.amazonaws.region.ecr.api                                             |
|  - com.amazonaws.region.ecr.dkr                                             |
|  - com.amazonaws.region.s3 (Gateway)                                        |
|  - com.amazonaws.region.logs                                                |
|                                                                             |
+-----------------------------------------------------------------------------+
```

---

## CI/CD Pipeline

```
Model Deployment Pipeline:

+-------------+     +-------------+     +-------------+     +-------------+
|   GitHub    |---->| CodeBuild   |---->| SageMaker   |---->| Approval    |
|   (Code)    |     | (Build &    |     | Pipeline    |     | (Manual)    |
|             |     |  Test)      |     | (Train)     |     |             |
+-------------+     +-------------+     +-------------+     +-------------+
                                                                   |
+-------------+     +-------------+     +-------------+     +------v------+
|   Monitor   |<----|  CloudWatch |<----| SageMaker   |<----|   Deploy    |
|   Dashboard |     |   Alarms    |     |   Endpoint  |     | (Blue/Green)|
+-------------+     +-------------+     +-------------+     +-------------+


buildspec.yml:

version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.9
    commands:
      - pip install -r requirements.txt
      - pip install pytest sagemaker

  pre_build:
    commands:
      - echo "Running unit tests..."
      - pytest tests/unit -v
      - echo "Running model validation..."
      - python scripts/validate_model.py

  build:
    commands:
      - echo "Building inference container..."
      - docker build -t $ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker push $ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION

  post_build:
    commands:
      - echo "Updating SageMaker model..."
      - python scripts/deploy_model.py --version $CODEBUILD_RESOLVED_SOURCE_VERSION

artifacts:
  files:
    - model_artifacts/**/*
    - deployment/**/*
```

---

## Terraform

See [terraform/aws/ml-inference/](../../../terraform/aws/ml-inference/) for complete implementation.
