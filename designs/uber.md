# Uber - Ride Sharing Platform

## Overview

Design a ride-sharing platform that matches riders with drivers in real-time, handles millions of trips daily, and provides accurate ETAs and pricing across global markets.

---

## Requirements

### Functional
- Rider can request a ride from current location to destination
- Match rider with nearby available drivers
- Real-time driver location tracking
- Dynamic pricing based on supply/demand
- ETA calculation for pickup and trip
- Payment processing
- Rating system for drivers and riders
- Trip history

### Non-Functional
- Low latency matching (< 1 second)
- High availability (99.99%)
- Accurate location tracking (< 5 meter precision)
- Handle 20M+ trips per day
- Real-time updates every 4 seconds

---

## Scale Estimation

```
Metrics:
- Daily trips: 20 million
- Active drivers: 5 million
- Active riders: 100 million
- Peak concurrent requests: 1 million
- Location updates: 5M drivers * 1 update/4s = 1.25M updates/sec

Storage:
- Trip data: 20M trips * 10KB = 200 GB/day
- Location history: 1.25M * 100 bytes * 86400 = 10 TB/day
- User data: 105M users * 5KB = 500 GB

Bandwidth:
- Location updates: 1.25M * 100 bytes = 125 MB/sec
```

---

## High-Level Architecture

```
    ┌─────────────────────────────────────────────────────────────────────────────────────┐
    │                                    CLIENTS                                          │
    │                                                                                     │
    │      ┌──────────────┐                                    ┌──────────────┐          │
    │      │  RIDER APP   │                                    │  DRIVER APP  │          │
    │      │              │                                    │              │          │
    │      │ - Request    │                                    │ - Go online  │          │
    │      │ - Track      │                                    │ - Accept     │          │
    │      │ - Pay        │                                    │ - Navigate   │          │
    │      │ - Rate       │                                    │ - Earn       │          │
    │      └──────┬───────┘                                    └──────┬───────┘          │
    │             │                                                   │                   │
    └─────────────┼───────────────────────────────────────────────────┼───────────────────┘
                  │                                                   │
                  │              ┌───────────────────┐                │
                  │              │   API GATEWAY     │                │
                  └──────────────┤                   ├────────────────┘
                                 │ - Authentication │
                                 │ - Rate Limiting  │
                                 │ - Load Balancing │
                                 └────────┬──────────┘
                                          │
          ┌───────────────┬───────────────┼───────────────┬───────────────┐
          │               │               │               │               │
          v               v               v               v               v
    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │   User   │    │  Trip    │    │  Match   │    │ Location │    │  Pricing │
    │ Service  │    │ Service  │    │ Service  │    │ Service  │    │ Service  │
    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
          │               │               │               │               │
          │               │               │               │               │
    ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
    │  Users DB │   │  Trips DB │   │   Redis   │   │  Cassandra│   │  Pricing  │
    │ (MySQL)   │   │ (MySQL)   │   │  Cluster  │   │  Cluster  │   │   Rules   │
    └───────────┘   └───────────┘   └───────────┘   └───────────┘   └───────────┘
```

---

## Geospatial Indexing

The core challenge is efficiently finding nearby drivers. Uber uses a combination of approaches:

### Google S2 Geometry

```
    S2 CELL HIERARCHY
    ─────────────────

    Level 0: Entire Earth (6 faces of a cube)
    Level 12: ~6.4 km x 6.4 km cells
    Level 14: ~1.6 km x 1.6 km cells (city blocks)
    Level 16: ~400m x 400m cells (neighborhoods)

    ┌─────────────────────────────────────────────────────────────┐
    │                     CITY VIEW (Level 12)                    │
    │                                                             │
    │    ┌─────────┬─────────┬─────────┬─────────┬─────────┐     │
    │    │  89c2a  │  89c2b  │  89c2c  │  89c2d  │  89c2e  │     │
    │    ├─────────┼─────────┼─────────┼─────────┼─────────┤     │
    │    │  89c2f  │  89c30  │  89c31  │  89c32  │  89c33  │     │
    │    ├─────────┼─────────┼─────────┼─────────┼─────────┤     │
    │    │  89c34  │  89c35  │  89c36★ │  89c37  │  89c38  │     │
    │    ├─────────┼─────────┼─────────┼─────────┼─────────┤     │
    │    │  89c39  │  89c3a  │  89c3b  │  89c3c  │  89c3d  │     │
    │    └─────────┴─────────┴─────────┴─────────┴─────────┘     │
    │                                                             │
    │    ★ = Rider location                                      │
    │    Search expands to neighboring cells                      │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘

    ZOOMED VIEW (Level 16)
    ──────────────────────

    ┌─────────────────────────────────────────────────────────────┐
    │                     CELL 89c36 (Level 16)                   │
    │                                                             │
    │    ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐       │
    │    │     │     │     │  D  │     │     │     │     │       │
    │    ├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤       │
    │    │     │  D  │     │     │     │     │  D  │     │       │
    │    ├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤       │
    │    │     │     │     │     │  ★  │     │     │     │       │
    │    ├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤       │
    │    │  D  │     │     │     │     │     │     │  D  │       │
    │    └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘       │
    │                                                             │
    │    ★ = Rider    D = Available Driver                       │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

### Implementation

```python
from s2geometry import S2CellId, S2LatLng, S2RegionCoverer

class LocationIndex:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.level = 16  # ~400m cells

    def update_driver_location(self, driver_id: str, lat: float, lng: float):
        """Update driver's position in the spatial index"""
        cell_id = self._get_cell_id(lat, lng)

        # Remove from old cell
        old_cell = self.redis.hget(f"driver:{driver_id}", "cell")
        if old_cell:
            self.redis.srem(f"cell:{old_cell}", driver_id)

        # Add to new cell
        self.redis.sadd(f"cell:{cell_id}", driver_id)
        self.redis.hset(f"driver:{driver_id}", mapping={
            "cell": cell_id,
            "lat": lat,
            "lng": lng,
            "updated_at": time.time()
        })

    def find_nearby_drivers(
        self,
        lat: float,
        lng: float,
        radius_km: float = 5.0,
        limit: int = 10
    ) -> list:
        """Find drivers within radius using S2 covering"""
        center = S2LatLng.from_degrees(lat, lng)
        cells = self._get_covering_cells(center, radius_km)

        candidates = []
        for cell in cells:
            driver_ids = self.redis.smembers(f"cell:{cell.id()}")
            for driver_id in driver_ids:
                driver_data = self.redis.hgetall(f"driver:{driver_id}")
                if driver_data:
                    distance = self._haversine_distance(
                        lat, lng,
                        float(driver_data['lat']),
                        float(driver_data['lng'])
                    )
                    if distance <= radius_km:
                        candidates.append({
                            'driver_id': driver_id,
                            'distance': distance,
                            'lat': driver_data['lat'],
                            'lng': driver_data['lng']
                        })

        # Sort by distance and return top N
        candidates.sort(key=lambda x: x['distance'])
        return candidates[:limit]

    def _get_cell_id(self, lat: float, lng: float) -> str:
        latlng = S2LatLng.from_degrees(lat, lng)
        cell = S2CellId(latlng).parent(self.level)
        return cell.to_token()

    def _get_covering_cells(self, center: S2LatLng, radius_km: float) -> list:
        # Create a cap (spherical circle) around the center
        cap = S2Cap.from_axis_height(
            center.to_point(),
            self._km_to_angle(radius_km)
        )

        coverer = S2RegionCoverer()
        coverer.set_min_level(self.level)
        coverer.set_max_level(self.level)

        return coverer.get_covering(cap)
```

---

## Real-Time Dispatch System

```
    DISPATCH FLOW
    ─────────────

    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   RIDER     │     │   SUPPLY    │     │   DEMAND    │     │   DISPATCH  │
    │   REQUEST   │────>│   SERVICE   │────>│   SERVICE   │────>│   ENGINE    │
    │             │     │             │     │             │     │             │
    └─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                       │
                        ┌──────────────────────────────────────────────┘
                        │
                        v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         MATCHING ALGORITHM                              │
    │                                                                         │
    │  1. Query nearby available drivers (S2 index)                          │
    │  2. Calculate ETA for each driver (routing service)                    │
    │  3. Score each driver-rider pair:                                      │
    │                                                                         │
    │     Score = w1 * ETA_score                                             │
    │           + w2 * driver_rating                                         │
    │           + w3 * acceptance_rate                                       │
    │           + w4 * trip_value_match                                      │
    │           + w5 * driver_destination_alignment                          │
    │                                                                         │
    │  4. Apply constraints:                                                  │
    │     - Driver vehicle type matches request                              │
    │     - Driver not in cool-down period                                   │
    │     - Driver preferences (no highways, etc.)                           │
    │                                                                         │
    │  5. Select optimal match                                               │
    │                                                                         │
    └────────────────────────────────────────────────────────────────────────┘
                        │
                        v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                         DISPATCH DECISION                               │
    │                                                                         │
    │  ┌───────────────────┐                     ┌───────────────────┐       │
    │  │    Send offer     │                     │   No match found  │       │
    │  │    to driver      │                     │   (expand radius) │       │
    │  └─────────┬─────────┘                     └───────────────────┘       │
    │            │                                                            │
    │            v                                                            │
    │  ┌───────────────────┐     ┌───────────────────┐                       │
    │  │  Driver accepts   │     │  Driver declines  │                       │
    │  │  (within 15 sec)  │     │  or times out     │                       │
    │  └─────────┬─────────┘     └─────────┬─────────┘                       │
    │            │                         │                                  │
    │            v                         v                                  │
    │  ┌───────────────────┐     ┌───────────────────┐                       │
    │  │   CREATE TRIP     │     │  TRY NEXT DRIVER  │                       │
    │  │   Notify rider    │     │  or expand search │                       │
    │  └───────────────────┘     └───────────────────┘                       │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## Dynamic Pricing (Surge)

```
    SURGE PRICING MODEL
    ───────────────────

    Supply/Demand Calculation per Zone:

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │  SUPPLY (S)                        DEMAND (D)                          │
    │  ─────────                         ──────────                          │
    │  - Active drivers in zone          - Open ride requests                │
    │  - Predicted arrivals              - Predicted requests                │
    │  - Historical patterns             - Event data                        │
    │                                    - Weather impact                    │
    │                                                                         │
    │                                                                         │
    │  SURGE MULTIPLIER                                                       │
    │  ─────────────────                                                      │
    │                                                                         │
    │  if D/S < 0.5:   multiplier = 1.0                                      │
    │  if D/S < 1.0:   multiplier = 1.0 + (D/S - 0.5) * 0.5                 │
    │  if D/S < 2.0:   multiplier = 1.25 + (D/S - 1.0) * 0.5                │
    │  if D/S < 4.0:   multiplier = 1.75 + (D/S - 2.0) * 0.375              │
    │  else:           multiplier = min(2.5 + (D/S - 4.0) * 0.25, 5.0)      │
    │                                                                         │
    │                                                                         │
    │  Price                                                                  │
    │    │                                                                    │
    │  5x│                                            ●●●●                   │
    │    │                                      ●●●●                         │
    │  4x│                                 ●●●                               │
    │    │                            ●●●                                    │
    │  3x│                        ●●                                         │
    │    │                    ●●                                             │
    │  2x│               ●●●                                                 │
    │    │          ●●●                                                      │
    │  1x│●●●●●●●●●                                                          │
    │    └────────────────────────────────────────────── D/S ratio           │
    │         1       2       3       4       5       6                      │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘

    ZONE EXAMPLE
    ────────────

    ┌─────────┬─────────┬─────────┐
    │  1.2x   │  1.0x   │  1.5x   │
    │ Airport │ Suburbs │Downtown │
    ├─────────┼─────────┼─────────┤
    │  1.0x   │  2.3x   │  1.8x   │
    │         │ Stadium │ Bars    │
    │         │ (Event) │ (Night) │
    ├─────────┼─────────┼─────────┤
    │  1.0x   │  1.0x   │  1.2x   │
    │         │         │         │
    └─────────┴─────────┴─────────┘
```

---

## ETA Prediction

```
    ETA CALCULATION PIPELINE
    ────────────────────────

    ┌─────────────┐
    │   INPUT     │
    │             │
    │ - Origin    │
    │ - Dest      │
    │ - Time      │
    │ - Day       │
    │             │
    └──────┬──────┘
           │
           v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                        ROUTING ENGINE                                   │
    │                                                                         │
    │  ┌────────────────────────────────────────────────────────────────┐    │
    │  │                    GRAPH STRUCTURE                             │    │
    │  │                                                                │    │
    │  │   Road network as directed weighted graph:                     │    │
    │  │   - Nodes: Intersections                                      │    │
    │  │   - Edges: Road segments                                      │    │
    │  │   - Weights: Travel time (dynamic)                            │    │
    │  │                                                                │    │
    │  │   ┌───┐     5min      ┌───┐     3min      ┌───┐              │    │
    │  │   │ A │──────────────>│ B │──────────────>│ C │              │    │
    │  │   └───┘               └───┘               └───┘              │    │
    │  │     │                   │                   │                 │    │
    │  │     │ 2min              │ 4min              │ 2min            │    │
    │  │     │                   │                   │                 │    │
    │  │     v                   v                   v                 │    │
    │  │   ┌───┐     6min      ┌───┐     2min      ┌───┐              │    │
    │  │   │ D │──────────────>│ E │──────────────>│ F │              │    │
    │  │   └───┘               └───┘               └───┘              │    │
    │  │                                                                │    │
    │  └────────────────────────────────────────────────────────────────┘    │
    │                                                                         │
    │  ALGORITHM: Contraction Hierarchies + A*                               │
    │  - Precompute shortcuts for fast queries                               │
    │  - Query time: < 1ms for city-scale graphs                            │
    │                                                                         │
    └────────────────────────────────────────────────────────────────────────┘
           │
           v
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                      TRAFFIC LAYER                                      │
    │                                                                         │
    │  Real-time edge weight updates:                                         │
    │                                                                         │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │  SOURCE              PROCESSING           UPDATE                │   │
    │  │                                                                  │   │
    │  │  Driver GPS ────────> Speed calc ────────> Edge weights         │   │
    │  │  (1.25M/sec)         (Spark Streaming)    (Every 60 sec)        │   │
    │  │                                                                  │   │
    │  │  Historical ────────> ML prediction ─────> Future weights       │   │
    │  │  patterns            (Traffic forecast)   (Next 1 hour)         │   │
    │  │                                                                  │   │
    │  │  External ──────────> Event impact ──────> Temporary            │   │
    │  │  (Events, weather)   estimation           adjustments           │   │
    │  │                                                                  │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                         │
    └────────────────────────────────────────────────────────────────────────┘
           │
           v
    ┌─────────────┐
    │   OUTPUT    │
    │             │
    │ - ETA       │
    │ - Route     │
    │ - Distance  │
    │             │
    └─────────────┘
```

---

## Trip State Machine

```
    TRIP LIFECYCLE
    ──────────────

    ┌──────────────┐
    │   CREATED    │
    │              │
    │ Rider        │
    │ requests     │
    └──────┬───────┘
           │
           v
    ┌──────────────┐     Timeout or      ┌──────────────┐
    │  MATCHING    │────────────────────>│  CANCELLED   │
    │              │     Cancel          │              │
    │ Finding      │                     │              │
    │ driver       │                     │              │
    └──────┬───────┘                     └──────────────┘
           │
           │ Driver accepts
           v
    ┌──────────────┐     Driver/Rider    ┌──────────────┐
    │  ACCEPTED    │────────────────────>│  CANCELLED   │
    │              │     cancels         │              │
    │ Driver       │                     │ Apply fees   │
    │ en route     │                     │ if needed    │
    └──────┬───────┘                     └──────────────┘
           │
           │ Driver arrives
           v
    ┌──────────────┐     No show         ┌──────────────┐
    │   ARRIVED    │────────────────────>│  CANCELLED   │
    │              │     (after 5 min)   │              │
    │ Waiting      │                     │ Charge       │
    │ for rider    │                     │ no-show fee  │
    └──────┬───────┘                     └──────────────┘
           │
           │ Rider enters vehicle
           v
    ┌──────────────┐
    │ IN_PROGRESS  │
    │              │
    │ Trip         │
    │ ongoing      │
    └──────┬───────┘
           │
           │ Arrive destination
           v
    ┌──────────────┐
    │  COMPLETED   │
    │              │
    │ Process      │
    │ payment      │
    └──────┬───────┘
           │
           v
    ┌──────────────┐
    │    RATED     │
    │              │
    │ Both parties │
    │ rate         │
    └──────────────┘
```

---

## Data Storage

| Data Type | Storage | Rationale |
|-----------|---------|-----------|
| User Profiles | MySQL (Vitess) | ACID, relational queries |
| Trip Data | MySQL (Vitess) | Transactional consistency |
| Location Stream | Kafka + Cassandra | High write throughput |
| Geospatial Index | Redis Cluster | Low latency reads |
| Analytics | HDFS + Hive | Batch processing |
| ML Features | Cassandra | Fast feature retrieval |
| Search | Elasticsearch | Full-text queries |
| Caching | Redis | Session, hot data |

---

## Reliability Patterns

```
    MULTI-REGION ARCHITECTURE
    ─────────────────────────

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                           GLOBAL LAYER                                  │
    │                                                                         │
    │  ┌─────────────────┐              ┌─────────────────┐                  │
    │  │   DNS (Route53) │              │    Global LB    │                  │
    │  │   Geo-routing   │──────────────│                 │                  │
    │  └─────────────────┘              └────────┬────────┘                  │
    │                                            │                            │
    └────────────────────────────────────────────┼────────────────────────────┘
                                                 │
              ┌──────────────────────────────────┼──────────────────────────────┐
              │                                  │                              │
              v                                  v                              v
    ┌─────────────────────┐          ┌─────────────────────┐          ┌─────────────────────┐
    │     US-WEST-2       │          │     US-EAST-1       │          │     EU-WEST-1       │
    │                     │          │                     │          │                     │
    │  ┌───────────────┐  │          │  ┌───────────────┐  │          │  ┌───────────────┐  │
    │  │   Services    │  │          │  │   Services    │  │          │  │   Services    │  │
    │  └───────┬───────┘  │          │  └───────┬───────┘  │          │  └───────┬───────┘  │
    │          │          │          │          │          │          │          │          │
    │  ┌───────┴───────┐  │          │  ┌───────┴───────┐  │          │  ┌───────┴───────┐  │
    │  │    MySQL      │  │          │  │    MySQL      │  │          │  │    MySQL      │  │
    │  │   (Primary)   │◄─┼──────────┼─►│   (Replica)   │◄─┼──────────┼─►│   (Replica)   │  │
    │  └───────────────┘  │  Async   │  └───────────────┘  │  Async   │  └───────────────┘  │
    │                     │  Repl    │                     │  Repl    │                     │
    │  ┌───────────────┐  │          │  ┌───────────────┐  │          │  ┌───────────────┐  │
    │  │    Redis      │  │          │  │    Redis      │  │          │  │    Redis      │  │
    │  │   (Local)     │  │          │  │   (Local)     │  │          │  │   (Local)     │  │
    │  └───────────────┘  │          │  └───────────────┘  │          │  └───────────────┘  │
    │                     │          │                     │          │                     │
    └─────────────────────┘          └─────────────────────┘          └─────────────────────┘

    FAILOVER: If US-WEST-2 fails:
    1. DNS removes US-WEST-2
    2. Traffic routes to US-EAST-1
    3. US-EAST-1 promoted to primary
    4. RTO < 5 minutes
```

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Daily Trips | 20M+ |
| Cities | 900+ |
| Countries | 70+ |
| Active Drivers | 5M+ |
| Matching Time | < 1 second |
| Location Updates | 1.25M/sec |
| Peak Requests | 1M concurrent |

---

## Lessons Learned

1. **Geospatial indexing is critical** - S2 geometry enables efficient nearby searches
2. **Real-time requires specialized infrastructure** - Kafka + Redis for high-frequency updates
3. **Dynamic pricing balances supply/demand** - But requires careful communication
4. **Multi-region is mandatory** - Users expect 100% availability
5. **ML enhances everything** - ETA, pricing, matching, fraud detection
6. **Mobile is the interface** - Optimize for battery, bandwidth, intermittent connectivity

---

## References

- Uber Engineering Blog
- Uber H3 Geospatial Index
- Ringpop for distributed systems
- Schemaless (Uber's MySQL sharding layer)
