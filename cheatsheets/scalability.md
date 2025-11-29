# Scalability Cheatsheet

Quick reference for capacity estimation, performance benchmarks, and scaling patterns.

---

## Numbers Every Developer Should Know

### Latency Comparison

| Operation | Latency | Relative |
|-----------|---------|----------|
| L1 cache reference | 0.5 ns | 1x |
| Branch mispredict | 5 ns | 10x |
| L2 cache reference | 7 ns | 14x |
| Mutex lock/unlock | 25 ns | 50x |
| Main memory reference | 100 ns | 200x |
| Compress 1K bytes with Zippy | 3,000 ns (3 us) | 6,000x |
| Send 1K bytes over 1 Gbps network | 10,000 ns (10 us) | 20,000x |
| Read 4K randomly from SSD | 150,000 ns (150 us) | 300,000x |
| Read 1 MB sequentially from memory | 250,000 ns (250 us) | 500,000x |
| Round trip within same datacenter | 500,000 ns (500 us) | 1,000,000x |
| Read 1 MB sequentially from SSD | 1,000,000 ns (1 ms) | 2,000,000x |
| Disk seek | 10,000,000 ns (10 ms) | 20,000,000x |
| Read 1 MB sequentially from disk | 20,000,000 ns (20 ms) | 40,000,000x |
| Send packet CA -> Netherlands -> CA | 150,000,000 ns (150 ms) | 300,000,000x |

### Powers of Two

| Power | Exact Value | Approx | Bytes |
|-------|-------------|--------|-------|
| 10 | 1,024 | 1 thousand | 1 KB |
| 20 | 1,048,576 | 1 million | 1 MB |
| 30 | 1,073,741,824 | 1 billion | 1 GB |
| 40 | 1,099,511,627,776 | 1 trillion | 1 TB |
| 50 | 1,125,899,906,842,624 | 1 quadrillion | 1 PB |

### Time Conversions

| Unit | Seconds | Per Day | Per Month | Per Year |
|------|---------|---------|-----------|----------|
| 1 second | 1 | 86,400 | 2.6M | 31.5M |
| 1 minute | 60 | 1,440 | 43,200 | 525,600 |
| 1 hour | 3,600 | 24 | 720 | 8,760 |
| 1 day | 86,400 | 1 | 30 | 365 |

---

## Capacity Estimation Formulas

### Traffic

```
Daily Active Users (DAU) = Monthly Active Users (MAU) * 0.2

Requests per Second (RPS) = DAU * requests_per_user / 86,400

Peak RPS = Average RPS * 2 to 3 (rule of thumb)

Example:
- MAU: 100 million
- DAU: 20 million
- Requests/user/day: 10
- Average RPS: 20M * 10 / 86,400 = 2,315 RPS
- Peak RPS: ~7,000 RPS
```

### Storage

```
Storage per Day = daily_records * record_size

Storage per Year = Storage per Day * 365 * replication_factor

Example (Twitter-like):
- 500M tweets/day
- 500 bytes/tweet (metadata)
- Daily: 500M * 500 = 250 GB
- Yearly: 250 GB * 365 * 3 (replication) = 274 TB
```

### Bandwidth

```
Bandwidth = RPS * average_response_size

Example:
- RPS: 10,000
- Avg response: 100 KB
- Bandwidth: 10,000 * 100 KB = 1 GB/s = 8 Gbps
```

### Memory (Caching)

```
Cache Size = working_set_size / cache_hit_ratio

80-20 Rule: 20% of data serves 80% of requests

Example:
- Daily unique requests: 100 million
- Cache 20%: 20 million items
- Item size: 1 KB
- Cache needed: 20 GB
```

### Server Estimation

```
Servers = Peak_RPS / RPS_per_server * (1 + buffer)

Example:
- Peak RPS: 50,000
- RPS per server: 500 (with DB access)
- Buffer: 30%
- Servers: 50,000 / 500 * 1.3 = 130 servers
```

---

## QPS Reference by System Type

| System Type | Typical QPS per Server |
|-------------|------------------------|
| Simple API (cache hit) | 10,000 - 50,000 |
| API with DB read | 500 - 2,000 |
| API with DB write | 100 - 500 |
| CPU-intensive computation | 10 - 100 |
| ML inference (CPU) | 10 - 50 |
| ML inference (GPU) | 100 - 1,000 |

---

## Database Scaling

### Read vs Write Patterns

```
READ-HEAVY (Social Media, News)
──────────────────────────────

    ┌─────────────┐
    │   WRITES    │  1x
    └─────────────┘

    ┌─────────────────────────────────────────────────┐
    │                    READS                         │  100x
    └─────────────────────────────────────────────────┘

    Solution: Read replicas, caching


WRITE-HEAVY (IoT, Logging)
──────────────────────────

    ┌─────────────────────────────────────────────────┐
    │                    WRITES                        │  100x
    └─────────────────────────────────────────────────┘

    ┌─────────────┐
    │    READS    │  1x
    └─────────────┘

    Solution: Sharding, async processing, time-series DB


BALANCED (E-commerce, Banking)
──────────────────────────────

    ┌─────────────────────────────────┐
    │             WRITES               │  1x
    └─────────────────────────────────┘

    ┌─────────────────────────────────────────┐
    │                 READS                    │  3-10x
    └─────────────────────────────────────────┘

    Solution: Primary-replica, read/write splitting
```

### Sharding Strategies

```
RANGE-BASED
───────────
user_id 1-1M      → Shard 1
user_id 1M-2M     → Shard 2
user_id 2M-3M     → Shard 3

Pros: Range queries efficient
Cons: Hotspots if recent data accessed more


HASH-BASED
──────────
shard = hash(user_id) % num_shards

Pros: Even distribution
Cons: Range queries require scatter-gather


DIRECTORY-BASED
───────────────
Lookup table maps key → shard

Pros: Flexible, easy rebalancing
Cons: Single point of failure (the directory)


CONSISTENT HASHING
──────────────────
Keys and servers on same hash ring
Key assigned to next server clockwise

Pros: Minimal redistribution on scale
Cons: Complexity, virtual nodes needed
```

---

## Caching Strategies

### Cache Patterns

```
CACHE-ASIDE (Lazy Loading)
──────────────────────────

    Read:
    1. Check cache
    2. If miss, read from DB
    3. Write to cache
    4. Return data

    Write:
    1. Write to DB
    2. Invalidate cache

    Pros: Only cache what's needed
    Cons: Cache miss penalty, stale data possible


WRITE-THROUGH
─────────────

    Write:
    1. Write to cache
    2. Cache writes to DB
    3. Return success

    Pros: Cache always consistent
    Cons: Write latency, cache churn


WRITE-BEHIND (Write-Back)
─────────────────────────

    Write:
    1. Write to cache
    2. Return success immediately
    3. Async write to DB

    Pros: Fast writes
    Cons: Data loss risk, complexity
```

### Cache Eviction

| Policy | Description | Use Case |
|--------|-------------|----------|
| LRU | Least Recently Used | General purpose |
| LFU | Least Frequently Used | Popularity-based |
| FIFO | First In First Out | Time-series |
| TTL | Time To Live | Session data |
| Random | Random eviction | Simple, uniform access |

---

## Load Balancing Algorithms

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| Round Robin | Sequential distribution | Homogeneous servers |
| Weighted Round Robin | Based on server capacity | Heterogeneous servers |
| Least Connections | To server with fewest connections | Long-lived connections |
| IP Hash | Based on client IP | Session affinity |
| Consistent Hashing | Minimal redistribution | Caching layers |
| Random | Random selection | Simple, stateless |

---

## Availability Calculations

### Uptime Targets

| Availability | Downtime/Year | Downtime/Month | Downtime/Week |
|--------------|---------------|----------------|---------------|
| 99% | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% | 8.76 hours | 43.2 minutes | 10.1 minutes |
| 99.99% | 52.6 minutes | 4.32 minutes | 1.01 minutes |
| 99.999% | 5.26 minutes | 26 seconds | 6 seconds |

### Calculating System Availability

```
SERIAL COMPONENTS (all must work)
─────────────────────────────────
A_total = A1 * A2 * A3 * ...

Example: Web -> App -> DB
A = 0.999 * 0.999 * 0.999 = 0.997 (99.7%)


PARALLEL COMPONENTS (at least one works)
────────────────────────────────────────
A_total = 1 - (1-A1) * (1-A2) * ...

Example: 2 redundant servers
A = 1 - (1-0.99) * (1-0.99) = 0.9999 (99.99%)


COMBINED
────────
Active-active: 2 app servers, 1 DB

App availability = 1 - (0.01)^2 = 0.9999
DB availability = 0.999

Total = 0.9999 * 0.999 = 0.9989 (99.89%)
```

---

## Message Queue Sizing

```
Queue Size = (Producer Rate - Consumer Rate) * Max Delay Tolerance

Example:
- Producer: 10,000 msg/sec
- Consumer: 8,000 msg/sec
- Max delay: 1 hour

Queue Size = (10,000 - 8,000) * 3,600 = 7.2 million messages
```

---

## Connection Pool Sizing

```
PostgreSQL formula:
connections = ((core_count * 2) + effective_spindle_count)

Example (8 core, SSD):
connections = (8 * 2) + 1 = 17 per server

For web app:
pool_size = connections / num_app_servers
```

---

## Quick Estimation Tips

1. **Round to powers of 10** for quick math
2. **Assume 100K RPS** per commodity server with caching
3. **1 million QPS** needs ~10-20 servers minimum
4. **1 TB storage** = 1 million records of 1 MB each
5. **Network is usually bottleneck** before CPU
6. **Memory access is 100x faster** than SSD
7. **Cross-datacenter is 1000x slower** than within
8. **80/20 rule** applies to most data access patterns
9. **3x replication** is standard for durability
10. **30% buffer** for capacity planning

---

## References

- Jeff Dean's "Numbers Everyone Should Know"
- High Scalability Blog
- System Design Primer
- Google SRE Book
