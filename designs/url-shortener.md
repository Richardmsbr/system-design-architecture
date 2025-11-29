# URL Shortener (bit.ly, TinyURL)

> 🎯 **Difficulty**: Easy | ⏱️ **Interview Time**: 35-45 min

## 1. Requirements Clarification

### Functional Requirements
- Given a long URL, generate a shorter unique alias
- When user accesses short URL, redirect to original URL
- Users can optionally pick custom short URL
- Links expire after default timespan (configurable)

### Non-Functional Requirements
- High availability (system should always be up)
- Low latency (URL redirection in real-time)
- Short URLs should not be predictable

### Extended Requirements
- Analytics (click count, location, referrer)
- REST API for developers

---

## 2. Capacity Estimation

### Traffic Estimates
```
Assumptions:
- 500M new URLs per month
- Read:Write ratio = 100:1
- 50B redirects per month

Calculations:
- Writes: 500M / (30 × 24 × 3600) ≈ 200 URLs/sec
- Reads: 200 × 100 = 20,000 URLs/sec
```

### Storage Estimates
```
- Each URL entry ≈ 500 bytes
- 500M × 500 bytes = 250 GB/month
- 5 years = 250 GB × 60 = 15 TB
```

### Bandwidth Estimates
```
- Incoming: 200 × 500 bytes = 100 KB/sec
- Outgoing: 20,000 × 500 bytes = 10 MB/sec
```

### Memory Estimates (Caching)
```
- Cache 20% of daily URLs (80-20 rule)
- Daily requests: 20,000 × 86,400 = 1.7B
- Unique URLs (20%): 340M × 500 bytes = 170 GB
```

---

## 3. System API Design

```http
POST /api/v1/shorten
Content-Type: application/json
Authorization: Bearer {api_key}

{
  "long_url": "https://example.com/very/long/path",
  "custom_alias": "my-link",      // optional
  "expiration_date": "2025-12-31" // optional
}

Response: 201 Created
{
  "short_url": "https://short.ly/abc123",
  "long_url": "https://example.com/very/long/path",
  "expires_at": "2025-12-31T00:00:00Z"
}
```

```http
GET /{short_code}

Response: 301 Moved Permanently
Location: https://example.com/very/long/path
```

```http
DELETE /api/v1/urls/{short_code}
Authorization: Bearer {api_key}

Response: 204 No Content
```

---

## 4. Database Design

### Schema

```sql
-- URLs Table
CREATE TABLE urls (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0,

    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);

-- Users Table
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    api_key VARCHAR(64) UNIQUE,
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    rate_limit INT DEFAULT 1000
);

-- Analytics Table (Optional)
CREATE TABLE clicks (
    id BIGINT PRIMARY KEY,
    url_id BIGINT REFERENCES urls(id),
    clicked_at TIMESTAMP DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT,
    referrer TEXT,
    country VARCHAR(2)
);
```

### Database Choice

| Option | Pros | Cons |
|--------|------|------|
| **PostgreSQL** | ACID, rich features | Scaling complexity |
| **DynamoDB** | Managed, scalable | Eventual consistency |
| **Cassandra** | High write throughput | Complex queries |

**Recommendation**: Start with PostgreSQL, migrate to DynamoDB for scale.

---

## 5. High-Level Design

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│    CDN      │────▶│   Load      │
│  (Browser)  │     │ (CloudFront)│     │  Balancer   │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                    ┌──────────────────────────┼──────────────────────────┐
                    │                          │                          │
             ┌──────▼──────┐           ┌──────▼──────┐           ┌──────▼──────┐
             │   App       │           │   App       │           │   App       │
             │  Server 1   │           │  Server 2   │           │  Server N   │
             └──────┬──────┘           └──────┬──────┘           └──────┬──────┘
                    │                          │                          │
                    └──────────────────────────┼──────────────────────────┘
                                               │
                              ┌────────────────┼────────────────┐
                              │                │                │
                       ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
                       │   Redis     │  │  Database   │  │   Key       │
                       │   Cache     │  │  (Primary)  │  │  Generator  │
                       └─────────────┘  └──────┬──────┘  └─────────────┘
                                               │
                                        ┌──────▼──────┐
                                        │  Database   │
                                        │  (Replica)  │
                                        └─────────────┘
```

---

## 6. Short URL Generation

### Option 1: Base62 Encoding

```python
import string

ALPHABET = string.digits + string.ascii_letters  # 0-9, a-z, A-Z

def encode_base62(num: int) -> str:
    if num == 0:
        return ALPHABET[0]

    result = []
    while num:
        num, remainder = divmod(num, 62)
        result.append(ALPHABET[remainder])
    return ''.join(reversed(result))

def decode_base62(s: str) -> int:
    num = 0
    for char in s:
        num = num * 62 + ALPHABET.index(char)
    return num

# Example: encode_base62(123456789) = "8M0kX"
```

### Option 2: MD5 Hash + Base62

```python
import hashlib

def generate_short_code(long_url: str, length: int = 7) -> str:
    hash_object = hashlib.md5(long_url.encode())
    hash_hex = hash_object.hexdigest()
    hash_int = int(hash_hex[:16], 16)
    return encode_base62(hash_int)[:length]
```

### Option 3: Pre-generated Keys (Recommended for Scale)

```python
class KeyGenerationService:
    """
    Pre-generates unique keys and stores them in a database.
    Application servers fetch keys in batches.
    """

    def __init__(self, db, batch_size=1000):
        self.db = db
        self.batch_size = batch_size
        self.keys = []

    def get_key(self) -> str:
        if not self.keys:
            self.keys = self.db.fetch_unused_keys(self.batch_size)
            self.db.mark_keys_as_used(self.keys)
        return self.keys.pop()
```

### Comparison

| Method | Pros | Cons |
|--------|------|------|
| Base62(ID) | Simple, fast | Predictable |
| MD5 Hash | Deterministic | Collisions possible |
| Pre-generated | Fast, unique | Storage overhead |

---

## 7. Detailed Component Design

### URL Shortening Flow

```
1. Client sends long URL
2. App server validates URL
3. Check cache for existing mapping
4. If not found:
   a. Get unique key from Key Generation Service
   b. Store mapping in database
   c. Update cache
5. Return short URL
```

### URL Redirection Flow

```
1. User clicks short URL
2. CDN checks cache (hit → redirect)
3. If miss, request hits Load Balancer
4. App server checks Redis cache
5. If miss, query database
6. Update cache with result
7. Return 301 redirect
8. Async: increment click counter
```

---

## 8. Caching Strategy

### Cache-Aside Pattern

```python
async def get_long_url(short_code: str) -> str:
    # 1. Check cache first
    cached = await redis.get(f"url:{short_code}")
    if cached:
        return cached

    # 2. Query database
    result = await db.query(
        "SELECT long_url FROM urls WHERE short_code = $1",
        short_code
    )

    if result:
        # 3. Update cache (TTL: 24 hours)
        await redis.setex(f"url:{short_code}", 86400, result.long_url)
        return result.long_url

    return None
```

### Cache Eviction
- **LRU** (Least Recently Used) - recommended
- TTL of 24 hours for URL mappings
- Invalidate on URL deletion

---

## 9. Analytics (Optional)

### Async Click Tracking

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(bootstrap_servers=['kafka:9092'])

async def track_click(short_code: str, request: Request):
    event = {
        "short_code": short_code,
        "timestamp": datetime.utcnow().isoformat(),
        "ip": request.client.host,
        "user_agent": request.headers.get("user-agent"),
        "referrer": request.headers.get("referer")
    }
    producer.send("click-events", json.dumps(event).encode())
```

### Analytics Pipeline

```
Clicks → Kafka → Spark Streaming → Aggregated Metrics → Analytics DB
                                                              ↓
                                                        Dashboard
```

---

## 10. Scaling Considerations

### Database Sharding

```
Shard by first character of short_code:
- Shard 0: 0-9 (10 characters)
- Shard 1: a-m (13 characters)
- Shard 2: n-z (13 characters)
- Shard 3: A-M (13 characters)
- Shard 4: N-Z (13 characters)
```

### Load Balancing
- Round-robin for stateless app servers
- Consistent hashing for cache servers

### Data Cleanup
- Cron job to delete expired URLs
- Batch deletion during off-peak hours

---

## 11. Security Considerations

1. **Rate Limiting**: Prevent abuse
2. **URL Validation**: Block malicious URLs
3. **Private URLs**: Require authentication
4. **Spam Detection**: Block known spam domains

---

## 12. Final Architecture

```
                                    ┌─────────────────────────────────────────────────────┐
                                    │                    GLOBAL                           │
                                    │  ┌─────────────┐         ┌─────────────┐           │
                                    │  │   Route 53  │────────▶│  CloudFront │           │
                                    │  │    (DNS)    │         │    (CDN)    │           │
                                    │  └─────────────┘         └──────┬──────┘           │
                                    └─────────────────────────────────┼───────────────────┘
                                                                      │
                    ┌─────────────────────────────────────────────────┼─────────────────────────────────────────────────┐
                    │                                    REGION: US-EAST-1                                              │
                    │                                                 │                                                 │
                    │                                          ┌──────▼──────┐                                         │
                    │                                          │     ALB     │                                         │
                    │                                          └──────┬──────┘                                         │
                    │                                                 │                                                 │
                    │              ┌──────────────────────────────────┼──────────────────────────────────┐             │
                    │              │                                  │                                  │             │
                    │       ┌──────▼──────┐                    ┌──────▼──────┐                    ┌──────▼──────┐      │
                    │       │   ECS/EKS   │                    │   ECS/EKS   │                    │   ECS/EKS   │      │
                    │       │  Service    │                    │  Service    │                    │  Service    │      │
                    │       └──────┬──────┘                    └──────┬──────┘                    └──────┬──────┘      │
                    │              │                                  │                                  │             │
                    │              └──────────────────────────────────┼──────────────────────────────────┘             │
                    │                                                 │                                                 │
                    │        ┌────────────────────────────────────────┼────────────────────────────────────────┐       │
                    │        │                                        │                                        │       │
                    │ ┌──────▼──────┐                          ┌──────▼──────┐                          ┌──────▼──────┐│
                    │ │   Redis     │                          │  DynamoDB   │                          │    Key      ││
                    │ │ ElastiCache │                          │  (URLs)     │                          │  Generator  ││
                    │ └─────────────┘                          └─────────────┘                          └─────────────┘│
                    │                                                                                                   │
                    └───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Interview Tips

1. **Start with requirements** - Clarify scope before designing
2. **Do back-of-envelope calculations** - Shows you think about scale
3. **Draw the diagram first** - Visual communication is key
4. **Discuss trade-offs** - There's no perfect solution
5. **Consider failure scenarios** - What if DB goes down?

---

## 14. Common Follow-up Questions

| Question | Key Points |
|----------|------------|
| How to handle collisions? | Append random suffix, retry |
| How to prevent abuse? | Rate limiting, CAPTCHA |
| How to support custom domains? | CNAME records, SSL certs |
| What if Key Generator fails? | Fallback to hash-based generation |
| How to handle hot URLs? | Cache at CDN level, replicate |

---

## References

- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [Bitly Engineering Blog](https://word.bitly.com/)
- [High Scalability](http://highscalability.com/)
