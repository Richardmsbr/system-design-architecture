# Data Lake Architecture

Scalable analytics platform on AWS for petabyte-scale data processing.

---

## Architecture Overview

```
                              DATA SOURCES
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │ Application │    │  Databases  │    │   Streams   │    │  External   │
    │    Logs     │    │  (CDC)      │    │  (Events)   │    │    APIs     │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │                  │
           └──────────────────┴──────────────────┴──────────────────┘
                                       │
                                       ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                          INGESTION LAYER                                │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
    │  │  Kinesis    │  │    DMS      │  │  AppFlow    │  │   Lambda    │   │
    │  │  Firehose   │  │  (CDC)      │  │             │  │  Ingestion  │   │
    │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │
    │         │                │                │                │          │
    └─────────┼────────────────┼────────────────┼────────────────┼──────────┘
              │                │                │                │
              └────────────────┴────────────────┴────────────────┘
                                       │
                                       ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                           S3 DATA LAKE                                  │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                         RAW ZONE                                 │   │
    │  │                    s3://datalake/raw/                           │   │
    │  │                                                                  │   │
    │  │  - Original format (JSON, CSV, Parquet)                         │   │
    │  │  - Partitioned by: source/year/month/day/hour                   │   │
    │  │  - Immutable, append-only                                       │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                 │                                       │
    │                                 │ ETL (Glue)                           │
    │                                 ▼                                       │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                       CURATED ZONE                               │   │
    │  │                   s3://datalake/curated/                        │   │
    │  │                                                                  │   │
    │  │  - Cleaned, validated, deduplicated                             │   │
    │  │  - Parquet format with Snappy compression                       │   │
    │  │  - Partitioned by business keys                                 │   │
    │  │  - Schema enforced via Glue Catalog                             │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                 │                                       │
    │                                 │ Aggregation                          │
    │                                 ▼                                       │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                       ANALYTICS ZONE                             │   │
    │  │                  s3://datalake/analytics/                       │   │
    │  │                                                                  │   │
    │  │  - Pre-aggregated metrics                                       │   │
    │  │  - Star schema for BI                                           │   │
    │  │  - Optimized for query patterns                                 │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         CONSUMPTION LAYER                               │
    │                                                                         │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
    │  │   Athena    │  │  Redshift   │  │ QuickSight  │  │  SageMaker  │   │
    │  │  (Ad-hoc)   │  │ (Warehouse) │  │    (BI)     │  │    (ML)     │   │
    │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## Data Zones

### Zone Definitions

| Zone | Purpose | Format | Retention |
|------|---------|--------|-----------|
| Raw | Landing zone, original data | Source format | 7 years |
| Curated | Cleaned, validated | Parquet | 3 years |
| Analytics | Aggregated, optimized | Parquet | 1 year |
| Sandbox | Experimentation | Any | 30 days |

### S3 Bucket Structure

```
s3://company-datalake-{env}/
├── raw/
│   ├── orders/
│   │   └── year=2024/month=01/day=15/hour=10/
│   │       └── orders-2024-01-15-10.json.gz
│   ├── users/
│   └── products/
│
├── curated/
│   ├── orders/
│   │   └── order_date=2024-01-15/
│   │       └── part-00000.parquet
│   ├── users/
│   └── products/
│
├── analytics/
│   ├── daily_sales/
│   ├── customer_360/
│   └── product_metrics/
│
└── sandbox/
    └── {user}/
```

---

## Ingestion Patterns

### Streaming Ingestion (Kinesis Firehose)

```hcl
resource "aws_kinesis_firehose_delivery_stream" "events" {
  name        = "events-stream"
  destination = "extended_s3"

  extended_s3_configuration {
    role_arn            = aws_iam_role.firehose.arn
    bucket_arn          = aws_s3_bucket.raw.arn
    prefix              = "raw/events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/"
    error_output_prefix = "errors/events/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"

    buffering_size     = 128  # MB
    buffering_interval = 300  # seconds

    compression_format = "GZIP"

    data_format_conversion_configuration {
      enabled = true

      input_format_configuration {
        deserializer {
          open_x_json_ser_de {}
        }
      }

      output_format_configuration {
        serializer {
          parquet_ser_de {
            compression = "SNAPPY"
          }
        }
      }

      schema_configuration {
        database_name = aws_glue_catalog_database.main.name
        table_name    = aws_glue_catalog_table.events.name
        role_arn      = aws_iam_role.firehose.arn
      }
    }
  }
}
```

### CDC with DMS

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Source DB     │────>│      DMS        │────>│    S3 Raw       │
│   (PostgreSQL)  │     │  Replication    │     │    (Parquet)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               │ Full load + CDC
                               │
                        Initial: Full snapshot
                        Ongoing: Change events only
```

---

## ETL Processing

### AWS Glue Job

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_path', 'target_path'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read from raw zone
raw_df = glueContext.create_dynamic_frame.from_options(
    connection_type="s3",
    connection_options={
        "paths": [args['source_path']],
        "recurse": True
    },
    format="json"
)

# Transform
# 1. Drop duplicates
# 2. Handle nulls
# 3. Type casting
# 4. Add derived columns

transformed_df = raw_df.toDF() \
    .dropDuplicates(['order_id']) \
    .na.fill({'quantity': 0, 'discount': 0}) \
    .withColumn('total', col('price') * col('quantity') - col('discount')) \
    .withColumn('order_date', to_date(col('created_at')))

# Convert back to DynamicFrame
output_df = DynamicFrame.fromDF(transformed_df, glueContext, "output")

# Write to curated zone
glueContext.write_dynamic_frame.from_options(
    frame=output_df,
    connection_type="s3",
    connection_options={
        "path": args['target_path'],
        "partitionKeys": ["order_date"]
    },
    format="parquet",
    format_options={"compression": "snappy"}
)

job.commit()
```

### Glue Workflow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GLUE WORKFLOW                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Trigger: Daily at 02:00 UTC                                               │
│                                                                             │
│  ┌──────────────┐                                                          │
│  │   Crawler    │ ─── Discover new partitions in raw zone                  │
│  │  (Raw Zone)  │                                                          │
│  └──────┬───────┘                                                          │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐                                                          │
│  │  ETL Job 1   │ ─── Raw → Curated (Orders)                              │
│  │   (Orders)   │                                                          │
│  └──────┬───────┘                                                          │
│         │                                                                   │
│         ├─────────────────┐                                                │
│         │                 │                                                │
│         ▼                 ▼                                                │
│  ┌──────────────┐  ┌──────────────┐                                       │
│  │  ETL Job 2   │  │  ETL Job 3   │                                       │
│  │   (Users)    │  │  (Products)  │                                       │
│  └──────┬───────┘  └──────┬───────┘                                       │
│         │                 │                                                │
│         └────────┬────────┘                                                │
│                  │                                                          │
│                  ▼                                                          │
│  ┌──────────────────────────────┐                                          │
│  │        ETL Job 4             │ ─── Join and aggregate                   │
│  │     (Daily Metrics)          │                                          │
│  └──────────────┬───────────────┘                                          │
│                 │                                                           │
│                 ▼                                                           │
│  ┌──────────────────────────────┐                                          │
│  │         Crawler              │ ─── Update Glue Catalog                  │
│  │      (Curated Zone)          │                                          │
│  └──────────────────────────────┘                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Query Layer

### Athena Configuration

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS analytics;

-- Create table with partitions
CREATE EXTERNAL TABLE analytics.orders (
    order_id STRING,
    customer_id STRING,
    product_id STRING,
    quantity INT,
    price DECIMAL(10,2),
    total DECIMAL(10,2),
    created_at TIMESTAMP
)
PARTITIONED BY (order_date DATE)
STORED AS PARQUET
LOCATION 's3://datalake/curated/orders/'
TBLPROPERTIES ('parquet.compression'='SNAPPY');

-- Load partitions
MSCK REPAIR TABLE analytics.orders;

-- Optimized query with partition pruning
SELECT
    customer_id,
    SUM(total) as total_spent,
    COUNT(*) as order_count
FROM analytics.orders
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31'
GROUP BY customer_id
ORDER BY total_spent DESC
LIMIT 100;
```

### Workgroups and Cost Control

```hcl
resource "aws_athena_workgroup" "analytics" {
  name = "analytics-team"

  configuration {
    enforce_workgroup_configuration    = true
    publish_cloudwatch_metrics_enabled = true

    result_configuration {
      output_location = "s3://${aws_s3_bucket.results.id}/athena-results/"

      encryption_configuration {
        encryption_option = "SSE_S3"
      }
    }

    bytes_scanned_cutoff_per_query = 10737418240  # 10 GB
  }
}
```

---

## Governance

### Glue Data Catalog

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          GLUE DATA CATALOG                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Databases:                                                                 │
│  ├── raw_db                                                                │
│  │   ├── orders (JSON, partitioned by date)                               │
│  │   ├── users (JSON)                                                     │
│  │   └── products (CSV)                                                   │
│  │                                                                         │
│  ├── curated_db                                                            │
│  │   ├── orders (Parquet, partitioned by order_date)                      │
│  │   ├── users (Parquet)                                                  │
│  │   └── products (Parquet)                                               │
│  │                                                                         │
│  └── analytics_db                                                          │
│      ├── daily_sales (Parquet)                                            │
│      ├── customer_360 (Parquet)                                           │
│      └── product_performance (Parquet)                                    │
│                                                                             │
│  Connections:                                                               │
│  ├── rds-production (JDBC to PostgreSQL)                                  │
│  └── redshift-warehouse (JDBC to Redshift)                                │
│                                                                             │
│  Crawlers:                                                                  │
│  ├── raw-zone-crawler (scheduled daily)                                   │
│  └── curated-zone-crawler (triggered by workflow)                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Lake Formation Permissions

```hcl
resource "aws_lakeformation_permissions" "analytics_team" {
  principal = aws_iam_role.analytics_team.arn

  permissions = ["SELECT"]

  table {
    database_name = aws_glue_catalog_database.curated.name
    name          = aws_glue_catalog_table.orders.name
  }

  # Column-level security
  table_with_columns {
    database_name = aws_glue_catalog_database.curated.name
    name          = aws_glue_catalog_table.customers.name
    column_names  = ["customer_id", "segment", "region"]
    # Excludes: email, phone, address (PII)
  }
}
```

---

## Cost Optimization

### S3 Storage Classes

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "datalake" {
  bucket = aws_s3_bucket.datalake.id

  rule {
    id     = "raw-zone-tiering"
    status = "Enabled"

    filter {
      prefix = "raw/"
    }

    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }

    transition {
      days          = 180
      storage_class = "GLACIER"
    }

    expiration {
      days = 2555  # 7 years
    }
  }

  rule {
    id     = "analytics-zone"
    status = "Enabled"

    filter {
      prefix = "analytics/"
    }

    transition {
      days          = 90
      storage_class = "INTELLIGENT_TIERING"
    }

    expiration {
      days = 365
    }
  }
}
```

### Athena Cost Optimization

```
1. Use columnar formats (Parquet, ORC)
   - 10-100x cheaper than JSON/CSV

2. Partition data
   - Scan only relevant partitions

3. Compress data
   - Snappy for balanced speed/compression

4. Use CTAS for repeated queries
   - CREATE TABLE AS SELECT

5. Set workgroup limits
   - bytes_scanned_cutoff_per_query
```

---

## Monitoring

### Key Metrics

| Component | Metric | Alert Threshold |
|-----------|--------|-----------------|
| Glue Jobs | Job duration | > 2x baseline |
| Glue Jobs | DPU hours | > budget |
| Athena | Bytes scanned | > 1TB/query |
| S3 | Request errors | > 1% |
| Kinesis Firehose | DeliveryToS3.Success | < 99% |

### Data Quality Monitoring

```python
# Glue Data Quality rules
from awsglue.context import GlueContext
from awsgluedq.transforms import EvaluateDataQuality

ruleset = """
Rules = [
    ColumnValues "order_id" != NULL,
    ColumnValues "total" > 0,
    ColumnValues "order_date" matches "\\d{4}-\\d{2}-\\d{2}",
    Completeness "customer_id" > 0.99,
    Uniqueness "order_id" > 0.9999,
    RowCount > 1000
]
"""

dq_results = EvaluateDataQuality.apply(
    frame=input_frame,
    ruleset=ruleset,
    publishing_options={
        "dataQualityEvaluationContext": "orders_quality",
        "enableDataQualityCloudWatchMetrics": True
    }
)
```

---

## Terraform

See [terraform/aws/data-lake/](../../../terraform/aws/data-lake/) for complete implementation.
