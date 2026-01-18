# kacapi Architecture

This document provides a comprehensive architectural overview of kacapi, explaining how the various components work together to enable code-first API gateway management.

## Table of Contents

- [High-Level Architecture](#high-level-architecture)
- [Core Components](#core-components)
- [Data Flow](#data-flow)
- [The Inversion Problem](#the-inversion-problem)
- [Design Principles](#design-principles)
- [Component Interactions](#component-interactions)

## High-Level Architecture

kacapi follows a **pipeline architecture** with four main stages:

```
┌──────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT STAGE                           │
│                                                                   │
│  Developer writes code with kacapi decorators                    │
│  Code deployed to platform (Modal, Vercel, etc.)                 │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                       DISCOVERY STAGE                             │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Platform Connectors                                      │    │
│  │  • Modal Connector: Uses Modal API for endpoint listing │    │
│  │  • Vercel Connector: Uses Vercel API                    │    │
│  │  • Supabase Connector: Queries Supabase projects        │    │
│  │  • Cloudflare Connector: Workers API                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Collector (Observatory)                                  │    │
│  │  • Fetches deployed endpoints                           │    │
│  │  • Extracts decorator metadata                          │    │
│  │  • Creates endpoint snapshots                           │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                       ANALYSIS STAGE                              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Differ (Observatory)                                     │    │
│  │  • Compares snapshots over time                         │    │
│  │  • Detects changes (new, modified, removed endpoints)   │    │
│  │  • Classifies changes (breaking vs. non-breaking)       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Versioner (Observatory)                                  │    │
│  │  • Proposes version numbers based on changes            │    │
│  │  • Strategies: semver, date-based, hash-based           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Governor (Observatory)                                   │    │
│  │  • Enforces governance policies                         │    │
│  │  • Validates changes against rules                      │    │
│  │  • Can block deployments                                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Traffic Analyzer (Observatory)                           │    │
│  │  • Analyzes usage patterns                              │    │
│  │  • Identifies dormant endpoints                         │    │
│  │  • Suggests optimizations                               │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                      GENERATION STAGE                             │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ KIR (Kacapi Intermediate Representation)                │    │
│  │  • Platform-agnostic representation of endpoints        │    │
│  │  • Gateway-agnostic representation of features          │    │
│  │  • JSON schema with full type information               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Gateway Adapters                                         │    │
│  │  • Kong Adapter: Generates Kong declarative config      │    │
│  │  • AWS API Gateway Adapter: Generates CloudFormation    │    │
│  │  • CloudFront Adapter: Generates Lambda@Edge functions  │    │
│  │  • APISIX Adapter: Generates APISIX config              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Spec Generator (Observatory)                             │    │
│  │  • Generates OpenAPI specifications                     │    │
│  │  • Creates SDK documentation                            │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│                       DEPLOYMENT STAGE                            │
│                                                                   │
│  CLI or CI/CD deploys generated configs to target gateways      │
└──────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Platform SDKs

**Purpose**: Provide decorators for developers to annotate their endpoints

**Implementations**:
- `@kacapi/modal` - Modal SDK (Python)
- `@kacapi/vercel` - Vercel SDK (JavaScript/TypeScript)
- `@kacapi/supabase` - Supabase Edge Functions SDK (TypeScript)
- `@kacapi/cloudflare` - Cloudflare Workers SDK (TypeScript)

**Key Responsibilities**:
- Provide type-safe decorator APIs
- Embed metadata in deployed functions
- Minimal runtime overhead
- Zero impact if kacapi tools not used

**Example**:
```python
from kacapi.modal import api_endpoint

@api_endpoint(
    auth=Clerk(),
    rate_limit=FixedWindow(requests=100, window="1m")
)
@modal.function()
def my_endpoint():
    pass
```

### 2. Platform Connectors

**Purpose**: Discover and fetch endpoint information from deployment platforms

**Interface** (`PlatformConnector`):
```typescript
interface PlatformConnector {
  // Authenticate with platform
  authenticate(credentials: PlatformCredentials): Promise<void>;
  
  // List all deployments
  listDeployments(): Promise<Deployment[]>;
  
  // Get endpoints for a deployment
  getEndpoints(deploymentId: string): Promise<Endpoint[]>;
  
  // Extract decorator metadata from endpoint
  getMetadata(endpoint: Endpoint): Promise<DecoratorMetadata>;
  
  // Validate endpoint URL is reachable
  validateEndpoint(url: string): Promise<boolean>;
}
```

**Key Responsibilities**:
- Platform API integration
- Endpoint URL discovery
- Metadata extraction
- Authentication management

**Why Only These Platforms?**

kacapi requires platforms where:
1. **URLs are discoverable** via API (not just localhost)
2. **Metadata is accessible** (source code or reflection)
3. **Deployments are enumerable** (can list all functions)

This excludes: traditional VMs, containers without orchestration, manually-managed servers.

### 3. Auth Connectors

**Purpose**: Integrate with authentication providers to validate tokens and extract user context

**Interface** (`AuthConnector`):
```typescript
interface AuthConnector {
  // Validate authentication token
  validateToken(token: string): Promise<AuthResult>;
  
  // Extract user context (ID, roles, etc.)
  extractContext(token: string): Promise<UserContext>;
  
  // Get provider configuration for gateway
  getGatewayConfig(): Promise<GatewayAuthConfig>;
}

interface AuthResult {
  valid: boolean;
  userId?: string;
  error?: string;
}

interface UserContext {
  userId: string;
  orgId?: string;
  roles: string[];
  metadata: Record<string, any>;
}
```

**Implementations**:
- Clerk
- Auth0
- AWS Cognito
- Okta
- Supabase Auth
- Firebase Auth
- WorkOS
- Generic JWT
- API Keys
- mTLS

### 4. Observatory

**Purpose**: Manage emergent specifications through change detection and version management

The Observatory is the heart of kacapi's "code-first" philosophy. It solves the **inversion problem** (see below).

#### 4.1 Collector

Periodically fetches endpoint snapshots:

```typescript
interface Snapshot {
  timestamp: string;
  platform: string;
  deployment: string;
  endpoints: EndpointSnapshot[];
}

interface EndpointSnapshot {
  url: string;
  method: string;
  decorators: DecoratorMetadata;
  hash: string; // Content-based hash for change detection
}
```

#### 4.2 Differ

Compares snapshots to detect changes:

```typescript
interface ChangeDetection {
  added: EndpointSnapshot[];
  removed: EndpointSnapshot[];
  modified: EndpointChange[];
}

interface EndpointChange {
  endpoint: string;
  changes: FieldChange[];
  breaking: boolean;
}

interface FieldChange {
  field: string; // e.g., "auth.provider"
  oldValue: any;
  newValue: any;
  breaking: boolean;
}
```

**Breaking Change Rules**:
- Removing an endpoint: **breaking**
- Adding required auth when none existed: **breaking**
- Reducing rate limit: **non-breaking** (more restrictive is safe)
- Increasing rate limit: **non-breaking**
- Adding new optional parameters: **non-breaking**
- Removing parameters: **breaking**
- Changing response schema: **depends** (uses JSON Schema diff)

#### 4.3 Versioner

Proposes version numbers based on detected changes:

**Strategies**:

1. **Semver** (default):
   - Breaking changes → major bump
   - New endpoints → minor bump
   - Config changes only → patch bump

2. **Date-based**: `YYYY.MM.DD.sequence`

3. **Hash-based**: Content hash of current state

4. **Manual**: Wait for human approval

#### 4.4 Governor

Enforces governance policies:

```typescript
interface Policy {
  name: string;
  level: 'error' | 'warning' | 'info';
  rule: string; // CEL expression
  environment?: string;
  versionConstraint?: string;
}
```

Example policies:
- All production endpoints must have auth
- Rate limits must not exceed 10,000 req/hour
- No breaking changes allowed in patch versions
- Deprecated endpoints must have sunset dates

#### 4.5 Traffic Analyzer

Analyzes gateway logs to understand usage:

```typescript
interface UsageAnalysis {
  endpoint: string;
  requests24h: number;
  requests7d: number;
  requests30d: number;
  lastSeen: string;
  dormant: boolean; // No traffic in 30 days
  percentile95Latency: number;
  errorRate: number;
}
```

**Use cases**:
- Identify dormant endpoints (candidates for deprecation)
- Detect traffic spikes (adjust rate limits)
- Find high-latency endpoints (add caching)

#### 4.6 Spec Generator

Generates OpenAPI specs from observed state:

```typescript
interface OpenAPIGenerator {
  generate(snapshot: Snapshot): Promise<OpenAPISpec>;
  
  // Infer response schemas from actual responses
  inferSchemas(trafficLogs: ResponseLog[]): Promise<SchemaMap>;
}
```

### 5. Gateway Adapters

**Purpose**: Translate KIR to gateway-specific configurations

**Interface** (`GatewayAdapter`):
```typescript
interface GatewayAdapter {
  // Convert KIR to gateway config
  generate(kir: KIR): Promise<GatewayConfig>;
  
  // Deploy config to gateway
  deploy(config: GatewayConfig, target: GatewayTarget): Promise<DeployResult>;
  
  // Validate config before deployment
  validate(config: GatewayConfig): Promise<ValidationResult>;
  
  // Generate diff between current and new config
  diff(current: GatewayConfig, proposed: GatewayConfig): Promise<ConfigDiff>;
}
```

**Implementations**:
- **Kong**: Generates declarative YAML config
- **AWS API Gateway**: Generates CloudFormation/CDK templates
- **CloudFront**: Generates Lambda@Edge functions and distributions
- **APISIX**: Generates APISIX YAML config

**Translation Examples**:

| Feature | Kong | AWS API Gateway | CloudFront |
|---------|------|-----------------|------------|
| Auth (JWT) | jwt plugin | JWT authorizer | Lambda@Edge validation |
| Rate limiting | rate-limiting plugin | Usage plan | Lambda@Edge counter |
| Caching | proxy-cache plugin | Cache policy | CloudFront cache behavior |
| CORS | cors plugin | CORS config | Lambda@Edge headers |

### 6. KIR (Kacapi Intermediate Representation)

**Purpose**: Platform-agnostic, gateway-agnostic representation of API endpoints and their configurations

```typescript
interface KIR {
  version: string;
  endpoints: KIREndpoint[];
  authProviders: AuthProvider[];
  metadata: KIRMetadata;
}

interface KIREndpoint {
  id: string;
  url: string;
  method: string;
  auth?: AuthConfig;
  rateLimit?: RateLimitConfig;
  cache?: CacheConfig;
  cors?: CORSConfig;
  security?: SecurityConfig;
  transforms?: TransformConfig;
  routing?: RoutingConfig;
  observability?: ObservabilityConfig;
  resilience?: ResilienceConfig;
  validation?: ValidationConfig;
}
```

See [docs/specs/kir.md](docs/specs/kir.md) for full schema.

### 7. CLI

**Purpose**: Command-line interface for all kacapi operations

```bash
kacapi connect <platform>       # Authenticate
kacapi discover <platform>      # Find endpoints
kacapi scan                     # Extract configs
kacapi generate <gateway>       # Generate gateway config
kacapi deploy <gateway>         # Deploy to gateway
kacapi spec                     # Generate OpenAPI spec
kacapi diff                     # Compare snapshots
kacapi governance check         # Run policy checks
kacapi version propose          # Propose next version
```

See [docs/cli/](docs/cli/) for detailed CLI documentation.

## Data Flow

### Happy Path: New Endpoint Deployment

```
1. Developer writes code with @api_endpoint decorator
2. Developer deploys to Modal
3. [Time passes]
4. kacapi collector runs (cron or CI/CD trigger)
5. Modal connector discovers new endpoint
6. Collector creates new snapshot
7. Differ detects "added" endpoint
8. Versioner proposes minor version bump
9. Governor validates (passes policies)
10. KIR generator creates intermediate representation
11. Kong adapter generates Kong config
12. CLI deploys to Kong gateway
13. Spec generator creates/updates OpenAPI spec
```

### Breaking Change Detection

```
1. Developer removes auth from existing endpoint
2. Developer deploys to Modal
3. kacapi collector runs
4. Collector creates new snapshot
5. Differ detects auth removal
6. Differ classifies as BREAKING change
7. Versioner proposes MAJOR version bump
8. Governor blocks deployment (policy: no breaking changes)
9. CLI exits with error
10. Developer notified via CI/CD
```

### Dormant Endpoint Detection

```
1. Traffic analyzer runs nightly
2. Analyzes gateway logs for past 30 days
3. Identifies endpoints with zero traffic
4. Tags endpoints as "dormant"
5. Generates report
6. Notifies team via webhook/Slack
7. Team decides: deprecate or investigate
```

## The Inversion Problem

Traditional API development follows this flow:

```
Spec → Code → Deployment
```

1. Write OpenAPI spec
2. Generate server stubs from spec
3. Implement business logic
4. Deploy
5. Keep spec in sync manually (often fails)

kacapi inverts this:

```
Code → Deployment → Spec
```

1. Write code with decorators
2. Deploy to platform
3. kacapi discovers deployed code
4. Spec emerges from observation

### Challenges This Creates

**Problem 1: When to version?**

Traditional: Developer increments version in spec.
kacapi: System detects changes and proposes version.

**Solution**: Observatory Versioner with pluggable strategies.

**Problem 2: What if code changes aren't deployed?**

Traditional: Spec is source of truth, code doesn't matter yet.
kacapi: Only deployed code matters.

**Solution**: kacapi only sees what's deployed. Local changes ignored.

**Problem 3: Breaking changes?**

Traditional: Spec diff tools detect breaking changes.
kacapi: Must infer breaking changes from decorator changes.

**Solution**: Observatory Differ with breaking change rules.

**Problem 4: Coordination across platforms?**

Traditional: One monolithic API.
kacapi: Functions scattered across Modal, Vercel, etc.

**Solution**: Collector aggregates across platforms into unified snapshot.

## Design Principles

### 1. **Single Source of Truth: Deployed Code**

The runtime environment is authoritative. If it's not deployed, it doesn't exist to kacapi.

### 2. **Eventual Consistency**

kacapi doesn't require immediate synchronization. Collector runs periodically (e.g., every 5 minutes). Slight lag is acceptable.

### 3. **Gateway Independence**

Features are described abstractly. Adapters handle gateway-specific translations. Adding a new gateway doesn't require changing platform SDKs.

### 4. **Platform Constraints as Features**

Only supporting platforms with discoverable URLs is intentional. This constraint enables the entire system.

### 5. **Policy as Code**

Governance rules are expressed as code (CEL expressions), not prose. This enables automated enforcement.

### 6. **Observability by Default**

All gateway interactions should be traced, logged, and metered. This enables traffic analysis and dormancy detection.

### 7. **Zero Trust on Metadata**

Never trust decorator metadata blindly. Governor validates all configurations before deployment.

## Component Interactions

### Authentication Flow

```
1. Request arrives at gateway
2. Gateway extracts token (header, cookie, etc.)
3. Gateway invokes auth connector
4. Auth connector validates with auth provider (Clerk, Auth0, etc.)
5. Auth connector returns user context
6. Gateway injects context into request headers
7. Request forwarded to backend endpoint
```

### Rate Limiting Flow

```
1. Request arrives at gateway
2. Gateway extracts rate limit key (user ID, IP, etc.)
3. Gateway checks rate limit counter (Redis, DynamoDB, etc.)
4. If under limit: increment counter, forward request
5. If over limit: return 429 Too Many Requests
```

### Caching Flow

```
1. Request arrives at gateway
2. Gateway computes cache key (URL + vary_by params)
3. Gateway checks cache (Redis, CloudFront, etc.)
4. If hit: return cached response
5. If miss: forward to backend
6. Backend responds
7. Gateway stores response in cache (respecting TTL)
8. Gateway returns response
```

### Change Detection Flow

```
1. Collector fetches current snapshot
2. Collector loads previous snapshot from storage
3. Differ compares snapshots
4. For each endpoint:
   a. Compute content hash
   b. If hash differs: deep diff decorators
   c. Classify each change (breaking/non-breaking)
5. Versioner receives change classification
6. Versioner proposes new version
7. Governor validates proposed version
8. If approved: update snapshot storage
9. If rejected: trigger alert/block deployment
```

## Deployment Modes

### 1. CLI Mode (Manual)

Developer runs commands manually:

```bash
kacapi discover modal
kacapi generate kong
kacapi deploy kong
```

### 2. CI/CD Mode (Automated)

GitHub Actions / GitLab CI runs kacapi:

```yaml
name: Deploy Gateway
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Discover endpoints
        run: kacapi discover modal --workspace prod
      - name: Check governance
        run: kacapi governance check
      - name: Generate config
        run: kacapi generate kong -o kong.yaml
      - name: Deploy
        run: kacapi deploy kong --config kong.yaml
```

### 3. Daemon Mode (Continuous)

kacapi runs as a service:

```bash
kacapi daemon start --interval 5m
```

Automatically:
- Discovers endpoints every 5 minutes
- Detects changes
- Proposes versions
- Deploys if policies pass

### 4. Observatory Mode (Passive)

kacapi observes but doesn't deploy:

```bash
kacapi observe --webhook https://slack.com/webhook
```

Reports changes but requires manual approval to deploy.

## Scalability Considerations

### Platform Connector Scaling

Each platform connector can be rate-limited by the platform API:
- Modal: 100 req/min
- Vercel: 100 req/min
- Supabase: 60 req/min
- Cloudflare: 1200 req/min

**Solution**: 
- Cache endpoint lists (TTL: 5 minutes)
- Batch requests where possible
- Parallelize across deployments

### Gateway Adapter Scaling

Some gateways have large config sizes:
- Kong: Declarative config can be 100MB+ for large APIs
- AWS API Gateway: CloudFormation templates have size limits

**Solution**:
- Incremental updates where supported
- Config splitting for large deployments
- Compression for storage/transport

### Observatory Storage

Snapshots grow over time:
- 1000 endpoints × 10KB each = 10MB per snapshot
- Hourly snapshots = 240MB/day = 7.2GB/month

**Solution**:
- Compress old snapshots
- Prune snapshots older than retention period
- Store diffs instead of full snapshots after first

### Traffic Analysis

Gateway logs can be massive:
- 1M requests/day × 1KB per log = 1GB/day

**Solution**:
- Sample logs (10% sampling = 100MB/day)
- Aggregate metrics (P95 latency, error rate) instead of raw logs
- Use gateway-native analytics where possible

## Security Considerations

### 1. Credential Management

Platform and gateway credentials are sensitive. kacapi:
- Never logs credentials
- Stores encrypted in system keychain (CLI mode)
- Uses environment variables (CI/CD mode)
- Supports secret management systems (AWS Secrets Manager, Vault)

### 2. Metadata Trust

Decorator metadata could be malicious:
- Rate limit of 999999999 (DoS the backend)
- IP whitelist of 0.0.0.0/0 (bypass security)

**Solution**: Governor validates all configurations before deployment.

### 3. Gateway Access

Deploying to gateways requires privileged access. kacapi:
- Uses least-privilege credentials
- Audits all deployments
- Supports approval workflows

### 4. Multi-Tenancy

Observatory must isolate snapshots between teams/projects:
- Separate storage per workspace
- API tokens scoped to specific projects
- No cross-project data leakage

## Future Considerations

### 1. Multi-Region Support

Currently assumes single region. Future:
- Detect endpoint regions
- Generate region-specific gateway configs
- Support geo-routing

### 2. Canary Deployment Automation

Currently requires manual canary configuration. Future:
- Automatic traffic shifting (5% → 50% → 100%)
- Rollback on error rate increase
- Integration with monitoring systems

### 3. Cost Optimization

Future traffic analyzer features:
- Identify expensive endpoints (high request rate × backend cost)
- Suggest caching opportunities
- Recommend rate limit adjustments

### 4. ML-Powered Versioning

Use machine learning to:
- Predict breaking changes more accurately
- Suggest optimal rate limits based on historical data
- Detect anomalous traffic patterns

## Conclusion

kacapi's architecture is designed around a core insight: **specifications should emerge from deployed code, not prescribe it**. Every component serves this goal:

- **Platform Connectors** discover what's actually deployed
- **Observatory** tracks changes over time and proposes versions
- **Gateway Adapters** translate to various gateway technologies
- **KIR** provides a stable intermediate representation

The system is complex because the problem is complex: inverting the traditional spec-first flow while maintaining safety, governance, and multi-gateway support.

This architecture documentation will evolve as we refine the design and encounter real-world use cases.
