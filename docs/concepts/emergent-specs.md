# Emergent Specifications

This document explores how kacapi inverts the traditional specification-first workflow by generating API specifications from observed, deployed code rather than prescribing them upfront. This approach creates unique challenges around versioning, change detection, and breaking change classification that the Observatory subsystem addresses.

## Table of Contents

- [The Traditional Spec-First Approach](#the-traditional-spec-first-approach)
- [The Inversion: Code-First to Emergent Specs](#the-inversion-code-first-to-emergent-specs)
- [The Versioning Challenge](#the-versioning-challenge)
- [How the Observatory Handles Change Detection](#how-the-observatory-handles-change-detection)
- [Breaking vs Non-Breaking Changes](#breaking-vs-non-breaking-changes)
- [Automatic Version Proposal Strategies](#automatic-version-proposal-strategies)
- [Comparison with Spec-First Approaches](#comparison-with-spec-first-approaches)
- [Edge Cases and Challenges](#edge-cases-and-challenges)
- [The Feedback Loop](#the-feedback-loop)
- [Best Practices](#best-practices)

## The Traditional Spec-First Approach

Traditional API development follows a **prescriptive** model:

```
1. Design API (write OpenAPI/Swagger spec)
   ↓
2. Review spec with stakeholders
   ↓
3. Generate server stubs from spec
   ↓
4. Implement business logic in stubs
   ↓
5. Deploy implementation
   ↓
6. (Optional) Validate implementation matches spec
   ↓
7. Generate client SDKs from spec
```

### The Ideal vs Reality

**The ideal:**
- Spec is always current
- Implementation matches spec exactly
- Clients trust the spec
- Breaking changes are deliberate and versioned

**The reality:**
- Specs become stale immediately after deployment
- Developers modify code without updating spec
- Spec says endpoint requires auth, but implementation doesn't check
- Version numbers bumped arbitrarily or forgotten
- Teams abandon specs entirely ("we'll update it later")

### Why Specs Drift

**1. Dual Maintenance Burden**

Every change requires two updates:
```python
# Update 1: Change the code
@app.post("/users")
def create_user(name: str, email: str, role: str):  # Added 'role' parameter
    pass

# Update 2: Change the spec (often forgotten)
```

```yaml
# openapi.yaml - Oops, forgot to add 'role'
/users:
  post:
    requestBody:
      properties:
        name: {type: string}
        email: {type: string}
        # role: {type: string}  ← Missing!
```

**2. No Automated Enforcement**

There's no system ensuring spec and code stay in sync. Validation is:
- Manual code review (humans miss things)
- CI/CD checks (often skipped or incomplete)
- Integration tests (rarely cover all edge cases)
- Runtime validation (added late or never)

**3. Spec Generation Tools Are Insufficient**

Tools like `swagger-jsdoc` or `drf-spectacular` generate specs from code annotations:

```python
@swagger_auto_schema(
    operation_description="Create a user",
    request_body=UserSerializer,
    responses={201: UserSerializer}
)
@api_view(['POST'])
def create_user(request):
    pass
```

**Problems:**
- Decorators are verbose, developers skip them
- Only captures request/response schemas, not gateway behavior (auth, rate limits, caching)
- Still requires manual maintenance
- Doesn't track changes over time

**4. Organizational Dynamics**

Specs are documentation. Documentation is:
- Always behind schedule
- First to be cut when deadlines loom
- Updated after the fact (if at all)
- Owned by no specific person

## The Inversion: Code-First to Emergent Specs

kacapi inverts the flow entirely:

```
1. Developer writes code with decorators
   ↓
2. Developer deploys to platform (Modal, Vercel, etc.)
   ↓
3. Platform hosts the endpoint (running code is truth)
   ↓
4. kacapi discovers endpoint via platform API
   ↓
5. kacapi extracts decorator metadata from deployed code
   ↓
6. kacapi generates gateway configuration
   ↓
7. kacapi deploys to gateway
   ↓
8. kacapi generates OpenAPI spec from observed state
   ↓
9. kacapi tracks changes over time (snapshots)
   ↓
10. kacapi proposes version when changes detected
```

### Key Insight: Deployed Code is the Only Truth

In kacapi's model:

> **The specification describes what IS deployed, not what SHOULD BE deployed.**

This is philosophical shift with profound implications:

**Traditional thinking:**
- Spec is prescriptive: "This is what the API should be"
- Code must conform to spec
- Validation checks implementation matches spec

**kacapi thinking:**
- Spec is descriptive: "This is what the API actually is"
- Code defines reality
- Validation checks nothing contradicts deployed state

### Example: New Endpoint

**Traditional approach:**

```yaml
# Step 1: Update openapi.yaml
/users/{id}:
  get:
    summary: Get user by ID
    parameters:
      - name: id
        in: path
        required: true
        schema: {type: string}
    responses:
      200:
        content:
          application/json:
            schema:
              type: object
              properties:
                id: {type: string}
                name: {type: string}
```

```python
# Step 2: Implement endpoint
@app.get("/users/{id}")
def get_user(id: str):
    return {"id": id, "name": "Alice"}
```

```bash
# Step 3: Remember to bump version in spec
sed -i 's/version: 1.0.0/version: 1.1.0/' openapi.yaml
```

**kacapi approach:**

```python
# Step 1: Implement endpoint with decorator
@api_endpoint(
    cache=Cache(ttl=300)
)
@modal.function()
def get_user(id: str):
    return {"id": id, "name": "Alice"}
```

```bash
# Step 2: Deploy
modal deploy

# Done. kacapi does the rest:
# - Discovers new endpoint
# - Extracts cache config
# - Proposes version 1.1.0 (new endpoint = minor bump)
# - Generates OpenAPI spec entry
# - Updates gateway config
```

### Why This Works

**1. Single Source of Truth**

Code is reality. Everything else is derived:

```
Deployed Code
    ↓
[Discovered Endpoints]
    ↓
[KIR Representation]
    ↓
[Gateway Config] + [OpenAPI Spec]
```

**2. No Drift Possible**

Spec cannot drift from code because spec is generated from code. The only way to change the spec is to change and deploy code.

**3. Enforced Discipline**

Want to add authentication? Must add it in code via decorator. No "we'll add it later" because:
- Decorator is in code (visible in code review)
- Decorator is type-checked (fails at deploy if invalid)
- Gateway won't be configured without decorator

## The Versioning Challenge

Traditional systems: developer increments version manually.

```yaml
# Developer decides when to bump version
version: 1.2.0 → 1.3.0
```

kacapi systems: **system** must decide when to bump version based on observed changes.

### The Core Problem

```
Time T1:               Time T2:
GET /users            GET /users (modified)
POST /users           POST /users
                      GET /users/{id} (new)
                      DELETE /users/{id} (new)

Question: What version should T2 be?
- T1 is version 1.0.0
- T2 has breaking changes? Non-breaking changes? Both?
- Should be 1.1.0? 2.0.0? 1.0.1?
```

kacapi must:

1. **Detect what changed**: Compare T1 snapshot to T2 snapshot
2. **Classify changes**: Breaking vs. non-breaking
3. **Propose version**: Apply versioning strategy
4. **Validate**: Ensure version follows governance rules

### Challenges Unique to Emergent Specs

**Challenge 1: No Human Intent**

Traditional: "I'm adding a new required field, so I'll make it version 2.0.0."

kacapi: Must infer intent from code changes:
- Added field to request schema
- Is field optional or required?
- If required → breaking change → major bump
- If optional → non-breaking → minor bump

**Challenge 2: Timing**

Traditional: Version bumped when spec is updated (explicit action).

kacapi: Version bumped when change is detected. But when should detection run?
- Every deployment? (might be too frequent)
- Hourly? (might miss rapid changes)
- On-demand? (might forget to run)

**Challenge 3: Coordinating Multiple Platforms**

Traditional: One monolithic API, one version.

kacapi: Endpoints scattered across Modal, Vercel, Supabase:

```
Modal:
  - GET /users (version 1.2.0)
  
Vercel:
  - GET /posts (version 1.0.0)
  
Supabase:
  - GET /comments (version 1.1.0)

Question: What's the version of the overall API?
```

**Solution**: Each platform can have separate versioning, or use unified versioning across platforms.

**Challenge 4: Distinguishing Code Changes from Config Changes**

```python
# Change 1: Code modification (affects behavior)
def get_user(id: str):
    return {"id": id, "name": "Bob"}  # Changed from "Alice"

# Change 2: Config modification (doesn't affect behavior)
@api_endpoint(
    rate_limit=FixedWindow(requests=200, window="1m")  # Changed from 100
)
```

**Question**: Do config-only changes warrant a version bump?

**Answer**: Depends on versioning strategy:
- Strict semver: Config changes are patch bumps
- Lenient: Only code changes trigger versions
- Custom: Defined in governance rules

## How the Observatory Handles Change Detection

The Observatory is kacapi's subsystem for managing emergent specifications. It consists of five components:

### 1. Collector

**Purpose**: Periodically capture snapshots of deployed state

**Workflow**:

```typescript
// Every N minutes (configurable)
async function collectSnapshot() {
  const endpoints = [];
  
  // For each platform connector
  for (const platform of ['modal', 'vercel', 'supabase']) {
    const connector = getConnector(platform);
    
    // Discover deployments
    const deployments = await connector.listDeployments();
    
    // For each deployment, get endpoints
    for (const deployment of deployments) {
      const deploymentEndpoints = await connector.getEndpoints(deployment.id);
      
      // Extract metadata from each endpoint
      for (const endpoint of deploymentEndpoints) {
        const metadata = await connector.getMetadata(endpoint);
        endpoints.push({
          url: endpoint.url,
          method: endpoint.method,
          platform: platform,
          deployment: deployment.id,
          decorators: metadata,
          hash: computeHash(endpoint.url, endpoint.method, metadata),
          timestamp: new Date().toISOString()
        });
      }
    }
  }
  
  // Save snapshot
  await storage.saveSnapshot({
    id: generateId(),
    timestamp: new Date().toISOString(),
    endpoints: endpoints,
    version: null  // Will be set by Versioner
  });
  
  return endpoints;
}
```

**Snapshot Structure**:

```json
{
  "id": "snapshot-2024-01-15T10-30-00Z",
  "timestamp": "2024-01-15T10:30:00Z",
  "platform": "modal",
  "endpoints": [
    {
      "url": "https://user--get-users.modal.run",
      "method": "GET",
      "deployment": "app-abc123",
      "decorators": {
        "auth": {"provider": "clerk"},
        "rate_limit": {"requests": 100, "window": "1m"},
        "cache": {"ttl": 300}
      },
      "hash": "a1b2c3d4"
    },
    {
      "url": "https://user--create-user.modal.run",
      "method": "POST",
      "deployment": "app-abc123",
      "decorators": {
        "auth": {"provider": "clerk"},
        "rate_limit": {"requests": 50, "window": "1m"}
      },
      "hash": "e5f6g7h8"
    }
  ],
  "version": "1.0.0"
}
```

### 2. Differ

**Purpose**: Compare snapshots to detect changes

**Workflow**:

```typescript
interface ChangeDetection {
  added: EndpointSnapshot[];
  removed: EndpointSnapshot[];
  modified: EndpointChange[];
  unchanged: EndpointSnapshot[];
}

interface EndpointChange {
  endpoint: string;
  method: string;
  changes: FieldChange[];
  breaking: boolean;
}

interface FieldChange {
  path: string;  // e.g., "decorators.auth.provider"
  oldValue: any;
  newValue: any;
  breaking: boolean;
  reason: string;
}

async function detectChanges(
  oldSnapshot: Snapshot,
  newSnapshot: Snapshot
): Promise<ChangeDetection> {
  const oldEndpoints = new Map(
    oldSnapshot.endpoints.map(e => [e.url + e.method, e])
  );
  const newEndpoints = new Map(
    newSnapshot.endpoints.map(e => [e.url + e.method, e])
  );
  
  const result: ChangeDetection = {
    added: [],
    removed: [],
    modified: [],
    unchanged: []
  };
  
  // Detect additions
  for (const [key, endpoint] of newEndpoints) {
    if (!oldEndpoints.has(key)) {
      result.added.push(endpoint);
    }
  }
  
  // Detect removals
  for (const [key, endpoint] of oldEndpoints) {
    if (!newEndpoints.has(key)) {
      result.removed.push(endpoint);
    }
  }
  
  // Detect modifications
  for (const [key, newEndpoint] of newEndpoints) {
    const oldEndpoint = oldEndpoints.get(key);
    if (oldEndpoint) {
      if (oldEndpoint.hash !== newEndpoint.hash) {
        // Something changed, deep diff
        const changes = deepDiff(oldEndpoint, newEndpoint);
        const isBreaking = changes.some(c => c.breaking);
        
        result.modified.push({
          endpoint: newEndpoint.url,
          method: newEndpoint.method,
          changes: changes,
          breaking: isBreaking
        });
      } else {
        result.unchanged.push(newEndpoint);
      }
    }
  }
  
  return result;
}
```

**Deep Diff Implementation**:

```typescript
function deepDiff(old: EndpointSnapshot, new: EndpointSnapshot): FieldChange[] {
  const changes: FieldChange[] = [];
  
  // Compare auth
  if (JSON.stringify(old.decorators.auth) !== JSON.stringify(new.decorators.auth)) {
    changes.push(diffAuth(old.decorators.auth, new.decorators.auth));
  }
  
  // Compare rate_limit
  if (JSON.stringify(old.decorators.rate_limit) !== JSON.stringify(new.decorators.rate_limit)) {
    changes.push(diffRateLimit(old.decorators.rate_limit, new.decorators.rate_limit));
  }
  
  // Compare cache
  if (JSON.stringify(old.decorators.cache) !== JSON.stringify(new.decorators.cache)) {
    changes.push(diffCache(old.decorators.cache, new.decorators.cache));
  }
  
  // ... compare other decorators
  
  return changes;
}
```

### 3. Change Classifier

**Purpose**: Determine if changes are breaking or non-breaking

This is the most critical component. Incorrect classification could:
- Allow breaking changes through (angry users)
- Block harmless changes (frustrated developers)

**Classification Rules**:

```typescript
const BREAKING_CHANGE_RULES = {
  // Endpoint lifecycle
  'endpoint.removed': {
    breaking: true,
    reason: 'Clients relying on this endpoint will fail'
  },
  'endpoint.added': {
    breaking: false,
    reason: 'Adding endpoints is always safe'
  },
  
  // Authentication
  'auth.added_when_none_existed': {
    breaking: true,
    reason: 'Previously public endpoint now requires auth'
  },
  'auth.removed': {
    breaking: true,
    reason: 'Security regression, could indicate accidental change'
  },
  'auth.provider_changed': {
    breaking: true,
    reason: 'Clients using old auth method will fail'
  },
  'auth.made_more_restrictive': {
    breaking: true,
    reason: 'Users who previously had access may lose it'
  },
  'auth.made_less_restrictive': {
    breaking: false,
    reason: 'More permissive auth is backward compatible'
  },
  
  // Rate limiting
  'rate_limit.reduced': {
    breaking: false,  // ⚠️ Controversial!
    reason: 'More restrictive limits are a service degradation, not a breaking change'
  },
  'rate_limit.increased': {
    breaking: false,
    reason: 'More permissive limits are safe'
  },
  'rate_limit.window_changed': {
    breaking: false,
    reason: 'Window changes affect behavior but don\'t break clients'
  },
  
  // Caching
  'cache.ttl_changed': {
    breaking: false,
    reason: 'Cache TTL is an optimization, not a contract'
  },
  'cache.added': {
    breaking: false,
    reason: 'Adding cache is transparent to clients'
  },
  'cache.removed': {
    breaking: false,
    reason: 'Removing cache may affect performance but not correctness'
  },
  
  // Request schema
  'request.field_added_required': {
    breaking: true,
    reason: 'Clients not sending field will fail validation'
  },
  'request.field_added_optional': {
    breaking: false,
    reason: 'Optional fields are backward compatible'
  },
  'request.field_removed': {
    breaking: true,
    reason: 'Clients sending removed field may fail or have unexpected behavior'
  },
  'request.field_type_changed': {
    breaking: true,
    reason: 'Type changes break clients sending old format'
  },
  
  // Response schema
  'response.field_added': {
    breaking: false,
    reason: 'Clients ignoring new fields are unaffected'
  },
  'response.field_removed': {
    breaking: true,
    reason: 'Clients expecting removed field will fail'
  },
  'response.field_type_changed': {
    breaking: true,
    reason: 'Type changes break client parsing'
  },
  'response.field_made_nullable': {
    breaking: true,
    reason: 'Clients not handling null will fail'
  },
  'response.field_made_non_nullable': {
    breaking: false,
    reason: 'Stronger guarantee is backward compatible'
  }
};
```

**Example Classifications**:

```typescript
// Example 1: Added auth to public endpoint
const change1 = {
  path: 'decorators.auth',
  oldValue: null,
  newValue: {provider: 'clerk'},
  breaking: true,
  reason: 'Previously public endpoint now requires auth'
};

// Example 2: Increased rate limit
const change2 = {
  path: 'decorators.rate_limit.requests',
  oldValue: 100,
  newValue: 200,
  breaking: false,
  reason: 'More permissive limits are safe'
};

// Example 3: Removed endpoint
const change3 = {
  path: 'endpoint',
  oldValue: 'GET /users/{id}',
  newValue: null,
  breaking: true,
  reason: 'Clients relying on this endpoint will fail'
};
```

### 4. Versioner

**Purpose**: Propose next version number based on detected changes

**Strategies**:

#### Strategy 1: Semantic Versioning (Default)

```typescript
function proposeVersionSemver(
  currentVersion: string,
  changes: ChangeDetection
): string {
  const [major, minor, patch] = currentVersion.split('.').map(Number);
  
  // Check for breaking changes
  const hasBreakingChanges = 
    changes.removed.length > 0 ||
    changes.modified.some(m => m.breaking);
  
  if (hasBreakingChanges) {
    return `${major + 1}.0.0`;
  }
  
  // Check for new features
  const hasNewFeatures = changes.added.length > 0;
  
  if (hasNewFeatures) {
    return `${major}.${minor + 1}.0`;
  }
  
  // Only non-breaking modifications
  const hasModifications = changes.modified.length > 0;
  
  if (hasModifications) {
    return `${major}.${minor}.${patch + 1}`;
  }
  
  // No changes
  return currentVersion;
}
```

**Examples**:

```
Current: 1.0.0

Changes:
- Added GET /users/{id}
Proposed: 1.1.0 (new endpoint = minor bump)

Changes:
- Modified POST /users: added required 'role' field
Proposed: 2.0.0 (breaking change = major bump)

Changes:
- Modified GET /users: increased rate limit
Proposed: 1.0.1 (config-only change = patch bump)
```

#### Strategy 2: Date-Based Versioning

```typescript
function proposeVersionDateBased(
  currentVersion: string,
  changes: ChangeDetection
): string {
  const now = new Date();
  const dateStr = `${now.getFullYear()}.${now.getMonth() + 1}.${now.getDate()}`;
  
  // Check if we already have a version for today
  if (currentVersion.startsWith(dateStr)) {
    // Increment sequence number
    const parts = currentVersion.split('.');
    const sequence = parseInt(parts[3] || '0') + 1;
    return `${dateStr}.${sequence}`;
  }
  
  // First version of the day
  return `${dateStr}.0`;
}
```

**Examples**:

```
Current: 2024.1.15.0

Changes:
- Any change
Proposed: 2024.1.15.1 (same day, increment sequence)

Next Day:
Proposed: 2024.1.16.0 (new day, reset sequence)
```

#### Strategy 3: Hash-Based Versioning

```typescript
function proposeVersionHashBased(
  currentVersion: string,
  snapshot: Snapshot
): string {
  // Compute hash of entire snapshot
  const hash = computeHash(JSON.stringify(snapshot.endpoints));
  
  // Use first 8 characters of hash
  return hash.substring(0, 8);
}
```

**Examples**:

```
Current: a1b2c3d4

Changes:
- Any change
Proposed: e5f6g7h8 (hash of new state)
```

**Advantage**: Deterministic, no need to track "previous version."

**Disadvantage**: No semantic meaning.

#### Strategy 4: Manual Approval

```typescript
function proposeVersionManual(
  currentVersion: string,
  changes: ChangeDetection
): string {
  // Don't propose automatically
  // Wait for human to decide
  return null;
}
```

When manual approval is configured:

```bash
$ kacapi diff
Changes detected:
- Added: GET /users/{id}
- Modified: POST /users (added auth)

Proposed version: [awaiting manual input]

$ kacapi version set 2.0.0
Version set to 2.0.0

$ kacapi deploy
Deploying version 2.0.0...
```

### 5. Governor

**Purpose**: Enforce governance policies before allowing deployment

**Policy Examples**:

```yaml
# kacapi.governance.yaml
policies:
  - name: require-auth-production
    level: error
    rule: all_endpoints.decorators.auth != null
    environment: production
    message: "All production endpoints must have authentication"
  
  - name: breaking-changes-major-only
    level: error
    rule: |
      breaking_changes.count > 0 implies
      version.major > previous_version.major
    message: "Breaking changes require major version bump"
  
  - name: rate-limit-ceiling
    level: warning
    rule: all_endpoints.decorators.rate_limit.requests <= 10000
    message: "Rate limits should not exceed 10,000 req/window"
  
  - name: cache-ttl-minimum
    level: info
    rule: |
      all_endpoints.decorators.cache != null implies
      all_endpoints.decorators.cache.ttl >= 60
    message: "Consider cache TTL of at least 60 seconds"
```

**Enforcement Workflow**:

```typescript
async function enforceGovernance(
  snapshot: Snapshot,
  changes: ChangeDetection,
  proposedVersion: string
): Promise<GovernanceResult> {
  const policies = await loadPolicies();
  const violations: PolicyViolation[] = [];
  
  for (const policy of policies) {
    // Evaluate policy rule (CEL expression)
    const result = await evaluateRule(policy.rule, {
      all_endpoints: snapshot.endpoints,
      changes: changes,
      version: parseVersion(proposedVersion),
      previous_version: parseVersion(snapshot.version),
      breaking_changes: changes.modified.filter(m => m.breaking)
    });
    
    if (!result) {
      violations.push({
        policy: policy.name,
        level: policy.level,
        message: policy.message
      });
    }
  }
  
  // Determine if deployment should be blocked
  const hasErrors = violations.some(v => v.level === 'error');
  
  return {
    passed: !hasErrors,
    violations: violations
  };
}
```

**Example Enforcement**:

```bash
$ kacapi deploy

Checking governance policies...

✅ require-auth-production: PASS
❌ breaking-changes-major-only: FAIL
   Breaking changes detected, but version 1.1.0 is not a major bump
   Expected: 2.0.0
⚠️  rate-limit-ceiling: WARNING
   Endpoint GET /analytics has rate limit of 50,000 req/hour
🔵 cache-ttl-minimum: INFO
   3 endpoints have cache TTL < 60 seconds

Deployment BLOCKED due to governance violations.
```

## Breaking vs Non-Breaking Changes

Determining what constitutes a "breaking change" is nuanced. kacapi follows these principles:

### Principle 1: Client Impact

> A change is breaking if it requires clients to modify their code to continue functioning correctly.

**Examples**:

**Breaking**:
- Removing an endpoint
- Removing a response field
- Making an optional request field required
- Changing response field type (string → number)
- Adding authentication to a public endpoint

**Non-Breaking**:
- Adding a new endpoint
- Adding a new optional request field
- Adding a new response field
- Making a required request field optional
- Removing authentication (⚠️ security concern, but not breaking)

### Principle 2: Gateway Behavior

> Changes to gateway configuration that affect request/response flow are evaluated separately from API schema changes.

**Rate Limiting**:

```python
# Before
@api_endpoint(rate_limit=FixedWindow(requests=100, window="1m"))

# After
@api_endpoint(rate_limit=FixedWindow(requests=50, window="1m"))
```

**Question**: Is reducing rate limit a breaking change?

**Arguments for "breaking"**:
- Clients expecting 100 req/min will now be throttled at 50
- This changes the service level agreement

**Arguments for "non-breaking"**:
- Rate limits are operational constraints, not API contract
- Clients should handle 429 responses gracefully regardless
- Service provider reserves right to adjust rate limits

**kacapi position**: **Non-breaking** (rate limit reductions are service degradations, not breaking changes)

However, governance policies can flag this:

```yaml
- name: rate-limit-reduction-warning
  level: warning
  rule: |
    changes.any(c => c.path == 'rate_limit.requests' && c.newValue < c.oldValue)
  message: "Rate limit reduced - consider notifying clients"
```

**Caching**:

```python
# Before
@api_endpoint(cache=Cache(ttl=300))

# After
@api_endpoint(cache=Cache(ttl=60))
```

**Question**: Is reducing cache TTL a breaking change?

**kacapi position**: **Non-breaking** (cache is transparent to clients, but governance can warn about performance impact)

### Principle 3: Security Changes

> Security-related changes are treated specially because they often have breaking impact even if technically non-breaking.

**Adding Authentication**:

```python
# Before
@api_endpoint()

# After
@api_endpoint(auth=Clerk())
```

**Technical analysis**: This is **breaking** because:
- Previously public endpoint now requires auth token
- Clients without tokens will receive 401 Unauthorized

**But consider**:

```python
# Before
@api_endpoint(auth=ApiKey())

# After
@api_endpoint(auth=any_of([Clerk(), ApiKey()]))
```

**This is non-breaking** because:
- Existing clients using API keys continue working
- New clients can use Clerk
- Auth became more permissive

**Rule**: Authentication changes are breaking if they make auth more restrictive, non-breaking if more permissive.

### Principle 4: Schema Evolution

Response schema changes follow specific rules:

**Adding Fields**:
```json
// Before
{"id": "123", "name": "Alice"}

// After
{"id": "123", "name": "Alice", "email": "alice@example.com"}
```

**Non-breaking** if clients are tolerant of unknown fields (most modern clients are).

**Removing Fields**:
```json
// Before
{"id": "123", "name": "Alice", "email": "alice@example.com"}

// After
{"id": "123", "name": "Alice"}
```

**Breaking** if any client depends on the removed field.

**Changing Field Types**:
```json
// Before
{"id": "123", "count": "100"}

// After
{"id": "123", "count": 100}
```

**Breaking** because clients expecting string will fail parsing number.

**Making Fields Nullable**:
```json
// Before
{"id": "123", "name": "Alice"}  // name always present

// After
{"id": "123", "name": null}  // name can be null
```

**Breaking** because clients not handling null will fail.

### Principle 5: Method Changes

Adding or removing HTTP methods to the same path:

```python
# Before
@api_endpoint()
@app.get("/users/{id}")

# After
@api_endpoint()
@app.get("/users/{id}")

@api_endpoint()
@app.patch("/users/{id}")  # Added
```

**Non-breaking** (adding PATCH doesn't affect existing GET).

But:

```python
# Before
@api_endpoint()
@app.post("/users")

# After
@api_endpoint()
@app.put("/users")  # Changed POST → PUT
```

**Breaking** if you consider this a removal of POST and addition of PUT.

### Edge Cases

**Case 1: Behavioral Change Without Schema Change**

```python
# Before
def get_users():
    return [user for user in db.users]  # Returns all users

# After
def get_users():
    return [user for user in db.users if user.active]  # Returns only active users
```

**Schema unchanged, but behavior changed significantly.**

**kacapi cannot detect this** because decorators and schema are unchanged. This is a limitation of emergent specs: behavioral changes without schema changes are invisible.

**Mitigation**: Rely on integration tests and monitoring to catch behavioral regressions.

**Case 2: Configuration Change with Behavior Impact**

```python
# Before
@api_endpoint(
    auth=Clerk(),
    validation=ValidateRequest(strict=False)  # Accepts extra fields
)

# After
@api_endpoint(
    auth=Clerk(),
    validation=ValidateRequest(strict=True)  # Rejects extra fields
)
```

**This is breaking** because clients sending extra fields will now get 400 Bad Request.

kacapi **can detect this** if the decorator differs properly handles validation config.

### Breaking Change Matrix

| Change | Breaking? | Rationale |
|--------|-----------|-----------|
| Remove endpoint | ✅ Yes | Clients calling endpoint will 404 |
| Add endpoint | ❌ No | New functionality, doesn't affect existing |
| Add required request field | ✅ Yes | Old requests missing field will fail validation |
| Add optional request field | ❌ No | Old requests still valid |
| Remove request field | ✅ Yes | Clients sending field may have unexpected behavior |
| Add response field | ❌ No | Clients ignore unknown fields |
| Remove response field | ✅ Yes | Clients expecting field will fail |
| Change field type | ✅ Yes | Clients parsing old type will fail |
| Make field nullable | ✅ Yes | Clients not handling null will fail |
| Make field non-nullable | ❌ No | Stronger guarantee |
| Add auth to public endpoint | ✅ Yes | Public access removed |
| Remove auth | ⚠️  Depends | Not breaking, but security concern |
| Make auth more restrictive | ✅ Yes | Users lose access |
| Make auth less restrictive | ❌ No | Users gain access |
| Reduce rate limit | ❌ No | Operational change, not API contract |
| Increase rate limit | ❌ No | More permissive |
| Change cache TTL | ❌ No | Transparent to clients |
| Add caching | ❌ No | Optimization, transparent |
| Remove caching | ❌ No | Performance impact, not breaking |
| Change CORS origins | ⚠️  Depends | Breaking if removes allowed origins |

## Automatic Version Proposal Strategies

kacapi supports multiple versioning strategies because different teams have different needs.

### Strategy Comparison

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| **Semantic Versioning** | Meaningful versions, industry standard, conveys compatibility | Requires accurate breaking change detection | Public APIs, SDKs |
| **Date-Based** | Easy to understand, chronological, no ambiguity | No compatibility information | Internal APIs, microservices |
| **Hash-Based** | Deterministic, no state needed, immutable | No human meaning | Immutable deployments, GitOps |
| **Manual** | Human judgment, flexible | Requires discipline, slower | Critical systems, regulated environments |

### Semantic Versioning (Detailed)

**Format**: `MAJOR.MINOR.PATCH`

**Rules**:
- **MAJOR**: Breaking changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes, config changes (backward compatible)

**Implementation**:

```typescript
interface SemverStrategy {
  type: 'semver';
  options: {
    // Start version if no previous snapshot
    initialVersion: string;  // default: "1.0.0"
    
    // How to handle config-only changes
    configChangesBumpPatch: boolean;  // default: true
    
    // Require explicit approval for major bumps
    majorBumpsRequireApproval: boolean;  // default: false
    
    // Pre-release suffixes
    preRelease?: string;  // e.g., "beta", "alpha"
  };
}

function proposeSemver(
  current: string,
  changes: ChangeDetection,
  options: SemverStrategy['options']
): string {
  const [major, minor, patch] = current.split('.').map(Number);
  
  // Determine bump type
  const breakingCount = changes.modified.filter(m => m.breaking).length +
                        changes.removed.length;
  
  if (breakingCount > 0) {
    const nextMajor = major + 1;
    const proposed = `${nextMajor}.0.0`;
    
    if (options.majorBumpsRequireApproval) {
      // Flag for manual approval
      return { proposed, requiresApproval: true };
    }
    
    return proposed;
  }
  
  const addedCount = changes.added.length;
  
  if (addedCount > 0) {
    return `${major}.${minor + 1}.0`;
  }
  
  const modifiedCount = changes.modified.length;
  
  if (modifiedCount > 0 && options.configChangesBumpPatch) {
    return `${major}.${minor}.${patch + 1}`;
  }
  
  // No changes
  return current;
}
```

**Pre-Release Versioning**:

```typescript
// For beta/alpha releases
const strategy: SemverStrategy = {
  type: 'semver',
  options: {
    preRelease: 'beta'
  }
};

// Versions produced:
// 1.0.0-beta.0
// 1.0.0-beta.1
// 1.0.0-beta.2
// 1.0.0  (when released)
```

### Date-Based Versioning (Detailed)

**Format**: `YYYY.MM.DD.SEQUENCE`

**Example**: `2024.1.15.0`

**Advantages**:
- Immediately clear when version was created
- No need to track "previous version" for comparison
- Simple increment logic
- Works well for continuous deployment

**Implementation**:

```typescript
interface DateBasedStrategy {
  type: 'date';
  options: {
    // Include time in version?
    includeTime: boolean;  // default: false
    // Format: YYYY.MM.DD.HH.MM
    
    // Timezone for date calculation
    timezone: string;  // default: "UTC"
    
    // Sequence reset period
    sequenceReset: 'daily' | 'weekly' | 'monthly';  // default: 'daily'
  };
}

function proposeDateBased(
  current: string,
  changes: ChangeDetection,
  options: DateBasedStrategy['options']
): string {
  const now = new Date();
  
  // Format date
  const year = now.getUTCFullYear();
  const month = now.getUTCMonth() + 1;
  const day = now.getUTCDate();
  
  const datePrefix = `${year}.${month}.${day}`;
  
  // Check if current version is from today
  if (current.startsWith(datePrefix)) {
    // Increment sequence
    const parts = current.split('.');
    const sequence = parseInt(parts[3] || '0') + 1;
    return `${datePrefix}.${sequence}`;
  }
  
  // New day, reset sequence
  return `${datePrefix}.0`;
}
```

**With Time**:

```typescript
// Version: 2024.1.15.14.30.0
//          YYYY.M.DD.HH.MM.Seq

// Allows multiple deployments per minute
```

### Hash-Based Versioning (Detailed)

**Format**: `[HASH]` (e.g., `a1b2c3d4`)

**Advantages**:
- Immutable: Same state always produces same hash
- No coordination needed: Multiple Observatories can independently compute same version
- Git-like: Familiar to developers
- Deterministic: No ambiguity about version

**Implementation**:

```typescript
interface HashBasedStrategy {
  type: 'hash';
  options: {
    // Hash algorithm
    algorithm: 'sha256' | 'sha1' | 'md5';  // default: 'sha256'
    
    // Hash length (first N characters)
    length: number;  // default: 8
    
    // Include metadata in hash?
    includeTimestamp: boolean;  // default: false
    includePlatform: boolean;  // default: true
  };
}

function proposeHashBased(
  snapshot: Snapshot,
  options: HashBasedStrategy['options']
): string {
  // Normalize snapshot for hashing
  const normalized = {
    endpoints: snapshot.endpoints.map(e => ({
      url: e.url,
      method: e.method,
      decorators: e.decorators,
      platform: options.includePlatform ? e.platform : undefined
    })).sort((a, b) => a.url.localeCompare(b.url))
  };
  
  // Compute hash
  const json = JSON.stringify(normalized);
  const hash = crypto.createHash(options.algorithm)
                     .update(json)
                     .digest('hex');
  
  return hash.substring(0, options.length);
}
```

**Collision Handling**:

With 8-character hashes (32 bits), collision probability is low but non-zero:
- 10,000 versions: ~1% collision chance
- 100,000 versions: ~10% collision chance

**Mitigation**: Increase hash length or append sequence number on collision.

### Manual Versioning (Detailed)

**Format**: Whatever the operator decides

**Workflow**:

```bash
# Step 1: Detect changes
$ kacapi diff
Changes detected:
- Added: GET /users/{id}
- Modified: POST /users (added required 'role' field)

Breaking changes: 1
Non-breaking changes: 1

Suggested version: 2.0.0 (based on semver strategy)

# Step 2: Human reviews and decides
$ kacapi version set 2.0.0
Version set to 2.0.0

# Step 3: Deploy with approved version
$ kacapi deploy
Deploying version 2.0.0...
```

**Approval Workflow**:

```yaml
# kacapi.governance.yaml
versioning:
  strategy: manual
  approval:
    required: true
    approvers:
      - team: platform-team
        minApprovals: 2
    notifyOn: breaking-changes
```

Integration with approval systems:
- GitHub PR reviews
- PagerDuty change requests
- ServiceNow tickets
- Slack approval bots

### Hybrid Strategies

**Example**: Semver for major/minor, date-based for patch

```typescript
interface HybridStrategy {
  type: 'hybrid';
  options: {
    majorMinor: 'semver',
    patch: 'date'
  };
}

// Versions produced:
// 1.0.20240115
// 1.0.20240116
// 1.1.20240116  (new feature added)
// 2.0.20240120  (breaking change)
```

## Comparison with Spec-First Approaches

### Spec-First (Traditional)

**Workflow**:

```
Design → Spec → Code → Deploy

1. Team designs API in meeting
2. Write OpenAPI spec
3. Review spec
4. Generate server stubs
5. Implement business logic
6. Deploy
7. (Manually keep spec updated)
```

**Advantages**:
- ✅ Contract established upfront
- ✅ Stakeholder alignment before coding
- ✅ Can generate docs/SDKs before implementation
- ✅ Clear versioning (developer explicitly bumps version)

**Disadvantages**:
- ❌ Spec inevitably drifts from implementation
- ❌ Dual maintenance burden
- ❌ Stale specs lead to distrust
- ❌ Manual synchronization is error-prone
- ❌ Doesn't capture gateway behavior (auth, rate limits, etc.)

### Code-First (kacapi)

**Workflow**:

```
Code → Deploy → Discover → Spec

1. Developer writes code with decorators
2. Deploy to platform
3. kacapi discovers endpoints
4. Spec emerges from observation
5. Versioning happens automatically
```

**Advantages**:
- ✅ Single source of truth (deployed code)
- ✅ Spec always reflects reality
- ✅ No drift possible
- ✅ Captures gateway behavior (auth, rate limits, etc.)
- ✅ Automatic version proposals

**Disadvantages**:
- ❌ No spec until after deployment
- ❌ Requires platform with discoverable endpoints
- ❌ Breaking change detection is heuristic-based
- ❌ Less upfront design (can be good or bad)

### Hybrid: Contract Testing

Some teams use contract testing (Pact, Spring Cloud Contract):

**Workflow**:

```
Code → Consumer Contract → Verify → Deploy

1. Consumers define expected API behavior
2. Contract tests verify provider matches expectations
3. Deploy if tests pass
```

**How kacapi fits**:
- kacapi generates OpenAPI spec from deployed code
- Consumers generate contract tests from spec
- Contract tests run against deployed endpoints
- Failures trigger alerts but don't block (emergent model)

### Side-by-Side Comparison

| Aspect | Spec-First | Code-First (kacapi) | Contract Testing |
|--------|-----------|---------------------|------------------|
| **Source of Truth** | OpenAPI spec | Deployed code | Consumer contracts |
| **When Spec Created** | Before coding | After deployment | During consumer development |
| **Spec Accuracy** | Often stale | Always accurate | As accurate as tests |
| **Versioning** | Manual | Automatic | Manual |
| **Breaking Detection** | Manual or tooling | Automatic (heuristic) | Test failures |
| **Gateway Behavior** | Not captured | Fully captured | Partially captured |
| **Upfront Design** | Required | Optional | Consumer-driven |
| **Maintenance Burden** | High (dual updates) | Low (code only) | Medium (maintain tests) |
| **Platform Support** | Any | Discoverable only | Any |

## Edge Cases and Challenges

### Edge Case 1: Simultaneous Deployments

**Scenario**:

```
Time 10:00: Developer A deploys new endpoint GET /users/{id}
Time 10:01: Developer B deploys new endpoint GET /posts/{id}
Time 10:05: Observatory collector runs
```

**Problem**: Observatory sees both changes at once. Should this be:
- Two minor version bumps? (1.0.0 → 1.1.0 → 1.2.0)
- One minor version bump? (1.0.0 → 1.1.0)

**Solution**: Treat as single changeset. Version 1.0.0 → 1.1.0.

**Rationale**: Observatory operates on snapshots, not individual deployments. All changes between snapshots are grouped.

### Edge Case 2: Rollback

**Scenario**:

```
Version 1.0.0: GET /users
Version 1.1.0: GET /users, GET /users/{id}
Rollback to 1.0.0: GET /users
```

**Problem**: Observatory sees GET /users/{id} was removed. This is a breaking change. Should it propose 2.0.0?

**Solution**: Detect rollback and restore previous version instead of bumping.

**Implementation**:

```typescript
function detectRollback(
  current: Snapshot,
  previous: Snapshot[]
): Snapshot | null {
  // Check if current matches any previous snapshot exactly
  for (const prev of previous) {
    if (snapshotsEqual(current, prev)) {
      return prev;  // This is a rollback
    }
  }
  return null;
}

// In versioner:
const rollback = detectRollback(newSnapshot, snapshotHistory);
if (rollback) {
  return rollback.version;  // Restore old version
}
```

### Edge Case 3: Multiple Platforms with Different States

**Scenario**:

```
Modal:
  - GET /users (version 1.0.0)
  - GET /users/{id} (version 1.1.0)

Vercel:
  - GET /posts (version 1.0.0)

Question: What's the overall version?
```

**Solution Options**:

**Option 1: Separate Versioning**
```
modal.api.version = 1.1.0
vercel.api.version = 1.0.0
```

**Option 2: Unified Versioning**
```
overall.api.version = 1.1.0
(max of all platform versions)
```

**Option 3: Platform-Prefixed**
```
modal-1.1.0
vercel-1.0.0
```

**kacapi default**: Option 1 (separate versioning per platform).

### Edge Case 4: Endpoint URL Changes

**Scenario**:

```
Before: https://user--get-users-v1.modal.run
After:  https://user--get-users-v2.modal.run
```

**Problem**: Is this a new endpoint or a modified endpoint?

**Solution**: Use endpoint "identity" based on semantic path, not URL:

```typescript
interface EndpointIdentity {
  semanticPath: string;  // e.g., "/users"
  method: string;        // e.g., "GET"
  platform: string;      // e.g., "modal"
}

// URLs can change, but identity is stable
```

### Edge Case 5: Metadata Extraction Failures

**Scenario**: Deployed function has kacapi decorators, but connector fails to extract metadata.

**Possible causes**:
- Decorator syntax error
- Unsupported decorator version
- Platform API changed
- Network error

**Solution**: Treat as "unknown" state and flag for manual review:

```typescript
interface EndpointSnapshot {
  url: string;
  method: string;
  decorators: DecoratorMetadata | null;  // null if extraction failed
  extractionError?: string;
}

// In differ:
if (!newEndpoint.decorators) {
  warnings.push({
    endpoint: newEndpoint.url,
    message: `Failed to extract metadata: ${newEndpoint.extractionError}`
  });
}
```

### Edge Case 6: Configuration Drift

**Scenario**: Developer modifies code locally but doesn't deploy.

**Local code**:
```python
@api_endpoint(auth=Clerk())  # Added auth
```

**Deployed code**:
```python
@api_endpoint()  # No auth
```

**Problem**: Observatory only sees deployed state. Local changes are invisible.

**This is intended behavior**: kacapi only manages what's deployed. Local changes don't exist in the kacapi model until deployed.

**Mitigation**: CI/CD pre-deployment checks can warn about local-vs-deployed drift.

## The Feedback Loop

kacapi creates a unique feedback loop:

```
Code → Deploy → Discover → Version → Spec → SDK → Clients → Telemetry → Analyze → Insights → Code

                                                                                    ↑___________|
```

### Loop Components

**1. Telemetry Collection**

Gateway collects metrics:
- Request count per endpoint
- Error rates
- Latency percentiles
- Rate limit hit rate

**2. Traffic Analysis**

Observatory analyzes telemetry:
- Which endpoints are heavily used?
- Which are dormant?
- Which have high error rates?
- Which hit rate limits frequently?

**3. Insights**

Observatory generates actionable insights:

```
Insight: Endpoint GET /analytics has P95 latency of 2.5s
Recommendation: Add caching with TTL=300

Insight: Endpoint GET /users hits rate limit 150 times/day
Recommendation: Increase rate limit from 100 to 200 req/min

Insight: Endpoint GET /reports has zero traffic in 30 days
Recommendation: Consider deprecating
```

**4. Code Changes**

Developer acts on insights:

```python
# Before
@api_endpoint(
    rate_limit=FixedWindow(requests=100, window="1m")
)

# After (responding to insight)
@api_endpoint(
    rate_limit=FixedWindow(requests=200, window="1m"),
    cache=Cache(ttl=300)
)
```

**5. Loop Continues**

- New code deployed
- Observatory detects changes
- Version bumped (config change = patch)
- Gateway updated
- Telemetry shows improved metrics
- New insights generated

### Continuous Improvement

This feedback loop enables **continuous, data-driven API improvement**:

- No guessing about rate limits (telemetry shows actual usage)
- No over-provisioning (identify dormant endpoints)
- No under-provisioning (detect rate limit hits)
- Objective deprecation decisions (based on usage data)

## Best Practices

### 1. Embrace Emergent Specifications

**Don't**: Try to write comprehensive specs upfront

**Do**: Write minimal decorators and let specs emerge

```python
# ✅ Good: Minimal, clear decorators
@api_endpoint(
    auth=Clerk(),
    rate_limit=FixedWindow(requests=100, window="1m")
)

# ❌ Bad: Over-specified (spec-first thinking)
# Just use decorators, don't write separate spec
```

### 2. Trust the Observatory

**Don't**: Manually bump version numbers in specs

**Do**: Let Observatory propose versions based on changes

```bash
# ❌ Bad: Manual versioning
echo "version: 2.0.0" >> openapi.yaml

# ✅ Good: Automatic versioning
kacapi deploy  # Observatory handles versioning
```

### 3. Use Governance to Encode Standards

**Don't**: Rely on code review to catch policy violations

**Do**: Encode policies in governance rules

```yaml
# ✅ Good: Automated enforcement
policies:
  - name: require-auth
    rule: all_endpoints.auth != null
    level: error
```

### 4. Monitor Traffic and Act on Insights

**Don't**: Ignore Observatory traffic analysis

**Do**: Regularly review insights and adjust code

```bash
# Weekly review
kacapi insights --since 7d

# Act on recommendations
# - Add caching to slow endpoints
# - Adjust rate limits based on usage
# - Deprecate dormant endpoints
```

### 5. Test Breaking Change Detection

**Don't**: Assume Observatory perfectly detects breaking changes

**Do**: Validate with integration tests

```python
# integration_test.py
def test_no_breaking_changes():
    """Ensure our changes don't break clients."""
    # Run against previous version
    old_response = call_endpoint_v1()
    
    # Run against new version
    new_response = call_endpoint_v2()
    
    # Validate compatibility
    assert new_response.schema >= old_response.schema
```

### 6. Document Intent in Code Comments

**Don't**: Rely on specs to document "why"

**Do**: Comment code with intent (especially for breaking changes)

```python
@api_endpoint(
    auth=Clerk()  # Added 2024-01-15: Public endpoint had abuse issues
)
def get_analytics():
    """
    Analytics endpoint.
    
    BREAKING CHANGE (v2.0.0): Now requires authentication.
    Reason: High volume of unauthenticated abuse traffic.
    Migration: Clients must obtain auth token before calling.
    """
    pass
```

### 7. Use Pre-Release Versions for Testing

**Don't**: Deploy breaking changes directly to production

**Do**: Use pre-release versions (beta, alpha) for validation

```bash
# Deploy to staging with beta version
kacapi deploy --env staging --version 2.0.0-beta.0

# Validate with subset of clients
# Promote to production when ready
kacapi deploy --env production --version 2.0.0
```

### 8. Maintain Changelog

**Don't**: Rely solely on generated specs for change history

**Do**: Maintain human-readable changelog

```markdown
# CHANGELOG.md

## [2.0.0] - 2024-01-15

### Breaking Changes
- GET /analytics: Now requires authentication (previously public)

### Added
- GET /users/{id}: Fetch individual user details

### Changed
- GET /users: Increased rate limit from 100 to 200 req/min
```

### 9. Coordinate Breaking Changes

**Don't**: Deploy breaking changes without client coordination

**Do**: Notify clients and provide migration window

```python
# Step 1: Add new endpoint with new behavior
@api_endpoint()
def get_analytics_v2():  # New endpoint, non-breaking
    pass

# Step 2: Deprecate old endpoint
@api_endpoint(
    deprecated=Deprecated(
        sunset="2024-03-01",
        migration="Use /analytics/v2 instead"
    )
)
def get_analytics():  # Old endpoint, deprecated but still works
    pass

# Step 3: After sunset date, remove old endpoint (breaking change)
```

### 10. Version Per Environment

**Don't**: Use same version across all environments

**Do**: Maintain separate versions for staging/production

```
staging.api.version = 2.1.0-beta.3
production.api.version = 2.0.0
```

This allows testing breaking changes in staging before promoting to production.

## Conclusion

Emergent specifications represent a fundamental inversion of traditional API development: instead of prescribing what an API should be, kacapi observes what it actually is. This approach:

- Eliminates spec drift (specs always match reality)
- Reduces maintenance burden (single source of truth: code)
- Enables automatic versioning (system proposes versions based on changes)
- Captures complete gateway behavior (auth, rate limits, caching, etc.)
- Creates feedback loops (telemetry → insights → code improvements)

The Observatory subsystem makes this possible through:
- **Collector**: Discovers deployed endpoints via platform APIs
- **Differ**: Compares snapshots and detects changes
- **Classifier**: Determines breaking vs non-breaking changes
- **Versioner**: Proposes versions using pluggable strategies
- **Governor**: Enforces governance policies before deployment

While emergent specifications have limitations (behavioral changes without schema changes are invisible, requires discoverable platforms), they offer a compelling alternative to spec-first approaches for teams building APIs on modern serverless platforms.

The key insight: **code is truth, specs are documentation**. By inverting the traditional flow, kacapi ensures documentation is always accurate because it's generated from the running system, not written before it.
