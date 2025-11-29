# System Design & Architecture Patterns

A comprehensive collection of production-ready system design blueprints, architectural patterns, and scalability strategies. Built from real-world experience designing systems that handle millions of requests per second.

---

## Overview

This repository provides battle-tested architectural patterns and system design templates used in production environments at scale. Each design includes capacity estimation, component deep-dives, failure scenarios, and infrastructure-as-code examples.

**Target Audience**: Senior Engineers, Solution Architects, Technical Leads, and Engineering Managers preparing for system design interviews or designing real systems.

---

## Table of Contents

1. [System Design Interview Templates](#system-design-interview-templates)
2. [Architectural Patterns](#architectural-patterns)
3. [Cloud Reference Architectures](#cloud-reference-architectures)
4. [Data Engineering](#data-engineering)
5. [Observability & Reliability](#observability--reliability)
6. [Infrastructure as Code](#infrastructure-as-code)
7. [Performance Engineering](#performance-engineering)

---

## System Design Interview Templates

Production-grade designs for common system design interview questions. Each template follows a structured approach: requirements gathering, capacity estimation, high-level design, deep-dive components, and failure analysis.

| System | Complexity | Key Technologies | Status |
|--------|------------|------------------|--------|
| [URL Shortener](designs/url-shortener.md) | Entry | DynamoDB, Redis, CloudFront | Complete |
| [Rate Limiter](designs/rate-limiter.md) | Entry | Redis, Token Bucket, Sliding Window | Complete |
| [Distributed Cache](designs/distributed-cache.md) | Intermediate | Consistent Hashing, Redis Cluster | Complete |
| [Message Queue](designs/message-queue.md) | Intermediate | Kafka, RabbitMQ, SQS | Complete |
| [Real-time Messaging](designs/whatsapp.md) | Intermediate | WebSocket, Cassandra, Signal Protocol | Complete |
| [News Feed](designs/news-feed.md) | Intermediate | Fan-out, Timeline Service, Graph DB | Complete |
| [Search Autocomplete](designs/autocomplete.md) | Intermediate | Trie, Elasticsearch, Prefix Trees | Complete |
| [Video Streaming](designs/video-streaming.md) | Advanced | Adaptive Bitrate, CDN, Transcoding | Complete |
| [Ride Sharing](designs/uber.md) | Advanced | Geospatial Index, Real-time Matching | Complete |
| [Distributed File System](designs/distributed-fs.md) | Advanced | GFS, HDFS, Erasure Coding | Complete |
| [Payment System](designs/payment-system.md) | Advanced | ACID, Idempotency, Reconciliation | Complete |
| [Ad Serving Platform](designs/ad-serving.md) | Expert | RTB, ML Inference, Sub-10ms Latency | Complete |

---

## Architectural Patterns

### Distributed Systems Patterns

```
patterns/
├── communication/
│   ├── api-gateway.md              # Request routing, auth, rate limiting
│   ├── service-mesh.md             # Istio, Linkerd, mTLS
│   ├── async-messaging.md          # Event-driven, pub/sub, queues
│   └── grpc-federation.md          # Schema stitching, federation
│
├── data-management/
│   ├── cqrs.md                     # Command Query Responsibility Segregation
│   ├── event-sourcing.md           # Append-only event log, projections
│   ├── saga-pattern.md             # Distributed transaction coordination
│   ├── outbox-pattern.md           # Reliable event publishing
│   └── change-data-capture.md      # Debezium, database replication
│
├── resilience/
│   ├── circuit-breaker.md          # Failure isolation, fallbacks
│   ├── bulkhead.md                 # Resource isolation
│   ├── retry-backoff.md            # Exponential backoff, jitter
│   └── timeout-patterns.md         # Cascading failure prevention
│
└── scaling/
    ├── horizontal-scaling.md       # Stateless services, auto-scaling
    ├── database-sharding.md        # Consistent hashing, range partitioning
    ├── read-replicas.md            # Read scaling, replication lag
    └── caching-strategies.md       # Write-through, write-behind, cache-aside
```

### Pattern: CQRS with Event Sourcing

```
                                    WRITE PATH
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   Client    │────>│  Command    │────>│   Domain    │────>│   Event     │
    │             │     │   Handler   │     │   Model     │     │    Store    │
    └─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                       │
                                                                       v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                           EVENT BUS (Kafka)                             │
    └─────────────────────────────────────────────────────────────────────────┘
                                       │
               ┌───────────────────────┼───────────────────────┐
               v                       v                       v
    ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
    │   Projection    │     │   Projection    │     │   Projection    │
    │   (List View)   │     │  (Analytics)    │     │   (Search)      │
    └────────┬────────┘     └────────┬────────┘     └────────┬────────┘
             │                       │                       │
             v                       v                       v
    ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
    │   PostgreSQL    │     │   ClickHouse    │     │  Elasticsearch  │
    │   (Read Model)  │     │   (OLAP)        │     │   (Full-text)   │
    └─────────────────┘     └─────────────────┘     └─────────────────┘
                                    ^
                                    │
                                READ PATH
    ┌─────────────┐     ┌─────────────┐     │
    │   Client    │────>│   Query     │─────┘
    │             │     │   Handler   │
    └─────────────┘     └─────────────┘
```

---

## Cloud Reference Architectures

### AWS Production Architecture: Multi-Region Active-Active

```
                                 GLOBAL
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │     ┌─────────────┐              ┌─────────────┐                       │
    │     │  Route 53   │              │  CloudFront │                       │
    │     │  (Latency)  │─────────────>│    (CDN)    │                       │
    │     └─────────────┘              └──────┬──────┘                       │
    │                                         │                               │
    └─────────────────────────────────────────┼───────────────────────────────┘
                                              │
              ┌───────────────────────────────┼───────────────────────────────┐
              │                               │                               │
              v                               v                               v
    ┌─────────────────────┐       ┌─────────────────────┐       ┌─────────────────────┐
    │    US-EAST-1        │       │    EU-WEST-1        │       │    AP-SOUTH-1       │
    │                     │       │                     │       │                     │
    │  ┌───────────────┐  │       │  ┌───────────────┐  │       │  ┌───────────────┐  │
    │  │      ALB      │  │       │  │      ALB      │  │       │  │      ALB      │  │
    │  └───────┬───────┘  │       │  └───────┬───────┘  │       │  └───────┬───────┘  │
    │          │          │       │          │          │       │          │          │
    │  ┌───────┴───────┐  │       │  ┌───────┴───────┐  │       │  ┌───────┴───────┐  │
    │  │     EKS       │  │       │  │     EKS       │  │       │  │     EKS       │  │
    │  │   Cluster     │  │       │  │   Cluster     │  │       │  │   Cluster     │  │
    │  └───────┬───────┘  │       │  └───────┬───────┘  │       │  └───────┬───────┘  │
    │          │          │       │          │          │       │          │          │
    │  ┌───────┴───────┐  │       │  ┌───────┴───────┐  │       │  ┌───────┴───────┐  │
    │  │    Aurora     │<─┼───────┼─>│    Aurora     │<─┼───────┼─>│    Aurora     │  │
    │  │   Global DB   │  │       │  │   Replica     │  │       │  │   Replica     │  │
    │  └───────────────┘  │       │  └───────────────┘  │       │  └───────────────┘  │
    │                     │       │                     │       │                     │
    │  ┌───────────────┐  │       │  ┌───────────────┐  │       │  ┌───────────────┐  │
    │  │ ElastiCache   │  │       │  │ ElastiCache   │  │       │  │ ElastiCache   │  │
    │  │   (Redis)     │  │       │  │   (Redis)     │  │       │  │   (Redis)     │  │
    │  └───────────────┘  │       │  └───────────────┘  │       │  └───────────────┘  │
    │                     │       │                     │       │                     │
    └─────────────────────┘       └─────────────────────┘       └─────────────────────┘
```

### Architecture Catalog

| Architecture | Use Case | Documentation | Terraform |
|--------------|----------|---------------|-----------|
| [3-Tier Web Application](cloud/aws/3-tier-webapp.md) | Standard web apps | [Docs](cloud/aws/3-tier-webapp.md) | [Code](terraform/aws/3-tier/) |
| [Serverless API](cloud/aws/serverless-api.md) | Event-driven APIs | [Docs](cloud/aws/serverless-api.md) | [Code](terraform/aws/serverless/) |
| [Event-Driven Microservices](cloud/aws/event-driven.md) | Decoupled services | [Docs](cloud/aws/event-driven.md) | [Code](terraform/aws/event-driven/) |
| [Data Lake](cloud/aws/data-lake.md) | Analytics platform | [Docs](cloud/aws/data-lake.md) | [Code](terraform/aws/data-lake/) |
| [Multi-Region Active-Active](cloud/aws/multi-region.md) | Global availability | [Docs](cloud/aws/multi-region.md) | [Code](terraform/aws/multi-region/) |
| [Kubernetes Platform](cloud/aws/eks-platform.md) | Container orchestration | [Docs](cloud/aws/eks-platform.md) | [Code](terraform/aws/eks/) |
| [ML Inference Pipeline](cloud/aws/ml-inference.md) | Real-time predictions | [Docs](cloud/aws/ml-inference.md) | [Code](terraform/aws/ml-inference/) |

---

## Data Engineering

### Database Selection Matrix

```
                            CONSISTENCY
                     Strong ◄─────────────► Eventual

              │
    Write     │   ┌─────────────────┐     ┌─────────────────┐
    Heavy     │   │   CockroachDB   │     │    Cassandra    │
              │   │   TiDB, Spanner │     │    ScyllaDB     │
              │   └─────────────────┘     └─────────────────┘
              │
    T         │
    H         │
    R         │   ┌─────────────────┐     ┌─────────────────┐
    O         │   │   PostgreSQL    │     │    MongoDB      │
    U         │   │   MySQL         │     │    DynamoDB     │
    G         │   └─────────────────┘     └─────────────────┘
    H         │
    P         │
    U         │   ┌─────────────────┐     ┌─────────────────┐
    T         │   │   SQLite        │     │    Redis        │
              │   │   (Embedded)    │     │    Memcached    │
    Read      │   └─────────────────┘     └─────────────────┘
    Heavy     │
              │
```

### Data Pipeline Architecture

```
    DATA SOURCES                      INGESTION                   PROCESSING
    ─────────────                    ──────────                   ───────────

    ┌─────────────┐
    │ Application │──┐
    │    Logs     │  │
    └─────────────┘  │
                     │       ┌─────────────┐       ┌─────────────┐
    ┌─────────────┐  │       │             │       │             │
    │  Database   │──┼──────>│   Kafka     │──────>│   Flink     │
    │    CDC      │  │       │  Connect    │       │   Spark     │
    └─────────────┘  │       │             │       │             │
                     │       └─────────────┘       └──────┬──────┘
    ┌─────────────┐  │                                    │
    │   API       │──┘                                    │
    │  Events     │                                       │
    └─────────────┘                                       │
                                                          │
                           ┌──────────────────────────────┴──────────────────────────────┐
                           │                              │                              │
                           v                              v                              v
                    ┌─────────────┐              ┌─────────────┐              ┌─────────────┐
                    │     S3      │              │ ClickHouse  │              │Elasticsearch│
                    │ (Data Lake) │              │   (OLAP)    │              │  (Search)   │
                    └─────────────┘              └─────────────┘              └─────────────┘

                                                 CONSUMPTION
                                                 ───────────

                    ┌─────────────┐              ┌─────────────┐              ┌─────────────┐
                    │   Athena    │              │   Grafana   │              │   Kibana    │
                    │   Presto    │              │   Superset  │              │             │
                    └─────────────┘              └─────────────┘              └─────────────┘
```

---

## Observability & Reliability

### Metrics That Matter

| Category | Metric | Target | Alert Threshold |
|----------|--------|--------|-----------------|
| Latency | P50 | < 50ms | - |
| Latency | P99 | < 200ms | > 500ms |
| Latency | P99.9 | < 1s | > 2s |
| Availability | Uptime | 99.99% | < 99.9% |
| Error Rate | 5xx | < 0.1% | > 1% |
| Saturation | CPU | < 70% | > 85% |
| Saturation | Memory | < 80% | > 90% |
| Saturation | Disk I/O | < 70% | > 85% |

### SLO/SLI Framework

```
    SERVICE LEVEL OBJECTIVES
    ────────────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │  SLI: Request Latency                                                   │
    │  ───────────────────                                                    │
    │  Definition: Time from request received to response sent                │
    │  Measurement: Histogram at load balancer                                │
    │                                                                         │
    │  SLO: 99% of requests complete in < 200ms                              │
    │  ───                                                                    │
    │  Error Budget: 1% (432 minutes/month)                                   │
    │                                                                         │
    │  Current Status:                                                        │
    │  ┌────────────────────────────────────────────────────────────┐        │
    │  │██████████████████████████████████████████████░░░░░░░░░░░░░░│ 78%    │
    │  └────────────────────────────────────────────────────────────┘        │
    │  Error Budget Remaining: 95 minutes                                     │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure as Code

### Terraform Module Structure

```
terraform/
├── modules/
│   ├── networking/
│   │   ├── vpc/
│   │   ├── subnets/
│   │   └── security-groups/
│   │
│   ├── compute/
│   │   ├── eks/
│   │   ├── ecs/
│   │   └── lambda/
│   │
│   ├── database/
│   │   ├── aurora/
│   │   ├── dynamodb/
│   │   └── elasticache/
│   │
│   └── observability/
│       ├── cloudwatch/
│       ├── prometheus/
│       └── grafana/
│
├── environments/
│   ├── dev/
│   ├── staging/
│   └── production/
│
└── examples/
    ├── 3-tier-webapp/
    ├── serverless-api/
    └── data-lake/
```

---

## Performance Engineering

### Latency Reference

| Operation | Time | Notes |
|-----------|------|-------|
| L1 cache reference | 0.5 ns | |
| Branch mispredict | 5 ns | |
| L2 cache reference | 7 ns | 14x L1 cache |
| Mutex lock/unlock | 25 ns | |
| Main memory reference | 100 ns | 20x L2 cache |
| Compress 1KB with Snappy | 3,000 ns | |
| Send 1KB over 1 Gbps | 10,000 ns | |
| Read 4KB randomly from SSD | 150,000 ns | 150 us |
| Read 1MB sequentially from memory | 250,000 ns | 250 us |
| Round trip within datacenter | 500,000 ns | 500 us |
| Read 1MB sequentially from SSD | 1,000,000 ns | 1 ms |
| Disk seek | 10,000,000 ns | 10 ms |
| Read 1MB sequentially from disk | 20,000,000 ns | 20 ms |
| Send packet CA to Netherlands | 150,000,000 ns | 150 ms |

### Capacity Estimation Formulas

```
Daily Active Users (DAU) = MAU * 0.2

Requests per Second = (DAU * avg_requests_per_user) / 86400

Peak RPS = Average RPS * 3 (rule of thumb)

Storage per Year = users * data_per_user * 365 * replication_factor

Bandwidth = RPS * average_response_size

Servers Required = Peak RPS / RPS_per_server * (1 + redundancy_factor)

Cache Size = Working Set * (1 / cache_hit_ratio - 1)
```

---

## Contributing

Contributions are welcome. Please review the [contribution guidelines](CONTRIBUTING.md) before submitting.

1. Fork the repository
2. Create a feature branch
3. Make your changes with appropriate documentation
4. Submit a pull request with a clear description

---

## License

MIT License - see [LICENSE](LICENSE) for details.

---

Maintained by [Richard](https://github.com/Richardmsbr)
