# Platform-First Architecture

This document explains why kacapi only supports platforms with discoverable URLs and why this constraint is a feature, not a limitation.

## Table of Contents

- [The Discoverability Requirement](#the-discoverability-requirement)
- [Supported Platforms](#supported-platforms)
- [Why Not Traditional Infrastructure?](#why-not-traditional-infrastructure)
- [Platform Viability Matrix](#platform-viability-matrix)
- [Technical Requirements](#technical-requirements)
- [Future Platform Considerations](#future-platform-considerations)

## The Discoverability Requirement

kacapi's entire architecture depends on one critical capability:

> **Can we programmatically discover all deployed endpoints and their URLs?**

If the answer is **yes**, the platform is supported. If **no**, it's not.

### Why This Matters

kacapi's workflow requires:

1. **Listing all deployments** (e.g., all Modal functions)
2. **Extracting endpoint URLs** (e.g., `https://user--fn-abc123.modal.run`)
3. **Fetching metadata** (decorator information)
4. **Validating endpoints** (checking they're reachable)

Without programmatic discovery, kacapi cannot function.

### Example: Modal (Supported)

```python
# Developer deploys this
@modal.function()
def my_endpoint():
    pass
```

kacapi can discover this because:

```python
# Using Modal's API
import modal

app = modal.App.lookup("my-workspace/my-app")
functions = app.list_functions()

for func in functions:
    url = func.web_url  # ✅ Modal provides the URL
    metadata = func.metadata  # ✅ Modal provides metadata
```

### Example: Traditional VM (Not Supported)

```python
# Developer runs this on a VM
@app.route('/my-endpoint')
def my_endpoint():
    pass
```

kacapi **cannot** discover this because:
- No API to list "all endpoints on this VM"
- URL might be `http://10.0.0.5:8080/my-endpoint` (internal IP)
- No standard way to extract metadata
- Multiple processes might be running on the same VM

## Supported Platforms

kacapi supports platforms that provide:

1. **API-based endpoint enumeration**
2. **Publicly accessible URLs** (or deterministic URL patterns)
3. **Metadata access** (source code, decorators, or reflection)

### ✅ Modal

**Why supported:**
- API for listing apps and functions
- Each function gets a deterministic web URL: `https://user--fn-name.modal.run`
- Can introspect function decorators
- Deployments are atomic and versioned

**Discovery method:**
```python
modal_client.apps.list()  # List all apps
modal_app.functions.list()  # List all functions
function.web_url  # Get URL
```

**Metadata extraction:**
- Function source code accessible via API
- Decorators embedded in function metadata
- Can parse Python AST to extract `@api_endpoint` decorators

### ✅ Vercel

**Why supported:**
- API for listing projects and deployments
- Each deployment has a unique URL: `https://project-abc123.vercel.app`
- Can fetch serverless function definitions
- Environment variables and config accessible

**Discovery method:**
```javascript
vercel.deployments.list()  // List all deployments
vercel.functions.list(deploymentId)  // List functions in deployment
deployment.url  // Get URL
```

**Metadata extraction:**
- Fetch function source from deployment
- Parse JSDoc or TypeScript decorators
- Extract `@apiEndpoint` from code

### ✅ Supabase Edge Functions

**Why supported:**
- API for listing projects and edge functions
- Functions have predictable URLs: `https://project-ref.supabase.co/functions/v1/function-name`
- Can fetch function source code
- Supabase CLI and Management API provide access

**Discovery method:**
```typescript
supabase.functions.list()  // List all edge functions
function.url  // Construct URL from project ref
```

**Metadata extraction:**
- Functions are TypeScript/JavaScript files
- Decorators or exported metadata objects
- Parse source via TypeScript compiler API

### ✅ Cloudflare Workers

**Why supported:**
- Workers API for listing all workers
- Each worker has a URL: `https://worker-name.user.workers.dev`
- Can fetch worker script source
- Routes and metadata accessible via API

**Discovery method:**
```javascript
cloudflare.workers.list()  // List all workers
worker.routes  // Get all routes
```

**Metadata extraction:**
- Worker scripts are JavaScript/TypeScript
- Parse exported handlers
- Extract metadata from script comments or exports

## Why Not Traditional Infrastructure?

### ❌ Kubernetes Pods

**Problem: No URL discoverability**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
```

Questions kacapi can't answer:
- What endpoints does this deployment expose?
- What are their URLs? (Internal cluster IPs? Load balancer? Ingress?)
- Which replica should we query for metadata?
- How do we extract decorator information?

**Possible future support:**
If we added a Kubernetes operator that:
- Watches for pods with kacapi annotations
- Queries pods for endpoint metadata
- Maintains a registry of endpoints

Then Kubernetes could be supported. But it's significantly more complex than platform-native discovery.

### ❌ AWS Lambda (Bare)

**Problem: No unified endpoint listing**

Lambda functions can be invoked via:
- API Gateway (multiple APIs possible)
- ALB (Application Load Balancer)
- Function URL (different per function)
- Direct invocation (no HTTP URL)

Questions kacapi can't answer:
- Which Lambda functions are HTTP endpoints?
- What are their URLs?
- Which API Gateway do they belong to?

**Possible future support:**
If we required:
- All Lambda functions use Function URLs
- Metadata stored in Lambda tags
- Specific naming conventions

Then bare Lambda could be supported. But it's fragile.

### ❌ Google Cloud Run

**Problem: Partial discoverability**

Cloud Run services have URLs: `https://service-abc123.run.app`

But:
- No API to discover "all services in organization"
- Need to know project ID upfront
- Service might have multiple endpoints (single service, multiple routes)
- Metadata extraction requires container introspection

**Possible future support:**
Cloud Run is closer to viable than Kubernetes/Lambda. Could support if:
- Organizations maintain a registry of Cloud Run services
- Services expose metadata via special endpoint (e.g., `/_kacapi/metadata`)
- kacapi polls known services

### ❌ Traditional VMs / EC2 Instances

**Problem: No endpoint discovery**

A VM could be running:
- Nginx with PHP
- Node.js with Express
- Python with FastAPI
- Multiple services on different ports
- Nothing at all

No standard way to:
- Enumerate endpoints
- Discover URLs
- Extract metadata

**Could never support** without significant agent installation and introspection.

## Platform Viability Matrix

| Platform | URL Discovery | Metadata Access | Atomic Deploys | Versioning | Status |
|----------|--------------|----------------|----------------|------------|--------|
| **Modal** | ✅ API | ✅ Source code | ✅ Yes | ✅ Built-in | ✅ Supported |
| **Vercel** | ✅ API | ✅ Source code | ✅ Yes | ✅ Built-in | ✅ Supported |
| **Supabase** | ✅ API | ✅ Source code | ✅ Yes | ✅ Git-based | ✅ Supported |
| **Cloudflare Workers** | ✅ API | ✅ Source code | ✅ Yes | ✅ Built-in | ✅ Supported |
| **AWS Lambda (bare)** | ⚠️ Partial | ⚠️ Limited | ✅ Yes | ⚠️ Manual | ❌ Not supported |
| **Google Cloud Run** | ⚠️ Partial | ❌ No | ✅ Yes | ✅ Built-in | ❌ Not supported |
| **Kubernetes** | ❌ No | ❌ No | ⚠️ Depends | ⚠️ Depends | ❌ Not supported |
| **Docker (bare)** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ Not supported |
| **VMs / EC2** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ Not supported |

### Legend

- ✅ **Yes**: Platform provides this capability natively
- ⚠️ **Partial**: Possible but requires additional setup
- ❌ **No**: Not available or not feasible

## Technical Requirements

For a platform to be viable for kacapi, it must provide:

### 1. **Endpoint Enumeration API**

```typescript
interface PlatformConnector {
  // Must be able to list all deployments
  listDeployments(): Promise<Deployment[]>;
  
  // Must be able to list endpoints per deployment
  getEndpoints(deploymentId: string): Promise<Endpoint[]>;
}
```

**Example: Modal**
```python
modal_client.list_apps()  # ✅ Can enumerate
```

**Counter-example: Kubernetes**
```bash
kubectl get pods  # ❌ Pods aren't endpoints
kubectl get services  # ❌ Services are load balancers, not endpoints
kubectl get ingress  # ⚠️ Closer, but incomplete
```

### 2. **Deterministic URL Generation**

Given an endpoint identifier, must be able to construct or retrieve its URL:

**Modal**: `https://{user}--{function-name}.modal.run`
**Vercel**: `https://{deployment-id}.vercel.app`
**Supabase**: `https://{project-ref}.supabase.co/functions/v1/{name}`
**Cloudflare**: `https://{worker-name}.{subdomain}.workers.dev`

**Why deterministic?** 
- Enables caching (don't need to query API every time)
- Allows parallel discovery (can construct URLs independently)
- Simplifies connector implementation

### 3. **Metadata Access**

Must be able to extract decorator metadata via one of:

**Option A: Source code access**
```python
# Platform provides function source
source = platform.get_function_source(function_id)
# Parse decorators from source
metadata = parse_decorators(source)
```

**Option B: Metadata API**
```python
# Platform stores metadata separately
metadata = platform.get_function_metadata(function_id)
# Metadata includes decorator information
```

**Option C: Runtime introspection**
```python
# Query running function for its metadata
response = requests.get(f"{function_url}/_kacapi/metadata")
metadata = response.json()
```

### 4. **Authentication**

Must provide API credentials for programmatic access:

- **Modal**: Token-based (`MODAL_TOKEN_ID`, `MODAL_TOKEN_SECRET`)
- **Vercel**: Bearer token (`VERCEL_TOKEN`)
- **Supabase**: Service role key (`SUPABASE_SERVICE_KEY`)
- **Cloudflare**: API token (`CF_API_TOKEN`)

## Discovery Workflow

Here's how kacapi discovers endpoints on supported platforms:

### Modal Example

```python
from kacapi.connectors.modal import ModalConnector

connector = ModalConnector(token=os.environ['MODAL_TOKEN'])

# 1. Authenticate
await connector.authenticate()

# 2. List all apps in workspace
apps = await connector.list_deployments()
# Returns: [
#   {'id': 'app-1', 'name': 'my-api'},
#   {'id': 'app-2', 'name': 'background-jobs'}
# ]

# 3. For each app, get endpoints
for app in apps:
    endpoints = await connector.get_endpoints(app['id'])
    # Returns: [
    #   {
    #     'id': 'fn-abc123',
    #     'name': 'get_users',
    #     'url': 'https://user--get-users.modal.run',
    #     'method': 'GET'
    #   }
    # ]
    
    # 4. For each endpoint, extract metadata
    for endpoint in endpoints:
        metadata = await connector.get_metadata(endpoint)
        # Returns: {
        #   'auth': {'provider': 'clerk'},
        #   'rate_limit': {'requests': 100, 'window': '1m'},
        #   ...
        # }
```

### Vercel Example

```javascript
const connector = new VercelConnector(token: process.env.VERCEL_TOKEN);

// 1. Authenticate
await connector.authenticate();

// 2. List all projects
const projects = await connector.listDeployments();

// 3. For each project, get latest deployment
for (const project of projects) {
  const deployment = await connector.getLatestDeployment(project.id);
  
  // 4. Get serverless functions from deployment
  const endpoints = await connector.getEndpoints(deployment.id);
  
  // 5. Extract metadata from function source
  for (const endpoint of endpoints) {
    const metadata = await connector.getMetadata(endpoint);
  }
}
```

## Constraint as Feature

The platform discoverability constraint is **intentional**. It ensures:

### 1. **Consistency**

All supported platforms follow the same discovery pattern:
- List deployments
- Get endpoints
- Extract metadata

This consistency simplifies implementation and testing.

### 2. **Reliability**

Platforms with discoverable URLs typically:
- Have stable APIs
- Provide versioning
- Maintain backward compatibility
- Have good documentation

These are exactly the platforms we want to support.

### 3. **Modern Architecture**

Platforms with discoverable endpoints are typically:
- Serverless or serverless-like
- Cloud-native
- Built for API-first development
- Have strong developer tooling

These platforms align with kacapi's philosophy.

### 4. **Clear Scope**

By establishing clear platform requirements, we avoid:
- Endless feature requests ("support my custom setup")
- Complex edge cases (manual URL configuration)
- Maintenance burden (supporting fragile platforms)

## Future Platform Considerations

### Potential Additions

**AWS App Runner**
- ✅ Has URL per service
- ✅ Has API for listing services
- ⚠️ Metadata extraction would need custom endpoint

**Fly.io**
- ✅ Has API for listing apps
- ✅ Has deterministic URLs
- ⚠️ Metadata extraction would need custom endpoint

**Railway**
- ✅ Has API for listing services
- ✅ Has URLs per deployment
- ⚠️ Metadata access unclear

**Deno Deploy**
- ✅ Has API for listing deployments
- ✅ Has deterministic URLs
- ✅ Source code accessible
- ✅ **Good candidate for future support**

### Platform Connector Interface

All platform connectors must implement:

```typescript
interface PlatformConnector {
  // Authenticate with platform
  authenticate(credentials: PlatformCredentials): Promise<void>;
  
  // List all deployments (apps, projects, services, etc.)
  listDeployments(): Promise<Deployment[]>;
  
  // Get endpoints for a specific deployment
  getEndpoints(deploymentId: string): Promise<Endpoint[]>;
  
  // Extract kacapi decorator metadata from endpoint
  getMetadata(endpoint: Endpoint): Promise<DecoratorMetadata>;
  
  // Validate that endpoint URL is reachable
  validateEndpoint(url: string): Promise<boolean>;
  
  // Get platform-specific configuration
  getPlatformConfig(): PlatformConfig;
}
```

See [connectors/overview.md](../connectors/overview.md) for detailed interface specification.

## Workarounds for Unsupported Platforms

If you're on an unsupported platform, options include:

### 1. **Metadata Endpoint**

Add a special endpoint to your service:

```python
@app.get("/_kacapi/metadata")
def kacapi_metadata():
    return {
        "endpoints": [
            {
                "path": "/users",
                "method": "GET",
                "decorators": {
                    "auth": {"provider": "clerk"},
                    "rate_limit": {"requests": 100}
                }
            }
        ]
    }
```

Then kacapi could poll known services for this metadata.

### 2. **Manual Registry**

Maintain a registry file:

```yaml
# kacapi-registry.yaml
endpoints:
  - url: https://my-server.com/api/users
    method: GET
    decorators:
      auth:
        provider: clerk
      rate_limit:
        requests: 100
```

kacapi reads this instead of discovering automatically.

### 3. **Service Mesh Integration**

If using a service mesh (Istio, Linkerd), extract endpoint information from mesh configuration.

**Note**: These workarounds defeat kacapi's core benefit (automatic discovery). Only consider if absolutely necessary.

## Conclusion

kacapi's platform-first constraint is a deliberate design decision. By only supporting platforms with discoverable URLs:

- We ensure reliable endpoint discovery
- We simplify connector implementation
- We target modern, cloud-native architectures
- We provide a consistent experience across platforms

This constraint enables kacapi's core value proposition: **automatic, code-first gateway management**.

If your platform isn't supported, consider:
1. Whether one of the supported platforms could work for your use case
2. Whether the platform could add discovery APIs
3. Whether a workaround (metadata endpoint, registry file) is acceptable

The platform-first approach is what makes kacapi possible.
