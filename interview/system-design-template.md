# System Design Interview Template

A structured framework for approaching any system design interview question.

---

## Interview Structure (45-60 minutes)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM DESIGN INTERVIEW TIMELINE                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  0-5 min    │  Requirements Clarification                              │
│             │  - Ask questions, don't assume                           │
│             │                                                           │
│  5-10 min   │  Capacity Estimation                                     │
│             │  - Back-of-envelope calculations                         │
│             │                                                           │
│  10-25 min  │  High-Level Design                                       │
│             │  - Draw the architecture                                 │
│             │  - Explain component interactions                        │
│             │                                                           │
│  25-40 min  │  Deep Dive                                               │
│             │  - Focus on 2-3 critical components                      │
│             │  - Discuss algorithms and data structures                │
│             │                                                           │
│  40-50 min  │  Scaling and Reliability                                 │
│             │  - Bottlenecks and solutions                             │
│             │  - Failure scenarios                                     │
│             │                                                           │
│  50-60 min  │  Wrap-up                                                 │
│             │  - Summarize trade-offs                                  │
│             │  - Answer follow-up questions                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Requirements Clarification

### Questions to Ask

```
FUNCTIONAL REQUIREMENTS
───────────────────────
- What are the core features?
- Who are the users? (B2B, B2C, internal)
- What operations are most frequent?
- Are there different user roles?
- Do we need real-time features?
- Is there a mobile app?


NON-FUNCTIONAL REQUIREMENTS
───────────────────────────
- What's the expected scale? (users, requests)
- What's the acceptable latency? (P50, P99)
- What availability is required? (99.9%, 99.99%)
- What consistency model? (strong, eventual)
- Are there compliance requirements? (GDPR, HIPAA)
- Is data retention specified?


CONSTRAINTS
───────────
- Budget constraints?
- Existing technology stack?
- Team expertise?
- Timeline?
```

### Template Response

```
"Before I start designing, I'd like to clarify a few things:

1. Scale: How many users are we expecting? Daily active users?
2. Features: Which features are must-haves for MVP?
3. Latency: What's our target latency for the main operations?
4. Availability: What's our uptime target?
5. Consistency: Is eventual consistency acceptable or do we need strong consistency?

Based on your answers, I'll focus on [specific area]."
```

---

## Step 2: Capacity Estimation

### Estimation Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CAPACITY ESTIMATION                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TRAFFIC                                                                │
│  ───────                                                                │
│  DAU = MAU × 0.2                                                       │
│  QPS = DAU × actions_per_user / 86,400                                 │
│  Peak QPS = Avg QPS × 2-3                                              │
│                                                                         │
│  Example: 100M MAU                                                      │
│  - DAU: 20M                                                            │
│  - Actions/user: 10                                                    │
│  - QPS: 20M × 10 / 86,400 ≈ 2,300                                     │
│  - Peak: ~7,000 QPS                                                    │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STORAGE                                                                │
│  ───────                                                                │
│  Daily = records_per_day × record_size                                 │
│  Yearly = Daily × 365 × replication_factor                             │
│                                                                         │
│  Example: 10M posts/day, 10KB each                                     │
│  - Daily: 10M × 10KB = 100GB                                          │
│  - Yearly: 100GB × 365 × 3 = 110TB                                    │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BANDWIDTH                                                              │
│  ─────────                                                              │
│  Bandwidth = QPS × response_size                                       │
│                                                                         │
│  Example: 5,000 QPS, 100KB responses                                   │
│  - Bandwidth: 5,000 × 100KB = 500 MB/s = 4 Gbps                       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  MEMORY (Cache)                                                         │
│  ─────────────                                                          │
│  Cache 20% of daily data (80-20 rule)                                  │
│                                                                         │
│  Example: 100M daily unique items, 1KB each                            │
│  - Cache: 100M × 0.2 × 1KB = 20GB                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Verbalization Template

```
"Let me do some quick calculations:

Users:
- 100 million monthly users
- About 20 million daily active users

Traffic:
- Each user does about 10 actions per day
- That's 200 million actions per day
- Or about 2,300 requests per second average
- Peak could be 2-3x, so let's design for 7,000 QPS

Storage:
- Each record is about 1KB
- 200 million per day = 200GB per day
- With 3x replication, that's 600GB per day
- Over 5 years: about 1 PB

This tells me we need:
- Horizontally scalable system
- Distributed storage
- Caching layer for hot data"
```

---

## Step 3: High-Level Design

### Drawing the Architecture

```
BASIC TEMPLATE
──────────────

                              ┌─────────────┐
                              │   Clients   │
                              │ (Web/Mobile)│
                              └──────┬──────┘
                                     │
                              ┌──────┴──────┐
                              │     CDN     │
                              └──────┬──────┘
                                     │
                              ┌──────┴──────┐
                              │    Load     │
                              │  Balancer   │
                              └──────┬──────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
       ┌──────┴──────┐        ┌──────┴──────┐        ┌──────┴──────┐
       │   Service   │        │   Service   │        │   Service   │
       │      A      │        │      B      │        │      C      │
       └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
              │                      │                      │
              └──────────────────────┼──────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
       ┌──────┴──────┐        ┌──────┴──────┐        ┌──────┴──────┐
       │    Cache    │        │  Database   │        │   Queue     │
       │   (Redis)   │        │ (Primary)   │        │  (Kafka)    │
       └─────────────┘        └──────┬──────┘        └─────────────┘
                                     │
                              ┌──────┴──────┐
                              │  Database   │
                              │  (Replica)  │
                              └─────────────┘
```

### Component Checklist

```
LAYER BY LAYER
──────────────

1. Client Layer
   □ Web application
   □ Mobile apps (iOS, Android)
   □ API clients (third-party)

2. Edge Layer
   □ CDN for static content
   □ DNS load balancing

3. Gateway Layer
   □ API Gateway
   □ Authentication/Authorization
   □ Rate limiting
   □ Request routing

4. Service Layer
   □ Core services (monolith or microservices)
   □ Background workers
   □ Scheduled jobs

5. Data Layer
   □ Primary database
   □ Read replicas
   □ Cache layer
   □ Search index
   □ Object storage

6. Infrastructure
   □ Message queue
   □ Service discovery
   □ Configuration management
   □ Logging/Monitoring
```

---

## Step 4: Deep Dive

### Component Selection Guide

```
WHEN TO GO DEEP
───────────────

Pick components that are:
1. Critical to the core use case
2. Have interesting technical challenges
3. Showcase your expertise

Examples by system type:

URL Shortener    → Key generation, redirect service
Twitter          → Fan-out strategy, timeline service
YouTube          → Video processing, recommendation engine
Uber             → Geospatial indexing, matching algorithm
WhatsApp         → Message delivery, presence service
Dropbox          → File sync, conflict resolution
```

### Deep Dive Framework

```
FOR EACH COMPONENT
──────────────────

1. API Design
   - What endpoints/methods?
   - Request/response format?
   - Error handling?

2. Data Model
   - Schema design
   - Indexes needed
   - Sharding key?

3. Algorithm
   - Core logic
   - Time/space complexity
   - Edge cases

4. Technology Choice
   - Why this technology?
   - Trade-offs considered?
   - Alternatives?

5. Failure Modes
   - What can go wrong?
   - How do we detect it?
   - How do we recover?
```

---

## Step 5: Scaling Discussion

### Scaling Strategies

```
HORIZONTAL SCALING
──────────────────

Stateless Services:
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│Server 1 │  │Server 2 │  │Server 3 │  │Server N │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
     ▲            ▲            ▲            ▲
     └────────────┴────────────┴────────────┘
                        │
                 Load Balancer


DATABASE SCALING
────────────────

Read Scaling:
┌──────────────┐
│   Primary    │ ←── Writes
└──────┬───────┘
       │ Replication
┌──────┴───────┬───────────────┐
│   Replica 1  │   Replica 2   │   Replica N  │ ←── Reads
└──────────────┴───────────────┘


Write Scaling (Sharding):
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │ Shard N  │
│ (A-G)    │  │ (H-N)    │  │ (O-T)    │  │ (U-Z)    │
└──────────┘  └──────────┘  └──────────┘  └──────────┘


CACHING LAYERS
──────────────

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Browser │ -> │   CDN   │ -> │  Redis  │ -> │Database │
│  Cache  │    │  Cache  │    │  Cache  │    │         │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
   ~ms           ~10ms          ~1ms           ~10ms
```

### Bottleneck Analysis

```
IDENTIFY BOTTLENECKS
────────────────────

Ask yourself:
1. What's the slowest component?
2. What's the most loaded component?
3. What fails first under load?

Common bottlenecks:

Database
- Too many queries → Add caching
- Slow queries → Add indexes, optimize
- Write contention → Sharding

Network
- Too much data → Compression, pagination
- Too many calls → Batching, caching

CPU
- Computation heavy → More servers, async processing
- Serialization → Optimize format (protobuf vs JSON)

Memory
- Not enough cache → More RAM, distributed cache
- Memory leaks → Monitoring, restart policy
```

---

## Step 6: Reliability and Trade-offs

### Failure Scenarios

```
FAILURE ANALYSIS TEMPLATE
─────────────────────────

For each component, ask:
1. What happens if it fails completely?
2. What if it's slow/degraded?
3. What if it returns wrong data?

Component        | Failure Mode      | Mitigation
─────────────────────────────────────────────────────
Load Balancer    | Single point of   | Multiple LBs,
                 | failure           | DNS failover
─────────────────────────────────────────────────────
Database Primary | Unavailable       | Auto-failover to
                 |                   | replica
─────────────────────────────────────────────────────
Cache            | Data loss         | Fallback to DB,
                 |                   | cache warming
─────────────────────────────────────────────────────
External API     | Timeout           | Circuit breaker,
                 |                   | fallback response
```

### Trade-off Discussion

```
ARTICULATE TRADE-OFFS
─────────────────────

"There are several trade-offs to consider:

1. Consistency vs Availability
   - We chose eventual consistency because...
   - Strong consistency would require... but adds latency

2. Latency vs Throughput
   - We batch writes for throughput
   - Real-time would require... but reduces efficiency

3. Simplicity vs Scalability
   - We started with a monolith for simplicity
   - Microservices add complexity but scale better

4. Cost vs Performance
   - Caching reduces DB load but adds infrastructure
   - The ROI makes sense at our scale because..."
```

---

## Common Patterns Reference

### Pattern Cheat Sheet

```
CACHING
───────
Cache-aside     : App manages cache
Write-through   : Cache manages writes
Write-behind    : Async cache-to-DB

MESSAGING
─────────
Pub/Sub         : Many consumers
Queue           : One consumer
Event sourcing  : Append-only log

DATA
────
Sharding        : Horizontal split
Replication     : Read scaling
CQRS            : Separate read/write

RELIABILITY
───────────
Circuit breaker : Fail fast
Retry w/ backoff: Handle transients
Bulkhead        : Isolate failures
```

---

## Red Flags to Avoid

```
DON'T
─────

✗ Jump into solution without understanding requirements
✗ Design for Google scale when asked about a startup
✗ Ignore the interviewer's hints
✗ Memorize solutions without understanding
✗ Use buzzwords you can't explain
✗ Forget about failure scenarios
✗ Design something you can't implement

DO
──

✓ Ask clarifying questions
✓ Think out loud
✓ Justify every decision
✓ Discuss trade-offs
✓ Be honest about unknowns
✓ Engage with interviewer's questions
✓ Cover both breadth and depth
```

---

## Practice Checklist

```
PREPARATION
───────────

□ Can explain CAP theorem with examples
□ Know when to use SQL vs NoSQL
□ Understand caching strategies
□ Can do capacity estimation quickly
□ Know common architectural patterns
□ Can draw clean system diagrams
□ Practiced explaining out loud

INTERVIEW DAY
─────────────

□ Listen carefully to the question
□ Confirm understanding before designing
□ Start high-level, then go deep
□ Drive the conversation
□ Check in with interviewer
□ Leave time for questions
□ Summarize at the end
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM DESIGN QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TIME MANAGEMENT                                                        │
│  • Requirements: 5 min                                                  │
│  • Estimation: 5 min                                                    │
│  • High-level: 15 min                                                   │
│  • Deep dive: 15 min                                                    │
│  • Scaling: 10 min                                                      │
│                                                                         │
│  KEY NUMBERS                                                            │
│  • 1 day = 86,400 seconds                                              │
│  • 1 month = 2.5M seconds                                              │
│  • 1 year = 31.5M seconds                                              │
│  • 1 QPS = 86,400 requests/day                                         │
│                                                                         │
│  LATENCY                                                                │
│  • Memory: 100ns                                                        │
│  • SSD: 100μs                                                          │
│  • Network (datacenter): 500μs                                         │
│  • Disk: 10ms                                                          │
│  • Network (cross-region): 150ms                                       │
│                                                                         │
│  STORAGE                                                                │
│  • 1 char = 1 byte (ASCII)                                             │
│  • UUID = 16 bytes                                                      │
│  • Timestamp = 8 bytes                                                  │
│  • Average tweet = 500 bytes                                           │
│  • Average image = 200KB-2MB                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
