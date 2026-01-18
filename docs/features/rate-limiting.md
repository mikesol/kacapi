# Rate Limiting in kacapi

This document provides a comprehensive guide to rate limiting in kacapi, covering all supported algorithms, dimensions, tiered limits, and integration patterns. Rate limiting is a first-class feature in kacapi's code-first approach, allowing you to declare rate limit requirements directly in your endpoint decorators and have them automatically enforced at the gateway level.

## Table of Contents

- [Overview](#overview)
- [Rate Limiting Philosophy](#rate-limiting-philosophy)
- [Rate Limiting Algorithms](#rate-limiting-algorithms)
  - [Fixed Window](#fixed-window)
  - [Sliding Window](#sliding-window)
  - [Token Bucket](#token-bucket)
  - [Leaky Bucket](#leaky-bucket)
- [Rate Limiting Dimensions](#rate-limiting-dimensions)
  - [By User](#by-user)
  - [By Organization](#by-organization)
  - [By IP Address](#by-ip-address)
  - [By API Key](#by-api-key)
  - [By Custom Header](#by-custom-header)
  - [Global Limits](#global-limits)
- [Tiered Rate Limits](#tiered-rate-limits)
  - [Subscription-Based Tiers](#subscription-based-tiers)
  - [Dynamic Tier Resolution](#dynamic-tier-resolution)
  - [Tier Overrides](#tier-overrides)
- [Combining Multiple Rate Limits](#combining-multiple-rate-limits)
  - [AND Logic](#and-logic)
  - [OR Logic](#or-logic)
  - [Hierarchical Limits](#hierarchical-limits)
- [Distributed Rate Limiting](#distributed-rate-limiting)
  - [Storage Backends](#storage-backends)
  - [Consistency Models](#consistency-models)
  - [Performance Optimization](#performance-optimization)
- [Response Headers](#response-headers)
  - [Standard Rate Limit Headers](#standard-rate-limit-headers)
  - [Custom Headers](#custom-headers)
- [Rate Limit Exceeded Handling](#rate-limit-exceeded-handling)
  - [Error Responses](#error-responses)
  - [Retry-After Behavior](#retry-after-behavior)
  - [Custom Error Handlers](#custom-error-handlers)
- [Gateway-Specific Implementations](#gateway-specific-implementations)
  - [Kong](#kong)
  - [AWS API Gateway](#aws-api-gateway)
  - [CloudFront](#cloudfront)
  - [APISIX](#apisix)
- [Best Practices](#best-practices)
- [Advanced Patterns](#advanced-patterns)
- [Troubleshooting](#troubleshooting)

## Overview

kacapi rate limiting moves rate limit configuration from gateway consoles and config files into your codebase as decorators. When you deploy an endpoint with a rate limit decorator, kacapi:

1. **Discovers** the endpoint and extracts rate limit requirements
2. **Validates** the rate limit configuration against capacity policies
3. **Generates** gateway-specific rate limit configurations
4. **Deploys** the rate limit rules to your API gateway
5. **Injects** rate limit headers into responses

This approach eliminates configuration drift between your code and gateway, ensures consistent rate limit enforcement, and provides automatic documentation of capacity constraints.

### Quick Example

```python
from kacapi.modal import api_endpoint
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Limited to 100 requests per minute per user"}
```

That's it. No separate gateway configuration, no manual rate limit tracking, no header management. kacapi handles everything.

## Rate Limiting Philosophy

### Code as Source of Truth

Traditional API development separates rate limit configuration from code:

```
Code (endpoint logic) ←─→ Gateway Config (rate limits)
     ↓                           ↓
  Deployed                   Manually synced
```

This creates problems:
- **Drift**: Code and config diverge over time
- **Coordination**: Teams must coordinate changes across systems
- **Documentation**: Rate limits hidden in gateway consoles
- **Testing**: Local development can't replicate gateway limits

kacapi inverts this:

```
Code (with @api_endpoint(rate_limit=...)) → Deployed → Gateway Config Generated
                ↓
        Single Source of Truth
```

Benefits:
- **Single Source**: Rate limits live in code, gateway reflects code
- **Self-Documenting**: Decorators document capacity constraints
- **Type-Safe**: SDKs provide autocomplete and validation
- **Testable**: Local middleware matches gateway behavior

### Gateway-Agnostic Abstraction

kacapi provides a unified rate limiting interface that works across different gateways:

```python
# Same decorator works with Kong, AWS API Gateway, CloudFront, APISIX
@api_endpoint(rate_limit=FixedWindow(requests=100, window="1m"))
```

Gateway adapters translate this abstract declaration into gateway-specific configurations:

| Algorithm | Kong | AWS API Gateway | CloudFront | APISIX |
|-----------|------|-----------------|------------|--------|
| Fixed Window | rate-limiting plugin | Usage plan | Lambda@Edge | limit-count plugin |
| Sliding Window | rate-limiting-advanced | Lambda authorizer | Lambda@Edge | limit-count plugin |
| Token Bucket | rate-limiting plugin | API throttling | Lambda@Edge | limit-req plugin |
| Leaky Bucket | Custom plugin | Lambda authorizer | Lambda@Edge | Custom plugin |

You write rate limits once, deploy to any gateway.

## Rate Limiting Algorithms

kacapi supports four primary rate limiting algorithms, each with different characteristics and use cases.

### Fixed Window

Fixed window is the simplest rate limiting algorithm. It counts requests in fixed time windows (e.g., 0:00-0:59, 1:00-1:59).

#### Basic Usage

**Python:**
```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m"
    )
)
@modal.function()
def simple_rate_limit():
    return {"message": "100 requests per minute"}
```

**TypeScript:**
```typescript
import { apiEndpoint } from '@kacapi/vercel';
import { FixedWindow } from '@kacapi/rate-limit';

export default apiEndpoint({
  rateLimit: new FixedWindow({
    requests: 100,
    window: '1m'
  })
}, async (req, res) => {
  res.json({ message: '100 requests per minute' });
});
```

#### Configuration Options

```python
from kacapi.rate_limit import FixedWindow

fixed_window = FixedWindow(
    # Number of requests allowed (required)
    requests=100,
    
    # Time window (required)
    # Supported: "1s", "10s", "1m", "5m", "1h", "24h"
    window="1m",
    
    # Rate limit dimension (optional)
    by_user=False,
    by_org=False,
    by_ip=False,
    by_key=False,
    
    # Custom dimension (optional)
    by_header=None,  # e.g., "X-Client-Id"
    
    # Storage backend (optional)
    storage="redis",  # or "memory", "postgres", "dynamodb"
    
    # Redis configuration (if storage="redis")
    redis_url=None,  # Defaults to REDIS_URL env var
    redis_key_prefix="rl:",
    
    # Behavior on limit exceeded (optional)
    on_exceeded="reject",  # or "queue", "custom"
    
    # Custom error response (optional)
    error_response={
        "error": "Rate limit exceeded",
        "retry_after": "{{retry_after}}"
    },
    
    # Status code on limit exceeded (optional)
    error_status=429,
    
    # Include rate limit headers in response (optional)
    include_headers=True
)
```

#### Window Format

Windows can be specified in various formats:

```python
# Seconds
FixedWindow(requests=10, window="1s")
FixedWindow(requests=100, window="30s")

# Minutes
FixedWindow(requests=100, window="1m")
FixedWindow(requests=500, window="5m")

# Hours
FixedWindow(requests=1000, window="1h")
FixedWindow(requests=5000, window="6h")

# Days
FixedWindow(requests=10000, window="24h")
FixedWindow(requests=50000, window="7d")
```

#### Characteristics

**Advantages:**
- Simple to understand and implement
- Low memory overhead
- Fast performance
- Predictable behavior

**Disadvantages:**
- Burst traffic at window boundaries
- Clients can game the system by timing requests
- Not smooth traffic distribution

**Best For:**
- Public APIs with generous limits
- Internal APIs with predictable traffic
- Simple use cases where precision isn't critical

#### Example: Burst at Window Boundary

```
Window 1 (0:00-0:59): 100 requests at 0:59
Window 2 (1:00-1:59): 100 requests at 1:00
Total: 200 requests in 2 seconds (instead of 100/minute average)
```

### Sliding Window

Sliding window provides smoother rate limiting by maintaining a rolling window of request timestamps.

#### Basic Usage

**Python:**
```python
from kacapi.rate_limit import SlidingWindow

@api_endpoint(
    rate_limit=SlidingWindow(
        requests=100,
        window="1m"
    )
)
@modal.function()
def smooth_rate_limit():
    return {"message": "100 requests per sliding minute"}
```

**TypeScript:**
```typescript
import { SlidingWindow } from '@kacapi/rate-limit';

export default apiEndpoint({
  rateLimit: new SlidingWindow({
    requests: 100,
    window: '1m'
  })
}, async (req, res) => {
  res.json({ message: '100 requests per sliding minute' });
});
```

#### Configuration Options

```python
from kacapi.rate_limit import SlidingWindow

sliding_window = SlidingWindow(
    # Number of requests allowed (required)
    requests=100,
    
    # Time window (required)
    window="1m",
    
    # Implementation strategy (optional)
    strategy="sliding_log",  # or "sliding_counter"
    
    # Rate limit dimension (optional)
    by_user=False,
    by_org=False,
    by_ip=False,
    
    # Storage backend (required for distributed)
    storage="redis",
    redis_url=None,
    
    # Sliding log options
    # Max timestamps to store per key
    max_timestamps=1000,
    
    # Sliding counter options
    # Number of sub-windows
    sub_windows=10,
    
    # Memory optimization
    # Expire old entries after window + buffer
    expire_after="2m",
    
    # Behavior on limit exceeded
    on_exceeded="reject",
    error_status=429,
    include_headers=True
)
```

#### Implementation Strategies

##### Sliding Log

Stores individual request timestamps:

```python
@api_endpoint(
    rate_limit=SlidingWindow(
        requests=100,
        window="1m",
        strategy="sliding_log"
    )
)
```

**Advantages:**
- Precise rate limiting
- Exact request tracking

**Disadvantages:**
- Higher memory usage
- More expensive for high request rates

##### Sliding Counter

Approximates sliding window using sub-windows:

```python
@api_endpoint(
    rate_limit=SlidingWindow(
        requests=100,
        window="1m",
        strategy="sliding_counter",
        sub_windows=10  # 6-second sub-windows
    )
)
```

**Advantages:**
- Lower memory usage
- Better performance

**Disadvantages:**
- Slightly less precise
- Small approximation errors

#### Characteristics

**Advantages:**
- Smooth traffic distribution
- No burst at window boundaries
- More fair to clients
- Better protects backend services

**Disadvantages:**
- Higher memory usage
- More complex implementation
- Slightly higher latency

**Best For:**
- Production APIs with strict capacity constraints
- Services sensitive to burst traffic
- Fair rate limiting across clients

#### Example: No Burst

```
Time 0:00: 50 requests
Time 0:30: 50 requests
Time 1:00: Rejected (100 requests in last 60s)
Time 1:01: Allowed (first request from 0:00 expired)
```

### Token Bucket

Token bucket allows burst traffic while maintaining an average rate over time.

#### Basic Usage

**Python:**
```python
from kacapi.rate_limit import TokenBucket

@api_endpoint(
    rate_limit=TokenBucket(
        capacity=100,
        refill_rate=10,
        refill_interval="1s"
    )
)
@modal.function()
def burst_allowed_endpoint():
    return {"message": "Burst up to 100, refill 10/second"}
```

**TypeScript:**
```typescript
import { TokenBucket } from '@kacapi/rate-limit';

export default apiEndpoint({
  rateLimit: new TokenBucket({
    capacity: 100,
    refillRate: 10,
    refillInterval: '1s'
  })
}, async (req, res) => {
  res.json({ message: 'Burst up to 100, refill 10/second' });
});
```

#### Configuration Options

```python
from kacapi.rate_limit import TokenBucket

token_bucket = TokenBucket(
    # Bucket capacity (max burst) (required)
    capacity=100,
    
    # Refill rate (required)
    refill_rate=10,
    
    # Refill interval (required)
    refill_interval="1s",
    
    # Initial tokens (optional)
    initial_tokens=100,  # Defaults to capacity
    
    # Cost per request (optional)
    tokens_per_request=1,
    
    # Custom cost function (optional)
    cost_function=lambda request: calculate_cost(request),
    
    # Rate limit dimension
    by_user=False,
    by_ip=False,
    
    # Storage backend
    storage="redis",
    redis_url=None,
    
    # Behavior on insufficient tokens
    on_exceeded="reject",  # or "wait"
    wait_timeout="5s",  # If on_exceeded="wait"
    
    # Headers
    include_headers=True,
    include_bucket_stats=True  # X-RateLimit-Bucket-Tokens
)
```

#### Variable Request Costs

Different requests can consume different amounts of tokens:

```python
def calculate_request_cost(request):
    """Calculate token cost based on request complexity"""
    if request.path.endswith("/search"):
        return 5  # Search is expensive
    elif request.path.endswith("/batch"):
        batch_size = len(request.json.get("items", []))
        return max(1, batch_size // 10)  # Cost based on batch size
    return 1  # Default cost

@api_endpoint(
    rate_limit=TokenBucket(
        capacity=100,
        refill_rate=10,
        refill_interval="1s",
        cost_function=calculate_request_cost
    )
)
@modal.function()
def variable_cost_endpoint():
    return {"message": "Variable rate limit costs"}
```

#### Characteristics

**Advantages:**
- Allows controlled burst traffic
- Maintains long-term average rate
- Flexible (variable costs per request)
- Smooth for bursty workloads

**Disadvantages:**
- More complex to understand
- Requires careful capacity tuning
- Can allow unexpected bursts if misconfigured

**Best For:**
- APIs with bursty traffic patterns
- Services that can handle short bursts
- Variable-cost operations (search, batch APIs)

#### Example: Burst Then Throttle

```
Initial: 100 tokens
Request 1-100: All succeed immediately (100 tokens consumed)
Request 101: Wait for refill
1 second later: 10 tokens refilled
Request 101-110: Succeed
Request 111: Wait for next refill
```

### Leaky Bucket

Leaky bucket smooths traffic by enforcing a constant outflow rate, queueing excess requests.

#### Basic Usage

**Python:**
```python
from kacapi.rate_limit import LeakyBucket

@api_endpoint(
    rate_limit=LeakyBucket(
        capacity=100,
        leak_rate=10,
        leak_interval="1s"
    )
)
@modal.function()
def smooth_traffic_endpoint():
    return {"message": "Smooth traffic at 10 req/s"}
```

**TypeScript:**
```typescript
import { LeakyBucket } from '@kacapi/rate-limit';

export default apiEndpoint({
  rateLimit: new LeakyBucket({
    capacity: 100,
    leakRate: 10,
    leakInterval: '1s'
  })
}, async (req, res) => {
  res.json({ message: 'Smooth traffic at 10 req/s' });
});
```

#### Configuration Options

```python
from kacapi.rate_limit import LeakyBucket

leaky_bucket = LeakyBucket(
    # Bucket capacity (max queue) (required)
    capacity=100,
    
    # Leak rate (requests processed) (required)
    leak_rate=10,
    
    # Leak interval (required)
    leak_interval="1s",
    
    # Queue behavior (optional)
    queue_timeout="30s",  # Max time request waits in queue
    queue_strategy="fifo",  # or "lifo", "priority"
    
    # Rate limit dimension
    by_user=False,
    by_ip=False,
    
    # Storage backend
    storage="redis",
    redis_url=None,
    
    # Behavior on queue full
    on_queue_full="reject",  # or "drop_oldest"
    error_status=429,
    
    # Queue position headers
    include_queue_position=True  # X-RateLimit-Queue-Position
)
```

#### Priority Queue

Process high-priority requests first:

```python
def get_request_priority(request):
    """Determine request priority"""
    if request.headers.get("X-Priority") == "high":
        return 10
    elif request.headers.get("X-User-Tier") == "enterprise":
        return 5
    return 1  # Normal priority

@api_endpoint(
    rate_limit=LeakyBucket(
        capacity=100,
        leak_rate=10,
        leak_interval="1s",
        queue_strategy="priority",
        priority_function=get_request_priority
    )
)
@modal.function()
def priority_queue_endpoint():
    return {"message": "Priority-based queuing"}
```

#### Characteristics

**Advantages:**
- Constant, predictable output rate
- Smooths burst traffic
- Prevents backend overload
- Can queue requests (better UX than rejection)

**Disadvantages:**
- Adds latency (queuing delay)
- Requires queue management
- More complex state management
- Not suitable for real-time APIs

**Best For:**
- Background job APIs
- Batch processing endpoints
- Services sensitive to traffic spikes
- Non-real-time operations

#### Example: Queue and Process

```
Time 0.0s: 50 requests arrive, queued
Time 0.1s: 50 more requests arrive, queued (100 total)
Time 0.1-10s: Process at 10 req/s
Time 10s: Queue empty, all requests processed
```

## Rate Limiting Dimensions

Rate limits can be applied across different dimensions to control access granularly.

### By User

Limit requests per authenticated user.

```python
from kacapi.auth import Clerk
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    auth=Clerk(),
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True
    )
)
@modal.function()
def per_user_limit():
    return {"message": "100 req/min per user"}
```

**Key Header**: `X-Kacapi-User-Id` (injected by auth connector)

**Use Cases:**
- Prevent individual user abuse
- Fair usage across users
- User-specific quotas

### By Organization

Limit requests per organization (multi-tenant apps).

```python
from kacapi.auth import Clerk
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    auth=Clerk(claims=["org_id"]),
    rate_limit=FixedWindow(
        requests=1000,
        window="1m",
        by_org=True
    )
)
@modal.function()
def per_org_limit():
    return {"message": "1000 req/min per organization"}
```

**Key Header**: `X-Kacapi-Org-Id` (injected by auth connector)

**Use Cases:**
- B2B SaaS applications
- Organization-level quotas
- Tenant isolation

### By IP Address

Limit requests per client IP address.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=10,
        window="1m",
        by_ip=True
    )
)
@modal.function()
def per_ip_limit():
    return {"message": "10 req/min per IP"}
```

**Key Header**: `X-Forwarded-For` (extracted by gateway)

**Use Cases:**
- Public APIs without authentication
- DDoS protection
- Prevent scraping/abuse

**Important**: Consider proxy/NAT scenarios where many users share one IP.

### By API Key

Limit requests per API key.

```python
from kacapi.auth import APIKey
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    auth=APIKey(key_validator=validate_key),
    rate_limit=FixedWindow(
        requests=1000,
        window="1h",
        by_key=True
    )
)
@modal.function()
def per_key_limit():
    return {"message": "1000 req/hour per API key"}
```

**Key Header**: `X-Kacapi-Key-Id` (injected by API key auth)

**Use Cases:**
- Third-party integrations
- Partner APIs
- Key-specific quotas

### By Custom Header

Limit requests by any custom header.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_header="X-Client-Id"
    )
)
@modal.function()
def per_client_limit():
    return {"message": "100 req/min per client ID"}
```

**Use Cases:**
- Custom client identification
- Mobile app versioning (by app version header)
- Geographic regions (by X-Region header)

### Global Limits

Apply rate limit globally across all requests.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=10000,
        window="1m"
    )
)
@modal.function()
def global_limit():
    return {"message": "10000 req/min globally"}
```

**Use Cases:**
- Backend capacity protection
- Emergency throttling
- Cost control (e.g., expensive LLM APIs)

## Tiered Rate Limits

Different users or customers often require different rate limits based on subscription tiers.

### Subscription-Based Tiers

Define rate limits that vary by subscription tier.

#### Basic Usage

**Python:**
```python
from kacapi.auth import Clerk
from kacapi.rate_limit import TieredLimit, FixedWindow

@api_endpoint(
    auth=Clerk(claims=["subscription_tier"]),
    rate_limit=TieredLimit(
        tier_key="X-Kacapi-Subscription-Tier",
        tiers={
            "free": FixedWindow(requests=100, window="1h"),
            "pro": FixedWindow(requests=1000, window="1h"),
            "enterprise": FixedWindow(requests=10000, window="1h")
        },
        default_tier="free"
    )
)
@modal.function()
def tiered_endpoint():
    return {"message": "Tier-specific rate limits"}
```

**TypeScript:**
```typescript
import { TieredLimit, FixedWindow } from '@kacapi/rate-limit';

export default apiEndpoint({
  auth: new Clerk({ claims: ['subscription_tier'] }),
  rateLimit: new TieredLimit({
    tierKey: 'X-Kacapi-Subscription-Tier',
    tiers: {
      free: new FixedWindow({ requests: 100, window: '1h' }),
      pro: new FixedWindow({ requests: 1000, window: '1h' }),
      enterprise: new FixedWindow({ requests: 10000, window: '1h' })
    },
    defaultTier: 'free'
  })
}, async (req, res) => {
  res.json({ message: 'Tier-specific rate limits' });
});
```

#### Configuration Options

```python
from kacapi.rate_limit import TieredLimit, FixedWindow, TokenBucket

tiered_limit = TieredLimit(
    # Header containing tier information (required)
    tier_key="X-Kacapi-Subscription-Tier",
    
    # Tier definitions (required)
    tiers={
        "free": FixedWindow(requests=100, window="1h", by_user=True),
        "pro": TokenBucket(capacity=1000, refill_rate=20, refill_interval="1m", by_user=True),
        "enterprise": FixedWindow(requests=10000, window="1h", by_user=True)
    },
    
    # Default tier for unknown values (optional)
    default_tier="free",
    
    # Fallback behavior if tier header missing (optional)
    on_missing_tier="use_default",  # or "reject", "allow"
    
    # Tier resolution function (optional)
    tier_resolver=None  # Custom function to resolve tier
)
```

### Dynamic Tier Resolution

Resolve tier from database or external service.

```python
from kacapi.rate_limit import TieredLimit, FixedWindow
import asyncpg

async def resolve_user_tier(headers):
    """Look up user tier from database"""
    user_id = headers.get("X-Kacapi-User-Id")
    
    async with asyncpg.create_pool(DATABASE_URL) as pool:
        async with pool.acquire() as conn:
            result = await conn.fetchrow(
                "SELECT subscription_tier FROM users WHERE id = $1",
                user_id
            )
            return result["subscription_tier"] if result else "free"

@api_endpoint(
    auth=Clerk(),
    rate_limit=TieredLimit(
        tier_key="X-Kacapi-Subscription-Tier",
        tier_resolver=resolve_user_tier,
        tiers={
            "free": FixedWindow(requests=100, window="1h", by_user=True),
            "pro": FixedWindow(requests=1000, window="1h", by_user=True),
            "enterprise": FixedWindow(requests=10000, window="1h", by_user=True)
        }
    )
)
@modal.function()
def dynamic_tier_endpoint():
    return {"message": "Database-backed tier resolution"}
```

### Tier Overrides

Override rate limits for specific users.

```python
from kacapi.rate_limit import TieredLimit, FixedWindow

@api_endpoint(
    auth=Clerk(),
    rate_limit=TieredLimit(
        tier_key="X-Kacapi-Subscription-Tier",
        tiers={
            "free": FixedWindow(requests=100, window="1h", by_user=True),
            "pro": FixedWindow(requests=1000, window="1h", by_user=True)
        },
        # Specific user overrides
        overrides={
            "user_123": FixedWindow(requests=5000, window="1h"),  # VIP user
            "user_456": FixedWindow(requests=50, window="1h")     # Rate limited user
        },
        override_key="X-Kacapi-User-Id"
    )
)
@modal.function()
def override_endpoint():
    return {"message": "User-specific overrides"}
```

## Combining Multiple Rate Limits

Apply multiple rate limits simultaneously for fine-grained control.

### AND Logic

All rate limits must be satisfied.

```python
from kacapi.rate_limit import CombinedLimit, FixedWindow

@api_endpoint(
    rate_limit=CombinedLimit(
        mode="all",  # All limits must be satisfied
        limits=[
            FixedWindow(requests=10, window="1s", by_user=True),    # Burst protection
            FixedWindow(requests=100, window="1m", by_user=True),   # Per-minute limit
            FixedWindow(requests=1000, window="1h", by_user=True)   # Hourly quota
        ]
    )
)
@modal.function()
def multi_limit_endpoint():
    return {"message": "Multiple rate limits enforced"}
```

**Use Case**: Prevent both burst attacks and quota exhaustion.

### OR Logic

Any rate limit can be satisfied.

```python
from kacapi.rate_limit import CombinedLimit, FixedWindow

@api_endpoint(
    rate_limit=CombinedLimit(
        mode="any",  # Any limit satisfied = allowed
        limits=[
            FixedWindow(requests=1000, window="1h", by_user=True),
            FixedWindow(requests=10000, window="1h", by_org=True)
        ]
    )
)
@modal.function()
def flexible_limit_endpoint():
    return {"message": "Either user or org limit"}
```

**Use Case**: Allow either individual or organizational quota.

### Hierarchical Limits

Combine global and dimension-specific limits.

```python
from kacapi.rate_limit import CombinedLimit, FixedWindow

@api_endpoint(
    rate_limit=CombinedLimit(
        mode="all",
        limits=[
            # Global limit (protect backend)
            FixedWindow(requests=10000, window="1m"),
            
            # Per-IP limit (prevent DDoS)
            FixedWindow(requests=100, window="1m", by_ip=True),
            
            # Per-user limit (fair usage)
            FixedWindow(requests=500, window="1m", by_user=True)
        ]
    )
)
@modal.function()
def hierarchical_limit_endpoint():
    return {"message": "Hierarchical rate limiting"}
```

**Example Scenario**:
1. Global limit prevents total traffic from exceeding 10k req/min
2. Per-IP limit prevents single IP from consuming >100 req/min
3. Per-user limit ensures no authenticated user exceeds 500 req/min

## Distributed Rate Limiting

For multi-instance deployments, rate limiting requires distributed state management.

### Storage Backends

kacapi supports multiple storage backends for distributed rate limiting.

#### Redis

Best for most distributed scenarios.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True,
        storage="redis",
        redis_url=os.environ["REDIS_URL"],
        redis_options={
            "max_connections": 50,
            "socket_timeout": 0.1,
            "socket_connect_timeout": 0.1,
            "retry_on_timeout": True
        }
    )
)
@modal.function()
def redis_backed_limit():
    return {"message": "Redis-backed distributed rate limiting"}
```

**Advantages:**
- Fast (sub-millisecond latency)
- Built-in TTL support
- Atomic operations
- High throughput

**Disadvantages:**
- Additional infrastructure dependency
- Network latency
- Single point of failure (use Redis Cluster)

#### PostgreSQL

Use existing database for rate limiting.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True,
        storage="postgres",
        postgres_url=os.environ["DATABASE_URL"],
        postgres_table="rate_limits"
    )
)
@modal.function()
def postgres_backed_limit():
    return {"message": "PostgreSQL-backed rate limiting"}
```

**Advantages:**
- No additional infrastructure
- ACID guarantees
- Persistent storage

**Disadvantages:**
- Slower than Redis (10-50ms latency)
- Higher database load
- Requires schema management

#### DynamoDB

AWS-native distributed rate limiting.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True,
        storage="dynamodb",
        dynamodb_table="RateLimits",
        dynamodb_region="us-east-1"
    )
)
@modal.function()
def dynamodb_backed_limit():
    return {"message": "DynamoDB-backed rate limiting"}
```

**Advantages:**
- Serverless, auto-scaling
- Pay-per-use
- Global replication

**Disadvantages:**
- Higher latency than Redis
- Cost at scale
- Eventual consistency (configurable)

#### In-Memory

Single-instance only (not distributed).

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True,
        storage="memory"
    )
)
@modal.function()
def memory_backed_limit():
    return {"message": "In-memory rate limiting"}
```

**Use Cases:**
- Development/testing
- Single-instance deployments
- Edge computing (Cloudflare Workers)

### Consistency Models

Different consistency models trade off accuracy for performance.

#### Strong Consistency

Exact rate limiting, slower performance.

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="redis",
        consistency="strong"
    )
)
```

**Characteristics:**
- Guaranteed accurate counts
- Higher latency (synchronous)
- Better for strict limits

#### Eventual Consistency

Approximate rate limiting, better performance.

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="redis",
        consistency="eventual"
    )
)
```

**Characteristics:**
- Slight over-limit allowed
- Lower latency (async replication)
- Better for loose limits

#### Best-Effort

Fastest, least accurate.

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="memory",
        consistency="best-effort"
    )
)
```

**Characteristics:**
- Per-instance counting
- No distributed state
- Useful for edge deployments

### Performance Optimization

#### Connection Pooling

Reuse connections to storage backend.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="redis",
        redis_url=os.environ["REDIS_URL"],
        redis_pool_size=50,
        redis_pool_timeout=0.1
    )
)
```

#### Lua Scripts (Redis)

Use Redis Lua scripts for atomic operations.

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="redis",
        use_lua_scripts=True  # Faster atomic operations
    )
)
```

#### Local Caching

Cache rate limit state locally for faster checks.

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="redis",
        local_cache=True,
        local_cache_ttl=1  # Cache for 1 second
    )
)
```

**Trade-off**: Slightly less accurate, much faster.

## Response Headers

kacapi injects standard rate limit headers into responses for client consumption.

### Standard Rate Limit Headers

Following the IETF draft standard:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1640000000
X-RateLimit-Window: 60
X-RateLimit-Policy: 100;w=60;by=user
Retry-After: 18
```

#### Header Descriptions

**`X-RateLimit-Limit`**: Maximum requests allowed in the window.

**`X-RateLimit-Remaining`**: Requests remaining in current window.

**`X-RateLimit-Reset`**: Unix timestamp when the window resets.

**`X-RateLimit-Window`**: Window duration in seconds.

**`X-RateLimit-Policy`**: Human-readable rate limit policy.

**`Retry-After`**: Seconds until the client can retry (when rate limited).

### Custom Headers

Configure custom header names.

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        include_headers=True,
        header_prefix="X-App-RateLimit-",  # Custom prefix
        headers={
            "limit": "Limit",
            "remaining": "Remaining",
            "reset": "Reset"
        }
    )
)
@modal.function()
def custom_headers_endpoint():
    return {"message": "Custom rate limit headers"}
```

**Response**:
```
X-App-RateLimit-Limit: 100
X-App-RateLimit-Remaining: 42
X-App-RateLimit-Reset: 1640000000
```

### Algorithm-Specific Headers

Different algorithms expose additional headers.

#### Token Bucket

```
X-RateLimit-Bucket-Capacity: 100
X-RateLimit-Bucket-Tokens: 42
X-RateLimit-Bucket-Refill-Rate: 10
X-RateLimit-Bucket-Refill-Interval: 1
```

#### Leaky Bucket

```
X-RateLimit-Queue-Capacity: 100
X-RateLimit-Queue-Size: 42
X-RateLimit-Queue-Position: 15
X-RateLimit-Queue-Wait-Time: 5
```

## Rate Limit Exceeded Handling

Customize behavior when rate limits are exceeded.

### Error Responses

Default error response:

```json
{
  "error": "Rate limit exceeded",
  "message": "You have exceeded the rate limit of 100 requests per minute",
  "retry_after": 42,
  "limit": 100,
  "window": "1m",
  "policy": "100;w=60;by=user"
}
```

**Status Code**: `429 Too Many Requests`

### Custom Error Responses

```python
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        error_response={
            "error": {
                "code": "RATE_LIMIT_EXCEEDED",
                "message": "Too many requests. Please slow down.",
                "retry_after": "{{retry_after}}",
                "docs": "https://docs.example.com/rate-limits"
            }
        },
        error_status=429
    )
)
@modal.function()
def custom_error_endpoint():
    return {"message": "Custom error responses"}
```

### Retry-After Behavior

**Retry-After Header**: Tells clients when to retry.

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        include_retry_after=True,
        retry_after_format="seconds"  # or "timestamp"
    )
)
```

**Response**:
```
HTTP/1.1 429 Too Many Requests
Retry-After: 42
X-RateLimit-Reset: 1640000000
```

### Custom Error Handlers

Handle rate limit exceeded with custom logic.

```python
from kacapi.rate_limit import FixedWindow

async def on_rate_limit_exceeded(request, limit_info):
    """Custom handler for rate limit exceeded"""
    user_id = request.headers.get("X-Kacapi-User-Id")
    
    # Log to analytics
    await log_rate_limit_event(user_id, limit_info)
    
    # Send notification
    if limit_info["remaining"] == 0:
        await notify_user(user_id, "Rate limit reached")
    
    # Return custom response
    return {
        "error": "Please upgrade your plan for higher limits",
        "upgrade_url": "https://example.com/upgrade",
        "retry_after": limit_info["retry_after"]
    }

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        on_exceeded_handler=on_rate_limit_exceeded
    )
)
@modal.function()
def custom_handler_endpoint():
    return {"message": "Custom rate limit handling"}
```

## Gateway-Specific Implementations

kacapi translates rate limit decorators into gateway-specific configurations.

### Kong

Kong uses the rate-limiting plugin.

#### Generated Configuration

```yaml
# From @api_endpoint(rate_limit=FixedWindow(requests=100, window="1m", by_user=True))
services:
  - name: my-service
    routes:
      - name: my-route
        paths: [/api/endpoint]
        plugins:
          - name: rate-limiting
            config:
              minute: 100
              policy: redis
              redis_host: redis.example.com
              redis_port: 6379
              redis_database: 0
              header_name: X-Kacapi-User-Id
              hide_client_headers: false
```

#### Algorithm Mapping

| kacapi | Kong Plugin |
|--------|-------------|
| FixedWindow | rate-limiting |
| SlidingWindow | rate-limiting-advanced |
| TokenBucket | Custom (rate-limiting with burst) |
| LeakyBucket | Custom plugin required |

### AWS API Gateway

AWS API Gateway uses usage plans and throttling.

#### Generated Configuration

```yaml
# From @api_endpoint(rate_limit=FixedWindow(requests=100, window="1m"))
UsagePlan:
  Type: AWS::ApiGateway::UsagePlan
  Properties:
    UsagePlanName: MyApiUsagePlan
    Throttle:
      RateLimit: 100
      BurstLimit: 200
    Quota:
      Limit: 10000
      Period: DAY
    ApiStages:
      - ApiId: !Ref MyApi
        Stage: prod
```

#### Lambda Authorizer

For dimension-specific limits (by_user, by_org):

```typescript
// Generated Lambda authorizer
export const handler = async (event: APIGatewayAuthorizerEvent) => {
  const userId = extractUserId(event.headers.Authorization);
  
  // Check rate limit in Redis/DynamoDB
  const rateLimitOk = await checkRateLimit(userId, {
    requests: 100,
    window: 60
  });
  
  if (!rateLimitOk) {
    throw new Error('Rate limit exceeded');
  }
  
  return {
    principalId: userId,
    policyDocument: generateAllowPolicy(event.methodArn),
    context: { userId }
  };
};
```

### CloudFront

CloudFront uses Lambda@Edge for rate limiting.

#### Generated Lambda@Edge Function

```typescript
// Generated Lambda@Edge viewer-request function
export const handler = async (event: CloudFrontRequestEvent) => {
  const request = event.Records[0].cf.request;
  const userId = request.headers['x-kacapi-user-id']?.[0]?.value;
  
  if (!userId) {
    return errorResponse(401, 'Unauthorized');
  }
  
  // Check rate limit in DynamoDB Global Tables
  const rateLimitKey = `rl:user:${userId}:minute`;
  const count = await incrementCounter(rateLimitKey, 60);
  
  if (count > 100) {
    return errorResponse(429, 'Rate limit exceeded', {
      'X-RateLimit-Limit': '100',
      'X-RateLimit-Remaining': '0',
      'Retry-After': '60'
    });
  }
  
  // Add rate limit headers
  request.headers['x-ratelimit-limit'] = [{ value: '100' }];
  request.headers['x-ratelimit-remaining'] = [{ value: String(100 - count) }];
  
  return request;
};
```

### APISIX

APISIX uses limit-count and limit-req plugins.

#### Generated Configuration

```yaml
# From @api_endpoint(rate_limit=FixedWindow(requests=100, window="1m", by_user=True))
routes:
  - uri: /api/endpoint
    plugins:
      limit-count:
        count: 100
        time_window: 60
        key_type: var
        key: http_x_kacapi_user_id
        rejected_code: 429
        rejected_msg: "Rate limit exceeded"
        policy: redis
        redis_host: redis.example.com
        redis_port: 6379
        redis_database: 0
```

## Best Practices

### 1. Choose the Right Algorithm

**Fixed Window**: Simple public APIs, generous limits  
**Sliding Window**: Production APIs, strict enforcement  
**Token Bucket**: Bursty traffic, variable costs  
**Leaky Bucket**: Batch processing, constant backend load

### 2. Set Appropriate Limits

```python
# ✅ GOOD - Reasonable limits
@api_endpoint(
    rate_limit=FixedWindow(requests=1000, window="1h", by_user=True)
)

# ❌ BAD - Too restrictive
@api_endpoint(
    rate_limit=FixedWindow(requests=10, window="1h", by_user=True)
)
```

### 3. Combine Dimensions

Protect against multiple attack vectors.

```python
@api_endpoint(
    rate_limit=CombinedLimit(
        mode="all",
        limits=[
            FixedWindow(requests=10000, window="1m"),           # Global
            FixedWindow(requests=100, window="1m", by_ip=True), # Per-IP
            FixedWindow(requests=500, window="1m", by_user=True) # Per-user
        ]
    )
)
```

### 4. Use Tiered Limits

Reward paying customers with higher limits.

```python
@api_endpoint(
    rate_limit=TieredLimit(
        tier_key="X-Kacapi-Subscription-Tier",
        tiers={
            "free": FixedWindow(requests=100, window="1h", by_user=True),
            "pro": FixedWindow(requests=1000, window="1h", by_user=True),
            "enterprise": FixedWindow(requests=10000, window="1h", by_user=True)
        }
    )
)
```

### 5. Always Include Headers

Help clients manage their usage.

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        include_headers=True,  # ✅ Always true
        include_retry_after=True
    )
)
```

### 6. Monitor Rate Limit Hits

Track when users hit limits.

```python
from kacapi.observability import Metrics

@api_endpoint(
    rate_limit=FixedWindow(requests=100, window="1m", by_user=True),
    metrics=Metrics(
        track_rate_limit_hits=True,
        alert_on_frequent_hits=True,
        hit_threshold=10  # Alert if user hits limit >10 times/day
    )
)
```

### 7. Test Rate Limits

Validate behavior under load.

```python
import pytest
from kacapi.testing import RateLimitTestClient

async def test_rate_limit():
    client = RateLimitTestClient(
        rate_limit=FixedWindow(requests=10, window="1m")
    )
    
    # First 10 requests should succeed
    for i in range(10):
        response = await client.request("/api/endpoint")
        assert response.status == 200
    
    # 11th request should be rate limited
    response = await client.request("/api/endpoint")
    assert response.status == 429
    assert "Retry-After" in response.headers
```

## Advanced Patterns

### Adaptive Rate Limiting

Adjust limits based on backend load.

```python
from kacapi.rate_limit import AdaptiveLimit

async def get_current_capacity():
    """Query backend for current capacity"""
    load = await get_backend_load()
    if load > 0.9:
        return 100  # Reduce under high load
    elif load > 0.7:
        return 500
    return 1000  # Normal capacity

@api_endpoint(
    rate_limit=AdaptiveLimit(
        base_limit=1000,
        window="1m",
        capacity_function=get_current_capacity,
        check_interval="10s"
    )
)
@modal.function()
def adaptive_endpoint():
    return {"message": "Adaptive rate limiting"}
```

### Quota Management

Track long-term quotas (daily/monthly).

```python
from kacapi.rate_limit import QuotaLimit

@api_endpoint(
    rate_limit=QuotaLimit(
        quota=100000,  # 100k requests per month
        period="30d",
        by_user=True,
        reset_day=1,  # Reset on 1st of month
        warn_threshold=0.8  # Warn at 80% usage
    )
)
@modal.function()
def quota_endpoint():
    return {"message": "Monthly quota tracking"}
```

### Rate Limit Exemptions

Exempt specific users from rate limits.

```python
from kacapi.rate_limit import FixedWindow

def is_exempt(headers):
    """Check if user is exempt from rate limits"""
    user_id = headers.get("X-Kacapi-User-Id")
    return user_id in ["admin_user", "service_account"]

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True,
        exempt_function=is_exempt
    )
)
@modal.function()
def exempt_endpoint():
    return {"message": "Exemptions supported"}
```

### Cost-Based Rate Limiting

Charge different costs for different operations.

```python
from kacapi.rate_limit import CostBasedLimit

def calculate_cost(request):
    """Calculate request cost based on complexity"""
    if "search" in request.path:
        query_length = len(request.query_params.get("q", ""))
        return max(1, query_length // 10)
    elif "batch" in request.path:
        batch_size = len(request.json.get("items", []))
        return batch_size
    return 1

@api_endpoint(
    rate_limit=CostBasedLimit(
        budget=1000,  # Total budget
        window="1h",
        by_user=True,
        cost_function=calculate_cost
    )
)
@modal.function()
def cost_based_endpoint():
    return {"message": "Cost-based rate limiting"}
```

## Troubleshooting

### Common Issues

#### 1. Rate Limits Not Enforced

**Problem**: Requests not being rate limited

**Solutions**:
- Verify gateway deployment succeeded
- Check storage backend connectivity (Redis, etc.)
- Validate dimension key headers are present
- Test with explicit user/IP values

```python
# Debug: Log rate limit checks
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True,
        debug=True  # Log all rate limit checks
    )
)
```

#### 2. Inconsistent Counts

**Problem**: Rate limit counts don't match expectations

**Solutions**:
- Check consistency model (strong vs. eventual)
- Verify clock synchronization across instances
- Inspect storage backend for stale data
- Use sliding window instead of fixed window

#### 3. High Latency

**Problem**: Rate limiting adds significant latency

**Solutions**:
- Enable local caching
- Use connection pooling
- Consider best-effort consistency
- Move to faster storage backend (Redis)

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="redis",
        local_cache=True,  # Cache locally
        local_cache_ttl=1,
        redis_pool_size=50
    )
)
```

#### 4. Redis Connection Errors

**Problem**: Rate limiting fails due to Redis errors

**Solutions**:
- Implement fallback behavior
- Add connection retry logic
- Use Redis Sentinel/Cluster for HA
- Configure connection timeouts

```python
@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        storage="redis",
        redis_url=os.environ["REDIS_URL"],
        redis_options={
            "socket_timeout": 0.1,
            "socket_connect_timeout": 0.1,
            "retry_on_timeout": True,
            "max_retries": 3
        },
        on_storage_error="allow"  # Allow requests if Redis down
    )
)
```

#### 5. Burst Traffic at Window Boundaries

**Problem**: Fixed window allows bursts at boundaries

**Solution**: Use sliding window instead

```python
# ❌ PROBLEM: Fixed window
@api_endpoint(
    rate_limit=FixedWindow(requests=100, window="1m")
)

# ✅ SOLUTION: Sliding window
@api_endpoint(
    rate_limit=SlidingWindow(requests=100, window="1m")
)
```

### Debug Mode

Enable comprehensive debugging:

```python
from kacapi.rate_limit import FixedWindow
from kacapi.observability import Logging

@api_endpoint(
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_user=True,
        debug=True
    ),
    logging=Logging(
        level="DEBUG",
        log_rate_limit_events=True,
        log_headers=True
    )
)
@modal.function()
def debug_endpoint():
    pass
```

### Testing Rate Limits Locally

```python
from kacapi.testing import MockGateway, RateLimitTestClient
from kacapi.rate_limit import FixedWindow

# Test rate limit behavior
async def test_rate_limit():
    test_client = RateLimitTestClient(
        rate_limit=FixedWindow(requests=10, window="1m"),
        storage="memory"  # Use in-memory for tests
    )
    
    # Simulate requests
    for i in range(10):
        response = await test_client.request("/api/endpoint")
        assert response.status == 200
        remaining = int(response.headers["X-RateLimit-Remaining"])
        assert remaining == 9 - i
    
    # Should be rate limited
    response = await test_client.request("/api/endpoint")
    assert response.status == 429
```

---

## Conclusion

kacapi rate limiting provides powerful, flexible, and distributed rate limiting capabilities with a simple decorator-based API. By declaring rate limits in code alongside your endpoints, you eliminate configuration drift, improve documentation, and enable automated gateway deployment.

Key takeaways:

1. **Code-first**: Rate limits live in decorators, not separate configs
2. **Algorithm choice**: Fixed/Sliding window, Token/Leaky bucket
3. **Multi-dimensional**: By user, org, IP, key, or custom header
4. **Tiered limits**: Different limits for different subscription tiers
5. **Combinable**: Apply multiple limits with AND/OR logic
6. **Distributed**: Redis, PostgreSQL, DynamoDB backends
7. **Observable**: Standard headers, custom error responses
8. **Gateway-agnostic**: Works with Kong, AWS API Gateway, CloudFront, APISIX
9. **Performant**: Local caching, connection pooling, Lua scripts
10. **Testable**: Comprehensive testing utilities

For more information, see:
- [Authentication](./authentication.md) - Combine auth with rate limiting
- [Caching](./caching.md) - Reduce backend load
- [Gateway Adapters](../adapters/) - Gateway-specific implementations
- [Observatory](../observatory/) - Monitor rate limit usage patterns
