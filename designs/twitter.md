# Twitter - Social Media Timeline

## Overview

Design a social media platform that supports real-time tweet posting, home timeline generation, search, and trending topics for 400+ million users.

---

## Requirements

### Functional
- Post tweets (280 characters, media attachments)
- Follow/unfollow users
- Home timeline (tweets from followed users)
- User timeline (user's own tweets)
- Retweets and likes
- Search tweets and users
- Trending topics
- Notifications

### Non-Functional
- Timeline generation < 500ms
- 99.99% availability
- Handle 500M+ tweets per day
- Support users with 50M+ followers
- Real-time updates

---

## Scale Estimation

```
Users: 400 million MAU, 200 million DAU
Tweets per day: 500 million
Average followers: 200
Celebrity followers: Up to 50 million

Timeline reads: 200M DAU * 10 reads/day = 2 billion reads/day
                = 23,000 reads/sec

Tweet writes: 500M / 86400 = 5,800 tweets/sec

Storage per tweet: 500 bytes (metadata) + media
Daily tweet storage: 500M * 500 bytes = 250 GB/day
```

---

## High-Level Architecture

```
    ┌─────────────────────────────────────────────────────────────────────────────────────┐
    │                                    CLIENTS                                          │
    │                                                                                     │
    │     ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                   │
    │     │   Mobile     │      │     Web      │      │   Desktop    │                   │
    │     └──────┬───────┘      └──────┬───────┘      └──────┬───────┘                   │
    │            │                     │                     │                            │
    └────────────┼─────────────────────┼─────────────────────┼────────────────────────────┘
                 │                     │                     │
                 └─────────────────────┼─────────────────────┘
                                       │
                              ┌────────┴────────┐
                              │   API GATEWAY   │
                              │                 │
                              │ - Auth          │
                              │ - Rate Limit    │
                              │ - Routing       │
                              └────────┬────────┘
                                       │
         ┌───────────────┬─────────────┼─────────────┬───────────────┐
         │               │             │             │               │
         v               v             v             v               v
    ┌─────────┐    ┌─────────┐   ┌─────────┐   ┌─────────┐    ┌─────────┐
    │  User   │    │  Tweet  │   │Timeline │   │ Search  │    │ Trends  │
    │ Service │    │ Service │   │ Service │   │ Service │    │ Service │
    └────┬────┘    └────┬────┘   └────┬────┘   └────┬────┘    └────┬────┘
         │              │             │             │               │
    ┌────┴────┐    ┌────┴────┐   ┌────┴────┐   ┌────┴────┐    ┌────┴────┐
    │  MySQL  │    │ Storage │   │  Redis  │   │  Elastic│    │  Redis  │
    │ Cluster │    │ Service │   │ Cluster │   │  Search │    │ Sorted  │
    └─────────┘    └─────────┘   └─────────┘   └─────────┘    │  Sets   │
                                                              └─────────┘
```

---

## The Fan-out Problem

The core challenge: When a celebrity with 50M followers tweets, how do we update 50 million timelines?

### Approach 1: Fan-out on Write (Push)

```
    USER POSTS TWEET
    ────────────────

    ┌─────────────┐
    │   Taylor    │
    │   Swift     │  Tweets: "Hello World!"
    │             │
    │ 50M followers
    └──────┬──────┘
           │
           v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        FAN-OUT SERVICE                                  │
    │                                                                         │
    │  1. Get follower list                                                   │
    │  2. For each follower: write tweet_id to their timeline cache          │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                    FOLLOWER TIMELINES                           │   │
    │  │                                                                  │   │
    │  │  User_1: [tweet_123, tweet_120, tweet_118, ...]                │   │
    │  │  User_2: [tweet_123, tweet_121, tweet_115, ...]                │   │
    │  │  User_3: [tweet_123, tweet_122, tweet_119, ...]                │   │
    │  │  ...                                                            │   │
    │  │  User_50M: [tweet_123, ...]                                    │   │
    │  │                                                                  │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                         │
    │  Problem: 50 million writes for one tweet                              │
    │  Latency: Minutes to complete fan-out                                  │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘

    Pros:
    - Timeline reads are fast (pre-computed)
    - Simple read path

    Cons:
    - Celebrity tweets cause write storms
    - Wasted work for inactive users
    - High latency for fan-out completion
```

### Approach 2: Fan-out on Read (Pull)

```
    USER READS TIMELINE
    ───────────────────

    ┌─────────────┐
    │    User     │
    │   opens     │
    │    app      │
    └──────┬──────┘
           │
           v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                       TIMELINE SERVICE                                  │
    │                                                                         │
    │  1. Get user's following list                                          │
    │  2. For each followed user: fetch recent tweets                        │
    │  3. Merge and sort all tweets                                          │
    │  4. Return top N                                                        │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                    MERGE OPERATION                              │   │
    │  │                                                                  │   │
    │  │  Following: [User_A, User_B, User_C, ..., User_500]            │   │
    │  │                                                                  │   │
    │  │  User_A tweets: [t1, t2, t3]                                   │   │
    │  │  User_B tweets: [t4, t5]                                       │   │
    │  │  User_C tweets: [t6, t7, t8, t9]                               │   │
    │  │  ...                                                            │   │
    │  │                                                                  │   │
    │  │  Merged timeline: [t6, t1, t4, t7, t2, t5, t8, t3, t9, ...]   │   │
    │  │                                                                  │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                         │
    │  Problem: Must query 500 users on every read                           │
    │  Latency: High for users following many accounts                       │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘

    Pros:
    - No write amplification
    - Always fresh data

    Cons:
    - Slow timeline reads
    - High read amplification
    - Expensive at scale
```

### Approach 3: Hybrid (Twitter's Actual Solution)

```
    HYBRID FAN-OUT STRATEGY
    ───────────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │  DECISION POINT: How many followers does the author have?              │
    │                                                                         │
    │                           Tweet Posted                                  │
    │                               │                                         │
    │                               v                                         │
    │                    ┌─────────────────────┐                             │
    │                    │  Followers > 10K?   │                             │
    │                    └──────────┬──────────┘                             │
    │                               │                                         │
    │               ┌───────────────┴───────────────┐                        │
    │               │                               │                        │
    │               v                               v                        │
    │         ┌─────────┐                     ┌─────────┐                    │
    │         │   NO    │                     │   YES   │                    │
    │         │         │                     │Celebrity│                    │
    │         └────┬────┘                     └────┬────┘                    │
    │              │                               │                         │
    │              v                               v                         │
    │    ┌─────────────────┐             ┌─────────────────┐                │
    │    │  FAN-OUT WRITE  │             │   STORE ONLY    │                │
    │    │                 │             │                 │                │
    │    │  Push to all    │             │  Don't fan-out  │                │
    │    │  follower       │             │  Store in       │                │
    │    │  timelines      │             │  author's       │                │
    │    │                 │             │  tweet list     │                │
    │    └─────────────────┘             └─────────────────┘                │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘


    TIMELINE READ (Hybrid Merge)
    ────────────────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │  User requests timeline:                                               │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                                                                  │   │
    │  │   1. Fetch pre-computed timeline from cache                     │   │
    │  │      (Contains tweets from non-celebrity followings)            │   │
    │  │                                                                  │   │
    │  │   2. Identify celebrity followings                              │   │
    │  │      (Users with > 10K followers)                               │   │
    │  │                                                                  │   │
    │  │   3. Fetch recent tweets from each celebrity                    │   │
    │  │      (Typically < 50 celebrities per user)                      │   │
    │  │                                                                  │   │
    │  │   4. Merge celebrity tweets with cached timeline                │   │
    │  │                                                                  │   │
    │  │   5. Return merged result                                       │   │
    │  │                                                                  │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                         │
    │  Result:                                                                │
    │  - Fast writes for celebrities (no fan-out)                            │
    │  - Fast reads for users (limited merge operations)                     │
    │  - Best of both approaches                                             │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## Timeline Cache Structure

```
    REDIS SORTED SETS FOR TIMELINES
    ────────────────────────────────

    Key: timeline:{user_id}
    Score: tweet timestamp (Unix epoch)
    Value: tweet_id

    ┌─────────────────────────────────────────────────────────────────────────┐
    │  timeline:12345                                                         │
    │                                                                         │
    │  Score (Timestamp)          Value (Tweet ID)                           │
    │  ─────────────────          ────────────────                           │
    │  1704067200000              tweet_789                                  │
    │  1704063600000              tweet_456                                  │
    │  1704060000000              tweet_123                                  │
    │  1704056400000              tweet_012                                  │
    │  ...                        ...                                        │
    │                                                                         │
    │  Operations:                                                            │
    │  - ZADD: Add new tweet O(log N)                                        │
    │  - ZRANGE: Get timeline page O(log N + M)                              │
    │  - ZREMRANGEBYRANK: Trim old tweets O(log N + M)                       │
    │                                                                         │
    │  Memory per user:                                                       │
    │  - 800 tweets * 8 bytes (tweet_id) = 6.4 KB                           │
    │  - 200M users * 6.4 KB = 1.28 TB                                       │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

### Fan-out Service Implementation

```python
class FanoutService:
    def __init__(self, redis_cluster, follower_service, celebrity_threshold=10000):
        self.redis = redis_cluster
        self.follower_service = follower_service
        self.celebrity_threshold = celebrity_threshold
        self.timeline_max_size = 800

    async def handle_new_tweet(self, tweet: Tweet):
        """Process a new tweet and fan-out if necessary"""
        follower_count = await self.follower_service.get_follower_count(
            tweet.author_id
        )

        # Store tweet in author's tweet list
        await self.redis.zadd(
            f"user_tweets:{tweet.author_id}",
            {tweet.id: tweet.created_at.timestamp()}
        )

        # Only fan-out for non-celebrities
        if follower_count < self.celebrity_threshold:
            await self._fan_out_to_followers(tweet)

    async def _fan_out_to_followers(self, tweet: Tweet):
        """Push tweet to all follower timelines"""
        followers = await self.follower_service.get_followers(tweet.author_id)

        # Batch writes for efficiency
        pipeline = self.redis.pipeline()
        for follower_id in followers:
            pipeline.zadd(
                f"timeline:{follower_id}",
                {tweet.id: tweet.created_at.timestamp()}
            )
            # Trim to prevent unbounded growth
            pipeline.zremrangebyrank(
                f"timeline:{follower_id}",
                0,
                -(self.timeline_max_size + 1)
            )

        await pipeline.execute()


class TimelineService:
    def __init__(self, redis_cluster, tweet_service, follower_service):
        self.redis = redis_cluster
        self.tweet_service = tweet_service
        self.follower_service = follower_service

    async def get_timeline(self, user_id: str, page: int = 0, size: int = 20):
        """Get user's home timeline with hybrid merge"""

        # 1. Get cached timeline (non-celebrity tweets)
        cached_tweet_ids = await self.redis.zrevrange(
            f"timeline:{user_id}",
            page * size,
            (page + 1) * size + 50  # Fetch extra for merge buffer
        )

        # 2. Get celebrity followings
        following = await self.follower_service.get_following(user_id)
        celebrities = [
            uid for uid in following
            if await self.follower_service.get_follower_count(uid) >= 10000
        ]

        # 3. Fetch recent tweets from celebrities
        celebrity_tweet_ids = []
        for celeb_id in celebrities[:50]:  # Limit celebrity merges
            tweets = await self.redis.zrevrange(
                f"user_tweets:{celeb_id}",
                0, 10  # Last 10 tweets per celebrity
            )
            celebrity_tweet_ids.extend(tweets)

        # 4. Merge and sort
        all_tweet_ids = list(set(cached_tweet_ids + celebrity_tweet_ids))
        tweets = await self.tweet_service.get_tweets_batch(all_tweet_ids)
        tweets.sort(key=lambda t: t.created_at, reverse=True)

        return tweets[page * size:(page + 1) * size]
```

---

## Tweet Storage

```
    TWEET DATA MODEL
    ────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                           TWEETS TABLE                                  │
    │                                                                         │
    │  tweet_id (Snowflake ID)                                               │
    │  ├── timestamp (41 bits) ─── Milliseconds since epoch                  │
    │  ├── datacenter (5 bits) ─── Datacenter ID                             │
    │  ├── worker (5 bits) ──────── Worker ID                                │
    │  └── sequence (12 bits) ───── Sequence number                          │
    │                                                                         │
    │  Schema:                                                                │
    │  ┌────────────────────────────────────────────────────────────────┐    │
    │  │  tweet_id         BIGINT PRIMARY KEY                           │    │
    │  │  author_id        BIGINT NOT NULL                              │    │
    │  │  content          VARCHAR(280)                                 │    │
    │  │  media_urls       JSON                                         │    │
    │  │  reply_to         BIGINT (nullable)                            │    │
    │  │  retweet_of       BIGINT (nullable)                            │    │
    │  │  quote_of         BIGINT (nullable)                            │    │
    │  │  like_count       INT DEFAULT 0                                │    │
    │  │  retweet_count    INT DEFAULT 0                                │    │
    │  │  reply_count      INT DEFAULT 0                                │    │
    │  │  created_at       TIMESTAMP                                    │    │
    │  └────────────────────────────────────────────────────────────────┘    │
    │                                                                         │
    │  Sharding: By tweet_id (time-ordered, ensures hot data locality)       │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## Search Architecture

```
    SEARCH PIPELINE
    ───────────────

    ┌─────────────┐
    │    Tweet    │
    │   Posted    │
    └──────┬──────┘
           │
           v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         INGESTION PIPELINE                              │
    │                                                                         │
    │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
    │  │    Kafka     │───>│   Earlybird  │───>│ Elasticsearch│             │
    │  │   (Buffer)   │    │  (Indexer)   │    │   Cluster    │             │
    │  └──────────────┘    └──────────────┘    └──────────────┘             │
    │                                                                         │
    │  Earlybird processing:                                                  │
    │  - Tokenization                                                         │
    │  - Language detection                                                   │
    │  - Entity extraction (hashtags, mentions, URLs)                        │
    │  - Sentiment analysis                                                   │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘

    SEARCH QUERY FLOW
    ─────────────────

    ┌─────────────┐
    │   "bitcoin" │
    │    query    │
    └──────┬──────┘
           │
           v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                          BLENDER SERVICE                                │
    │                                                                         │
    │  Query Analysis:                                                        │
    │  ┌────────────────────────────────────────────────────────────────┐    │
    │  │  - Parse query intent                                          │    │
    │  │  - Expand synonyms                                             │    │
    │  │  - Determine time range                                        │    │
    │  └────────────────────────────────────────────────────────────────┘    │
    │                                                                         │
    │  Parallel Search:                                                       │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │
    │  │  Realtime   │  │   Recent    │  │  Historical │                    │
    │  │   Index     │  │   Index     │  │    Index    │                    │
    │  │  (< 1 min)  │  │  (< 7 days) │  │  (all time) │                    │
    │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                    │
    │         │                │                │                            │
    │         └────────────────┼────────────────┘                            │
    │                          │                                              │
    │                          v                                              │
    │  ┌────────────────────────────────────────────────────────────────┐    │
    │  │                      RESULT RANKING                            │    │
    │  │                                                                 │    │
    │  │  Score = text_relevance * recency_boost * engagement_score    │    │
    │  │        * author_reputation * personalization_factor           │    │
    │  │                                                                 │    │
    │  └────────────────────────────────────────────────────────────────┘    │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## Trending Topics

```
    TRENDS CALCULATION
    ──────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         TRENDS PIPELINE                                 │
    │                                                                         │
    │  Real-time stream processing (Apache Storm/Flink):                     │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                                                                  │   │
    │  │  1. EXTRACT: Parse hashtags, keywords, phrases from tweets      │   │
    │  │                                                                  │   │
    │  │  2. COUNT: Sliding window counts (5min, 1hr, 24hr)             │   │
    │  │                                                                  │   │
    │  │  3. NORMALIZE: Adjust for baseline popularity                   │   │
    │  │                                                                  │   │
    │  │     trend_score = (current_rate - baseline_rate) / baseline_rate│   │
    │  │                                                                  │   │
    │  │  4. FILTER: Remove spam, sensitive content                      │   │
    │  │                                                                  │   │
    │  │  5. LOCALIZE: Group by geography, language                      │   │
    │  │                                                                  │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                         │
    │  Output: Top 10 trends per location, updated every 5 minutes           │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘

    TREND DETECTION ALGORITHM
    ─────────────────────────

    For each term:

    baseline = avg_count_last_30_days
    current = count_last_5_minutes * 288  # Normalize to daily rate

    z_score = (current - baseline) / std_dev

    if z_score > 3.0 and current > min_threshold:
        mark_as_trending()

    Example:
    ─────────
    Term: #SuperBowl
    Baseline: 10,000 mentions/day
    Current rate: 500,000 mentions/day (projected)
    Z-score: 15.2
    Result: TRENDING
```

---

## Notification System

```
    NOTIFICATION FLOW
    ─────────────────

    ┌─────────────┐
    │   Action    │
    │  (Like,     │
    │  Reply,     │
    │  Mention)   │
    └──────┬──────┘
           │
           v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      NOTIFICATION SERVICE                               │
    │                                                                         │
    │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
    │  │    Kafka     │───>│  Aggregator  │───>│   Delivery   │             │
    │  │   (Events)   │    │  (Batching)  │    │   (Push)     │             │
    │  └──────────────┘    └──────────────┘    └──────────────┘             │
    │                                                                         │
    │  Aggregation Rules:                                                     │
    │  - Multiple likes → "User and 5 others liked your tweet"              │
    │  - Batch within 5 minutes                                              │
    │  - Priority: mentions > replies > likes > follows                      │
    │                                                                         │
    │  Delivery Channels:                                                     │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
    │  │  │  Push   │  │  Email  │  │   SMS   │  │ In-App  │            │   │
    │  │  │  (APNs/ │  │(Batched)│  │(Critical│  │  Feed   │            │   │
    │  │  │  FCM)   │  │         │  │  only)  │  │         │            │   │
    │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘            │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## Data Storage Summary

| Data | Storage | Notes |
|------|---------|-------|
| Tweets | MySQL + Gizzard (sharded) | Sharded by tweet_id |
| User Data | MySQL | Partitioned by user_id |
| Timelines | Redis Cluster | Sorted sets, 1.28 TB |
| Social Graph | FlockDB (deprecated) / MySQL | Follow relationships |
| Search Index | Elasticsearch | Real-time indexing |
| Media | Blob Storage + CDN | Images, videos |
| Analytics | HDFS + Hive | Batch processing |

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Monthly Active Users | 400M+ |
| Tweets per Day | 500M+ |
| Timeline Reads/sec | 23,000 |
| Search Queries/sec | 10,000+ |
| Median Timeline Latency | < 200ms |
| P99 Timeline Latency | < 500ms |

---

## Key Lessons

1. **Fan-out is the fundamental tradeoff** - No perfect solution, hybrid approach wins
2. **Caching is essential** - Redis stores entire timeline working set
3. **Real-time is hard** - Requires specialized infrastructure (Kafka, Storm)
4. **Celebrity problem** - Special handling for high-follower accounts
5. **ID generation matters** - Snowflake enables time-ordered distributed IDs
6. **Search is separate** - Different infrastructure than timeline

---

## References

- Twitter Engineering Blog
- The Architecture of Twitter
- Timelines at Scale (InfoQ talk)
- How Twitter Uses Redis
