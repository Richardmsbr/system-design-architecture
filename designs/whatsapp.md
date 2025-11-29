# WhatsApp / Messaging System Design

> 🎯 **Difficulty**: Medium | ⏱️ **Interview Time**: 45-60 min

## 1. Requirements Clarification

### Functional Requirements
- One-on-one messaging
- Group chats (up to 256 members)
- Message delivery status (sent, delivered, read)
- Online/offline status
- Media sharing (images, videos, documents)
- End-to-end encryption

### Non-Functional Requirements
- Real-time messaging (< 100ms latency)
- High availability (99.99%)
- Message ordering guaranteed
- Eventual consistency acceptable
- Scalable to billions of users

---

## 2. Capacity Estimation

```
Assumptions:
- 2 billion users
- 500 million DAU
- 40 messages/user/day
- Average message size: 100 bytes

Calculations:
- Messages/day: 500M × 40 = 20 billion
- Messages/second: 20B / 86400 ≈ 230,000 msg/sec
- Storage/day: 20B × 100 bytes = 2 TB
- Storage/year: 2 TB × 365 = 730 TB (text only)

Bandwidth:
- Incoming: 230K × 100 bytes = 23 MB/sec
- With media (avg 200KB): 10% messages × 200KB = ~4.6 GB/sec
```

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                      CLIENTS                                         │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐                     │
│    │ iOS App  │    │ Android  │    │   Web    │    │ Desktop  │                     │
│    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘                     │
└─────────┼───────────────┼───────────────┼───────────────┼───────────────────────────┘
          │               │               │               │
          └───────────────┴───────────────┴───────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   Load Balancer     │
                         │   (WebSocket LB)    │
                         └──────────┬──────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Chat Server 1   │     │  Chat Server 2   │     │  Chat Server N   │
│  (WebSocket)     │     │  (WebSocket)     │     │  (WebSocket)     │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    ▼             ▼             ▼
          ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
          │   Redis     │ │   Kafka     │ │   User      │
          │  (Sessions) │ │ (Messages)  │ │  Service    │
          └─────────────┘ └──────┬──────┘ └─────────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
          ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
          │  Cassandra  │ │    S3       │ │   Push      │
          │ (Messages)  │ │  (Media)    │ │ Notification│
          └─────────────┘ └─────────────┘ └─────────────┘
```

---

## 4. Core Components

### 4.1 Connection Management

```python
# WebSocket Connection Handler
class ChatServer:
    def __init__(self):
        self.connections = {}  # user_id -> websocket
        self.redis = Redis()
        self.kafka = KafkaProducer()

    async def connect(self, user_id: str, websocket: WebSocket):
        await websocket.accept()
        self.connections[user_id] = websocket

        # Register in Redis for cross-server routing
        await self.redis.hset(
            "user:connections",
            user_id,
            f"{self.server_id}:{websocket.id}"
        )

        # Update online status
        await self.redis.set(f"user:online:{user_id}", "1", ex=300)

    async def disconnect(self, user_id: str):
        if user_id in self.connections:
            del self.connections[user_id]
            await self.redis.hdel("user:connections", user_id)
            await self.redis.delete(f"user:online:{user_id}")
```

### 4.2 Message Flow

```
┌────────┐    ┌────────────┐    ┌─────────┐    ┌────────────┐    ┌────────┐
│ Sender │───▶│Chat Server │───▶│  Kafka  │───▶│Chat Server │───▶│Receiver│
│        │    │  (Sender)  │    │         │    │ (Receiver) │    │        │
└────────┘    └────────────┘    └─────────┘    └────────────┘    └────────┘
     │              │                │                │               │
     │   1. Send    │                │                │               │
     │─────────────▶│                │                │               │
     │              │  2. Publish    │                │               │
     │              │───────────────▶│                │               │
     │              │                │  3. Consume    │               │
     │              │                │───────────────▶│               │
     │              │                │                │  4. Deliver   │
     │              │                │                │──────────────▶│
     │              │                │                │               │
     │◀─────────────│                │                │               │
     │   5. ACK     │                │                │               │
```

### 4.3 Message Structure

```json
{
  "message_id": "uuid-v7",
  "conversation_id": "conv_123",
  "sender_id": "user_456",
  "recipient_id": "user_789",
  "type": "text|image|video|document|voice",
  "content": {
    "text": "Hello!",
    "media_url": null,
    "thumbnail_url": null
  },
  "metadata": {
    "reply_to": null,
    "forwarded": false
  },
  "encryption": {
    "protocol": "signal",
    "key_id": "key_123"
  },
  "status": "sent|delivered|read",
  "timestamps": {
    "sent_at": "2024-01-15T10:30:00Z",
    "delivered_at": null,
    "read_at": null
  }
}
```

---

## 5. Database Design

### 5.1 Cassandra Schema (Messages)

```sql
-- Messages by conversation (for chat history)
CREATE TABLE messages_by_conversation (
    conversation_id UUID,
    message_id TIMEUUID,
    sender_id UUID,
    message_type TEXT,
    content TEXT,
    media_url TEXT,
    status TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Messages by user (for syncing)
CREATE TABLE messages_by_user (
    user_id UUID,
    message_id TIMEUUID,
    conversation_id UUID,
    sender_id UUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
AND default_time_to_live = 2592000;  -- 30 days

-- User's conversations
CREATE TABLE conversations_by_user (
    user_id UUID,
    conversation_id UUID,
    last_message_id TIMEUUID,
    last_message_preview TEXT,
    unread_count INT,
    updated_at TIMESTAMP,
    PRIMARY KEY (user_id, updated_at, conversation_id)
) WITH CLUSTERING ORDER BY (updated_at DESC);
```

### 5.2 Redis Data Structures

```
# User sessions (which server they're connected to)
HSET user:connections user_123 "server_5:ws_789"

# Online status with heartbeat
SETEX user:online:user_123 300 "1"

# Typing indicators (expires in 5 seconds)
SETEX typing:conv_456:user_123 5 "1"

# Unread message counts
HINCRBY unread:user_789 conv_456 1

# Last seen timestamp
SET lastseen:user_123 "2024-01-15T10:30:00Z"
```

---

## 6. Message Delivery Guarantees

### 6.1 Delivery States

```
┌──────────────────────────────────────────────────────────────┐
│                    MESSAGE LIFECYCLE                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│   ┌─────┐     ┌─────────┐     ┌───────────┐     ┌──────┐    │
│   │ ✓   │────▶│ ✓✓      │────▶│ ✓✓ (blue) │────▶│ 🗑️   │    │
│   │Sent │     │Delivered│     │   Read    │     │Deleted│    │
│   └─────┘     └─────────┘     └───────────┘     └──────┘    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 Delivery Flow

```python
async def deliver_message(message: Message):
    # 1. Store message in Cassandra
    await cassandra.execute(
        "INSERT INTO messages_by_conversation (...) VALUES (...)"
    )

    # 2. Check if recipient is online
    server_info = await redis.hget("user:connections", message.recipient_id)

    if server_info:
        # 3a. Recipient online - deliver via WebSocket
        server_id, ws_id = server_info.split(":")

        if server_id == MY_SERVER_ID:
            # Same server - deliver directly
            await self.connections[message.recipient_id].send(message)
        else:
            # Different server - publish to Kafka
            await kafka.send(f"server-{server_id}", message)

        # Update status to delivered
        await update_message_status(message.id, "delivered")
    else:
        # 3b. Recipient offline - queue for push notification
        await send_push_notification(message.recipient_id, message)
```

---

## 7. Group Messaging

### 7.1 Group Schema

```sql
CREATE TABLE groups (
    group_id UUID PRIMARY KEY,
    name TEXT,
    description TEXT,
    avatar_url TEXT,
    created_by UUID,
    created_at TIMESTAMP,
    member_count INT
);

CREATE TABLE group_members (
    group_id UUID,
    user_id UUID,
    role TEXT,  -- admin, member
    joined_at TIMESTAMP,
    PRIMARY KEY (group_id, user_id)
);

CREATE TABLE user_groups (
    user_id UUID,
    group_id UUID,
    last_read_message_id TIMEUUID,
    muted_until TIMESTAMP,
    PRIMARY KEY (user_id, group_id)
);
```

### 7.2 Fan-out Strategy

```python
async def send_group_message(group_id: str, message: Message):
    # Get all group members
    members = await get_group_members(group_id)

    # Store message once
    await store_message(message)

    # Fan-out to online members
    online_members = []
    offline_members = []

    for member in members:
        if member.id != message.sender_id:
            if await is_online(member.id):
                online_members.append(member)
            else:
                offline_members.append(member)

    # Batch deliver to online members
    await asyncio.gather(*[
        deliver_to_user(member.id, message)
        for member in online_members
    ])

    # Queue push notifications for offline members
    await queue_push_notifications(offline_members, message)
```

---

## 8. Media Handling

### 8.1 Media Upload Flow

```
┌────────┐     ┌────────────┐     ┌─────────┐     ┌─────────┐
│ Client │────▶│ API Server │────▶│   S3    │────▶│   CDN   │
└────────┘     └────────────┘     └─────────┘     └─────────┘
     │               │                  │               │
     │ 1. Upload     │                  │               │
     │   Request     │                  │               │
     │──────────────▶│                  │               │
     │               │                  │               │
     │◀──────────────│                  │               │
     │ 2. Presigned  │                  │               │
     │    URL        │                  │               │
     │               │                  │               │
     │ 3. Upload to S3                  │               │
     │─────────────────────────────────▶│               │
     │               │                  │               │
     │◀─────────────────────────────────│               │
     │ 4. Success    │                  │               │
     │               │                  │               │
     │ 5. Send message with media_url   │               │
     │──────────────▶│                  │               │
```

### 8.2 Media Processing

```python
async def process_media(media_id: str, media_type: str):
    # Download original
    original = await s3.download(f"uploads/{media_id}")

    if media_type == "image":
        # Generate thumbnail
        thumbnail = resize_image(original, 200, 200)
        await s3.upload(f"thumbnails/{media_id}", thumbnail)

        # Compress for mobile
        compressed = compress_image(original, quality=80)
        await s3.upload(f"compressed/{media_id}", compressed)

    elif media_type == "video":
        # Transcode to multiple resolutions
        for resolution in ["480p", "720p", "1080p"]:
            transcoded = transcode_video(original, resolution)
            await s3.upload(f"videos/{media_id}/{resolution}", transcoded)

        # Generate thumbnail
        thumbnail = extract_frame(original, second=1)
        await s3.upload(f"thumbnails/{media_id}", thumbnail)
```

---

## 9. End-to-End Encryption

### 9.1 Signal Protocol Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    SIGNAL PROTOCOL                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐                              ┌──────────┐         │
│  │  Alice   │                              │   Bob    │         │
│  │          │                              │          │         │
│  │ Identity │                              │ Identity │         │
│  │   Key    │                              │   Key    │         │
│  │          │                              │          │         │
│  │ Signed   │                              │ Signed   │         │
│  │ PreKey   │                              │ PreKey   │         │
│  │          │                              │          │         │
│  │ One-Time │                              │ One-Time │         │
│  │ PreKeys  │                              │ PreKeys  │         │
│  └────┬─────┘                              └────┬─────┘         │
│       │                                         │                │
│       │     1. Fetch Bob's PreKey Bundle        │                │
│       │────────────────────────────────────────▶│                │
│       │                                         │                │
│       │     2. Generate Shared Secret (X3DH)    │                │
│       │◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │                │
│       │                                         │                │
│       │     3. Encrypt with Double Ratchet      │                │
│       │────────────────────────────────────────▶│                │
│       │                                         │                │
└───────┴─────────────────────────────────────────┴────────────────┘
```

### 9.2 Key Storage

```sql
-- Store on server (public keys only)
CREATE TABLE user_keys (
    user_id UUID PRIMARY KEY,
    identity_key BLOB,          -- Public identity key
    signed_prekey BLOB,         -- Signed prekey
    signed_prekey_sig BLOB,     -- Signature
    registration_id INT
);

CREATE TABLE one_time_prekeys (
    user_id UUID,
    key_id INT,
    prekey BLOB,
    PRIMARY KEY (user_id, key_id)
);
```

---

## 10. Push Notifications

### 10.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   PUSH NOTIFICATION FLOW                     │
└─────────────────────────────────────────────────────────────┘

    Message arrives     Check online       Queue notification
         │                   │                    │
         ▼                   ▼                    ▼
    ┌─────────┐        ┌─────────┐         ┌─────────┐
    │  Kafka  │───────▶│  Redis  │────────▶│  Queue  │
    └─────────┘        └─────────┘         └────┬────┘
                                                │
                         ┌──────────────────────┴──────────────────────┐
                         │                      │                      │
                         ▼                      ▼                      ▼
                   ┌──────────┐           ┌──────────┐           ┌──────────┐
                   │   APNs   │           │   FCM    │           │  Huawei  │
                   │  (iOS)   │           │(Android) │           │   Push   │
                   └──────────┘           └──────────┘           └──────────┘
```

### 10.2 Notification Payload

```python
async def send_push_notification(user_id: str, message: Message):
    # Get user's push tokens
    tokens = await get_push_tokens(user_id)

    # Prepare notification (encrypted content hint)
    notification = {
        "title": get_sender_name(message.sender_id),
        "body": "New message",  # Don't reveal content
        "data": {
            "conversation_id": message.conversation_id,
            "message_id": message.message_id,
        },
        "priority": "high",
        "ttl": 86400  # 24 hours
    }

    for token in tokens:
        if token.platform == "ios":
            await apns.send(token.value, notification)
        elif token.platform == "android":
            await fcm.send(token.value, notification)
```

---

## 11. Scaling Strategies

### 11.1 Horizontal Scaling

```
                              ┌─────────────────┐
                              │  Load Balancer  │
                              │  (Consistent    │
                              │   Hashing)      │
                              └────────┬────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
        ▼                              ▼                              ▼
┌───────────────┐              ┌───────────────┐              ┌───────────────┐
│ Chat Server   │              │ Chat Server   │              │ Chat Server   │
│ Cluster A     │              │ Cluster B     │              │ Cluster N     │
│               │              │               │              │               │
│ Users: A-H    │              │ Users: I-P    │              │ Users: Q-Z    │
└───────────────┘              └───────────────┘              └───────────────┘
```

### 11.2 Database Sharding

```python
def get_shard(user_id: str) -> int:
    """Consistent hashing for user data"""
    hash_value = murmurhash3(user_id)
    return hash_value % NUM_SHARDS

# Shard by conversation for messages
def get_message_shard(conversation_id: str) -> int:
    hash_value = murmurhash3(conversation_id)
    return hash_value % NUM_MESSAGE_SHARDS
```

---

## 12. Reliability & Fault Tolerance

### 12.1 Message Retry Logic

```python
async def send_with_retry(message: Message, max_retries: int = 3):
    for attempt in range(max_retries):
        try:
            await deliver_message(message)
            return
        except DeliveryError as e:
            if attempt == max_retries - 1:
                # Store in dead letter queue
                await dlq.push(message)
                raise
            await asyncio.sleep(2 ** attempt)  # Exponential backoff
```

### 12.2 Health Checks

```python
@app.get("/health")
async def health_check():
    checks = await asyncio.gather(
        check_redis(),
        check_cassandra(),
        check_kafka(),
        return_exceptions=True
    )

    all_healthy = all(c == True for c in checks)

    return {
        "status": "healthy" if all_healthy else "degraded",
        "redis": checks[0],
        "cassandra": checks[1],
        "kafka": checks[2]
    }
```

---

## 13. Interview Tips

1. **Start with requirements** - Real-time vs near-real-time matters
2. **Focus on the happy path first** - Then handle edge cases
3. **Discuss trade-offs** - Why WebSocket over polling?
4. **Consider mobile constraints** - Battery, bandwidth
5. **Security is critical** - E2E encryption, key management

---

## 14. Common Follow-up Questions

| Question | Key Points |
|----------|------------|
| How to handle millions of groups? | Shard by group_id, limit fan-out |
| What if a server crashes? | Reconnect to different server, sync pending messages |
| How to implement read receipts at scale? | Batch updates, eventual consistency |
| How to prevent spam? | Rate limiting, ML-based detection |
| How to sync across devices? | Message versioning, conflict resolution |

---

## References

- [WhatsApp Architecture (InfoQ)](https://www.infoq.com/presentations/whatsapp-scalability/)
- [Signal Protocol](https://signal.org/docs/)
- [Building Messaging Systems at Scale](https://engineering.fb.com/2015/05/26/core-data/building-mobile-first-infrastructure-for-messenger/)
