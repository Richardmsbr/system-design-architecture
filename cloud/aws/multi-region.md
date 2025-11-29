# Multi-Region Active-Active Architecture

Global availability architecture with active-active deployments across AWS regions.

---

## Architecture Overview

```
                                    GLOBAL LAYER
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │                         ┌─────────────────┐                            │
    │                         │    Route 53     │                            │
    │                         │  (Latency DNS)  │                            │
    │                         └────────┬────────┘                            │
    │                                  │                                      │
    │                         ┌────────┴────────┐                            │
    │                         │   CloudFront    │                            │
    │                         │  (Global CDN)   │                            │
    │                         └────────┬────────┘                            │
    │                                  │                                      │
    └──────────────────────────────────┼──────────────────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼

    US-EAST-1                     EU-WEST-1                    AP-SOUTHEAST-1
    (Primary)                     (Active)                     (Active)
    ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
    │                 │          │                 │          │                 │
    │  ┌───────────┐  │          │  ┌───────────┐  │          │  ┌───────────┐  │
    │  │    ALB    │  │          │  │    ALB    │  │          │  │    ALB    │  │
    │  └─────┬─────┘  │          │  └─────┬─────┘  │          │  └─────┬─────┘  │
    │        │        │          │        │        │          │        │        │
    │  ┌─────┴─────┐  │          │  ┌─────┴─────┐  │          │  ┌─────┴─────┐  │
    │  │    EKS    │  │          │  │    EKS    │  │          │  │    EKS    │  │
    │  │  Cluster  │  │          │  │  Cluster  │  │          │  │  Cluster  │  │
    │  └─────┬─────┘  │          │  └─────┬─────┘  │          │  └─────┬─────┘  │
    │        │        │          │        │        │          │        │        │
    │  ┌─────┴─────┐  │          │  ┌─────┴─────┐  │          │  ┌─────┴─────┐  │
    │  │  Aurora   │◄─┼──────────┼─►│  Aurora   │◄─┼──────────┼─►│  Aurora   │  │
    │  │ Global DB │  │  Async   │  │ Secondary │  │  Async   │  │ Secondary │  │
    │  │ (Writer)  │  │  Repl    │  │ (Reader)  │  │  Repl    │  │ (Reader)  │  │
    │  └───────────┘  │  <1s     │  └───────────┘  │  <1s     │  └───────────┘  │
    │                 │          │                 │          │                 │
    │  ┌───────────┐  │          │  ┌───────────┐  │          │  ┌───────────┐  │
    │  │ElastiCache│  │          │  │ElastiCache│  │          │  │ElastiCache│  │
    │  │  Global   │◄─┼──────────┼─►│  Global   │◄─┼──────────┼─►│  Global   │  │
    │  │ Datastore │  │          │  │ Datastore │  │          │  │ Datastore │  │
    │  └───────────┘  │          │  └───────────┘  │          │  └───────────┘  │
    │                 │          │                 │          │                 │
    └─────────────────┘          └─────────────────┘          └─────────────────┘
```

---

## DNS Configuration

### Route 53 Latency-Based Routing

```hcl
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  # US-EAST-1
  set_identifier = "us-east-1"
  latency_routing_policy {
    region = "us-east-1"
  }

  alias {
    name                   = aws_lb.us_east_1.dns_name
    zone_id                = aws_lb.us_east_1.zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.us_east_1.id
}

resource "aws_route53_record" "api_eu" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  # EU-WEST-1
  set_identifier = "eu-west-1"
  latency_routing_policy {
    region = "eu-west-1"
  }

  alias {
    name                   = aws_lb.eu_west_1.dns_name
    zone_id                = aws_lb.eu_west_1.zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.eu_west_1.id
}
```

### Health Checks

```hcl
resource "aws_route53_health_check" "us_east_1" {
  fqdn              = aws_lb.us_east_1.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  regions = [
    "us-east-1",
    "eu-west-1",
    "ap-southeast-1"
  ]

  tags = {
    Name = "api-health-us-east-1"
  }
}
```

---

## Database Replication

### Aurora Global Database

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AURORA GLOBAL DATABASE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   US-EAST-1 (Primary)                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         PRIMARY CLUSTER                              │  │
│   │                                                                      │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │  │
│   │   │   Writer    │  │   Reader    │  │   Reader    │                │  │
│   │   │  Instance   │  │  Instance   │  │  Instance   │                │  │
│   │   └──────┬──────┘  └─────────────┘  └─────────────┘                │  │
│   │          │                                                          │  │
│   │          │ Synchronous                                              │  │
│   │          ▼                                                          │  │
│   │   ┌─────────────────────────────────────────────────────────────┐  │  │
│   │   │                   Aurora Storage (6 copies)                  │  │  │
│   │   └─────────────────────────────────────────────────────────────┘  │  │
│   │                                                                      │  │
│   └──────────────────────────────────────┬──────────────────────────────┘  │
│                                          │                                  │
│                                          │ Asynchronous Replication        │
│                                          │ (typically < 1 second)          │
│                                          │                                  │
│   ┌──────────────────────────────────────┼──────────────────────────────┐  │
│   │                                      │                              │  │
│   │   EU-WEST-1                          ▼         AP-SOUTHEAST-1       │  │
│   │   ┌──────────────────────┐    ┌──────────────────────┐             │  │
│   │   │   SECONDARY CLUSTER  │    │   SECONDARY CLUSTER  │             │  │
│   │   │                      │    │                      │             │  │
│   │   │   ┌───────────┐      │    │   ┌───────────┐      │             │  │
│   │   │   │  Reader   │      │    │   │  Reader   │      │             │  │
│   │   │   │ Instance  │      │    │   │ Instance  │      │             │  │
│   │   │   └───────────┘      │    │   └───────────┘      │             │  │
│   │   │                      │    │                      │             │  │
│   │   │   Can be promoted    │    │   Can be promoted    │             │  │
│   │   │   to writer in       │    │   to writer in       │             │  │
│   │   │   < 1 minute         │    │   < 1 minute         │             │  │
│   │   │                      │    │                      │             │  │
│   │   └──────────────────────┘    └──────────────────────┘             │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Terraform Configuration

```hcl
# Primary cluster (US-EAST-1)
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "global-database"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  database_name             = "app"
  storage_encrypted         = true
}

resource "aws_rds_cluster" "primary" {
  provider                  = aws.us_east_1
  cluster_identifier        = "primary-cluster"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = aws_rds_global_cluster.main.engine
  engine_version            = aws_rds_global_cluster.main.engine_version
  database_name             = "app"
  master_username           = var.db_username
  master_password           = var.db_password
  db_subnet_group_name      = aws_db_subnet_group.primary.name
  vpc_security_group_ids    = [aws_security_group.db_primary.id]
}

# Secondary cluster (EU-WEST-1)
resource "aws_rds_cluster" "secondary" {
  provider                  = aws.eu_west_1
  cluster_identifier        = "secondary-cluster-eu"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = aws_rds_global_cluster.main.engine
  engine_version            = aws_rds_global_cluster.main.engine_version
  db_subnet_group_name      = aws_db_subnet_group.secondary_eu.name
  vpc_security_group_ids    = [aws_security_group.db_secondary_eu.id]

  depends_on = [aws_rds_cluster_instance.primary]
}
```

---

## Data Consistency Patterns

### Write Forwarding

```
All regions accept reads locally.
Writes are forwarded to primary region.

┌─────────────┐     Write      ┌─────────────┐
│   Client    │ ─────────────> │  EU-WEST-1  │
│   (Europe)  │                │   (Local)   │
└─────────────┘                └──────┬──────┘
                                      │
                                      │ Forward to Primary
                                      ▼
                               ┌─────────────┐
                               │  US-EAST-1  │
                               │  (Primary)  │
                               └──────┬──────┘
                                      │
                                      │ Replicate
                                      ▼
                               ┌─────────────┐
                               │  EU-WEST-1  │
                               │ (Confirmed) │
                               └─────────────┘
```

### Conflict Resolution

```python
# Last-writer-wins with vector clocks
class ConflictResolver:
    def resolve(self, record1, record2):
        # Compare vector clocks
        if record1.vector_clock > record2.vector_clock:
            return record1
        elif record2.vector_clock > record1.vector_clock:
            return record2
        else:
            # Concurrent writes - use timestamp as tiebreaker
            return record1 if record1.timestamp > record2.timestamp else record2

# Application-level conflict resolution
class OrderConflictResolver:
    def resolve(self, order1, order2):
        # Merge order items (union)
        merged_items = {}
        for item in order1.items + order2.items:
            if item.id in merged_items:
                # Take higher quantity
                merged_items[item.id].quantity = max(
                    merged_items[item.id].quantity,
                    item.quantity
                )
            else:
                merged_items[item.id] = item

        return Order(items=list(merged_items.values()))
```

---

## Caching Strategy

### ElastiCache Global Datastore

```hcl
resource "aws_elasticache_global_replication_group" "main" {
  global_replication_group_id_suffix = "global-cache"
  primary_replication_group_id       = aws_elasticache_replication_group.primary.id

  # Automatic failover
  automatic_failover_enabled = true
}

resource "aws_elasticache_replication_group" "primary" {
  provider                   = aws.us_east_1
  replication_group_id       = "cache-primary"
  description                = "Primary cache cluster"
  node_type                  = "cache.r6g.large"
  num_cache_clusters         = 2
  automatic_failover_enabled = true
  multi_az_enabled           = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}

resource "aws_elasticache_replication_group" "secondary" {
  provider                       = aws.eu_west_1
  replication_group_id           = "cache-secondary-eu"
  description                    = "Secondary cache cluster EU"
  global_replication_group_id    = aws_elasticache_global_replication_group.main.global_replication_group_id
  num_cache_clusters             = 2
  automatic_failover_enabled     = true
  multi_az_enabled               = true
}
```

---

## Failover Procedures

### Automated Failover

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          FAILOVER DETECTION                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Route 53 health checks fail (3 consecutive)                            │
│                                                                             │
│  2. CloudWatch alarm triggers                                              │
│     - ALB HealthyHostCount = 0                                             │
│     - Custom health endpoint fails                                          │
│                                                                             │
│  3. Automatic actions:                                                      │
│     - Route 53 removes unhealthy endpoint                                  │
│     - Traffic routes to remaining regions                                   │
│     - SNS notification sent                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                        DATABASE FAILOVER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  If PRIMARY region fails:                                                   │
│                                                                             │
│  1. Detect failure (Aurora monitors replication lag)                       │
│  2. Initiate managed planned failover (if possible)                        │
│  3. Or: Initiate unplanned failover                                        │
│                                                                             │
│     aws rds failover-global-cluster \                                      │
│       --global-cluster-identifier global-database \                         │
│       --target-db-cluster-identifier secondary-cluster-eu                  │
│                                                                             │
│  4. Secondary promoted to primary (< 1 minute)                             │
│  5. Application connection strings updated via DNS                          │
│  6. Former primary becomes secondary after recovery                        │
│                                                                             │
│  RPO: < 1 second (replication lag)                                         │
│  RTO: < 1 minute                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Failover Runbook

```yaml
# failover-runbook.yaml
name: Region Failover
description: Failover from primary to secondary region

steps:
  - name: Assess situation
    actions:
      - Check CloudWatch dashboards
      - Verify health check failures
      - Confirm regional outage (not application bug)

  - name: Initiate traffic failover
    actions:
      - Route 53 automatically redirects (if health checks configured)
      - Or manually update DNS weights to 0 for failed region

  - name: Database failover (if needed)
    actions:
      - Check Aurora global database status
      - Run: aws rds failover-global-cluster
      - Verify new primary accepts writes

  - name: Verify services
    actions:
      - Check application health endpoints
      - Verify database connectivity
      - Run smoke tests

  - name: Communication
    actions:
      - Update status page
      - Notify stakeholders
      - Create incident ticket

  - name: Post-incident
    actions:
      - Monitor for data consistency issues
      - Plan failback when original region recovers
      - Conduct post-mortem
```

---

## Cost Considerations

### Multi-Region Cost Factors

| Component | Cost Factor | Optimization |
|-----------|-------------|--------------|
| Data Transfer | Cross-region ~$0.02/GB | Compress, batch |
| Aurora | 2-3x single region | Use serverless |
| ElastiCache | Per-region cluster | Right-size nodes |
| NAT Gateway | Per-region | VPC endpoints |
| CloudFront | Already global | Cache more |

### Estimated Monthly Cost (3 Regions)

```
Compute (EKS):
- 3 regions x 4 nodes x $150 = $1,800

Database (Aurora Global):
- Primary: 2 ACU min = $150
- Secondary x 2: 2 ACU min each = $300
- Storage: 100GB x $0.10 x 3 = $30
- Total: $480

Cache (ElastiCache Global):
- 3 regions x 2 nodes x $100 = $600

Data Transfer:
- Cross-region replication: ~$200
- CDN to origin: ~$100

Load Balancers:
- 3 x $25 = $75

Total: ~$3,300/month
```

---

## Monitoring

### Global Dashboard

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        GLOBAL HEALTH DASHBOARD                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Region Status                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                        │
│  │ US-EAST-1   │  │ EU-WEST-1   │  │AP-SOUTHEAST │                        │
│  │   [OK]      │  │   [OK]      │  │    [OK]     │                        │
│  │             │  │             │  │             │                        │
│  │ Latency:45ms│  │ Latency:52ms│  │ Latency:48ms│                        │
│  │ RPS: 15,000 │  │ RPS: 8,000  │  │ RPS: 5,000  │                        │
│  │ Errors: 0.1%│  │ Errors: 0.1%│  │ Errors: 0.2%│                        │
│  └─────────────┘  └─────────────┘  └─────────────┘                        │
│                                                                             │
│  Database Replication                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ US-EAST-1 (Primary) ──────> EU-WEST-1: Lag 0.3s [OK]               │  │
│  │                     ──────> AP-SOUTHEAST: Lag 0.5s [OK]            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Active Alarms: 0                                                          │
│  Last Failover Test: 7 days ago                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Terraform

See [terraform/aws/multi-region/](../../../terraform/aws/multi-region/) for complete implementation.
