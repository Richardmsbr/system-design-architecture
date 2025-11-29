# SQL vs NoSQL Decision Guide

A practical guide for choosing between relational and non-relational databases.

---

## Quick Decision Matrix

```
                         STRUCTURED        UNSTRUCTURED
                         SCHEMA            SCHEMA
                            │                  │
                            │                  │
    ACID Required ──────────┼──────────────────┼──────────────
                            │                  │
                         SQL DB            Document DB
                      (PostgreSQL)         (MongoDB)
                            │                  │
                            │                  │
    Eventual OK ────────────┼──────────────────┼──────────────
                            │                  │
                       NewSQL             Key-Value
                    (CockroachDB)          (Redis)
                            │                  │
                            │                  │
                         STRONG            FLEXIBLE
                       CONSISTENCY         SCHEMA
```

---

## Database Categories

### Relational (SQL)

```
BEST FOR:
- Complex queries with JOINs
- ACID transactions
- Structured data with relationships
- Reporting and analytics
- Data integrity requirements

EXAMPLES:
┌─────────────────┬─────────────────────────────────────────────────────────┐
│   PostgreSQL    │ Feature-rich, extensible, JSON support                  │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   MySQL         │ Widely adopted, mature ecosystem                        │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   SQL Server    │ Enterprise, Windows ecosystem                           │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   Oracle        │ Enterprise, high performance                            │
└─────────────────┴─────────────────────────────────────────────────────────┘

SCALING PATTERN:
- Vertical scaling (bigger machine)
- Read replicas for read scaling
- Sharding for write scaling (complex)
```

### Document Stores

```
BEST FOR:
- Flexible, evolving schemas
- Semi-structured data
- Content management
- Catalogs and user profiles
- Nested data structures

EXAMPLES:
┌─────────────────┬─────────────────────────────────────────────────────────┐
│   MongoDB       │ Popular, flexible querying, aggregation pipeline        │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   Firestore     │ Real-time sync, serverless                              │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   CouchDB       │ Multi-master replication                                │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   DynamoDB      │ Managed, single-digit millisecond latency               │
└─────────────────┴─────────────────────────────────────────────────────────┘

DATA MODEL:
{
  "_id": "user123",
  "name": "John Doe",
  "email": "john@example.com",
  "orders": [
    {"id": "ord1", "total": 99.99},
    {"id": "ord2", "total": 149.99}
  ],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}
```

### Key-Value Stores

```
BEST FOR:
- Caching
- Session management
- Real-time leaderboards
- Rate limiting
- Simple lookups by key

EXAMPLES:
┌─────────────────┬─────────────────────────────────────────────────────────┐
│   Redis         │ In-memory, rich data structures, pub/sub               │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   Memcached     │ Simple caching, multi-threaded                         │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   DynamoDB      │ Managed, consistent performance at scale               │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   etcd          │ Distributed configuration, consensus                   │
└─────────────────┴─────────────────────────────────────────────────────────┘

OPERATIONS:
GET key           O(1)
SET key value     O(1)
DEL key           O(1)
INCR key          O(1)
```

### Wide-Column Stores

```
BEST FOR:
- Time-series data
- IoT sensor data
- Write-heavy workloads
- Large-scale analytics
- Event logging

EXAMPLES:
┌─────────────────┬─────────────────────────────────────────────────────────┐
│   Cassandra     │ No single point of failure, linear scaling             │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   ScyllaDB      │ Cassandra-compatible, better performance               │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   HBase         │ Hadoop ecosystem, random read/write                    │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   Bigtable      │ Google managed, high throughput                        │
└─────────────────┴─────────────────────────────────────────────────────────┘

DATA MODEL:
Row Key         Column Family: user_info        Column Family: activity
──────────────────────────────────────────────────────────────────────────
user:123        name:John | email:j@e.com       login:1704067200 | ...
user:456        name:Jane | email:j@e.org       login:1704153600 | ...
```

### Graph Databases

```
BEST FOR:
- Social networks
- Recommendation engines
- Fraud detection
- Knowledge graphs
- Network analysis

EXAMPLES:
┌─────────────────┬─────────────────────────────────────────────────────────┐
│   Neo4j         │ Native graph, Cypher query language                    │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   Neptune       │ AWS managed, Gremlin and SPARQL                        │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   JanusGraph    │ Distributed, multiple backends                         │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   TigerGraph    │ Real-time analytics, massive graphs                    │
└─────────────────┴─────────────────────────────────────────────────────────┘

QUERY EXAMPLE (Cypher):
MATCH (user:User)-[:FOLLOWS]->(friend:User)-[:LIKES]->(product:Product)
WHERE user.id = 'user123'
RETURN product.name, COUNT(friend) as recommendation_strength
ORDER BY recommendation_strength DESC
```

### Time-Series Databases

```
BEST FOR:
- Metrics and monitoring
- IoT data
- Financial data
- Log aggregation
- Real-time analytics

EXAMPLES:
┌─────────────────┬─────────────────────────────────────────────────────────┐
│   InfluxDB      │ Purpose-built, high ingest rate                        │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   TimescaleDB   │ PostgreSQL extension, SQL interface                    │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   Prometheus    │ Metrics-focused, pull model                            │
├─────────────────┼─────────────────────────────────────────────────────────┤
│   ClickHouse    │ Column-oriented, fast aggregations                     │
└─────────────────┴─────────────────────────────────────────────────────────┘

TYPICAL QUERY:
SELECT mean(cpu_usage)
FROM metrics
WHERE host = 'server01'
  AND time > now() - 1h
GROUP BY time(5m)
```

---

## Comparison Table

| Criteria | SQL | Document | Key-Value | Wide-Column | Graph |
|----------|-----|----------|-----------|-------------|-------|
| Schema | Fixed | Flexible | None | Flexible | Flexible |
| Transactions | ACID | Limited | Limited | Eventual | ACID |
| Scaling | Vertical | Horizontal | Horizontal | Horizontal | Vertical |
| Queries | Complex | Medium | Simple | Simple | Complex |
| Joins | Native | Manual | No | No | Native |
| Consistency | Strong | Tunable | Tunable | Tunable | Strong |

---

## Decision Tree

```
START
  │
  ▼
Do you need ACID transactions across multiple records?
  │
  ├── YES ──────────────────────────────────────────────────────┐
  │                                                              │
  │                                                              ▼
  │                                            Do you need global scale?
  │                                                              │
  │                                              ├── YES ─> NewSQL
  │                                              │          (CockroachDB,
  │                                              │           Spanner)
  │                                              │
  │                                              └── NO ──> SQL
  │                                                         (PostgreSQL,
  │                                                          MySQL)
  │
  └── NO
        │
        ▼
   What's your primary access pattern?
        │
        ├── Key-based lookups ──────────────────────> Key-Value
        │                                             (Redis, DynamoDB)
        │
        ├── Time-series data ───────────────────────> Time-Series
        │                                             (InfluxDB, TimescaleDB)
        │
        ├── Relationship traversal ─────────────────> Graph
        │                                             (Neo4j, Neptune)
        │
        ├── Wide, sparse data ──────────────────────> Wide-Column
        │                                             (Cassandra, Bigtable)
        │
        └── Flexible documents ─────────────────────> Document
                                                      (MongoDB, Firestore)
```

---

## CAP Theorem Reference

```
                              CONSISTENCY
                                   ▲
                                  /│\
                                 / │ \
                                /  │  \
                               /   │   \
                              /    │    \
                             /     │     \
                            /  CP  │  CA  \
                           / Mongo │ MySQL \
                          / HBase  │ Postgres\
                         /─────────┼─────────\
                        /          │          \
                       /           │           \
                      /     AP     │            \
                     /  Cassandra  │             \
                    /   DynamoDB   │              \
                   /    Riak       │               \
                  /________________│________________\
       AVAILABILITY ─────────────────────────── PARTITION TOLERANCE


Reality: In a distributed system, you MUST handle partitions.
         The real choice is between CP and AP.

CP: Refuse to respond if uncertain (banking, inventory)
AP: Respond with potentially stale data (social media, caching)
```

---

## Polyglot Persistence Example

```
E-COMMERCE PLATFORM
───────────────────

┌─────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION                                   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│  PostgreSQL   │       │   MongoDB     │       │    Redis      │
│               │       │               │       │               │
│ - Orders      │       │ - Product     │       │ - Sessions    │
│ - Payments    │       │   Catalog     │       │ - Cart        │
│ - Inventory   │       │ - Reviews     │       │ - Rate limits │
│               │       │ - User prefs  │       │ - Leaderboard │
│               │       │               │       │               │
│ (ACID needed) │       │ (Flexible     │       │ (Low latency) │
│               │       │  schema)      │       │               │
└───────────────┘       └───────────────┘       └───────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│ Elasticsearch │       │   Cassandra   │       │    Neo4j      │
│               │       │               │       │               │
│ - Search      │       │ - Click       │       │ - Recommen-   │
│ - Facets      │       │   stream      │       │   dations     │
│ - Autocomplete│       │ - Analytics   │       │ - Social      │
│               │       │ - Time-series │       │   graph       │
│               │       │               │       │               │
│ (Full-text)   │       │ (Write-heavy) │       │ (Relations)   │
└───────────────┘       └───────────────┘       └───────────────┘
```

---

## Migration Considerations

### SQL to NoSQL

```
WHEN TO MIGRATE:
- Read/write ratio heavily skewed
- Schema changes are painful
- Horizontal scaling required
- Flexible data model needed

CHALLENGES:
- Loss of JOINs (denormalization required)
- Transaction boundaries
- Query complexity shifts to application
- Data modeling paradigm shift

PATTERN:
1. Add NoSQL alongside SQL
2. Dual-write to both
3. Read from NoSQL, validate against SQL
4. Cut over reads
5. Remove SQL writes
```

### NoSQL to SQL

```
WHEN TO MIGRATE:
- Need complex queries/reporting
- ACID requirements increased
- Data relationships became complex
- Consistency issues

CHALLENGES:
- Schema design from flexible data
- Handling schema variations
- JOIN performance
- Migration downtime

PATTERN:
1. Design normalized schema
2. ETL from NoSQL to SQL
3. Dual-read validation
4. Gradual traffic shift
```

---

## References

- Martin Fowler - NoSQL Distilled
- Database Internals by Alex Petrov
- Designing Data-Intensive Applications by Martin Kleppmann
