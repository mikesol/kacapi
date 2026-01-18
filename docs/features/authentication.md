# Authentication in kacapi

This document provides a comprehensive guide to authentication in kacapi, covering all supported authentication providers, strategies, and integration patterns. Authentication is a first-class feature in kacapi's code-first approach, allowing you to declare auth requirements directly in your endpoint decorators and have them automatically enforced at the gateway level.

## Table of Contents

- [Overview](#overview)
- [Authentication Philosophy](#authentication-philosophy)
- [Auth Connectors](#auth-connectors)
- [Provider Connectors](#provider-connectors)
  - [Clerk](#clerk)
  - [Auth0](#auth0)
  - [AWS Cognito](#aws-cognito)
  - [Okta](#okta)
  - [Supabase Auth](#supabase-auth)
  - [Firebase Auth](#firebase-auth)
  - [WorkOS](#workos)
- [Generic Authentication](#generic-authentication)
  - [JWT Authentication](#jwt-authentication)
  - [API Keys](#api-keys)
  - [mTLS (Mutual TLS)](#mtls-mutual-tls)
- [Combining Strategies](#combining-strategies)
  - [any_of (OR Logic)](#any_of-or-logic)
  - [all_of (AND Logic)](#all_of-and-logic)
- [AuthConnector Interface](#authconnector-interface)
- [Auth Context Injection](#auth-context-injection)
- [Security Considerations](#security-considerations)
- [Gateway Integration](#gateway-integration)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Overview

kacapi authentication moves auth configuration from gateway consoles and config files into your codebase as decorators. When you deploy an endpoint with an auth decorator, kacapi:

1. **Discovers** the endpoint and extracts auth requirements
2. **Validates** the auth configuration against security policies
3. **Generates** gateway-specific auth configurations
4. **Deploys** the auth rules to your API gateway
5. **Injects** authenticated user context into backend requests

This approach eliminates configuration drift between your code and gateway, ensures consistent auth enforcement, and provides automatic documentation of auth requirements.

### Quick Example

```python
from kacapi.modal import api_endpoint
from kacapi.auth import Clerk

@api_endpoint(auth=Clerk())
@modal.function()
def protected_endpoint():
    return {"message": "This endpoint requires Clerk authentication"}
```

That's it. No separate gateway configuration, no manual JWT validation, no context injection logic. kacapi handles everything.

## Authentication Philosophy

### Code as Source of Truth

Traditional API development separates auth configuration from code:

```
Code (endpoint logic) ←─→ Gateway Config (auth rules)
     ↓                           ↓
  Deployed                   Manually synced
```

This creates problems:
- **Drift**: Code and config diverge over time
- **Coordination**: Teams must coordinate changes across systems
- **Documentation**: Auth requirements hidden in gateway consoles
- **Testing**: Local development can't replicate gateway auth

kacapi inverts this:

```
Code (with @api_endpoint(auth=...)) → Deployed → Gateway Config Generated
                ↓
        Single Source of Truth
```

Benefits:
- **Single Source**: Auth lives in code, gateway reflects code
- **Self-Documenting**: Decorators document auth requirements
- **Type-Safe**: SDKs provide autocomplete and validation
- **Testable**: Local auth middleware matches gateway behavior

### Gateway-Agnostic Abstraction

kacapi provides a unified auth interface that works across different gateways:

```python
# Same decorator works with Kong, AWS API Gateway, CloudFront, APISIX
@api_endpoint(auth=Auth0(audience="https://api.example.com"))
```

Gateway adapters translate this abstract declaration into gateway-specific configurations:

| Provider | Kong | AWS API Gateway | CloudFront | APISIX |
|----------|------|-----------------|------------|--------|
| Clerk | jwt plugin + Clerk JWKS | Lambda authorizer | Lambda@Edge | jwt-auth plugin |
| Auth0 | jwt plugin + Auth0 JWKS | Lambda authorizer | Lambda@Edge | jwt-auth plugin |
| API Keys | key-auth plugin | API key source | Lambda@Edge | key-auth plugin |
| mTLS | mtls-auth plugin | Mutual TLS config | CloudFront HTTPS policy | mTLS plugin |

You write auth once, deploy to any gateway.

## Auth Connectors

Auth connectors are kacapi's abstraction layer between authentication providers and API gateways. Each connector implements a standard interface for token validation, context extraction, and gateway configuration generation.

### AuthConnector Interface

All auth connectors implement this TypeScript interface:

```typescript
interface AuthConnector {
  /**
   * Unique identifier for this auth provider
   */
  readonly provider: string;

  /**
   * Validate an authentication token
   * 
   * @param token - The authentication token (JWT, API key, etc.)
   * @param context - Optional request context (IP, headers, etc.)
   * @returns Promise resolving to validation result
   */
  validateToken(
    token: string,
    context?: RequestContext
  ): Promise<AuthValidationResult>;

  /**
   * Extract user context from a validated token
   * 
   * @param token - The validated authentication token
   * @returns Promise resolving to user context
   */
  extractContext(token: string): Promise<UserContext>;

  /**
   * Get gateway-specific configuration for this auth provider
   * 
   * @param gatewayType - Target gateway (kong, aws-apigw, cloudfront, apisix)
   * @returns Promise resolving to gateway auth configuration
   */
  getGatewayConfig(gatewayType: GatewayType): Promise<GatewayAuthConfig>;

  /**
   * Validate connector configuration
   * 
   * @returns Promise resolving to validation result
   */
  validateConfig(): Promise<ValidationResult>;

  /**
   * Get JWKS (JSON Web Key Set) URL if applicable
   * 
   * @returns JWKS URL or null if not applicable
   */
  getJwksUrl(): string | null;

  /**
   * Get claims to extract from token
   * 
   * @returns Array of claim names to extract
   */
  getClaimsToExtract(): string[];

  /**
   * Get token location configuration
   * 
   * @returns Token location settings (header, cookie, query param)
   */
  getTokenLocation(): TokenLocation;
}

interface AuthValidationResult {
  valid: boolean;
  userId?: string;
  error?: AuthError;
  expiresAt?: number;
  metadata?: Record<string, any>;
}

interface UserContext {
  userId: string;
  email?: string;
  username?: string;
  orgId?: string;
  tenantId?: string;
  roles: string[];
  permissions: string[];
  metadata: Record<string, any>;
  sessionId?: string;
  deviceId?: string;
}

interface GatewayAuthConfig {
  type: 'jwt' | 'oauth2' | 'api-key' | 'mtls' | 'custom';
  config: Record<string, any>;
  plugins?: GatewayPlugin[];
  headers?: HeaderInjection[];
}

interface TokenLocation {
  headers?: string[];  // e.g., ["Authorization", "X-API-Key"]
  cookies?: string[];  // e.g., ["session_token"]
  queryParams?: string[];  // e.g., ["api_key"]
  extractionPattern?: string;  // Regex pattern for extraction
}

interface RequestContext {
  ip: string;
  userAgent: string;
  headers: Record<string, string>;
  path: string;
  method: string;
}
```

### Python AuthConnector Interface

The Python SDK provides an equivalent abstract base class:

```python
from abc import ABC, abstractmethod
from typing import Optional, Dict, Any, List
from dataclasses import dataclass

@dataclass
class AuthValidationResult:
    valid: bool
    user_id: Optional[str] = None
    error: Optional[str] = None
    expires_at: Optional[int] = None
    metadata: Optional[Dict[str, Any]] = None

@dataclass
class UserContext:
    user_id: str
    email: Optional[str] = None
    username: Optional[str] = None
    org_id: Optional[str] = None
    tenant_id: Optional[str] = None
    roles: List[str] = None
    permissions: List[str] = None
    metadata: Dict[str, Any] = None
    session_id: Optional[str] = None
    device_id: Optional[str] = None

class AuthConnector(ABC):
    """Base class for all authentication connectors"""
    
    @property
    @abstractmethod
    def provider(self) -> str:
        """Unique identifier for this auth provider"""
        pass
    
    @abstractmethod
    async def validate_token(
        self,
        token: str,
        context: Optional[Dict[str, Any]] = None
    ) -> AuthValidationResult:
        """Validate an authentication token"""
        pass
    
    @abstractmethod
    async def extract_context(self, token: str) -> UserContext:
        """Extract user context from validated token"""
        pass
    
    @abstractmethod
    async def get_gateway_config(
        self,
        gateway_type: str
    ) -> Dict[str, Any]:
        """Get gateway-specific auth configuration"""
        pass
    
    @abstractmethod
    async def validate_config(self) -> Dict[str, Any]:
        """Validate connector configuration"""
        pass
    
    def get_jwks_url(self) -> Optional[str]:
        """Get JWKS URL if applicable"""
        return None
    
    def get_claims_to_extract(self) -> List[str]:
        """Get claims to extract from token"""
        return ["sub", "email", "roles"]
    
    def get_token_location(self) -> Dict[str, Any]:
        """Get token location configuration"""
        return {
            "headers": ["Authorization"],
            "extraction_pattern": r"Bearer (.+)"
        }
```

## Provider Connectors

kacapi includes built-in connectors for the most popular authentication providers. Each connector is pre-configured with provider-specific settings and optimized for that provider's API.

### Clerk

[Clerk](https://clerk.com) is a complete authentication and user management solution with beautiful pre-built UI components.

#### Basic Usage

**Python:**
```python
from kacapi.modal import api_endpoint
from kacapi.auth import Clerk

@api_endpoint(auth=Clerk())
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with Clerk"}
```

**TypeScript:**
```typescript
import { apiEndpoint } from '@kacapi/vercel';
import { Clerk } from '@kacapi/auth';

export default apiEndpoint({
  auth: new Clerk()
}, async (req, res) => {
  res.json({ message: 'Authenticated with Clerk' });
});
```

#### Configuration Options

```python
from kacapi.auth import Clerk

clerk = Clerk(
    # Frontend API URL (required)
    frontend_api="clerk.example.com",
    
    # Optional: JWT verification settings
    jwt_key="your-jwt-key",  # If not using JWKS
    
    # Optional: Session token name
    session_token_name="__session",
    
    # Optional: Claims to extract and inject
    claims=["org_id", "org_role", "org_slug"],
    
    # Optional: Allow unsigned tokens in development
    development_mode=False,
    
    # Optional: Custom token location
    token_location=TokenLocation(
        cookies=["__session"],
        headers=["Authorization"]
    )
)

@api_endpoint(auth=clerk)
@modal.function()
def endpoint():
    pass
```

#### Advanced: Organization-Based Access

```python
from kacapi.auth import Clerk

# Require specific organization role
@api_endpoint(
    auth=Clerk(
        required_org_role="admin",
        allow_org_roles=["admin", "member"]
    )
)
@modal.function()
def admin_endpoint():
    return {"message": "Admin only"}

# Require specific permission
@api_endpoint(
    auth=Clerk(
        required_permissions=["org:billing:manage"]
    )
)
@modal.function()
def billing_endpoint():
    return {"message": "Billing management"}
```

#### TypeScript Example

```typescript
import { apiEndpoint } from '@kacapi/vercel';
import { Clerk } from '@kacapi/auth';

const clerk = new Clerk({
  frontendApi: 'clerk.example.com',
  jwtKey: process.env.CLERK_JWT_KEY,
  claims: ['org_id', 'org_role'],
  developmentMode: process.env.NODE_ENV === 'development'
});

export default apiEndpoint({
  auth: clerk
}, async (req, res) => {
  // Clerk context automatically injected
  const userId = req.headers['x-kacapi-user-id'];
  const orgId = req.headers['x-kacapi-org-id'];
  
  res.json({ userId, orgId });
});
```

#### Gateway Configuration

When deployed to Kong, kacapi generates:

```yaml
plugins:
  - name: jwt
    config:
      uri_param_names: []
      cookie_names:
        - __session
      header_names:
        - Authorization
      claims_to_verify:
        - exp
      key_claim_name: sub
      secret_is_base64: false
      run_on_preflight: false
  - name: request-transformer
    config:
      add:
        headers:
          - X-Kacapi-User-Id:$(jwt.sub)
          - X-Kacapi-Org-Id:$(jwt.org_id)
```

### Auth0

[Auth0](https://auth0.com) is a flexible authentication and authorization platform supporting numerous identity providers and protocols.

#### Basic Usage

**Python:**
```python
from kacapi.auth import Auth0

@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with Auth0"}
```

**TypeScript:**
```typescript
import { Auth0 } from '@kacapi/auth';

export default apiEndpoint({
  auth: new Auth0({
    domain: 'example.auth0.com',
    audience: 'https://api.example.com'
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with Auth0' });
});
```

#### Configuration Options

```python
from kacapi.auth import Auth0, TokenLocation

auth0 = Auth0(
    # Auth0 domain (required)
    domain="example.auth0.com",
    
    # API audience (required)
    audience="https://api.example.com",
    
    # Optional: Custom issuer
    issuer="https://example.auth0.com/",
    
    # Optional: JWKS cache TTL in seconds
    jwks_cache_ttl=3600,
    
    # Optional: Allowed algorithms
    algorithms=["RS256"],
    
    # Optional: Required scopes
    required_scopes=["read:users", "write:users"],
    
    # Optional: Claims to extract
    claims=["sub", "email", "permissions", "app_metadata"],
    
    # Optional: Validate audience
    validate_audience=True,
    
    # Optional: Custom token location
    token_location=TokenLocation(
        headers=["Authorization", "X-Auth-Token"]
    )
)
```

#### Advanced: Scope-Based Access Control

```python
from kacapi.auth import Auth0

# Require specific scopes
@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com",
        required_scopes=["read:users", "write:users"]
    )
)
@modal.function()
def user_management():
    return {"message": "User management endpoint"}

# Require ANY of multiple scopes
@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com",
        required_scopes=["admin:all", "admin:users"],
        scope_match_mode="any"  # Match ANY scope (default is ALL)
    )
)
@modal.function()
def admin_endpoint():
    return {"message": "Admin endpoint"}
```

#### TypeScript Example with Custom Claims

```typescript
import { Auth0 } from '@kacapi/auth';

const auth0 = new Auth0({
  domain: 'example.auth0.com',
  audience: 'https://api.example.com',
  claims: [
    'sub',
    'email',
    'https://example.com/roles',  // Custom namespace claim
    'https://example.com/permissions'
  ]
});

export default apiEndpoint({
  auth: auth0
}, async (req, res) => {
  const roles = req.headers['x-kacapi-https://example.com/roles'];
  res.json({ roles });
});
```

### AWS Cognito

[AWS Cognito](https://aws.amazon.com/cognito/) is AWS's managed authentication service with user pools and identity pools.

#### Basic Usage

**Python:**
```python
from kacapi.auth import Cognito

@api_endpoint(
    auth=Cognito(
        region="us-east-1",
        user_pool_id="us-east-1_ABC123"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with Cognito"}
```

**TypeScript:**
```typescript
import { Cognito } from '@kacapi/auth';

export default apiEndpoint({
  auth: new Cognito({
    region: 'us-east-1',
    userPoolId: 'us-east-1_ABC123'
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with Cognito' });
});
```

#### Configuration Options

```python
from kacapi.auth import Cognito

cognito = Cognito(
    # AWS region (required)
    region="us-east-1",
    
    # User pool ID (required)
    user_pool_id="us-east-1_ABC123",
    
    # Optional: App client ID (for additional validation)
    app_client_id="7example23exampleid",
    
    # Optional: Token use (id or access)
    token_use="access",  # or "id"
    
    # Optional: Claims to extract
    claims=["sub", "email", "cognito:groups", "custom:org_id"],
    
    # Optional: Required groups
    required_groups=["Admins", "Users"],
    
    # Optional: Custom JWKS URL (if not using standard Cognito)
    jwks_url=None,
    
    # Optional: Cache JWKS
    cache_jwks=True
)
```

#### Advanced: Group-Based Access

```python
from kacapi.auth import Cognito

# Require specific Cognito groups
@api_endpoint(
    auth=Cognito(
        region="us-east-1",
        user_pool_id="us-east-1_ABC123",
        required_groups=["Admins"]
    )
)
@modal.function()
def admin_endpoint():
    return {"message": "Admin only"}

# Require ANY of multiple groups
@api_endpoint(
    auth=Cognito(
        region="us-east-1",
        user_pool_id="us-east-1_ABC123",
        required_groups=["Admins", "Moderators"],
        group_match_mode="any"
    )
)
@modal.function()
def moderator_endpoint():
    return {"message": "Admin or moderator"}
```

### Okta

[Okta](https://www.okta.com) is an enterprise identity and access management platform.

#### Basic Usage

**Python:**
```python
from kacapi.auth import Okta

@api_endpoint(
    auth=Okta(
        domain="example.okta.com",
        audience="api://default"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with Okta"}
```

**TypeScript:**
```typescript
import { Okta } from '@kacapi/auth';

export default apiEndpoint({
  auth: new Okta({
    domain: 'example.okta.com',
    audience: 'api://default'
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with Okta' });
});
```

### Supabase Auth

[Supabase Auth](https://supabase.com/auth) is an open-source authentication system built on PostgreSQL.

#### Basic Usage

**Python:**
```python
from kacapi.auth import SupabaseAuth

@api_endpoint(
    auth=SupabaseAuth(
        project_url="https://abc123.supabase.co",
        jwt_secret="your-jwt-secret"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with Supabase"}
```

**TypeScript:**
```typescript
import { SupabaseAuth } from '@kacapi/auth';

export default apiEndpoint({
  auth: new SupabaseAuth({
    projectUrl: 'https://abc123.supabase.co',
    jwtSecret: process.env.SUPABASE_JWT_SECRET
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with Supabase' });
});
```

### Firebase Auth

[Firebase Auth](https://firebase.google.com/products/auth) is Google's authentication system integrated with Firebase.

#### Basic Usage

**Python:**
```python
from kacapi.auth import FirebaseAuth

@api_endpoint(
    auth=FirebaseAuth(
        project_id="my-firebase-project"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with Firebase"}
```

**TypeScript:**
```typescript
import { FirebaseAuth } from '@kacapi/auth';

export default apiEndpoint({
  auth: new FirebaseAuth({
    projectId: 'my-firebase-project'
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with Firebase' });
});
```

### WorkOS

[WorkOS](https://workos.com) provides enterprise-ready authentication with SSO, directory sync, and more.

#### Basic Usage

**Python:**
```python
from kacapi.auth import WorkOS

@api_endpoint(
    auth=WorkOS(
        api_key="sk_test_123",
        client_id="client_123"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with WorkOS"}
```

**TypeScript:**
```typescript
import { WorkOS } from '@kacapi/auth';

export default apiEndpoint({
  auth: new WorkOS({
    apiKey: process.env.WORKOS_API_KEY,
    clientId: process.env.WORKOS_CLIENT_ID
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with WorkOS' });
});
```

## Generic Authentication

In addition to provider-specific connectors, kacapi supports generic authentication mechanisms for custom requirements.

### JWT Authentication

Generic JWT validation for custom token issuers.

#### Basic Usage

**Python:**
```python
from kacapi.auth import JWT

@api_endpoint(
    auth=JWT(
        secret="your-secret-key",
        algorithm="HS256"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with JWT"}
```

**TypeScript:**
```typescript
import { JWT } from '@kacapi/auth';

export default apiEndpoint({
  auth: new JWT({
    secret: process.env.JWT_SECRET,
    algorithm: 'HS256'
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with JWT' });
});
```

#### Configuration Options

```python
from kacapi.auth import JWT, TokenLocation

jwt = JWT(
    # Secret key for HMAC or public key for RSA (required)
    secret="your-secret-key",
    
    # Algorithm (required)
    algorithm="HS256",  # HS256, HS384, HS512, RS256, RS384, RS512, ES256, etc.
    
    # Optional: Public key for asymmetric algorithms
    public_key="""-----BEGIN PUBLIC KEY-----
    ...
    -----END PUBLIC KEY-----""",
    
    # Optional: JWKS URL for key rotation
    jwks_url="https://example.com/.well-known/jwks.json",
    
    # Optional: Issuer validation
    issuer="https://example.com",
    
    # Optional: Audience validation
    audience="https://api.example.com",
    
    # Optional: Claims to extract
    claims=["sub", "email", "roles", "permissions"],
    
    # Optional: Custom claim validation
    required_claims={"role": "admin"},
    
    # Optional: Leeway for exp/nbf validation (seconds)
    leeway=10,
    
    # Optional: Token location
    token_location=TokenLocation(
        headers=["Authorization", "X-JWT-Token"],
        extraction_pattern=r"Bearer (.+)"
    )
)
```

#### Advanced: RSA with JWKS

```python
from kacapi.auth import JWT

# Use JWKS for automatic key rotation
@api_endpoint(
    auth=JWT(
        algorithm="RS256",
        jwks_url="https://example.com/.well-known/jwks.json",
        issuer="https://example.com",
        audience="https://api.example.com",
        cache_jwks_ttl=3600  # Cache JWKS for 1 hour
    )
)
@modal.function()
def jwks_endpoint():
    return {"message": "JWT with JWKS rotation"}
```

### API Keys

Traditional API key authentication.

#### Basic Usage

**Python:**
```python
from kacapi.auth import APIKey

@api_endpoint(
    auth=APIKey(
        keys=["key_123", "key_456"],
        header="X-API-Key"
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with API key"}
```

**TypeScript:**
```typescript
import { APIKey } from '@kacapi/auth';

export default apiEndpoint({
  auth: new APIKey({
    keys: ['key_123', 'key_456'],
    header: 'X-API-Key'
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with API key' });
});
```

#### Configuration Options

```python
from kacapi.auth import APIKey, TokenLocation

api_key = APIKey(
    # API keys (required) - can be list or function
    keys=["key_123", "key_456"],
    
    # Or use a key validation function
    key_validator=lambda key: validate_key_in_database(key),
    
    # Optional: Token location
    token_location=TokenLocation(
        headers=["X-API-Key", "Authorization"],
        query_params=["api_key"],
        extraction_pattern=r"Bearer (.+)"  # For Authorization header
    ),
    
    # Optional: Key metadata extraction
    extract_metadata=True,  # Extracts user_id, rate_limit, etc. from key
    
    # Optional: Hash keys before comparison
    hash_keys=True,
    hash_algorithm="sha256",
    
    # Optional: Key rotation grace period
    grace_period_seconds=86400  # Allow old keys for 24 hours after rotation
)
```

#### Advanced: Database-Backed Keys

```python
from kacapi.auth import APIKey
import hashlib
import asyncpg

async def validate_api_key(key: str) -> dict:
    """Validate API key against database"""
    key_hash = hashlib.sha256(key.encode()).hexdigest()
    
    async with asyncpg.create_pool(DATABASE_URL) as pool:
        async with pool.acquire() as conn:
            result = await conn.fetchrow(
                "SELECT user_id, rate_limit, metadata FROM api_keys WHERE key_hash = $1 AND active = true",
                key_hash
            )
            
            if result:
                return {
                    "valid": True,
                    "user_id": result["user_id"],
                    "rate_limit": result["rate_limit"],
                    "metadata": result["metadata"]
                }
            return {"valid": False}

@api_endpoint(
    auth=APIKey(
        key_validator=validate_api_key,
        extract_metadata=True
    )
)
@modal.function()
async def db_backed_endpoint():
    return {"message": "Database-backed API key auth"}
```

### mTLS (Mutual TLS)

Client certificate authentication for high-security scenarios.

#### Basic Usage

**Python:**
```python
from kacapi.auth import MTLS

@api_endpoint(
    auth=MTLS(
        ca_cert="/path/to/ca.pem",
        verify_client=True
    )
)
@modal.function()
def protected_endpoint():
    return {"message": "Authenticated with mTLS"}
```

**TypeScript:**
```typescript
import { MTLS } from '@kacapi/auth';

export default apiEndpoint({
  auth: new MTLS({
    caCert: '/path/to/ca.pem',
    verifyClient: true
  })
}, async (req, res) => {
  res.json({ message: 'Authenticated with mTLS' });
});
```

#### Configuration Options

```python
from kacapi.auth import MTLS

mtls = MTLS(
    # CA certificate for client cert verification (required)
    ca_cert="/path/to/ca.pem",
    
    # Or CA certificate content
    ca_cert_content="""-----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----""",
    
    # Verify client certificate (required)
    verify_client=True,
    
    # Optional: Require specific CN (Common Name)
    required_cn="client.example.com",
    
    # Optional: Require specific OU (Organizational Unit)
    required_ou="Engineering",
    
    # Optional: Allowed certificate serial numbers
    allowed_serials=["123456", "789012"],
    
    # Optional: Certificate revocation list
    crl_url="https://example.com/crl.pem",
    
    # Optional: OCSP responder URL
    ocsp_url="https://ocsp.example.com",
    
    # Optional: Extract certificate attributes
    extract_attributes=["CN", "OU", "O", "serialNumber"],
    
    # Optional: Verify certificate chain
    verify_chain=True,
    
    # Optional: Maximum chain depth
    max_chain_depth=3
)
```

## Combining Strategies

kacapi supports combining multiple authentication strategies using logical operators.

### any_of (OR Logic)

Accept requests authenticated by ANY of the specified methods.

#### Basic Usage

**Python:**
```python
from kacapi.auth import any_of, APIKey, JWT

@api_endpoint(
    auth=any_of(
        APIKey(keys=["key_123"]),
        JWT(secret="jwt-secret", algorithm="HS256")
    )
)
@modal.function()
def flexible_auth_endpoint():
    return {"message": "Authenticated with API key OR JWT"}
```

**TypeScript:**
```typescript
import { anyOf, APIKey, JWT } from '@kacapi/auth';

export default apiEndpoint({
  auth: anyOf(
    new APIKey({ keys: ['key_123'] }),
    new JWT({ secret: 'jwt-secret', algorithm: 'HS256' })
  )
}, async (req, res) => {
  res.json({ message: 'Authenticated with API key OR JWT' });
});
```

#### Advanced: Multiple Providers

```python
from kacapi.auth import any_of, Clerk, Auth0, APIKey

# Accept Clerk, Auth0, or API key
@api_endpoint(
    auth=any_of(
        Clerk(frontend_api="clerk.example.com"),
        Auth0(domain="example.auth0.com", audience="https://api.example.com"),
        APIKey(keys=["key_123"])
    )
)
@modal.function()
def multi_provider_endpoint():
    return {"message": "Multiple auth providers supported"}
```

### all_of (AND Logic)

Require ALL specified authentication methods.

#### Basic Usage

**Python:**
```python
from kacapi.auth import all_of, JWT, MTLS

@api_endpoint(
    auth=all_of(
        JWT(secret="jwt-secret", algorithm="HS256"),
        MTLS(ca_cert="/path/to/ca.pem", verify_client=True)
    )
)
@modal.function()
def high_security_endpoint():
    return {"message": "Requires BOTH JWT AND client certificate"}
```

**TypeScript:**
```typescript
import { allOf, JWT, MTLS } from '@kacapi/auth';

export default apiEndpoint({
  auth: allOf(
    new JWT({ secret: 'jwt-secret', algorithm: 'HS256' }),
    new MTLS({ caCert: '/path/to/ca.pem', verifyClient: true })
  )
}, async (req, res) => {
  res.json({ message: 'Requires BOTH JWT AND client certificate' });
});
```

#### Nested Combinations

```python
from kacapi.auth import any_of, all_of, Clerk, Auth0, MTLS

# Complex: (Clerk OR Auth0) AND mTLS
@api_endpoint(
    auth=all_of(
        any_of(
            Clerk(frontend_api="clerk.example.com"),
            Auth0(domain="example.auth0.com", audience="https://api.example.com")
        ),
        MTLS(ca_cert="/path/to/ca.pem", verify_client=True)
    )
)
@modal.function()
def complex_auth_endpoint():
    return {"message": "Complex auth combination"}
```

## Auth Context Injection

After successful authentication, kacapi injects user context into requests as HTTP headers.

### Standard Headers

kacapi injects these standard headers for all auth providers:

```
X-Kacapi-User-Id: user_123
X-Kacapi-Email: user@example.com
X-Kacapi-Roles: admin,user
X-Kacapi-Auth-Provider: clerk
X-Kacapi-Auth-Method: jwt
X-Kacapi-Session-Id: session_abc
```

### Accessing Context in Code

**Python:**
```python
@api_endpoint(auth=Clerk())
@modal.function()
def get_user_profile(request_headers: dict):
    user_id = request_headers.get("X-Kacapi-User-Id")
    email = request_headers.get("X-Kacapi-Email")
    roles = request_headers.get("X-Kacapi-Roles", "").split(",")
    
    return {
        "user_id": user_id,
        "email": email,
        "roles": roles
    }
```

**TypeScript:**
```typescript
export default apiEndpoint({
  auth: new Clerk()
}, async (req, res) => {
  const userId = req.headers['x-kacapi-user-id'];
  const email = req.headers['x-kacapi-email'];
  const roles = (req.headers['x-kacapi-roles'] || '').split(',');
  
  res.json({ userId, email, roles });
});
```

### Provider-Specific Headers

Each provider may inject additional headers:

**Clerk:**
```
X-Kacapi-Org-Id: org_123
X-Kacapi-Org-Role: admin
X-Kacapi-Org-Slug: acme-inc
```

**Auth0:**
```
X-Kacapi-Permissions: read:users,write:users
X-Kacapi-Scopes: openid profile email
```

**Cognito:**
```
X-Kacapi-Groups: Admins,Users
X-Kacapi-Token-Use: access
X-Kacapi-Username: john.doe
```

**API Key:**
```
X-Kacapi-Key-Id: key_123
X-Kacapi-Rate-Limit: 1000
X-Kacapi-Environment: production
```

**mTLS:**
```
X-Kacapi-Cert-CN: client.example.com
X-Kacapi-Cert-OU: Engineering
X-Kacapi-Cert-Serial: 123456
```

### Custom Claim Mapping

Configure which claims to extract and inject:

```python
from kacapi.auth import Auth0

@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com",
        claims=[
            "sub",
            "email",
            "https://example.com/roles",  # Custom claim
            "https://example.com/tenant_id"
        ],
        claim_header_map={
            "https://example.com/roles": "X-Kacapi-App-Roles",
            "https://example.com/tenant_id": "X-Kacapi-Tenant-Id"
        }
    )
)
@modal.function()
def custom_claims_endpoint(request_headers: dict):
    app_roles = request_headers.get("X-Kacapi-App-Roles")
    tenant_id = request_headers.get("X-Kacapi-Tenant-Id")
    
    return {"app_roles": app_roles, "tenant_id": tenant_id}
```

## Security Considerations

### Token Storage

**Never store secrets in code:**
```python
# ❌ BAD
@api_endpoint(auth=JWT(secret="hardcoded-secret"))

# ✅ GOOD
@api_endpoint(auth=JWT(secret=os.environ["JWT_SECRET"]))
```

### Token Transmission

Always use HTTPS for token transmission. Configure CORS appropriately:

```python
from kacapi.modal import api_endpoint
from kacapi.auth import Clerk
from kacapi.cors import CORS

@api_endpoint(
    auth=Clerk(),
    cors=CORS(
        origins=["https://app.example.com"],
        credentials=True,  # Allow cookies
        expose_headers=["X-Kacapi-User-Id"]
    )
)
@modal.function()
def secure_endpoint():
    pass
```

### Rate Limiting by Auth

Combine auth with rate limiting to prevent abuse:

```python
from kacapi.modal import api_endpoint
from kacapi.auth import APIKey
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    auth=APIKey(keys=["key_123"]),
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_key="X-Kacapi-Key-Id"  # Rate limit per API key
    )
)
@modal.function()
def rate_limited_endpoint():
    pass
```

## Gateway Integration

kacapi translates auth decorators into gateway-specific configurations.

### Kong

Kong uses the JWT plugin for most auth providers:

```yaml
# Generated Kong configuration for @api_endpoint(auth=Auth0(...))
services:
  - name: my-service
    routes:
      - name: my-route
        paths: [/api/endpoint]
        plugins:
          - name: jwt
            config:
              uri_param_names: []
              cookie_names: []
              header_names: [Authorization]
              claims_to_verify: [exp, nbf]
              key_claim_name: sub
              secret_is_base64: false
              run_on_preflight: false
          - name: request-transformer
            config:
              add:
                headers:
                  - X-Kacapi-User-Id:$(jwt.sub)
                  - X-Kacapi-Email:$(jwt.email)
```

### AWS API Gateway

AWS API Gateway uses Lambda authorizers:

```typescript
// Generated Lambda authorizer for @api_endpoint(auth=Clerk(...))
export const handler = async (event: APIGatewayAuthorizerEvent) => {
  const token = extractToken(event.headers.Authorization);
  
  const clerkClient = new ClerkClient({
    frontendApi: process.env.CLERK_FRONTEND_API
  });
  
  try {
    const verifiedToken = await clerkClient.verifyToken(token);
    
    return {
      principalId: verifiedToken.sub,
      policyDocument: {
        Version: '2012-10-17',
        Statement: [{
          Action: 'execute-api:Invoke',
          Effect: 'Allow',
          Resource: event.methodArn
        }]
      },
      context: {
        userId: verifiedToken.sub,
        email: verifiedToken.email,
        orgId: verifiedToken.org_id
      }
    };
  } catch (error) {
    throw new Error('Unauthorized');
  }
};
```

## Best Practices

### 1. Use Environment Variables

Never hardcode secrets:

```python
# ✅ GOOD
@api_endpoint(
    auth=Clerk(
        frontend_api=os.environ["CLERK_FRONTEND_API"],
        jwt_key=os.environ.get("CLERK_JWT_KEY")
    )
)
```

### 2. Principle of Least Privilege

Only extract necessary claims:

```python
# ✅ GOOD - minimal claims
@api_endpoint(
    auth=Auth0(
        domain=os.environ["AUTH0_DOMAIN"],
        audience=os.environ["AUTH0_AUDIENCE"],
        claims=["sub", "email"]  # Only what you need
    )
)
```

### 3. Combine Auth with Rate Limiting

Protect authenticated endpoints from abuse:

```python
from kacapi.auth import Clerk
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    auth=Clerk(),
    rate_limit=FixedWindow(
        requests=100,
        window="1m",
        by_key="X-Kacapi-User-Id"  # Per-user limits
    )
)
@modal.function()
def protected_endpoint():
    pass
```

### 4. Validate in Backend

Don't blindly trust gateway headers:

```python
@api_endpoint(auth=Clerk())
@modal.function()
def validated_endpoint(request_headers: dict):
    user_id = request_headers.get("X-Kacapi-User-Id")
    
    # Additional validation
    if not user_id or not user_exists(user_id):
        raise ValueError("Invalid user context")
    
    return {"data": "secure"}
```

### 5. Monitor Auth Failures

Track failed authentication attempts:

```python
from kacapi.observability import Metrics

@api_endpoint(
    auth=Clerk(),
    metrics=Metrics(
        track_auth_failures=True,
        alert_on_spike=True,
        failure_threshold=100  # Alert if >100 failures/min
    )
)
@modal.function()
def monitored_endpoint():
    pass
```

## Troubleshooting

### Common Issues

#### 1. "Invalid token" errors

**Problem**: Token validation fails at gateway

**Solutions**:
- Verify token format matches provider
- Check clock skew (use `leeway` parameter)
- Validate JWKS URL is accessible
- Ensure algorithm matches token

```python
# Add clock skew tolerance
@api_endpoint(
    auth=JWT(
        secret=os.environ["JWT_SECRET"],
        algorithm="HS256",
        leeway=30  # 30 second tolerance
    )
)
```

#### 2. Missing user context in backend

**Problem**: `X-Kacapi-*` headers not present

**Solutions**:
- Verify auth is configured on endpoint
- Check gateway deployment succeeded
- Ensure gateway strips incoming `X-Kacapi-*` headers
- Validate token is being sent

```python
# Debug: log all headers
@api_endpoint(auth=Clerk())
@modal.function()
def debug_endpoint(request_headers: dict):
    print("All headers:", request_headers)
    return {"headers": request_headers}
```

#### 3. CORS errors with authenticated requests

**Problem**: Browser blocks authenticated requests

**Solutions**:
- Enable credentials in CORS
- Set specific origins (not `*`)
- Include auth headers in exposed headers

```python
from kacapi.cors import CORS

@api_endpoint(
    auth=Clerk(),
    cors=CORS(
        origins=["https://app.example.com"],
        credentials=True,
        allow_headers=["Authorization", "Content-Type"],
        expose_headers=["X-Kacapi-User-Id"]
    )
)
```

### Debug Mode

Enable debug logging:

```python
from kacapi.auth import Clerk
from kacapi.observability import Logging

@api_endpoint(
    auth=Clerk(development_mode=True),  # Relaxed validation
    logging=Logging(
        level="DEBUG",
        log_auth_events=True,
        log_headers=True  # Log all headers
    )
)
@modal.function()
def debug_endpoint():
    pass
```

### Testing Authentication Locally

kacapi provides local middleware that mimics gateway auth:

```python
from kacapi.testing import MockGateway
from kacapi.auth import Clerk

# In your test file
mock_gateway = MockGateway(
    auth=Clerk(development_mode=True)
)

# Simulates gateway auth locally
response = mock_gateway.request(
    "/api/endpoint",
    headers={"Authorization": "Bearer test-token"}
)

assert response.status == 200
assert response.headers["X-Kacapi-User-Id"] == "user_123"
```

---



## Advanced Authentication Patterns

### Multi-Tenancy with Authentication

kacapi provides powerful patterns for building multi-tenant applications with tenant-scoped authentication.

#### Tenant Isolation at Gateway

```python
from kacapi.auth import Clerk
from kacapi.routing import TenantRouter

# Route requests to tenant-specific backends based on auth context
@api_endpoint(
    auth=Clerk(claims=["org_id", "org_slug"]),
    routing=TenantRouter(
        tenant_key="X-Kacapi-Org-Id",
        backend_map={
            "org_123": "https://tenant1.api.example.com",
            "org_456": "https://tenant2.api.example.com"
        },
        default_backend="https://default.api.example.com"
    )
)
@modal.function()
def multi_tenant_endpoint():
    return {"message": "Tenant-specific response"}
```

#### Database Tenant Isolation

```python
from kacapi.auth import Auth0

@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com",
        claims=["sub", "email", "https://example.com/tenant_id"]
    )
)
@modal.function()
def tenant_data_endpoint(request_headers: dict):
    tenant_id = request_headers.get("X-Kacapi-Https://example.com/tenant_id")
    
    # Use tenant ID for database query isolation
    conn = get_database_connection()
    data = conn.query(
        "SELECT * FROM resources WHERE tenant_id = %s",
        [tenant_id]
    )
    
    return {"data": data, "tenant_id": tenant_id}
```

#### Tenant-Specific Rate Limits

```python
from kacapi.auth import APIKey
from kacapi.rate_limit import TenantRateLimit

@api_endpoint(
    auth=APIKey(extract_metadata=True),
    rate_limit=TenantRateLimit(
        by_tenant_key="X-Kacapi-Tenant-Id",
        limits={
            "tier_free": FixedWindow(requests=100, window="1h"),
            "tier_pro": FixedWindow(requests=1000, window="1h"),
            "tier_enterprise": FixedWindow(requests=10000, window="1h")
        },
        default_tier="tier_free"
    )
)
@modal.function()
def tiered_endpoint():
    return {"message": "Tier-specific rate limiting"}
```

### Progressive Authentication

Implement progressive authentication where endpoints work without auth but provide enhanced features when authenticated.

#### Optional Authentication

```python
from kacapi.auth import Clerk, optional

# Endpoint accessible without auth, but extracts user context if present
@api_endpoint(
    auth=optional(Clerk(frontend_api="clerk.example.com"))
)
@modal.function()
def public_with_perks_endpoint(request_headers: dict):
    user_id = request_headers.get("X-Kacapi-User-Id")
    
    if user_id:
        # Authenticated user gets personalized response
        return {
            "message": "Welcome back!",
            "personalized": True,
            "user_id": user_id
        }
    else:
        # Anonymous user gets generic response
        return {
            "message": "Welcome!",
            "personalized": False
        }
```

#### Conditional Features

```python
from kacapi.auth import Auth0, optional
from kacapi.rate_limit import FixedWindow

@api_endpoint(
    auth=optional(Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com"
    )),
    rate_limit=FixedWindow(
        requests=lambda headers: 1000 if headers.get("X-Kacapi-User-Id") else 10,
        window="1h"
    )
)
@modal.function()
def conditional_limit_endpoint():
    # Authenticated users get higher rate limits
    return {"message": "Conditional rate limiting"}
```

### Service-to-Service Authentication

Secure backend-to-backend communication with service accounts and machine-to-machine tokens.

#### Service Account Pattern

```python
from kacapi.auth import JWT

# Service account JWT with specific claims
service_jwt = JWT(
    algorithm="RS256",
    public_key=os.environ["SERVICE_PUBLIC_KEY"],
    required_claims={
        "iss": "https://auth.example.com",
        "aud": "https://api.example.com",
        "scope": "service:internal"
    },
    claim_validators={
        "service_name": lambda name: name in ["analytics", "billing", "notifications"]
    }
)

@api_endpoint(auth=service_jwt)
@modal.function()
def internal_service_endpoint(request_headers: dict):
    service_name = request_headers.get("X-Kacapi-Service-Name")
    return {"message": f"Internal service call from {service_name}"}
```

#### Machine-to-Machine with OAuth2 Client Credentials

```python
from kacapi.auth import OAuth2ClientCredentials

@api_endpoint(
    auth=OAuth2ClientCredentials(
        token_url="https://auth.example.com/oauth/token",
        client_id=os.environ["M2M_CLIENT_ID"],
        client_secret=os.environ["M2M_CLIENT_SECRET"],
        audience="https://api.example.com",
        required_scopes=["read:data", "write:data"]
    )
)
@modal.function()
def m2m_endpoint():
    return {"message": "Machine-to-machine authenticated"}
```

### Authentication Middleware Chain

Build complex authentication flows with middleware chains.

#### Pre-Authentication Hook

```python
from kacapi.auth import Clerk
from kacapi.middleware import PreAuthHook

async def check_maintenance_mode(request):
    """Block all requests during maintenance"""
    if os.environ.get("MAINTENANCE_MODE") == "true":
        raise MaintenanceError("System under maintenance")
    return request

async def check_ip_whitelist(request):
    """Only allow requests from whitelisted IPs"""
    client_ip = request.headers.get("X-Forwarded-For", "").split(",")[0]
    if client_ip not in ALLOWED_IPS:
        raise ForbiddenError(f"IP {client_ip} not allowed")
    return request

@api_endpoint(
    auth=Clerk(),
    pre_auth_hooks=[check_maintenance_mode, check_ip_whitelist]
)
@modal.function()
def protected_endpoint():
    return {"message": "Passed pre-auth checks"}
```

#### Post-Authentication Hook

```python
from kacapi.auth import Auth0
from kacapi.middleware import PostAuthHook

async def enrich_user_context(request, user_context):
    """Enrich user context with database info"""
    user_id = user_context.get("user_id")
    db_user = await fetch_user_from_db(user_id)
    
    user_context["subscription_tier"] = db_user.subscription_tier
    user_context["features_enabled"] = db_user.features
    
    return request, user_context

async def audit_log_access(request, user_context):
    """Log all authenticated accesses"""
    await log_audit_event(
        user_id=user_context["user_id"],
        endpoint=request.path,
        timestamp=datetime.utcnow()
    )
    return request, user_context

@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com"
    ),
    post_auth_hooks=[enrich_user_context, audit_log_access]
)
@modal.function()
def audited_endpoint():
    return {"message": "Access logged"}
```

### Token Refresh and Rotation

Handle token expiration and rotation gracefully.

#### Automatic Token Refresh

```python
from kacapi.auth import RefreshableJWT

@api_endpoint(
    auth=RefreshableJWT(
        access_token_header="Authorization",
        refresh_token_header="X-Refresh-Token",
        refresh_endpoint="https://auth.example.com/refresh",
        auto_refresh=True,
        refresh_buffer_seconds=300  # Refresh 5 minutes before expiration
    )
)
@modal.function()
def auto_refresh_endpoint():
    return {"message": "Token automatically refreshed if needed"}
```

#### Graceful Degradation on Token Expiry

```python
from kacapi.auth import JWT
from kacapi.cache import Cache

@api_endpoint(
    auth=JWT(
        secret=os.environ["JWT_SECRET"],
        algorithm="HS256",
        on_expired_token="allow_with_cache"
    ),
    cache=Cache(
        ttl=300,
        vary_by=["X-Kacapi-User-Id"],
        serve_stale_on_auth_failure=True
    )
)
@modal.function()
def graceful_degradation_endpoint():
    # If token is expired but cache is fresh, serve cached response
    return {"message": "Cached response on token expiry"}
```

### Geographic Authentication Policies

Implement location-based authentication rules.

#### Geo-Fencing

```python
from kacapi.auth import Clerk
from kacapi.geo import GeoFence

@api_endpoint(
    auth=Clerk(),
    geo_fence=GeoFence(
        allowed_countries=["US", "CA", "MX"],
        blocked_countries=["KP", "IR", "SY"],
        on_violation="reject"
    )
)
@modal.function()
def geo_restricted_endpoint():
    return {"message": "Available in North America only"}
```

#### Location-Based MFA

```python
from kacapi.auth import Auth0
from kacapi.geo import RequireMFA

@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com"
    ),
    mfa=RequireMFA(
        when=lambda request: request.geo_country not in ["US", "CA"],
        mfa_provider="auth0",
        challenge_type="sms"
    )
)
@modal.function()
def location_aware_mfa_endpoint():
    return {"message": "MFA required for non-US/CA requests"}
```

### Authentication Analytics and Monitoring

Track authentication patterns and detect anomalies.

#### Auth Metrics

```python
from kacapi.auth import Clerk
from kacapi.observability import AuthMetrics

@api_endpoint(
    auth=Clerk(),
    metrics=AuthMetrics(
        track_login_frequency=True,
        track_device_fingerprints=True,
        alert_on_anomaly=True,
        anomaly_threshold=5  # Alert if >5 failed attempts in 5 minutes
    )
)
@modal.function()
def monitored_endpoint():
    return {"message": "Auth metrics tracked"}
```

#### Session Analytics

```python
from kacapi.auth import Auth0
from kacapi.observability import SessionAnalytics

@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com"
    ),
    session_analytics=SessionAnalytics(
        track_session_duration=True,
        track_concurrent_sessions=True,
        max_concurrent_sessions=3,
        on_limit_exceeded="terminate_oldest"
    )
)
@modal.function()
def session_limited_endpoint():
    return {"message": "Session limits enforced"}
```

### Custom Authentication Providers

Extend kacapi with your own authentication providers.

#### Custom Auth Connector Implementation

```python
from kacapi.auth import AuthConnector, AuthValidationResult, UserContext
import aiohttp

class CustomAuthProvider(AuthConnector):
    """Custom authentication provider"""
    
    def __init__(self, api_url: str, api_key: str):
        self.api_url = api_url
        self.api_key = api_key
    
    @property
    def provider(self) -> str:
        return "custom"
    
    async def validate_token(
        self,
        token: str,
        context: Optional[Dict[str, Any]] = None
    ) -> AuthValidationResult:
        """Validate token against custom API"""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.api_url}/validate",
                headers={"X-API-Key": self.api_key},
                json={"token": token}
            ) as response:
                if response.status == 200:
                    data = await response.json()
                    return AuthValidationResult(
                        valid=True,
                        user_id=data["user_id"],
                        expires_at=data["exp"]
                    )
                else:
                    return AuthValidationResult(
                        valid=False,
                        error="Invalid token"
                    )
    
    async def extract_context(self, token: str) -> UserContext:
        """Extract user context from token"""
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.api_url}/user",
                headers={"Authorization": f"Bearer {token}"}
            ) as response:
                data = await response.json()
                return UserContext(
                    user_id=data["id"],
                    email=data["email"],
                    roles=data["roles"],
                    permissions=data["permissions"],
                    metadata=data.get("metadata", {})
                )
    
    async def get_gateway_config(
        self,
        gateway_type: str
    ) -> Dict[str, Any]:
        """Generate gateway-specific config"""
        if gateway_type == "kong":
            return {
                "type": "custom",
                "config": {
                    "validation_endpoint": f"{self.api_url}/validate",
                    "api_key": self.api_key
                }
            }
        # Add other gateway types...
        return {}
    
    async def validate_config(self) -> Dict[str, Any]:
        """Validate connector configuration"""
        # Test connection to custom auth API
        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(
                    f"{self.api_url}/health",
                    headers={"X-API-Key": self.api_key}
                ) as response:
                    if response.status == 200:
                        return {"valid": True}
        except Exception as e:
            return {"valid": False, "error": str(e)}

# Use custom provider
@api_endpoint(
    auth=CustomAuthProvider(
        api_url="https://auth.mycompany.com",
        api_key=os.environ["CUSTOM_AUTH_KEY"]
    )
)
@modal.function()
def custom_auth_endpoint():
    return {"message": "Custom authentication"}
```

## Authentication Performance Optimization

### Token Caching

Reduce authentication overhead by caching validated tokens.

#### In-Memory Token Cache

```python
from kacapi.auth import JWT
from kacapi.cache import TokenCache

@api_endpoint(
    auth=JWT(
        secret=os.environ["JWT_SECRET"],
        algorithm="HS256",
        cache=TokenCache(
            backend="memory",
            ttl=300,  # Cache for 5 minutes
            max_size=10000  # Cache up to 10k tokens
        )
    )
)
@modal.function()
def cached_auth_endpoint():
    return {"message": "Fast auth with caching"}
```

#### Redis Token Cache

```python
from kacapi.auth import Auth0
from kacapi.cache import TokenCache

@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com",
        cache=TokenCache(
            backend="redis",
            redis_url=os.environ["REDIS_URL"],
            ttl=600,
            key_prefix="auth:"
        )
    )
)
@modal.function()
def redis_cached_endpoint():
    return {"message": "Distributed token cache"}
```

### JWKS Caching

Cache JSON Web Key Sets to reduce external API calls.

```python
from kacapi.auth import JWT

@api_endpoint(
    auth=JWT(
        algorithm="RS256",
        jwks_url="https://example.com/.well-known/jwks.json",
        jwks_cache_ttl=3600,  # Cache JWKS for 1 hour
        jwks_cache_backend="redis",
        jwks_cache_redis_url=os.environ["REDIS_URL"]
    )
)
@modal.function()
def jwks_cached_endpoint():
    return {"message": "JWKS cached for performance"}
```

### Batch Token Validation

Validate multiple tokens in a single request for performance.

```python
from kacapi.auth import BatchAuthValidator

batch_validator = BatchAuthValidator(
    auth=Clerk(frontend_api="clerk.example.com"),
    batch_size=100,
    batch_timeout_ms=50
)

@api_endpoint(auth=batch_validator)
@modal.function()
def batch_validated_endpoint():
    return {"message": "Efficient batch validation"}
```

## Authentication Testing

### Unit Testing Auth Connectors

```python
import pytest
from kacapi.auth import Clerk
from kacapi.testing import MockAuthProvider

@pytest.fixture
def mock_clerk():
    return MockAuthProvider(
        provider="clerk",
        valid_tokens={
            "valid_token_123": {
                "user_id": "user_123",
                "email": "test@example.com",
                "org_id": "org_456"
            }
        }
    )

async def test_clerk_validation(mock_clerk):
    result = await mock_clerk.validate_token("valid_token_123")
    assert result.valid is True
    assert result.user_id == "user_123"

async def test_clerk_invalid_token(mock_clerk):
    result = await mock_clerk.validate_token("invalid_token")
    assert result.valid is False
```

### Integration Testing with Real Providers

```python
import pytest
from kacapi.auth import Auth0
from kacapi.testing import AuthTestClient

@pytest.fixture
def auth0_test_client():
    return AuthTestClient(
        auth=Auth0(
            domain=os.environ["AUTH0_TEST_DOMAIN"],
            audience=os.environ["AUTH0_TEST_AUDIENCE"]
        )
    )

async def test_auth0_integration(auth0_test_client):
    # Get test token from Auth0
    token = await auth0_test_client.get_test_token(
        username="test@example.com",
        password=os.environ["AUTH0_TEST_PASSWORD"]
    )
    
    # Validate token
    result = await auth0_test_client.validate_token(token)
    assert result.valid is True
```

### End-to-End Gateway Testing

```python
import pytest
from kacapi.testing import GatewayTestClient
from kacapi.auth import Clerk

@pytest.fixture
def gateway_client():
    return GatewayTestClient(
        gateway_type="kong",
        gateway_url="http://localhost:8000",
        auth=Clerk(frontend_api="clerk.test.com")
    )

async def test_end_to_end_auth(gateway_client):
    # Request with valid token
    response = await gateway_client.get(
        "/api/endpoint",
        headers={"Authorization": "Bearer valid_token"}
    )
    assert response.status == 200
    assert "X-Kacapi-User-Id" in response.headers
    
    # Request without token
    response = await gateway_client.get("/api/endpoint")
    assert response.status == 401
```

## Migration Guides

### Migrating from Manual Gateway Config

If you're currently managing auth in gateway configs, here's how to migrate to kacapi:

#### Before (Kong YAML):

```yaml
services:
  - name: my-api
    routes:
      - name: protected-route
        paths: [/api/protected]
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
          key_claim_name: sub
```

#### After (kacapi):

```python
from kacapi.modal import api_endpoint
from kacapi.auth import JWT

@api_endpoint(
    auth=JWT(
        secret=os.environ["JWT_SECRET"],
        algorithm="HS256"
    )
)
@modal.function()
def protected_route():
    return {"message": "Protected"}
```

### Migrating Auth Providers

#### From Auth0 to Clerk:

```python
# Before: Auth0
@api_endpoint(
    auth=Auth0(
        domain="example.auth0.com",
        audience="https://api.example.com",
        claims=["sub", "email", "app_metadata"]
    )
)

# After: Clerk
@api_endpoint(
    auth=Clerk(
        frontend_api="clerk.example.com",
        claims=["sub", "email", "public_metadata"]
    )
)
```

Migration checklist:
1. Update auth decorator
2. Map claim names (Auth0 `app_metadata` → Clerk `public_metadata`)
3. Update frontend auth library
4. Test with both providers using `any_of` during transition
5. Remove old provider after verification

## Conclusion

kacapi authentication provides a powerful, flexible, and secure way to manage API authentication. By declaring auth requirements in code alongside your endpoints, you eliminate configuration drift, improve documentation, and enable automated gateway deployment.

Key takeaways:

1. **Code-first**: Auth lives in decorators, not separate configs
2. **Provider-agnostic**: Same interface works with Clerk, Auth0, Cognito, etc.
3. **Gateway-agnostic**: Deploy to Kong, AWS API Gateway, CloudFront, APISIX
4. **Composable**: Combine strategies with `any_of` and `all_of`
5. **Secure**: Built-in best practices and security validations
6. **Observable**: Automatic logging, metrics, and audit trails
7. **Performant**: Built-in caching and optimization strategies
8. **Testable**: Comprehensive testing utilities for unit and integration tests
9. **Extensible**: Create custom auth connectors for any provider

For more information, see:
- [Auth Connector Specification](../specs/auth-connectors.md)
- [Gateway Adapters](../adapters/)
- [Security Best Practices](../concepts/security.md)
- [Observatory](../observatory/)
