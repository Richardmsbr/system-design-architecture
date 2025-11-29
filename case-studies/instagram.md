# Instagram Engineering Case Study

How Instagram scaled from startup to 2 billion users while maintaining engineering velocity.

---

## Company Overview

| Metric | Value |
|--------|-------|
| Monthly Active Users | 2+ billion |
| Daily Active Users | 500+ million |
| Photos uploaded daily | 100+ million |
| Stories viewed daily | 500+ million |
| Peak likes per second | 1+ million |
| Engineers (at scale) | ~2,000 |

---

## Architecture Evolution

### Phase 1: Startup (2010)

```
ORIGINAL STACK (13 employees, millions of users)
────────────────────────────────────────────────

    ┌─────────────┐
    │   Users     │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │   NGINX     │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │   Django    │  (Python)
    │   Gunicorn  │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │ PostgreSQL  │  (Single primary)
    └─────────────┘

Key Insight: Started simple, scaled later
```

### Phase 2: Rapid Growth (2011-2012)

```
SCALING CHALLENGES
──────────────────

1. Database bottleneck
   Solution: Read replicas + pgbouncer

2. Photo storage
   Solution: Amazon S3 + CloudFront

3. Feed generation
   Solution: Redis for timeline cache

4. Real-time updates
   Solution: Push notifications via APNS/GCM


ARCHITECTURE AT ACQUISITION (13 engineers, 30M users)
─────────────────────────────────────────────────────

    ┌─────────────────────────────────────────────────────────────┐
    │                        CLIENTS                              │
    └─────────────────────────────────────────────────────────────┘
                                │
                                ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                      AMAZON ELB                             │
    └─────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
    ┌─────────────────────┐     ┌─────────────────────┐
    │      NGINX          │     │      NGINX          │
    │   (Load Balancer)   │     │   (Load Balancer)   │
    └─────────┬───────────┘     └─────────┬───────────┘
              │                           │
              └───────────┬───────────────┘
                          │
    ┌─────────────────────┴─────────────────────┐
    │              DJANGO APP SERVERS           │
    │           (25 EC2 instances)              │
    └─────────────────────┬─────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│  PostgreSQL   │ │    Redis      │ │  Memcached    │
│   Primary +   │ │   Cluster     │ │   Cluster     │
│   Replicas    │ │               │ │               │
└───────────────┘ └───────────────┘ └───────────────┘
        │
        ▼
┌───────────────┐
│   Amazon S3   │  (Photo storage)
│  + CloudFront │  (CDN)
└───────────────┘
```

### Phase 3: Facebook Scale (2013-Present)

```
CURRENT ARCHITECTURE (Simplified)
─────────────────────────────────

    ┌─────────────────────────────────────────────────────────────┐
    │                       MOBILE APPS                           │
    │                   (iOS, Android, Web)                       │
    └─────────────────────────────────────────────────────────────┘
                                │
                                ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                     FACEBOOK EDGE                           │
    │              (Global CDN + Edge Computing)                  │
    └─────────────────────────────────────────────────────────────┘
                                │
                                ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                      API GATEWAY                            │
    │             (Thrift + GraphQL Federation)                   │
    └─────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│    Feed       │       │    Media      │       │    Social     │
│   Service     │       │   Service     │       │   Graph       │
└───────────────┘       └───────────────┘       └───────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│    TAO        │       │   Haystack    │       │     TAO       │
│ (Graph Store) │       │(Photo Storage)│       │ (Graph Store) │
└───────────────┘       └───────────────┘       └───────────────┘
```

---

## Key Technical Decisions

### 1. Feed Ranking

```
FEED GENERATION PIPELINE
────────────────────────

┌─────────────────────────────────────────────────────────────────────────┐
│                         CANDIDATE GENERATION                            │
│                                                                         │
│   Sources:                                                              │
│   - Following feed (posts from accounts you follow)                    │
│   - Explore/Discover (recommended content)                             │
│   - Ads inventory                                                       │
│                                                                         │
│   Output: 500+ candidates                                               │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         FIRST PASS RANKING                              │
│                                                                         │
│   Lightweight model:                                                    │
│   - User engagement history                                            │
│   - Content freshness                                                   │
│   - Author relationship strength                                        │
│                                                                         │
│   Output: ~100 candidates                                               │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         FINAL RANKING                                   │
│                                                                         │
│   Deep neural network:                                                  │
│   - Predicted engagement (like, comment, share, save)                  │
│   - Value model (time well spent)                                      │
│   - Integrity signals (misinformation, policy violations)             │
│                                                                         │
│   Output: Ranked feed                                                   │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         POST-RANKING ADJUSTMENTS                        │
│                                                                         │
│   - Diversity injection (not all photos from same person)             │
│   - Format mixing (photos, reels, stories)                             │
│   - Ads insertion                                                       │
│   - Integrity demotion                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2. Photo Storage (Haystack)

```
THE PHOTO STORAGE PROBLEM
─────────────────────────

Challenge:
- 100M+ photos uploaded daily
- Each photo: 4 sizes (thumbnail, low, medium, original)
- Traditional storage: 1 file per photo = massive metadata overhead

Solution: Haystack
- Bundle multiple photos into large files ("haystack")
- Store metadata in memory
- Single disk read per photo


HAYSTACK ARCHITECTURE
─────────────────────

    Photo Request: photo_id=12345
           │
           ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                          HAYSTACK DIRECTORY                         │
    │                                                                     │
    │   Maps: photo_id → (store_id, volume_id, offset, size)            │
    │                                                                     │
    │   Lookup: O(1) in-memory hash table                                │
    └───────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │                          HAYSTACK STORE                             │
    │                                                                     │
    │   Volume File (100GB):                                             │
    │   ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐       │
    │   │Photo1│Photo2│Photo3│Photo4│Photo5│Photo6│Photo7│ ...  │       │
    │   └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘       │
    │         ▲                                                          │
    │         │                                                          │
    │   Direct seek to offset, single read                               │
    │                                                                     │
    └─────────────────────────────────────────────────────────────────────┘


PHOTO ENTRY FORMAT
──────────────────

┌────────────────────────────────────────────────────────────┐
│  Header (fixed)    │  Key      │  Data         │  Footer  │
│  - magic number    │  - photo  │  - JPEG       │  - CRC   │
│  - cookie          │    _id    │    bytes      │          │
│  - size            │           │               │          │
└────────────────────────────────────────────────────────────┘

Efficiency:
- 1 disk I/O per photo (vs. 3+ for filesystem)
- 40% metadata reduction
- Linear scaling with storage nodes
```

### 3. Django at Scale

```
PYTHON PERFORMANCE OPTIMIZATIONS
────────────────────────────────

1. Cython for hot paths
   - C extensions for critical code
   - 10-100x speedup for CPU-bound operations

2. Memory optimizations
   - __slots__ for frequently instantiated classes
   - Object pooling for request handling

3. Async where appropriate
   - Celery for background tasks
   - uvloop for async I/O

4. Connection pooling
   - pgbouncer for PostgreSQL
   - Connection pools for Redis/Memcached


SCALING DJANGO
──────────────

┌─────────────────────────────────────────────────────────────────────────┐
│                              DJANGO APP                                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         REQUEST PATH                             │   │
│  │                                                                   │   │
│  │   1. Middleware (auth, rate limit) ─────── ~1ms                  │   │
│  │   2. URL routing ───────────────────────── ~0.1ms                │   │
│  │   3. View function ─────────────────────── ~5-50ms               │   │
│  │   4. Database queries ──────────────────── ~10-100ms             │   │
│  │   5. Serialization ─────────────────────── ~1-5ms                │   │
│  │   6. Response ──────────────────────────── ~0.1ms                │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Key optimization: Database is always the bottleneck                   │
│                                                                         │
│  Solutions:                                                             │
│  - Read replicas                                                       │
│  - Query optimization (select_related, prefetch_related)              │
│  - Aggressive caching (Memcached)                                      │
│  - Denormalization where justified                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure Lessons

### 1. Keep It Simple

```
SIMPLICITY PRINCIPLES
─────────────────────

1. "Do the simplest thing that could possibly work"
   - Started with single PostgreSQL instance
   - Added complexity only when forced

2. Monolith first
   - Django monolith served billions of requests
   - Microservices adopted gradually, with clear boundaries

3. Boring technology
   - PostgreSQL over newer databases
   - Redis for caching (proven at scale)
   - Python despite "performance concerns"

4. Measure, then optimize
   - Profile before assuming bottlenecks
   - Data-driven decisions
```

### 2. Database Scaling Journey

```
POSTGRESQL SCALING PATH
───────────────────────

Stage 1: Vertical Scaling
- Bigger instance
- More RAM for caching
- Faster disks (SSD)

Stage 2: Read Replicas
- Primary for writes
- Replicas for reads (90% of queries)
- PgBouncer for connection pooling

Stage 3: Sharding
- User-based sharding
- Application-level routing
- Cross-shard queries minimized

Stage 4: Specialized Databases
- TAO for social graph (Facebook)
- Cassandra for feed storage
- Elasticsearch for search


CURRENT STATE
─────────────

    ┌─────────────────────────────────────────────────────────────────┐
    │                          WRITES                                 │
    │                                                                 │
    │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
    │  │  Shard 1  │  │  Shard 2  │  │  Shard 3  │  │  Shard N  │   │
    │  │ (Users    │  │ (Users    │  │ (Users    │  │ (Users    │   │
    │  │  A-D)     │  │  E-H)     │  │  I-L)     │  │  ...)     │   │
    │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘   │
    │        │              │              │              │          │
    └────────┼──────────────┼──────────────┼──────────────┼──────────┘
             │              │              │              │
             ▼              ▼              ▼              ▼
    ┌────────────────────────────────────────────────────────────────┐
    │                          READS                                  │
    │                                                                 │
    │   Each shard has 3+ read replicas                              │
    │   - Sync replication within region                             │
    │   - Async replication across regions                           │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### 3. Reliability Practices

```
RELIABILITY PHILOSOPHY
──────────────────────

1. Design for failure
   - Assume any component can fail
   - Graceful degradation over hard failure
   - Circuit breakers everywhere

2. Observability first
   - Metrics on everything
   - Distributed tracing
   - Error budgets (SLO-based)

3. Incremental rollouts
   - 0.1% → 1% → 10% → 50% → 100%
   - Automatic rollback on anomaly detection
   - Feature flags for everything

4. Chaos engineering
   - Regular failure injection
   - Game days for incident practice
   - Disaster recovery testing
```

---

## Key Metrics

| Metric | Target | Actual |
|--------|--------|--------|
| Feed latency P99 | < 500ms | ~300ms |
| Photo upload success | > 99.9% | 99.95% |
| App crash rate | < 0.1% | 0.05% |
| API availability | 99.99% | 99.995% |

---

## Engineering Culture

```
CORE PRINCIPLES
───────────────

1. Move fast
   - Continuous deployment (100+ deploys/day)
   - Minimal process, maximum ownership
   - "If it hurts, do it more often"

2. Data-driven decisions
   - A/B test everything
   - Metrics over opinions
   - Kill features that don't move metrics

3. Owner mentality
   - Engineers own services end-to-end
   - On-call for what you build
   - No "throw it over the wall"

4. Continuous learning
   - Post-mortems without blame
   - Knowledge sharing (tech talks, docs)
   - 20% time for exploration
```

---

## Key Takeaways

1. **Start simple** - Instagram handled 30M users with 13 engineers
2. **Scale what breaks** - Don't pre-optimize
3. **Boring technology works** - PostgreSQL, Python, Redis
4. **Caching is crucial** - Multiple caching layers
5. **Measure everything** - Data-driven decisions
6. **Own your infrastructure** - Custom solutions where needed (Haystack)
7. **Culture matters** - Fast iteration, ownership, learning

---

## References

- Instagram Engineering Blog
- "Scaling Instagram Infrastructure" (InfoQ)
- "How Instagram Moved to Python 3" (PyCon)
- "Building a Photo Storage System" (Facebook Engineering)
