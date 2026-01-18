# Core Concepts: Overview

This document introduces the foundational philosophy behind kacapi and explains why it takes a radically different approach to API gateway management.

## Table of Contents

- [The Traditional Approach](#the-traditional-approach)
- [The kacapi Philosophy](#the-kacapi-philosophy)
- [Core Tenets](#core-tenets)
- [Why This Matters](#why-this-matters)
- [Trade-offs](#trade-offs)

## The Traditional Approach

Traditional API gateway management follows a **spec-first** workflow:

```
1. Design API (write OpenAPI spec)
2. Configure gateway (add routes, auth, rate limiting)
3. Implement backend endpoints
4. Deploy
5. Keep spec and gateway in sync manually
```

### Problems with This Approach

**1. Dual Source of Truth**

You have two places where the API is defined:
- The OpenAPI spec (what the API should be)
- The actual code (what the API actually is)

These inevitably drift apart.

**2. Manual Synchronization**

Every time you:
- Add an endpoint
- Change authentication requirements
- Modify rate limits
- Update CORS settings

You must update both code AND gateway configuration. Humans forget.

**3. Stale Specifications**

OpenAPI specs become outdated the moment they're written. Teams lack time/discipline to keep them current.

**4. Configuration Sprawl**

Gateway configuration lives in different places:
- Kong: YAML files
- AWS API Gateway: CloudFormation templates
- CloudFront: CDK code
- APISIX: YAML files

Migrating gateways means rewriting all configuration.

**5. Testing Challenges**

How do you test that your gateway config matches your code? Usually, you don't. Issues discovered in production.

## The kacapi Philosophy

kacapi inverts the traditional approach:

```
1. Implement backend endpoints with decorators
2. Deploy to platform (Modal, Vercel, etc.)
3. kacapi discovers endpoints automatically
4. kacapi generates gateway configuration
5. kacapi deploys to gateway
6. Specs emerge from observed reality
```

### Key Insight: Code as Single Source of Truth

In kacapi, **deployed code is the only source of truth**. Everything else is derived:

```
Deployed Code
    ↓
Endpoint Discovery
    ↓
Gateway Configuration
    ↓
OpenAPI Specification
```

If it's not deployed, it doesn't exist. If it is deployed, kacapi will find it.

## Core Tenets

### 1. **Code-First, Not Config-First**

Gateway policies are expressed as decorators in code:

```python
@api_endpoint(
    auth=Clerk(),
    rate_limit=FixedWindow(requests=100, window="1m"),
    cache=Cache(ttl=300)
)
@modal.function()
def my_endpoint():
    pass
```

Not as separate configuration:

```yaml
# ❌ Traditional approach
routes:
  - path: /my-endpoint
    auth:
      provider: clerk
    rate_limit:
      requests: 100
      window: 1m
```

**Why?** Code and configuration stay in sync because they're in the same place.

### 2. **Emergent, Not Prescriptive**

Specifications **emerge** from observing deployed code:

```
Code Deployed → kacapi Observes → Spec Generated
```

Not prescribed upfront:

```
Spec Written → Code Generated → Code Deployed
```

**Why?** Eliminates stale specs. The spec always reflects reality.

### 3. **Declarative, Not Imperative**

Decorators declare **what** you want, not **how** to achieve it:

```python
# ✅ Declarative: "I want authentication"
@api_endpoint(auth=Clerk())

# ❌ Imperative: "Call Clerk API with these parameters"
@api_endpoint(
    before_request=lambda: clerk.verify(request.headers['Authorization'])
)
```

Gateway adapters translate declarations to gateway-specific implementations.

**Why?** Same decorators work with Kong, AWS API Gateway, CloudFront, etc.

### 4. **Platform-Aware**

kacapi only supports platforms with **discoverable URLs**:
- Modal: Can list all functions via API
- Vercel: Can list all deployments via API
- Supabase: Can query edge functions
- Cloudflare: Can list workers

kacapi does **not** support:
- Traditional VMs (no endpoint enumeration)
- Docker containers without orchestration
- Self-hosted servers without service discovery

**Why?** If we can't discover endpoints programmatically, we can't manage them automatically.

### 5. **Gateway-Agnostic**

Your decorators don't mention Kong, AWS API Gateway, or CloudFront. They describe features abstractly:

```python
@api_endpoint(
    auth=Clerk(),  # Not "Kong JWT plugin"
    rate_limit=FixedWindow(...)  # Not "AWS usage plan"
)
```

Gateway adapters translate to specific implementations:
- Kong adapter → Kong plugins
- AWS adapter → API Gateway authorizers
- CloudFront adapter → Lambda@Edge functions

**Why?** Switch gateways without changing code.

### 6. **Observable by Default**

All gateway interactions are logged, traced, and metered automatically:

```python
@api_endpoint(
    observability=Observability(
        log_body=True,
        trace_sampling=0.1,
        metrics=["latency", "error_rate"]
    )
)
```

**Why?** Enables traffic analysis, dormancy detection, and optimization suggestions.

### 7. **Change-Aware**

kacapi tracks changes over time:

```
Snapshot 1 (Monday)    Snapshot 2 (Tuesday)
- GET /users           - GET /users
- POST /users          - POST /users (auth added)
                       - GET /users/{id} (new)
```

The Observatory detects:
- New endpoints
- Modified endpoints (breaking vs. non-breaking)
- Removed endpoints

**Why?** Automatic versioning and breaking change detection.

## Why This Matters

### For Developers

**Before kacapi:**
```bash
# 1. Update code
vim api.py

# 2. Update gateway config
vim kong.yaml

# 3. Deploy code
git push

# 4. Deploy gateway config
kubectl apply -f kong.yaml

# 5. Update OpenAPI spec
vim openapi.yaml

# 6. Regenerate SDK
openapi-generator generate ...
```

**With kacapi:**
```bash
# 1. Update code with decorators
vim api.py

# 2. Deploy code
git push

# Done. kacapi handles the rest automatically.
```

### For Platform Teams

**Before kacapi:**
- Manually review gateway config changes
- Hunt for stale endpoints
- Manually enforce policies ("all endpoints need auth")
- Manually generate OpenAPI specs
- Answer "what changed?" questions by diffing YAML

**With kacapi:**
- Governance rules enforce policies automatically
- Traffic analysis identifies stale endpoints
- Change detection tracks all modifications
- Specs generated automatically
- Observatory provides rich change history

### For Security Teams

**Before kacapi:**
- Gateway config and code can be inconsistent
- Hard to audit what's actually deployed
- Manual checks for security policies

**With kacapi:**
- Gateway config guaranteed to match deployed code
- Observatory provides complete audit trail
- Governor enforces security policies before deployment

## Trade-offs

kacapi makes deliberate trade-offs:

### ✅ Gains

1. **Single source of truth**: Code is authoritative
2. **Automatic sync**: Gateway config always matches code
3. **Change tracking**: Complete history of modifications
4. **Multi-gateway**: Same code works with different gateways
5. **Policy enforcement**: Automated governance

### ❌ Limitations

1. **Platform constraints**: Only works with discoverable platforms
2. **Eventual consistency**: Slight lag between deployment and gateway update
3. **Less control**: Can't hand-edit gateway config (must use decorators)
4. **New paradigm**: Team must learn decorator-based approach

### Is kacapi Right For You?

**Good fit if:**
- You use Modal, Vercel, Supabase, or Cloudflare
- You want infrastructure-as-code (but for gateways)
- You struggle keeping gateway config in sync with code
- You need to support multiple gateways
- You want automated change detection

**Not a good fit if:**
- You need full control over gateway config
- You use platforms without API-based endpoint discovery
- You require real-time synchronization (< 1 second)
- You prefer traditional spec-first workflows

## Conceptual Model

Think of kacapi as a **compiler for API gateways**:

```
Source Code (Python/TypeScript with decorators)
            ↓
    [kacapi compiler]
            ↓
Target Code (Kong YAML, CloudFormation, etc.)
```

Just as a compiler translates high-level code to machine code, kacapi translates high-level intent (decorators) to low-level gateway configuration.

The **Observatory** is like a version control system:
- Tracks changes over time
- Detects conflicts (breaking changes)
- Proposes versions
- Provides audit trail

## Mental Model Shift

| Traditional Thinking | kacapi Thinking |
|---------------------|-----------------|
| Write spec, then code | Write code, spec emerges |
| Gateway config separate from code | Gateway config embedded in code |
| Manual versioning | Automatic version proposals |
| Manual sync required | Sync happens automatically |
| Spec is source of truth | Deployed code is source of truth |
| Platform-agnostic code | Platform-aware, gateway-agnostic |

## Next Steps

- Read [platform-first.md](platform-first.md) to understand why we only support certain platforms
- Read [emergent-specs.md](emergent-specs.md) to dive deep into how specs emerge from code
- Explore [features documentation](../features/) to see what capabilities kacapi provides
- Review [decorator API](../specs/decorator-api.md) to see all available decorators

## Philosophy in Practice

Here's a complete example showing the philosophy in action:

```python
from kacapi.modal import api_endpoint
from kacapi.auth import Clerk, any_of, ApiKey
from kacapi.rate_limit import FixedWindow, tier

@api_endpoint(
    # Declarative: WHAT you want, not HOW
    auth=any_of([Clerk(), ApiKey(header="X-API-Key")]),
    
    # Gateway-agnostic: Works with Kong, AWS, etc.
    rate_limit=FixedWindow(
        requests=tier(free=100, pro=1000, enterprise=10000),
        window="1h",
        by_user=True
    ),
    
    # Observable: Automatically logged and traced
    observability=Observability(log_body=True),
    
    # Platform-aware: Modal-specific
    modal=ModalConfig(timeout=300, retries=3)
)
@modal.function()
def get_analytics(user_id: str, date_range: str):
    """
    This endpoint:
    - Is automatically discovered via Modal API
    - Has config extracted from decorator
    - Gets deployed to gateway automatically
    - Appears in generated OpenAPI spec
    - Is tracked for changes over time
    - Enforces governance policies
    
    All from a single @api_endpoint decorator.
    """
    return {"user_id": user_id, "data": [...]}
```

When you deploy this:

1. Modal hosts the function
2. kacapi Collector discovers it via Modal API
3. Observatory takes a snapshot
4. KIR represents it in gateway-agnostic form
5. Kong Adapter generates Kong configuration
6. CLI deploys to Kong
7. Spec Generator adds to OpenAPI spec
8. Traffic Analyzer monitors its usage

All automatically. This is the kacapi philosophy in action.
