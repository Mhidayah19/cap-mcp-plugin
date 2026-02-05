# OAuth Token Exchange Uses Provider URL Instead of Subscriber-Specific XSUAA URL in Multi-Tenant Deployments

## Current Behaviour

In multi-tenant BTP applications, the MCP server's OAuth token exchange flow incorrectly uses the **provider's XSUAA URL** instead of the **subscriber tenant's XSUAA URL**. This causes authentication tokens to be issued at the provider level rather than the subscriber level, breaking tenant isolation and preventing proper access to subscriber-specific resources.

**Root Cause:**
The OAuth endpoints construct URLs using `req.get("host")`, which returns the internal Cloud Foundry application URL (e.g., `myapp.cfapps.eu10.hana.ondemand.com`) instead of the subscriber's approuter URL (e.g., `subscriber-tenant.myapp.cfapps.eu10.hana.ondemand.com`).

**Example of incorrect behavior:**

```javascript
// Current implementation - WRONG
const baseUrl = `${protocol}://${req.get("host")}`;
const issuer = credentials.url;  // Provider's XSUAA URL

// OAuth metadata response
{
  "issuer": "https://provider.authentication.eu10.hana.ondemand.com",  // ❌ Provider level
  "authorization_endpoint": "https://myapp.cfapps.eu10.hana.ondemand.com/oauth/authorize",  // ❌ Internal CF URL
  "token_endpoint": "https://myapp.cfapps.eu10.hana.ondemand.com/oauth/token"  // ❌ Internal CF URL
}
```

**Problems:**
- Token is fetched from provider's XSUAA instead of subscriber's XSUAA
- Subscriber acts as a tenant for the provider, but authentication doesn't respect tenant boundaries
- Despite having subscriber-specific authentication configured, tokens are still issued at provider level
- OAuth callbacks redirect to internal CF URLs instead of subscriber approuter URLs
- Breaks multi-tenant isolation and security model
- Destination lookups fail because tenant context is incorrect
- Token propagation to downstream services uses wrong tenant credentials

## Expected Behaviour

OAuth flows should use the **subscriber tenant's XSUAA URL** derived from the request's effective host (typically set by the approuter via `x-forwarded-host` header):

```javascript
// Correct implementation
const effectiveHost = getEffectiveHost(req);  // e.g., "subscriber-tenant.myapp.cfapps.eu10.hana.ondemand.com"
const subdomain = effectiveHost.split('.')[0];  // "subscriber-tenant"
const subscriberXsuaaUrl = `https://${subdomain}.${uaadomain}`;

// OAuth metadata response
{
  "issuer": "https://subscriber-tenant.authentication.eu10.hana.ondemand.com",  // ✅ Subscriber level
  "authorization_endpoint": "https://subscriber-tenant.myapp.cfapps.eu10.hana.ondemand.com/oauth/authorize",  // ✅ Subscriber URL
  "token_endpoint": "https://subscriber-tenant.myapp.cfapps.eu10.hana.ondemand.com/oauth/token"  // ✅ Subscriber URL
}
```

**Expected token exchange flow:**
1. Client requests authorization → Redirects to **subscriber's** XSUAA authorization endpoint
2. User authenticates → XSUAA issues authorization code for **subscriber tenant**
3. Client exchanges code for token → Token request goes to **subscriber's** XSUAA token endpoint
4. Token is issued with **subscriber tenant context** (correct `zid`, `tenant`, scopes)
5. Downstream services receive tokens with proper tenant isolation

## Steps to Reproduce

1. Deploy the MCP server in a multi-tenant BTP environment with XSUAA authentication
2. Configure a subscriber tenant with its own subdomain (e.g., `subscriber-a.myapp.cfapps.eu10.hana.ondemand.com`)
3. Access the OAuth metadata endpoint from the subscriber tenant:
   ```bash
   curl https://subscriber-a.myapp.cfapps.eu10.hana.ondemand.com/.well-known/oauth-authorization-server
   ```
4. Observe that the `issuer` field points to the provider's XSUAA URL instead of the subscriber's
5. Initiate an OAuth authorization flow from the subscriber tenant
6. Complete the authentication and observe the token exchange
7. Decode the JWT access token and inspect the `zid` (zone ID) claim:
   ```bash
   echo "<access_token>" | cut -d. -f2 | base64 -d | jq .
   ```
8. Verify that the `zid` is the provider's tenant ID instead of the subscriber's tenant ID

**Expected vs Actual:**
- **Expected `zid`**: `subscriber-a-tenant-guid`
- **Actual `zid`**: `provider-tenant-guid` ❌

## Environment

- **Environment**: SAP BTP Cloud Foundry (Multi-Tenant)
- **Database Type**: N/A (Authentication layer issue)
- **Release Version**: Affects versions before 1.4.1
- **Authentication Type**: JWT / XSUAA
- **Configuration**: Multi-tenant application with managed approuter
- **Deployment Model**: Provider application with multiple subscriber tenants
- **Customizations**: N/A

## Proposed Solution

### 1. Add `getEffectiveHost()` Function

Create a utility function that resolves the correct subscriber host from request headers:

```typescript
function getEffectiveHost(req) {
  const xForwardedHost = req.headers["x-forwarded-host"];
  const xCustomHost = req.headers["x-custom-host"];
  const xApprouterHost = req.headers["x-approuter-host"];
  const rawHost = req.get("host") || "";
  
  // Check if rawHost is internal CF URL
  const isCfAppUrl = rawHost.includes('.cfapps.') || rawHost.includes('.hana.ondemand.com');
  
  // Priority: x-forwarded-host (set by approuter) > custom headers > raw host
  if (xForwardedHost) {
    logger.debug("[MCP-HOST] Using x-forwarded-host", { host: xForwardedHost });
    return xForwardedHost;
  }
  
  if (xCustomHost) {
    logger.debug("[MCP-HOST] Using x-custom-host", { host: xCustomHost });
    return xCustomHost;
  }
  
  if (xApprouterHost) {
    logger.debug("[MCP-HOST] Using x-approuter-host", { host: xApprouterHost });
    return xApprouterHost;
  }
  
  logger.debug("[MCP-HOST] Using raw host", { host: rawHost, isCfAppUrl });
  return rawHost;
}
```

### 2. Add `getSubscriberXsuaaUrl()` Function

Build the subscriber-specific XSUAA URL from the effective host:

```typescript
function getSubscriberXsuaaUrl(req, credentials, urlPath) {
  let subdomain = getEffectiveHost(req).split(".")[0];
  const originalSubdomain = subdomain;

  // Handle local development
  const isLocalDev = subdomain.includes(':') || subdomain === 'localhost' || subdomain.startsWith('127');
  
  if (isLocalDev) {
    subdomain = credentials.identityzone || credentials.zid || subdomain;
    logger.debug("[MCP-XSUAA] Local development - using identity zone", { 
      originalSubdomain,
      identityZone: subdomain
    });
  }

  const domain = credentials.uaadomain || "authentication.eu10.hana.ondemand.com";
  const xsuaaUrl = `https://${subdomain}.${domain}${urlPath}`;
  
  logger.info("[MCP-XSUAA] Built subscriber XSUAA URL", {
    xsuaaUrl,
    subdomain,
    domain,
    urlPath,
    isLocalDev
  });
  
  return xsuaaUrl;
}
```

### 3. Update OAuth Endpoints

Replace all instances of provider URLs with subscriber-specific URLs:

```typescript
// OAuth Authorization Server Metadata
app.get("/.well-known/oauth-authorization-server", (req, res) => {
  const base = `${getProtocol(req)}://${getEffectiveHost(req)}`;
  const issuer = getSubscriberXsuaaUrl(req, credentials, "");  // ✅ Subscriber XSUAA
  
  res.json({
    issuer,  // ✅ Now points to subscriber's XSUAA
    authorization_endpoint: `${base}/oauth/authorize`,  // ✅ Subscriber approuter URL
    token_endpoint: `${base}/oauth/token`,  // ✅ Subscriber approuter URL
    // ... rest of metadata
  });
});

// OAuth Authorization Endpoint
app.get("/oauth/authorize", (req, res) => {
  const xsuaaUrl = `${getSubscriberXsuaaUrl(req, credentials, "/oauth/authorize")}?${params}`;
  res.redirect(xsuaaUrl);  // ✅ Redirects to subscriber's XSUAA
});

// OAuth Token Exchange
app.post("/oauth/token", async (req, res) => {
  const tokenUrl = getSubscriberXsuaaUrl(req, credentials, "/oauth/token");
  // Exchange code at subscriber's XSUAA token endpoint
  const response = await fetch(tokenUrl, { /* ... */ });
  // ✅ Token issued by subscriber's XSUAA with correct tenant context
});
```

## Affected Files

- `src/auth/utils.ts` - Main changes for host resolution and subscriber URL building
- `src/auth/factory.ts` - OAuth endpoint handlers
- `src/auth/xsuaa-service.ts` - Token exchange logic
- `lib/auth/utils.js` - Compiled output
- `lib/auth/factory.js` - Compiled output

## Affected Scenarios

All OAuth flows in multi-tenant deployments:
- ❌ OAuth authorization server metadata discovery
- ❌ Authorization code flow initiation
- ❌ Token exchange (authorization code → access token)
- ❌ Refresh token flow
- ❌ Client registration endpoints
- ❌ Downstream service calls requiring tenant-scoped tokens
- ❌ Destination service lookups (requires correct tenant context)

## Technical Context

### Multi-Tenant Architecture in BTP

In SAP BTP multi-tenant applications:
- **Provider** = The application owner/developer
- **Subscriber** = A tenant that subscribes to the provider's application
- Each subscriber gets their own subdomain: `{subscriber-subdomain}.{app-domain}`
- Each subscriber has their own XSUAA instance: `https://{subscriber-subdomain}.{uaa-domain}`

### Why This Matters

1. **Tenant Isolation**: Tokens must be issued by the subscriber's XSUAA to ensure proper tenant isolation
2. **Token Claims**: The `zid` (zone ID) claim in the JWT must match the subscriber's tenant ID
3. **Destination Resolution**: The Destination service requires tenant-scoped tokens to fetch subscriber-specific destinations
4. **Token Propagation**: Downstream CAP services validate tokens against the subscriber's XSUAA
5. **Security**: Using provider-level tokens for subscriber requests violates the BTP security model

### Request Flow

```
User → Approuter (subscriber-a.myapp.com) → MCP Server
       [Sets x-forwarded-host header]
```

The approuter sets `x-forwarded-host: subscriber-a.myapp.com`, which the MCP server must use to:
1. Determine the subscriber subdomain (`subscriber-a`)
2. Build the subscriber's XSUAA URL (`https://subscriber-a.authentication.eu10.hana.ondemand.com`)
3. Exchange tokens at the subscriber's XSUAA endpoint

## Testing Checklist

- [ ] `getEffectiveHost()` returns subscriber approuter URL (not internal CF URL)
- [ ] `getEffectiveHost()` respects `x-forwarded-host` header priority
- [ ] `getSubscriberXsuaaUrl()` builds correct subscriber XSUAA URL
- [ ] OAuth metadata `issuer` field points to subscriber's XSUAA
- [ ] Authorization endpoint redirects to subscriber's XSUAA
- [ ] Token exchange happens at subscriber's XSUAA token endpoint
- [ ] JWT access token contains correct subscriber `zid` claim
- [ ] JWT access token contains subscriber-specific scopes
- [ ] Local development works (falls back to `identityzone` or `zid`)
- [ ] Destination service calls succeed with subscriber-scoped tokens
- [ ] Downstream CAP services accept the subscriber-scoped tokens
- [ ] Multiple subscribers can authenticate independently

## Related Issues

- Prerequisite for proper multi-tenant authentication
- Required for destination service integration
- Enables proper token propagation to downstream services
- Related to RFC 9728 compliance (Issue #1)

## Implementation Status

✅ **Fixed in patch**: `@gavdi+cap-mcp+1.4.1.patch`

## Additional Context

This issue is **critical for multi-tenant BTP deployments**. Without proper subscriber URL resolution, the entire OAuth flow operates at the provider level, which:

### Security Implications
- ✗ Breaks tenant isolation (subscriber A could potentially access subscriber B's data)
- ✗ Violates BTP security model and compliance requirements
- ✗ Tokens lack proper tenant context for authorization decisions

### Functional Implications
- ✗ Destination service lookups fail (wrong tenant context)
- ✗ Token propagation to downstream services fails validation
- ✗ CAP service authorization checks fail (wrong tenant in token)
- ✗ Multi-tenant data isolation cannot be enforced

### Example Token Comparison

**Provider-level token (WRONG):**
```json
{
  "zid": "provider-12345-guid",
  "zone_id": "provider-12345-guid",
  "scope": ["app!t123.Admin"],
  "ext_attr": {
    "zdn": "provider-zone"
  }
}
```

**Subscriber-level token (CORRECT):**
```json
{
  "zid": "subscriber-a-67890-guid",
  "zone_id": "subscriber-a-67890-guid", 
  "scope": ["app!t456.Admin"],
  "ext_attr": {
    "zdn": "subscriber-a",
    "serviceinstanceid": "subscriber-a-instance-id"
  }
}
```

### Impact Assessment

**Severity**: **Critical** - Breaks multi-tenant deployments completely

**Affected Users**: All multi-tenant BTP applications using MCP with XSUAA authentication

**Workaround**: None - requires code changes to fix

**Priority Justification**: This is a **blocker** for production multi-tenant deployments as it violates fundamental BTP security and isolation requirements.
