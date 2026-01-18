# kacapi

> **Code-first, declarative API gateway system** — Add decorators to your API endpoints and automatically get an enterprise-grade API gateway.

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-specification-orange.svg)]()

## Overview

kacapi is a revolutionary approach to API gateway management where **specifications emerge from code** rather than being prescribed upfront. Developers add simple decorators to their API endpoints deployed on platforms like Modal, Vercel, Supabase, and Cloudflare, and kacapi automatically generates and deploys complete API gateway configurations to Kong, AWS API Gateway, CloudFront, and more.

### The Problem

Traditional API gateways require you to:
1. Write code for your endpoints
2. Separately configure authentication, rate limiting, caching, etc. in the gateway
3. Keep these two sources of truth in sync manually
4. Deal with version mismatches and drift

### The kacapi Solution

```python
from kacapi.modal import api_endpoint

@api_endpoint(
    auth=Clerk(),
    rate_limit=FixedWindow(requests=100, window="1m", by_user=True),
    cache=Cache(ttl=300, vary_by=["user_id"]),
    cors=CORS(origins=["https://example.com"])
)
@modal.function()
def get_user_profile(user_id: str):
    return {"user_id": user_id, "name": "Alice"}
```

That's it. kacapi:
- Discovers this endpoint automatically
- Extracts gateway configuration from decorators
- Generates optimized gateway configs (Kong, AWS API Gateway, etc.)
- Deploys to your chosen gateway
- Tracks changes and proposes versions
- Detects breaking changes automatically

## Key Philosophy

### 1. **Code-First, Not Config-First**

Gateway configuration lives alongside the code as decorators. The "source of truth" is your running application, not a separate config file.

### 2. **Emergent Specifications**

Instead of writing an OpenAPI spec upfront, kacapi **observes** your deployed endpoints and generates specs from reality. Versioning happens when the system detects changes, not when developers remember to bump a version number.

### 3. **Platform-First**

kacapi only supports platforms with **discoverable URLs** (Modal, Vercel, Supabase, Cloudflare). If we can't programmatically discover your endpoints, we can't manage them. This constraint is a feature, not a limitation.

### 4. **Gateway-Agnostic**

Your decorators describe **what** you want (authentication, rate limiting, etc.), not **how** to configure a specific gateway. kacapi adapters translate to Kong, AWS API Gateway, CloudFront, APISIX, etc.

## Quick Start

### Installation

```bash
npm install -g kacapi
# or
pip install kacapi
```

### 1. Connect to Your Platform

```bash
kacapi connect modal --token $MODAL_TOKEN
```

### 2. Discover Endpoints

```bash
kacapi discover modal --workspace my-workspace
```

### 3. Generate Gateway Configuration

```bash
kacapi generate kong --output kong.yaml
```

### 4. Deploy to Gateway

```bash
kacapi deploy kong --config kong.yaml --gateway https://api.example.com
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Your Application                         │
│  (Modal, Vercel, Supabase, Cloudflare + kacapi decorators) │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ Platform API
                        ↓
┌─────────────────────────────────────────────────────────────┐
│                  kacapi Observatory                          │
│  • Collector: Discovers endpoints via platform APIs         │
│  • Scanner: Extracts decorator metadata                     │
│  • Differ: Detects changes, proposes versions               │
│  • Governor: Enforces policies, blocks breaking changes     │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ KIR (Kacapi Intermediate Representation)
                        ↓
┌─────────────────────────────────────────────────────────────┐
│                  Gateway Adapters                            │
│           Kong | AWS API Gateway | CloudFront                │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
                 Deployed Gateway
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed architecture documentation.

## Features

kacapi provides comprehensive API gateway capabilities:

- **Authentication**: Clerk, Auth0, Cognito, Okta, JWT, API keys, mTLS
- **Rate Limiting**: Fixed window, sliding window, token bucket; by user/org/IP/tier
- **Caching**: TTL, vary_by, stale-while-revalidate
- **Security**: CORS, IP allow/deny, geo-blocking, bot protection, WAF
- **Transforms**: Header manipulation, context injection
- **Routing**: Canary deployments, weighted routing, A/B testing
- **Observability**: Structured logging, metrics, distributed tracing
- **Resilience**: Timeouts, retries, circuit breakers, fallbacks
- **Validation**: JSON Schema, OpenAPI validation

See [docs/features/](docs/features/) for detailed feature specifications.

## Supported Platforms

| Platform | SDK | Connector | Status |
|----------|-----|-----------|--------|
| Modal | ✅ | ✅ | Planned |
| Vercel | ✅ | ✅ | Planned |
| Supabase Edge Functions | ✅ | ✅ | Planned |
| Cloudflare Workers | ✅ | ✅ | Planned |

See [docs/sdks/](docs/sdks/) for platform-specific documentation.

## Supported Gateways

| Gateway | Adapter | Status |
|---------|---------|--------|
| Kong | ✅ | Planned |
| AWS API Gateway | ✅ | Planned |
| CloudFront + Lambda@Edge | ✅ | Planned |
| Apache APISIX | ✅ | Planned |

See [docs/adapters/](docs/adapters/) for gateway-specific documentation.

## Documentation

- **[Core Concepts](docs/concepts/)** - Philosophy and architectural decisions
- **[Features](docs/features/)** - Comprehensive feature specifications
- **[Platform SDKs](docs/sdks/)** - Platform-specific decorator APIs
- **[Connectors](docs/connectors/)** - Platform integration specifications
- **[Auth Connectors](docs/auth-connectors/)** - Authentication provider integrations
- **[Gateway Adapters](docs/adapters/)** - Target gateway specifications
- **[Observatory](docs/observatory/)** - Emergent spec management system
- **[CLI Reference](docs/cli/)** - Command-line interface documentation
- **[Technical Specs](docs/specs/)** - KIR, decorator API, governance schemas

## Why kacapi?

### vs. Traditional API Gateways (Kong, AWS API Gateway)

Traditional gateways require separate configuration management. With kacapi:
- Configuration lives in code as decorators
- No drift between code and gateway config
- Automatic change detection and versioning
- Type-safe configuration (in TypeScript/Python)

### vs. Code-Generation Tools (OpenAPI generators)

Code generators create code from specs. kacapi does the opposite:
- Specs emerge from real, deployed code
- No stale specifications
- Versions proposed automatically based on observed changes
- Works with existing codebases

### vs. Service Meshes (Istio, Linkerd)

Service meshes operate at the infrastructure layer. kacapi:
- Works with serverless platforms (no infrastructure to manage)
- Platform-agnostic (Modal, Vercel, Supabase, Cloudflare)
- Simpler mental model (decorators, not YAML)
- Focuses on API gateway concerns, not service-to-service mesh

## Real-World Example

```python
from kacapi.modal import api_endpoint
from kacapi.auth import Clerk, any_of, ApiKey
from kacapi.rate_limit import FixedWindow, tier
from kacapi.routing import weighted, Canary

@api_endpoint(
    # Auth: Allow either Clerk users OR API key
    auth=any_of([Clerk(), ApiKey(header="X-API-Key")]),
    
    # Rate limiting: Tiered by user's subscription
    rate_limit=FixedWindow(
        requests=tier(free=100, pro=1000, enterprise=10000),
        window="1h",
        by_user=True
    ),
    
    # Caching with user-specific keys
    cache=Cache(ttl=300, vary_by=["user_id", "tier"]),
    
    # Canary deployment: 95% old, 5% new
    routing=Canary(weight=0.05, target="v2-endpoint"),
    
    # Security
    cors=CORS(origins=["https://example.com"], credentials=True),
    security=Security(
        ip_whitelist=["10.0.0.0/8"],
        bot_protection=True
    ),
    
    # Observability
    observability=Observability(
        log_body=True,
        trace_sampling=0.1
    )
)
@modal.function()
def get_analytics(user_id: str, date_range: str):
    return {"user_id": user_id, "data": [...]}
```

## Observatory: The Inversion Problem

The biggest challenge in kacapi is what we call **the inversion problem**: traditional systems start with a spec and generate code; kacapi starts with code and generates specs.

The Observatory subsystem solves this:

1. **Collector**: Periodically scans platform APIs to discover endpoints
2. **Differ**: Compares current state to previous snapshots
3. **Versioner**: Proposes new versions when changes detected
4. **Governor**: Enforces policies (e.g., "no breaking changes in minor versions")
5. **Traffic Analyzer**: Identifies dormant endpoints, usage patterns

See [docs/observatory/](docs/observatory/) for detailed documentation.

## Governance

Create a `kacapi.governance.yaml` to enforce organizational policies:

```yaml
policies:
  # Require auth on all production endpoints
  - name: require-auth
    level: error
    rule: all_endpoints.auth != null
    environment: production
  
  # Warn if rate limit exceeds threshold
  - name: rate-limit-max
    level: warning
    rule: all_endpoints.rate_limit.requests <= 10000
  
  # Block breaking changes in minor versions
  - name: no-breaking-changes
    level: error
    rule: breaking_changes.count == 0
    version_constraint: "~"
```

```bash
kacapi governance check --config kacapi.governance.yaml
```

See [docs/specs/governance-config.md](docs/specs/governance-config.md) for full specification.

## Status

⚠️ **This project is currently in the specification phase.** All documentation reflects the planned architecture and features. Implementation has not yet begun.

This repository contains comprehensive documentation that serves as the complete specification for kacapi. We're refining the design before writing any code.

## Contributing

Since we're in the specification phase, contributions should focus on:
- Reviewing and refining documentation
- Identifying edge cases or missing features
- Proposing alternative approaches
- Validating the design against real-world use cases

Please open issues for discussion or submit PRs to improve documentation.

## License

MIT License - see [LICENSE](LICENSE) for details.

## Roadmap

1. **Phase 1**: Finalize specification (current phase)
2. **Phase 2**: Implement core abstractions (KIR, interfaces)
3. **Phase 3**: Build first platform SDK (Modal)
4. **Phase 4**: Build first platform connector (Modal)
5. **Phase 5**: Build first gateway adapter (Kong)
6. **Phase 6**: Implement Observatory subsystem
7. **Phase 7**: Additional platforms and gateways
8. **Phase 8**: Governance and policy enforcement

## Contact

- GitHub Issues: [github.com/mikesol/kacapi/issues](https://github.com/mikesol/kacapi/issues)
- Documentation: [github.com/mikesol/kacapi/tree/main/docs](https://github.com/mikesol/kacapi/tree/main/docs)
