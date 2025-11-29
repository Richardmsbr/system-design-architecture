# Circuit Breaker Pattern

## Problem Statement

When a service depends on another service that becomes slow or unavailable, the calling service can exhaust its resources (threads, connections) waiting for responses that never come. This leads to cascading failures across the system.

---

## Solution

Implement a circuit breaker that monitors for failures and "trips" when a threshold is exceeded, immediately failing subsequent requests without attempting the operation.

---

## State Machine

```
                              ┌─────────────────────────────────────────────────────────────┐
                              │                                                             │
                              │                    CIRCUIT BREAKER                          │
                              │                                                             │
                              │    ┌──────────────────────────────────────────────────┐    │
                              │    │                                                  │    │
                              │    │                    CLOSED                        │    │
                              │    │              (Normal Operation)                  │    │
                              │    │                                                  │    │
                              │    │  - All requests pass through                    │    │
                              │    │  - Track success/failure counts                 │    │
                              │    │  - Reset counts after time window               │    │
                              │    │                                                  │    │
                              │    └────────────────────┬─────────────────────────────┘    │
                              │                         │                                   │
                              │                         │ Failure threshold exceeded        │
                              │                         │                                   │
                              │                         v                                   │
                              │    ┌──────────────────────────────────────────────────┐    │
                              │    │                                                  │    │
                              │    │                     OPEN                         │    │
                              │    │              (Fail Fast Mode)                    │    │
                              │    │                                                  │    │
                              │    │  - All requests fail immediately                │    │
                              │    │  - No calls to downstream service               │    │
                              │    │  - Return fallback or error                     │    │
                              │    │                                                  │    │
                              │    └────────────────────┬─────────────────────────────┘    │
                              │                         │                                   │
                              │                         │ Timeout expires                   │
                              │                         │                                   │
                              │                         v                                   │
                              │    ┌──────────────────────────────────────────────────┐    │
                              │    │                                                  │    │
        Success               │    │                  HALF-OPEN                       │    │
     ┌────────────────────────│────│               (Testing Mode)                     │    │
     │                        │    │                                                  │    │
     │                        │    │  - Allow limited test requests                  │    │
     │                        │    │  - If success → CLOSED                          │    │
     │                        │    │  - If failure → OPEN                            │    │
     │                        │    │                                                  │    │
     │                        │    └────────────────────┬─────────────────────────────┘    │
     │                        │                         │                                   │
     │                        │                         │ Failure                           │
     │                        │                         │                                   │
     │                        └─────────────────────────┴───────────────────────────────────┘
     │                                                  │
     │                                                  │
     └──────────────────────────────────────────────────┘
```

---

## Implementation

```python
import time
from enum import Enum
from threading import Lock
from typing import Callable, TypeVar, Optional
from dataclasses import dataclass

T = TypeVar('T')

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5
    success_threshold: int = 3
    timeout: float = 30.0
    half_open_max_calls: int = 3


class CircuitBreaker:
    def __init__(self, config: CircuitBreakerConfig = None):
        self.config = config or CircuitBreakerConfig()
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = 0
        self.half_open_calls = 0
        self.lock = Lock()

    def call(self, func: Callable[[], T], fallback: Callable[[], T] = None) -> T:
        with self.lock:
            if self.state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self.state = CircuitState.HALF_OPEN
                    self.half_open_calls = 0
                else:
                    return self._handle_open_circuit(fallback)

            if self.state == CircuitState.HALF_OPEN:
                if self.half_open_calls >= self.config.half_open_max_calls:
                    return self._handle_open_circuit(fallback)
                self.half_open_calls += 1

        try:
            result = func()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            if fallback:
                return fallback()
            raise

    def _should_attempt_reset(self) -> bool:
        return time.time() - self.last_failure_time >= self.config.timeout

    def _on_success(self):
        with self.lock:
            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.config.success_threshold:
                    self._reset()
            else:
                self.failure_count = 0

    def _on_failure(self):
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()

            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.OPEN
            elif self.failure_count >= self.config.failure_threshold:
                self.state = CircuitState.OPEN

    def _reset(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.half_open_calls = 0

    def _handle_open_circuit(self, fallback: Callable[[], T]) -> T:
        if fallback:
            return fallback()
        raise CircuitOpenError("Circuit breaker is open")


class CircuitOpenError(Exception):
    pass
```

---

## Usage Example

```python
# Configuration
config = CircuitBreakerConfig(
    failure_threshold=5,      # Open after 5 failures
    success_threshold=3,      # Close after 3 successes in half-open
    timeout=30.0,             # Try again after 30 seconds
    half_open_max_calls=3     # Allow 3 test calls in half-open
)

circuit_breaker = CircuitBreaker(config)

# Service call with fallback
def get_user_data(user_id: str) -> dict:
    return circuit_breaker.call(
        func=lambda: external_service.get_user(user_id),
        fallback=lambda: {"user_id": user_id, "cached": True, "data": cache.get(user_id)}
    )
```

---

## Integration with HTTP Client

```python
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

class ResilientHttpClient:
    def __init__(self, base_url: str, circuit_breaker: CircuitBreaker):
        self.client = httpx.Client(base_url=base_url, timeout=10.0)
        self.circuit_breaker = circuit_breaker

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10)
    )
    def _make_request(self, method: str, path: str, **kwargs) -> httpx.Response:
        response = self.client.request(method, path, **kwargs)
        response.raise_for_status()
        return response

    def get(self, path: str, fallback=None, **kwargs) -> httpx.Response:
        return self.circuit_breaker.call(
            func=lambda: self._make_request("GET", path, **kwargs),
            fallback=fallback
        )

    def post(self, path: str, fallback=None, **kwargs) -> httpx.Response:
        return self.circuit_breaker.call(
            func=lambda: self._make_request("POST", path, **kwargs),
            fallback=fallback
        )
```

---

## Metrics and Monitoring

```python
from prometheus_client import Counter, Gauge, Histogram

# Metrics
circuit_state = Gauge(
    'circuit_breaker_state',
    'Current state of circuit breaker',
    ['service']
)

circuit_calls_total = Counter(
    'circuit_breaker_calls_total',
    'Total calls through circuit breaker',
    ['service', 'result']  # result: success, failure, rejected
)

circuit_state_transitions = Counter(
    'circuit_breaker_transitions_total',
    'Circuit breaker state transitions',
    ['service', 'from_state', 'to_state']
)


class ObservableCircuitBreaker(CircuitBreaker):
    def __init__(self, service_name: str, config: CircuitBreakerConfig = None):
        super().__init__(config)
        self.service_name = service_name

    def _on_success(self):
        circuit_calls_total.labels(self.service_name, 'success').inc()
        super()._on_success()
        self._update_state_metric()

    def _on_failure(self):
        old_state = self.state
        circuit_calls_total.labels(self.service_name, 'failure').inc()
        super()._on_failure()
        self._record_transition(old_state)
        self._update_state_metric()

    def _handle_open_circuit(self, fallback):
        circuit_calls_total.labels(self.service_name, 'rejected').inc()
        return super()._handle_open_circuit(fallback)

    def _update_state_metric(self):
        state_values = {
            CircuitState.CLOSED: 0,
            CircuitState.HALF_OPEN: 1,
            CircuitState.OPEN: 2
        }
        circuit_state.labels(self.service_name).set(state_values[self.state])

    def _record_transition(self, old_state):
        if old_state != self.state:
            circuit_state_transitions.labels(
                self.service_name,
                old_state.value,
                self.state.value
            ).inc()
```

---

## Configuration Guidelines

| Parameter | Low Sensitivity | Medium | High Sensitivity |
|-----------|-----------------|--------|------------------|
| failure_threshold | 10 | 5 | 3 |
| success_threshold | 5 | 3 | 1 |
| timeout | 60s | 30s | 10s |
| half_open_max_calls | 5 | 3 | 1 |

### Factors to Consider

1. Service criticality
2. Downstream service SLA
3. Recovery time expectations
4. Fallback availability
5. Request volume

---

## Combined with Other Patterns

```
    REQUEST
       │
       v
    ┌─────────────────┐
    │   RATE LIMITER  │──── Exceeded ──> 429
    └────────┬────────┘
             │
             v
    ┌─────────────────┐
    │ CIRCUIT BREAKER │──── Open ──> Fallback
    └────────┬────────┘
             │
             v
    ┌─────────────────┐
    │     TIMEOUT     │──── Exceeded ──> Fallback
    └────────┬────────┘
             │
             v
    ┌─────────────────┐
    │      RETRY      │──── Max Retries ──> Error
    │  (with backoff) │
    └────────┬────────┘
             │
             v
    ┌─────────────────┐
    │    BULKHEAD     │──── Saturated ──> 503
    │   (Semaphore)   │
    └────────┬────────┘
             │
             v
    ┌─────────────────┐
    │ DOWNSTREAM CALL │
    └─────────────────┘
```

---

## Distributed Circuit Breaker

For microservices, share circuit state across instances:

```python
class DistributedCircuitBreaker:
    def __init__(self, redis_client, service_name: str, config: CircuitBreakerConfig):
        self.redis = redis_client
        self.service_name = service_name
        self.config = config
        self.key_prefix = f"circuit:{service_name}"

    async def get_state(self) -> CircuitState:
        state = await self.redis.get(f"{self.key_prefix}:state")
        return CircuitState(state) if state else CircuitState.CLOSED

    async def record_failure(self):
        pipe = self.redis.pipeline()
        pipe.incr(f"{self.key_prefix}:failures")
        pipe.expire(f"{self.key_prefix}:failures", 60)
        results = await pipe.execute()

        failure_count = results[0]
        if failure_count >= self.config.failure_threshold:
            await self._trip_circuit()

    async def _trip_circuit(self):
        await self.redis.setex(
            f"{self.key_prefix}:state",
            self.config.timeout,
            CircuitState.OPEN.value
        )
```

---

## References

- Martin Fowler - Circuit Breaker
- Release It! by Michael Nygard
- Netflix Hystrix (archived but influential)
- Resilience4j documentation
