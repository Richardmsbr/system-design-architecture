# Rate Limiter

## Overview

A rate limiter controls the rate of requests that clients can make to an API. This is essential for protecting services from abuse, ensuring fair usage, and maintaining system stability under load.

---

## Requirements

### Functional
- Limit requests per client (by IP, API key, or user ID)
- Support different rate limits for different endpoints
- Return appropriate error codes when limit exceeded
- Configurable time windows and request counts

### Non-Functional
- Low latency (sub-millisecond overhead)
- High availability
- Distributed (work across multiple servers)
- Accurate (no false positives/negatives)

---

## Algorithms

### 1. Token Bucket

```
    CONFIGURATION
    ─────────────
    Bucket Size: 10 tokens
    Refill Rate: 2 tokens/second

    TIME 0                          TIME 5s (after 3 requests)
    ──────                          ────────────────────────────

    ┌───────────────────┐           ┌───────────────────┐
    │ ●  ●  ●  ●  ●     │           │ ●  ●  ●  ●  ●     │
    │ ●  ●  ●  ●  ●     │    ───>   │ ●  ●  ○  ○  ○     │
    └───────────────────┘           └───────────────────┘
         10 tokens                    7 tokens remaining
                                     (3 used, 0 refilled yet)

    ● = available token
    ○ = consumed token
```

Implementation:

```python
class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()

    def allow_request(self, tokens: int = 1) -> bool:
        self._refill()

        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
```

### 2. Sliding Window Log

```
    WINDOW: 1 minute
    LIMIT: 5 requests

    Timeline (requests marked with X)
    ─────────────────────────────────────────────────────────────────────

    00:00    00:15    00:30    00:45    01:00    01:15    01:30
      │        │        │        │        │        │        │
      X        X        X                 X        X
      │        │        │                 │        │
      │        │        │                 │        │
      └────────┴────────┴─────────────────┴────────┴────────┘
              ▲                                    ▲
              │                                    │
         Window at 00:45                    Window at 01:15
         Count: 3 (ALLOW)                   Count: 4 (ALLOW)

    At each request, count entries in [now - window, now]
```

Implementation:

```python
class SlidingWindowLog:
    def __init__(self, window_size: int, max_requests: int):
        self.window_size = window_size  # seconds
        self.max_requests = max_requests
        self.requests = []  # sorted timestamps

    def allow_request(self) -> bool:
        now = time.time()
        window_start = now - self.window_size

        # Remove expired entries
        self.requests = [ts for ts in self.requests if ts > window_start]

        if len(self.requests) < self.max_requests:
            self.requests.append(now)
            return True
        return False
```

### 3. Sliding Window Counter

Combines fixed window efficiency with sliding window accuracy.

```
    WINDOW: 1 minute
    LIMIT: 100 requests

    Previous Window          Current Window
    (00:00 - 01:00)         (01:00 - 02:00)
    ┌─────────────────┐     ┌─────────────────┐
    │   84 requests   │     │   36 requests   │
    └─────────────────┘     └─────────────────┘
                      ▲
                      │
               Current time: 01:15
               (25% into current window)

    Weighted count = 84 * 0.75 + 36 * 0.25 = 63 + 9 = 72
    72 < 100 → ALLOW
```

Implementation:

```python
class SlidingWindowCounter:
    def __init__(self, window_size: int, max_requests: int):
        self.window_size = window_size
        self.max_requests = max_requests
        self.current_window_start = 0
        self.current_count = 0
        self.previous_count = 0

    def allow_request(self) -> bool:
        now = time.time()
        current_window = int(now // self.window_size)

        if current_window != self.current_window_start:
            self.previous_count = self.current_count
            self.current_count = 0
            self.current_window_start = current_window

        # Calculate weighted count
        window_elapsed = (now % self.window_size) / self.window_size
        weighted_count = (
            self.previous_count * (1 - window_elapsed) +
            self.current_count
        )

        if weighted_count < self.max_requests:
            self.current_count += 1
            return True
        return False
```

### Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling |
|-----------|--------|----------|----------------|
| Token Bucket | O(1) | High | Allows controlled bursts |
| Sliding Window Log | O(N) | Exact | No bursts |
| Sliding Window Counter | O(1) | Approximate | Smoothed |
| Fixed Window | O(1) | Low | Edge case issues |
| Leaky Bucket | O(1) | High | Strict rate |

---

## Distributed Rate Limiting

### Architecture

```
                    ┌─────────────────────────────────────────────────────┐
                    │                    CLIENTS                          │
                    └─────────────────────────────────────────────────────┘
                                            │
                                            v
                    ┌─────────────────────────────────────────────────────┐
                    │                  API GATEWAY                         │
                    │            (Rate Limit Check)                        │
                    └─────────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    │                       │                       │
                    v                       v                       v
            ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
            │   Server 1    │       │   Server 2    │       │   Server N    │
            └───────────────┘       └───────────────┘       └───────────────┘
                    │                       │                       │
                    └───────────────────────┼───────────────────────┘
                                            │
                                            v
                    ┌─────────────────────────────────────────────────────┐
                    │              REDIS CLUSTER                           │
                    │         (Centralized Counter Store)                  │
                    │                                                      │
                    │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
                    │  │ Node 1  │  │ Node 2  │  │ Node 3  │             │
                    │  │(Primary)│  │(Primary)│  │(Primary)│             │
                    │  └────┬────┘  └────┬────┘  └────┬────┘             │
                    │       │            │            │                   │
                    │  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐             │
                    │  │ Replica │  │ Replica │  │ Replica │             │
                    │  └─────────┘  └─────────┘  └─────────┘             │
                    │                                                      │
                    └─────────────────────────────────────────────────────┘
```

### Redis Implementation

```lua
-- rate_limiter.lua
-- Token Bucket implementation in Redis

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Calculate tokens to add
local elapsed = now - last_refill
local tokens_to_add = elapsed * refill_rate
tokens = math.min(capacity, tokens + tokens_to_add)

local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

-- Update bucket state
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) * 2)

return allowed
```

Python client:

```python
class DistributedRateLimiter:
    def __init__(self, redis_client, capacity: int, refill_rate: float):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.script = self._load_script()

    def _load_script(self):
        with open('rate_limiter.lua') as f:
            return self.redis.register_script(f.read())

    def allow_request(self, client_id: str, tokens: int = 1) -> bool:
        key = f"rate_limit:{client_id}"
        now = time.time()

        result = self.script(
            keys=[key],
            args=[self.capacity, self.refill_rate, tokens, now]
        )
        return bool(result)
```

---

## Rate Limit Rules Engine

### Rule Configuration

```yaml
rate_limits:
  # Global default
  default:
    requests: 100
    window: 60  # seconds
    algorithm: sliding_window_counter

  # By endpoint
  endpoints:
    /api/v1/search:
      requests: 30
      window: 60
      algorithm: token_bucket
      burst: 10

    /api/v1/upload:
      requests: 10
      window: 3600
      algorithm: sliding_window_log

  # By client tier
  tiers:
    free:
      requests: 100
      window: 3600
    premium:
      requests: 10000
      window: 3600
    enterprise:
      requests: 100000
      window: 3600
```

### Rule Evaluation Order

```
    REQUEST
       │
       v
    ┌──────────────────┐
    │ 1. Check IP ban  │──── Banned ──> 403 Forbidden
    │    list          │
    └────────┬─────────┘
             │ OK
             v
    ┌──────────────────┐
    │ 2. Check global  │──── Exceeded ──> 429 Too Many Requests
    │    rate limit    │
    └────────┬─────────┘
             │ OK
             v
    ┌──────────────────┐
    │ 3. Check tier    │──── Exceeded ──> 429 Too Many Requests
    │    rate limit    │
    └────────┬─────────┘
             │ OK
             v
    ┌──────────────────┐
    │ 4. Check endpoint│──── Exceeded ──> 429 Too Many Requests
    │    rate limit    │
    └────────┬─────────┘
             │ OK
             v
    ┌──────────────────┐
    │ 5. Process       │
    │    request       │
    └──────────────────┘
```

---

## Response Headers

Standard headers for rate limit information:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1640995200
Retry-After: 30
```

When limit exceeded:

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995200
Retry-After: 30

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry after 30 seconds.",
  "retry_after": 30
}
```

---

## High Availability Considerations

### Local + Remote Fallback

```
    REQUEST
       │
       v
    ┌──────────────────────────────────────────────────────────────┐
    │                      RATE LIMITER                            │
    │                                                              │
    │  ┌─────────────────┐         ┌─────────────────┐            │
    │  │   Local Cache   │ ──X──>  │  Redis Cluster  │            │
    │  │  (In-Memory)    │         │  (Source of     │            │
    │  │                 │         │   Truth)        │            │
    │  └────────┬────────┘         └─────────────────┘            │
    │           │                                                  │
    │           │ Redis unavailable?                               │
    │           │                                                  │
    │           v                                                  │
    │  ┌─────────────────┐                                        │
    │  │ Fallback to     │                                        │
    │  │ conservative    │                                        │
    │  │ local limits    │                                        │
    │  └─────────────────┘                                        │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Synchronization Strategy

```python
class HybridRateLimiter:
    def __init__(self, redis_client, local_capacity: int, sync_interval: float):
        self.redis = redis_client
        self.local = TokenBucket(local_capacity, local_capacity / 60)
        self.sync_interval = sync_interval
        self.last_sync = 0

    async def allow_request(self, client_id: str) -> bool:
        now = time.time()

        # Try Redis first
        try:
            if now - self.last_sync > self.sync_interval:
                result = await self._check_redis(client_id)
                self.last_sync = now
                return result
        except RedisError:
            pass  # Fallback to local

        # Local fallback (more conservative)
        return self.local.allow_request()

    async def _check_redis(self, client_id: str) -> bool:
        # Distributed check
        return await self.distributed_limiter.allow_request(client_id)
```

---

## Monitoring

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| rate_limit_requests_total | Total requests checked | - |
| rate_limit_exceeded_total | Requests rejected | > 5% of total |
| rate_limit_latency_ms | Check latency | P99 > 5ms |
| redis_connection_errors | Redis failures | > 0 |

### Dashboard Queries (Prometheus)

```promql
# Rate limit hit rate
sum(rate(rate_limit_exceeded_total[5m])) /
sum(rate(rate_limit_requests_total[5m])) * 100

# Top rate-limited clients
topk(10,
  sum by (client_id) (
    rate(rate_limit_exceeded_total[1h])
  )
)

# P99 latency
histogram_quantile(0.99,
  rate(rate_limit_latency_bucket[5m])
)
```

---

## Interview Discussion Points

1. Why not use a simple fixed window counter?
   - Edge case: 100 requests at 0:59 + 100 at 1:01 = 200 in 2 seconds

2. How to handle distributed clock skew?
   - Use Redis server time, not client time
   - Tolerate small drift with buffer

3. What if Redis goes down?
   - Local fallback with conservative limits
   - Circuit breaker pattern

4. How to prevent gaming the system?
   - Rate limit by multiple dimensions (IP + user + fingerprint)
   - Implement exponential backoff on repeated violations
