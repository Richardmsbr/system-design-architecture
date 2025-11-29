# 3-Tier Web Application Architecture

Production-ready architecture for scalable web applications on AWS.

---

## Architecture Overview

```
                                    INTERNET
                                        │
                                        ▼
                              ┌─────────────────┐
                              │    Route 53     │
                              │   (DNS + Health)│
                              └────────┬────────┘
                                       │
                              ┌────────┴────────┐
                              │   CloudFront    │
                              │   (CDN + WAF)   │
                              └────────┬────────┘
                                       │
                              ┌────────┴────────┐
                              │ ACM Certificate │
                              │   (HTTPS/TLS)   │
                              └────────┬────────┘
                                       │
    ┌──────────────────────────────────┴──────────────────────────────────┐
    │                              VPC                                     │
    │                         10.0.0.0/16                                  │
    │                                                                      │
    │   PUBLIC SUBNETS                                                     │
    │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
    │   │  10.0.1.0/24    │  │  10.0.2.0/24    │  │  10.0.3.0/24    │    │
    │   │    us-east-1a   │  │    us-east-1b   │  │    us-east-1c   │    │
    │   │                 │  │                 │  │                 │    │
    │   │   ┌─────────┐   │  │   ┌─────────┐   │  │   ┌─────────┐   │    │
    │   │   │   NAT   │   │  │   │   NAT   │   │  │   │   NAT   │   │    │
    │   │   │ Gateway │   │  │   │ Gateway │   │  │   │ Gateway │   │    │
    │   │   └─────────┘   │  │   └─────────┘   │  │   └─────────┘   │    │
    │   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │
    │            │                    │                    │              │
    │   ┌────────┴────────────────────┴────────────────────┴────────┐    │
    │   │                  APPLICATION LOAD BALANCER                │    │
    │   │                    (Cross-zone enabled)                   │    │
    │   └────────┬────────────────────┬────────────────────┬────────┘    │
    │            │                    │                    │              │
    │   PRIVATE SUBNETS (Application)                                     │
    │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
    │   │  10.0.11.0/24   │  │  10.0.12.0/24   │  │  10.0.13.0/24   │    │
    │   │    us-east-1a   │  │    us-east-1b   │  │    us-east-1c   │    │
    │   │                 │  │                 │  │                 │    │
    │   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │    │
    │   │  │ECS Fargate│  │  │  │ECS Fargate│  │  │  │ECS Fargate│  │    │
    │   │  │  Tasks    │  │  │  │  Tasks    │  │  │  │  Tasks    │  │    │
    │   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │    │
    │   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │
    │            │                    │                    │              │
    │            └────────────────────┼────────────────────┘              │
    │                                 │                                   │
    │   ┌─────────────────────────────┴─────────────────────────────┐    │
    │   │                      ElastiCache                          │    │
    │   │                    (Redis Cluster)                        │    │
    │   └─────────────────────────────┬─────────────────────────────┘    │
    │                                 │                                   │
    │   PRIVATE SUBNETS (Database)                                        │
    │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
    │   │  10.0.21.0/24   │  │  10.0.22.0/24   │  │  10.0.23.0/24   │    │
    │   │    us-east-1a   │  │    us-east-1b   │  │    us-east-1c   │    │
    │   │                 │  │                 │  │                 │    │
    │   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │    │
    │   │  │  Aurora   │  │  │  │  Aurora   │  │  │  │  Aurora   │  │    │
    │   │  │ Primary   │  │  │  │ Replica   │  │  │  │ Replica   │  │    │
    │   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │    │
    │   └─────────────────┘  └─────────────────┘  └─────────────────┘    │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. DNS and CDN Layer

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| Route 53 | DNS management, health checks | Latency-based routing |
| CloudFront | Content delivery, caching | Origin: ALB, TTL: 86400s for static |
| WAF | Web Application Firewall | OWASP rules, rate limiting |
| ACM | SSL/TLS certificates | Auto-renewal enabled |

### 2. Network Layer

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| VPC | Network isolation | 10.0.0.0/16, DNS enabled |
| Public Subnets | NAT, bastion hosts | 3 AZs, /24 each |
| Private Subnets | Application tier | 3 AZs, /24 each |
| Database Subnets | Data tier | 3 AZs, /24 each, no internet |
| NAT Gateway | Outbound internet | 1 per AZ (HA) or 1 (cost) |

### 3. Compute Layer

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| ECS Fargate | Container orchestration | Serverless, auto-scaling |
| ALB | Load balancing | Cross-zone, health checks |
| Auto Scaling | Capacity management | CPU/Memory target tracking |

### 4. Data Layer

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| Aurora PostgreSQL | Primary database | Serverless v2, Multi-AZ |
| ElastiCache Redis | Caching, sessions | Cluster mode, encryption |
| S3 | Static assets, backups | Versioning, lifecycle rules |

---

## Security

### Security Groups

```
ALB Security Group
──────────────────
Inbound:  443 from 0.0.0.0/0 (HTTPS)
          80 from 0.0.0.0/0 (HTTP redirect)
Outbound: All to ECS SG

ECS Security Group
──────────────────
Inbound:  8080 from ALB SG
Outbound: 5432 to Database SG
          6379 to Redis SG
          443 to 0.0.0.0/0 (external APIs)

Database Security Group
───────────────────────
Inbound:  5432 from ECS SG
Outbound: None

Redis Security Group
────────────────────
Inbound:  6379 from ECS SG
Outbound: None
```

### IAM Roles

```
ECS Execution Role
──────────────────
- ecr:GetDownloadUrlForLayer
- ecr:BatchGetImage
- logs:CreateLogStream
- logs:PutLogEvents
- ssm:GetParameters
- secretsmanager:GetSecretValue

ECS Task Role
─────────────
- s3:GetObject (for app assets)
- sqs:SendMessage (if needed)
- Custom permissions for app
```

---

## Scaling Configuration

### ECS Auto Scaling

```hcl
# Target Tracking - CPU
resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

### Scaling Thresholds

| Metric | Target | Scale Out | Scale In |
|--------|--------|-----------|----------|
| CPU | 70% | +2 tasks | -1 task |
| Memory | 80% | +2 tasks | -1 task |
| Request Count | 1000/target | +2 tasks | -1 task |

---

## Cost Optimization

### Development Environment

```
- Single NAT Gateway (not HA)
- t3.micro for bastion
- Aurora Serverless (scales to 0)
- Single AZ for non-critical
- Spot instances for workers
```

### Production Environment

```
- Multi-AZ everything
- Reserved capacity for baseline
- Savings Plans for compute
- S3 Intelligent-Tiering
- CloudFront for static content
```

### Estimated Monthly Cost (Production)

| Component | Configuration | Cost |
|-----------|---------------|------|
| ECS Fargate | 4 tasks, 1 vCPU, 2GB | $120 |
| Aurora Serverless | 2-8 ACU | $150 |
| ElastiCache | cache.t3.medium x 2 | $100 |
| NAT Gateway | 3 gateways | $100 |
| ALB | Standard usage | $25 |
| CloudFront | 100GB/month | $10 |
| **Total** | | **~$500/month** |

---

## Deployment

### CI/CD Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   GitHub    │───>│  CodeBuild  │───>│    ECR      │───>│    ECS      │
│   Push      │    │   Build     │    │   Push      │    │   Deploy    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │   Tests     │
                   │  Security   │
                   │   Scans     │
                   └─────────────┘
```

### Blue-Green Deployment

```hcl
deployment_controller {
  type = "CODE_DEPLOY"
}

# CodeDeploy handles traffic shifting
# 10% -> 50% -> 100% with health checks
```

---

## Monitoring

### CloudWatch Dashboards

```
Key Metrics:
- ALB: RequestCount, TargetResponseTime, HTTPCode_Target_5XX
- ECS: CPUUtilization, MemoryUtilization, RunningTaskCount
- Aurora: DatabaseConnections, CPUUtilization, FreeableMemory
- Redis: CacheHits, CacheMisses, CurrConnections
```

### Alarms

| Metric | Threshold | Action |
|--------|-----------|--------|
| ALB 5XX | > 1% for 5 min | SNS alert |
| ECS CPU | > 85% for 5 min | Scale out + alert |
| Aurora CPU | > 80% for 10 min | SNS alert |
| Redis Memory | > 80% | SNS alert |

---

## Disaster Recovery

### Backup Strategy

| Component | Backup | Retention | RTO | RPO |
|-----------|--------|-----------|-----|-----|
| Aurora | Automated snapshots | 30 days | 1 hour | 5 min |
| Redis | Daily snapshots | 7 days | 1 hour | 24 hours |
| S3 | Cross-region replication | Indefinite | 0 | 0 |
| ECS Task Def | Version history | 100 versions | Minutes | 0 |

### Recovery Procedures

```
Database Failure:
1. Aurora auto-failover to replica (< 30 seconds)
2. Update DNS if needed
3. Verify application connectivity

Region Failure:
1. Route 53 health check fails
2. DNS fails over to DR region
3. DR environment scales up
4. Verify data consistency
```

---

## Terraform

See [terraform/aws/3-tier/](../../../terraform/aws/3-tier/) for complete implementation.

```bash
# Deploy
cd terraform/aws/3-tier
terraform init
terraform plan -var-file=production.tfvars
terraform apply -var-file=production.tfvars
```
