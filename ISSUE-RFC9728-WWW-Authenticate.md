# Missing RFC 9728 Compliant `WWW-Authenticate` Header in 401 Responses

## Current Behaviour

The MCP server returns 401 Unauthorized responses without the required `WWW-Authenticate` header, preventing MCP clients from discovering OAuth authentication metadata as specified in RFC 9728 (OAuth 2.0 Protected Resource Metadata).

When authentication fails, the server returns a plain 401 JSON-RPC error response:

```javascript
res.status(401).json({
  jsonrpc: "2.0",
  error: {
    code: 10,
    message: "Unauthorized",
    id: null
  }
});
```

**Problems:**
- No `WWW-Authenticate` header is included
- MCP clients cannot discover the OAuth authorization server metadata endpoint
- Violates RFC 9728 requirements for protected resources
- Clients must manually configure OAuth endpoints instead of auto-discovery

## Expected Behaviour

401 responses should include a `WWW-Authenticate` header pointing to the OAuth protected resource metadata:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://{host}/.well-known/oauth-protected-resource"
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "error": {
    "code": 10,
    "message": "Unauthorized: Missing authorization header",
    "id": null
  }
}
```

This will allow MCP clients to auto-discover OAuth metadata as required by RFC 9728.

## Steps to Reproduce

1. Configure the MCP server with JWT/XSUAA authentication enabled
2. Send a request to the MCP server without an `Authorization` header:
   ```bash
   curl -X POST http://localhost:4004/mcp \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
   ```
3. Observe the HTTP response headers using `-v` flag or browser DevTools
4. Note the absence of a `WWW-Authenticate` header in the 401 response
5. Verify that MCP clients cannot auto-discover OAuth endpoints

**Alternative reproduction:**
1. Send a request with an expired or invalid JWT token
2. Observe the same missing `WWW-Authenticate` header in the 401 response

## Environment

- **Environment**: SAP BTP Cloud Foundry / Local Development
- **Database Type**: N/A (Authentication layer issue)
- **Release Version**: Affects versions before 1.4.1
- **Authentication Type**: JWT / XSUAA / IAS
- **Configuration**: Any MCP server with authentication enabled
- **Customizations**: N/A

## Proposed Solution

Add a new utility function `set401WithWwwAuthenticate()` in `src/auth/factory.ts`:

```typescript
function set401WithWwwAuthenticate(req, res, message) {
  const baseUrl = `${getProtocol(req)}://${getEffectiveHost(req)}`;
  const resourceMetadataUrl = `${baseUrl}/.well-known/oauth-protected-resource`;

  res.setHeader(
    'WWW-Authenticate', 
    `Bearer resource_metadata="${resourceMetadataUrl}"`
  );

  logger.debug("[MCP-AUTH] Returning 401 with WWW-Authenticate header", { 
    resourceMetadataUrl,
    message 
  });

  res.status(401).json({
    jsonrpc: "2.0",
    error: { code: RPC_UNAUTHORIZED, message, id: null }
  });
}
```

**Replace all 401 responses in the authentication middleware:**

```typescript
// Before
res.status(401).json({
  jsonrpc: "2.0",
  error: { code: 10, message: "Unauthorized", id: null }
});

// After
return set401WithWwwAuthenticate(req, res, "Unauthorized: Missing authorization header");
```

## Affected Files

- `src/auth/factory.ts` - Authentication middleware (main changes)
- `src/auth/utils.ts` - OAuth endpoint registration
- `lib/auth/factory.js` - Compiled output

## Affected Scenarios

All 401 responses in the authentication flow:
- ❌ Missing authorization header
- ❌ Invalid or expired JWT token
- ❌ Anonymous user access attempts
- ❌ Failed XSUAA security context creation

## Standards Reference

- **RFC 9728**: [OAuth 2.0 Protected Resource Metadata](https://www.rfc-editor.org/rfc/rfc9728.html)
  - Section 3: Protected Resource Metadata Request
  - Requires `WWW-Authenticate` header with `resource_metadata` parameter pointing to `/.well-known/oauth-protected-resource`
- **RFC 6750**: [The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)
  - Section 3: The WWW-Authenticate Response Header Field

## Testing Checklist

- [ ] 401 response includes `WWW-Authenticate` header
- [ ] Header format: `Bearer resource_metadata="<url>"`
- [ ] Header points to correct `/.well-known/oauth-protected-resource` endpoint
- [ ] Works in multi-tenant scenarios (uses subscriber host, not internal CF URL)
- [ ] Works in local development (uses localhost)
- [ ] MCP clients can auto-discover OAuth metadata from the header
- [ ] Error messages are descriptive (not just generic "Unauthorized")
- [ ] All 401 scenarios use the new function consistently

## Related Issues

- Depends on proper host resolution for multi-tenant environments
- Enables OAuth auto-discovery for MCP clients
- Prerequisite for MCP protocol compliance

## Implementation Status

✅ **Fixed in patch**: `@gavdi+cap-mcp+1.4.1.patch`

## Additional Context

This issue is **critical for MCP protocol compliance**. The MCP specification requires that servers support OAuth 2.0 discovery, and RFC 9728 mandates the `WWW-Authenticate` header for protected resources to enable this discovery flow.

Without this header, MCP clients cannot automatically discover:
- ✗ Authorization endpoint (`/oauth/authorize`)
- ✗ Token endpoint (`/oauth/token`)
- ✗ Supported grant types (`authorization_code`, `refresh_token`)
- ✗ Supported scopes
- ✗ Client registration endpoint (`/oauth/register`)

**Impact:**
- MCP clients must manually configure OAuth endpoints (poor UX)
- Non-compliant with OAuth 2.0 standards
- Breaks auto-discovery workflows in modern OAuth clients

**Priority Justification:**
This is a **High Priority** issue because it affects all authenticated MCP deployments and prevents proper OAuth integration with compliant clients.
