# Netflix - Video Streaming Platform

## Overview

Design a video streaming platform capable of serving 200+ million subscribers globally with personalized content recommendations, adaptive streaming, and 99.99% availability.

---

## Requirements

### Functional
- User registration and authentication
- Video upload and processing (studio partners)
- Video streaming with adaptive bitrate
- Search and discovery
- Personalized recommendations
- Watch history and continue watching
- Multiple profiles per account
- Offline downloads

### Non-Functional
- Global availability with low latency
- Support 200M+ concurrent users
- 99.99% uptime SLA
- Adaptive quality based on bandwidth
- Sub-second start time for playback

---

## Scale Estimation

```
Users: 200 million subscribers
Concurrent viewers (peak): 10 million
Average video size: 5GB (multiple resolutions)
Total content: 15,000 titles
Storage: 15,000 * 5GB = 75 PB (with redundancy: 225 PB)
Bandwidth (peak): 10M * 5 Mbps = 50 Tbps
```

---

## High-Level Architecture

```
                                           GLOBAL EDGE
    ┌──────────────────────────────────────────────────────────────────────────────────────┐
    │                                                                                      │
    │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
    │  │   Client    │    │   Client    │    │   Client    │    │   Client    │          │
    │  │  (Smart TV) │    │   (Mobile)  │    │    (Web)    │    │  (Console)  │          │
    │  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘          │
    │         │                  │                  │                  │                  │
    │         └──────────────────┴──────────────────┴──────────────────┘                  │
    │                                       │                                              │
    │                              ┌────────┴────────┐                                    │
    │                              │   OPEN CONNECT  │                                    │
    │                              │   (Edge Cache)  │                                    │
    │                              │                 │                                    │
    │                              │  10,000+ nodes  │                                    │
    │                              │  in ISP networks│                                    │
    │                              └────────┬────────┘                                    │
    │                                       │                                              │
    └───────────────────────────────────────┼──────────────────────────────────────────────┘
                                            │
                                            │ Cache Miss
                                            │
    ┌───────────────────────────────────────┼──────────────────────────────────────────────┐
    │                              AWS INFRASTRUCTURE                                      │
    │                                       │                                              │
    │                              ┌────────┴────────┐                                    │
    │                              │    CloudFront   │                                    │
    │                              │  (Fallback CDN) │                                    │
    │                              └────────┬────────┘                                    │
    │                                       │                                              │
    │         ┌─────────────────────────────┼─────────────────────────────┐               │
    │         │                             │                             │               │
    │         v                             v                             v               │
    │  ┌─────────────┐              ┌─────────────┐              ┌─────────────┐         │
    │  │   Zuul      │              │   Zuul      │              │   Zuul      │         │
    │  │  (Gateway)  │              │  (Gateway)  │              │  (Gateway)  │         │
    │  └──────┬──────┘              └──────┬──────┘              └──────┬──────┘         │
    │         │                            │                            │                 │
    │         └────────────────────────────┼────────────────────────────┘                 │
    │                                      │                                              │
    │    ┌───────────┬───────────┬─────────┼─────────┬───────────┬───────────┐           │
    │    │           │           │         │         │           │           │           │
    │    v           v           v         v         v           v           v           │
    │ ┌──────┐   ┌──────┐   ┌──────┐  ┌──────┐  ┌──────┐   ┌──────┐   ┌──────┐          │
    │ │User  │   │Play  │   │Search│  │Reco  │  │Video │   │Billing│  │Analytics│       │
    │ │Service│  │back  │   │Service│ │Service│ │Service│  │Service│  │Service│         │
    │ └──────┘   └──────┘   └──────┘  └──────┘  └──────┘   └──────┘   └──────┘          │
    │                                                                                     │
    │    ┌───────────────────────────────────────────────────────────────────────┐       │
    │    │                         DATA LAYER                                     │       │
    │    │                                                                        │       │
    │    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐              │       │
    │    │  │Cassandra │  │  MySQL   │  │   EVCache│  │   S3     │              │       │
    │    │  │(Metadata)│  │(Billing) │  │  (Redis) │  │ (Videos) │              │       │
    │    │  └──────────┘  └──────────┘  └──────────┘  └──────────┘              │       │
    │    │                                                                        │       │
    │    └───────────────────────────────────────────────────────────────────────┘       │
    │                                                                                     │
    └─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Video Processing Pipeline

```
    CONTENT INGESTION                    PROCESSING                         DELIVERY
    ─────────────────                    ──────────                         ────────

    ┌─────────────┐
    │   Studio    │
    │   Upload    │
    │  (Mezzanine │
    │   4K/8K)    │
    └──────┬──────┘
           │
           v
    ┌─────────────┐      ┌─────────────────────────────────────────────────────────┐
    │    S3       │      │                   ENCODING PIPELINE                     │
    │  (Source)   │─────>│                                                         │
    └─────────────┘      │  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
                         │  │ Quality  │───>│ Parallel │───>│ Package  │          │
                         │  │ Analysis │    │ Encoding │    │ (DASH/   │          │
                         │  │          │    │          │    │  HLS)    │          │
                         │  └──────────┘    └──────────┘    └──────────┘          │
                         │                                                         │
                         │  Per-title optimization:                                │
                         │  - Analyze complexity per scene                         │
                         │  - Allocate bitrate dynamically                        │
                         │  - Generate encoding ladder                            │
                         │                                                         │
                         └─────────────────────────┬───────────────────────────────┘
                                                   │
                                                   v
                         ┌─────────────────────────────────────────────────────────┐
                         │                   OUTPUT PROFILES                       │
                         │                                                         │
                         │  Resolution    Bitrate      Codec                       │
                         │  ──────────    ───────      ─────                       │
                         │  4K (2160p)    16 Mbps      HEVC/VP9                   │
                         │  1080p         5.8 Mbps     H.264/HEVC                 │
                         │  720p          2.35 Mbps    H.264                      │
                         │  480p          1.05 Mbps    H.264                      │
                         │  360p          0.56 Mbps    H.264                      │
                         │  240p          0.27 Mbps    H.264                      │
                         │                                                         │
                         └─────────────────────────┬───────────────────────────────┘
                                                   │
                                                   v
                                            ┌─────────────┐
                                            │    S3       │
                                            │  (Encoded)  │
                                            └──────┬──────┘
                                                   │
                                    ┌──────────────┼──────────────┐
                                    │              │              │
                                    v              v              v
                             ┌──────────┐   ┌──────────┐   ┌──────────┐
                             │  Open    │   │  Open    │   │  Open    │
                             │ Connect  │   │ Connect  │   │ Connect  │
                             │  (ISP 1) │   │  (ISP 2) │   │  (ISP N) │
                             └──────────┘   └──────────┘   └──────────┘
```

---

## Adaptive Bitrate Streaming

```
    CLIENT BANDWIDTH DETECTION
    ──────────────────────────

    Time ──────────────────────────────────────────────────────────────────>

    Bandwidth
    (Mbps)
       │
    20 │                              ████████
       │                         █████        ████
    15 │                    █████                  ████
       │               █████                           ████
    10 │          █████                                    ████████████
       │     █████
     5 │█████
       │
     0 └───────────────────────────────────────────────────────────────────
         │        │         │         │         │         │
       Start    720p      1080p     1080p      720p      1080p
              upgrade    upgrade   maintain  downgrade  upgrade

    ABR ALGORITHM (Buffer-Based + Throughput)
    ─────────────────────────────────────────

    if buffer_level < CRITICAL_THRESHOLD:
        select_lowest_quality()
    elif buffer_level < SAFE_THRESHOLD:
        quality = estimate_from_throughput()
        if quality < current_quality:
            downgrade()
    else:
        quality = estimate_from_throughput()
        if quality > current_quality and buffer_stable:
            upgrade()
```

---

## Open Connect CDN

Netflix's custom CDN deployed directly in ISP networks:

```
    INTERNET EXCHANGE POINTS                    ISP NETWORKS
    ────────────────────────                    ────────────

    ┌─────────────────────────┐          ┌─────────────────────────────────┐
    │     NETFLIX ORIGIN      │          │           ISP NETWORK           │
    │                         │          │                                 │
    │  ┌───────────────────┐  │          │  ┌─────────────────────────┐   │
    │  │    S3 Storage     │  │          │  │    Open Connect Box     │   │
    │  │    (All Content)  │  │          │  │                         │   │
    │  └─────────┬─────────┘  │          │  │  - 100TB+ storage       │   │
    │            │            │          │  │  - 100 Gbps capacity    │   │
    │            v            │          │  │  - Serves 90%+ traffic  │   │
    │  ┌───────────────────┐  │          │  │                         │   │
    │  │   Fill Servers    │──┼──────────┼─>│  Popular content        │   │
    │  │  (Proactive Fill) │  │  Night   │  │  pre-positioned         │   │
    │  └───────────────────┘  │  Fill    │  │                         │   │
    │                         │          │  └────────────┬────────────┘   │
    └─────────────────────────┘          │               │                │
                                         │               v                │
                                         │  ┌─────────────────────────┐   │
                                         │  │      ISP Router         │   │
                                         │  └────────────┬────────────┘   │
                                         │               │                │
                                         │    ┌──────────┴──────────┐    │
                                         │    │                     │    │
                                         │    v                     v    │
                                         │ ┌──────┐             ┌──────┐│
                                         │ │ Home │             │ Home ││
                                         │ └──────┘             └──────┘│
                                         │                               │
                                         └───────────────────────────────┘

    CONTENT PLACEMENT ALGORITHM
    ───────────────────────────
    1. Analyze viewing patterns per region
    2. Predict popularity for next 24 hours
    3. Pre-position content during off-peak hours
    4. Maintain hot/warm/cold tiers
    5. Real-time popularity adjustments
```

---

## Recommendation System

```
    DATA COLLECTION                    PROCESSING                      SERVING
    ───────────────                    ──────────                      ───────

    ┌─────────────┐
    │  Viewing    │───┐
    │  History    │   │
    └─────────────┘   │
                      │      ┌───────────────────────────────────────────────┐
    ┌─────────────┐   │      │              SPARK CLUSTER                    │
    │  Ratings    │───┼─────>│                                               │
    │             │   │      │  ┌─────────────────────────────────────────┐ │
    └─────────────┘   │      │  │         FEATURE ENGINEERING              │ │
                      │      │  │                                          │ │
    ┌─────────────┐   │      │  │  - User features (demographics, history)│ │
    │  Search     │───┼─────>│  │  - Item features (genre, cast, tags)    │ │
    │  Queries    │   │      │  │  - Context (time, device, location)     │ │
    └─────────────┘   │      │  │                                          │ │
                      │      │  └──────────────────┬──────────────────────┘ │
    ┌─────────────┐   │      │                     │                        │
    │  Browse     │───┘      │                     v                        │
    │  Behavior   │          │  ┌─────────────────────────────────────────┐ │
    └─────────────┘          │  │         ML MODELS (Offline)             │ │
                             │  │                                          │ │
                             │  │  - Collaborative Filtering              │ │
                             │  │  - Matrix Factorization                 │ │
                             │  │  - Deep Neural Networks                 │ │
                             │  │  - Contextual Bandits                   │ │
                             │  │                                          │ │
                             │  └──────────────────┬──────────────────────┘ │
                             │                     │                        │
                             └─────────────────────┼────────────────────────┘
                                                   │
                                                   v
                             ┌─────────────────────────────────────────────┐
                             │            CANDIDATE GENERATION             │
                             │                                             │
                             │  1. Generate 10,000 candidates per user    │
                             │  2. Score with lightweight model           │
                             │  3. Apply business rules                   │
                             │  4. Diversify results                      │
                             │                                             │
                             └──────────────────────┬──────────────────────┘
                                                    │
                                                    v
                             ┌─────────────────────────────────────────────┐
                             │              RANKING SERVICE                │
                             │                                             │
                             │  - Real-time personalization               │
                             │  - A/B test integration                    │
                             │  - Freshness boosting                      │
                             │                                             │
                             └──────────────────────┬──────────────────────┘
                                                    │
                                                    v
                             ┌─────────────────────────────────────────────┐
                             │               CLIENT DISPLAY                │
                             │                                             │
                             │  ┌─────────────────────────────────────┐   │
                             │  │  Because you watched Breaking Bad   │   │
                             │  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐│   │
                             │  │  │    │ │    │ │    │ │    │ │    ││   │
                             │  │  └────┘ └────┘ └────┘ └────┘ └────┘│   │
                             │  └─────────────────────────────────────┘   │
                             │                                             │
                             │  ┌─────────────────────────────────────┐   │
                             │  │  Trending Now                       │   │
                             │  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐│   │
                             │  │  │    │ │    │ │    │ │    │ │    ││   │
                             │  │  └────┘ └────┘ └────┘ └────┘ └────┘│   │
                             │  └─────────────────────────────────────┘   │
                             │                                             │
                             └─────────────────────────────────────────────┘
```

---

## Microservices Architecture

```
    SERVICE MESH (Zuul + Eureka)
    ────────────────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                              ZUUL GATEWAY                               │
    │                                                                         │
    │  - Dynamic routing                                                      │
    │  - Load balancing                                                       │
    │  - Authentication                                                       │
    │  - Rate limiting                                                        │
    │  - Request/Response filtering                                           │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    v                   v                   v
    ┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────────┐
    │    USER SERVICE       │ │   PLAYBACK SERVICE    │ │    RECO SERVICE       │
    │                       │ │                       │ │                       │
    │ - Authentication      │ │ - Stream manifest     │ │ - Personalization     │
    │ - Profile management  │ │ - License/DRM         │ │ - Row generation      │
    │ - Preferences         │ │ - Quality selection   │ │ - A/B testing         │
    │ - Watch history       │ │ - CDN selection       │ │ - Diversity           │
    │                       │ │                       │ │                       │
    │ Tech: Java + Cassandra│ │ Tech: Java + MySQL    │ │ Tech: Python + Spark  │
    └───────────────────────┘ └───────────────────────┘ └───────────────────────┘

                    ┌───────────────────────────────────────┐
                    │           EUREKA (Service Discovery)  │
                    │                                       │
                    │  - Service registration              │
                    │  - Health checks                     │
                    │  - Client-side load balancing        │
                    │                                       │
                    └───────────────────────────────────────┘
```

---

## Chaos Engineering

Netflix pioneered chaos engineering with the Simian Army:

```
    CHAOS ENGINEERING TOOLS
    ───────────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                           SIMIAN ARMY                                   │
    │                                                                         │
    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
    │  │  CHAOS MONKEY   │  │ LATENCY MONKEY  │  │ CONFORMITY      │        │
    │  │                 │  │                 │  │ MONKEY          │        │
    │  │ Randomly kills  │  │ Introduces      │  │ Finds instances │        │
    │  │ EC2 instances   │  │ artificial      │  │ that don't      │        │
    │  │ in production   │  │ delays          │  │ follow best     │        │
    │  │                 │  │                 │  │ practices       │        │
    │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │
    │                                                                         │
    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
    │  │ CHAOS GORILLA   │  │ CHAOS KONG      │  │ DOCTOR MONKEY   │        │
    │  │                 │  │                 │  │                 │        │
    │  │ Kills entire    │  │ Kills entire    │  │ Performs health │        │
    │  │ availability    │  │ AWS regions     │  │ checks and      │        │
    │  │ zones           │  │                 │  │ removes sick    │        │
    │  │                 │  │                 │  │ instances       │        │
    │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘

    PRINCIPLES
    ──────────
    1. Build a hypothesis around steady state
    2. Vary real-world events
    3. Run experiments in production
    4. Automate experiments to run continuously
    5. Minimize blast radius
```

---

## Data Storage

| Data Type | Storage | Scale |
|-----------|---------|-------|
| Video Files | S3 + Open Connect | 225 PB |
| User Data | Cassandra | 10+ PB |
| Viewing History | Cassandra | Billions of records |
| Billing | MySQL (Vitess) | Millions of transactions |
| Search Index | Elasticsearch | Billions of documents |
| Cache | EVCache (Redis) | Petabytes |
| Analytics | Druid + Spark | Petabytes |

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Global Subscribers | 230M+ |
| Countries | 190+ |
| Peak Concurrent Streams | 10M+ |
| Daily Video Hours | 1B+ |
| Original Content | 50% of library |
| Uptime | 99.99% |
| Start Time | < 2 seconds |

---

## Lessons Learned

1. **Everything fails** - Design for failure from day one
2. **Edge is king** - Move content as close to users as possible
3. **Personalization drives engagement** - 80% of content watched comes from recommendations
4. **Per-title encoding** - Optimize for content, not one-size-fits-all
5. **Chaos engineering** - Test failure in production before it happens to you
6. **Microservices enable velocity** - But require mature infrastructure

---

## References

- Netflix Tech Blog
- The Netflix Simian Army
- Open Connect Overview
- Zuul Gateway Documentation
