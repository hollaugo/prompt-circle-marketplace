---
name: mcp-builder:add-auth
description: Add OAuth authentication to your MCP server using Auth0 or custom providers
---

# Add Authentication

You are helping the user add authentication to their MCP server using the MCP Authorization Specification.

## Overview

MCP supports OAuth 2.1 with PKCE for authentication. This enables:
- User-specific data access
- Scoped permissions per tool
- Audit logging with user context

## When to Add Auth

Add authentication when:
- Tools access user-specific data
- Tools perform actions on behalf of users
- You need audit trails with user identity
- Multi-tenant data isolation is required

## MCP Authorization Spec

The spec requires:
1. **Protected Resource Metadata** endpoint
2. **OAuth Metadata** (authorization server discovery)
3. **Token verification** in tool handlers
4. **WWW-Authenticate** challenges on 401

## Workflow

### Phase 1: Choose Auth Provider

| Provider | Best For | Complexity |
|----------|----------|------------|
| **Auth0** | Quick setup, managed service | Low |
| **Supabase Auth** | Already using Supabase | Low |
| **Custom OAuth** | Full control, enterprise | High |

### Phase 2: Configure Auth0

1. **Create Auth0 Application**
   - Go to Auth0 Dashboard → Applications → Create
   - Choose "Regular Web Application"
   - Note: Client ID, Client Secret, Domain

2. **Configure Allowed Callbacks**
   ```
   Allowed Callback URLs: https://your-server.com/callback
   Allowed Logout URLs: https://your-server.com
   ```

3. **Create API in Auth0**
   - APIs → Create API
   - Identifier: `https://your-server.com/api`
   - Add scopes: `read:tasks`, `write:tasks`, `delete:tasks`

### Phase 3: Implement Server Endpoints

**Python (FastMCP):**

```python
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
import httpx
import jwt

AUTH0_DOMAIN = os.environ["AUTH0_DOMAIN"]
AUTH0_AUDIENCE = os.environ["AUTH0_AUDIENCE"]

# Protected Resource Metadata
@mcp.custom_route("/.well-known/oauth-protected-resource", methods=["GET"])
async def protected_resource_metadata(request: Request):
    return JSONResponse({
        "resource": AUTH0_AUDIENCE,
        "authorization_servers": [f"https://{AUTH0_DOMAIN}/"],
        "scopes_supported": ["read:tasks", "write:tasks", "delete:tasks"],
        "bearer_methods_supported": ["header"],
    })

# Token verification middleware
async def verify_token(request: Request) -> dict:
    auth_header = request.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        raise HTTPException(
            status_code=401,
            headers={"WWW-Authenticate": f'Bearer realm="{AUTH0_AUDIENCE}"'}
        )

    token = auth_header[7:]

    # Get JWKS
    jwks_url = f"https://{AUTH0_DOMAIN}/.well-known/jwks.json"
    async with httpx.AsyncClient() as client:
        jwks = (await client.get(jwks_url)).json()

    # Verify token
    try:
        payload = jwt.decode(
            token,
            jwks,
            algorithms=["RS256"],
            audience=AUTH0_AUDIENCE,
            issuer=f"https://{AUTH0_DOMAIN}/"
        )
        return payload
    except jwt.InvalidTokenError as e:
        raise HTTPException(status_code=401, detail=str(e))

# Get user from token in tool handlers
def get_user_id(request: Request) -> str:
    """Extract user ID from verified token."""
    token_payload = request.state.token_payload
    return token_payload.get("sub")
```

### Phase 4: Scope Tool Access

Implement tool-level permissions:

```python
TOOL_SCOPES = {
    "search-tasks": ["read:tasks"],
    "create-task": ["write:tasks"],
    "delete-task": ["delete:tasks"],
}

async def check_scope(request: Request, tool_name: str):
    """Verify user has required scope for tool."""
    required_scopes = TOOL_SCOPES.get(tool_name, [])
    token_scopes = request.state.token_payload.get("scope", "").split()

    for scope in required_scopes:
        if scope not in token_scopes:
            raise HTTPException(
                status_code=403,
                detail=f"Missing required scope: {scope}"
            )
```

### Phase 5: User-Scoped Data

Filter data by authenticated user:

```python
@mcp.tool()
async def search_tasks(query: str, request: Request) -> str:
    """Search tasks for the authenticated user."""
    user_id = get_user_id(request)

    # Query only this user's tasks
    results = await db.fetch(
        "SELECT * FROM tasks WHERE user_id = $1 AND subject ILIKE $2",
        user_id, f"%{query}%"
    )

    return json.dumps(results)
```

## Environment Variables

```bash
# .env
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_AUDIENCE=https://your-server.com/api
AUTH0_CLIENT_ID=your-client-id
AUTH0_CLIENT_SECRET=your-client-secret
```

## Testing Auth

1. **Get test token from Auth0:**
   ```bash
   curl --request POST \
     --url https://YOUR_DOMAIN/oauth/token \
     --header 'content-type: application/json' \
     --data '{"client_id":"YOUR_CLIENT_ID","client_secret":"YOUR_CLIENT_SECRET","audience":"YOUR_API_AUDIENCE","grant_type":"client_credentials"}'
   ```

2. **Test authenticated request:**
   ```bash
   curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://localhost:8000/mcp
   ```

## Update State

```json
{
  "auth": {
    "provider": "auth0",
    "domain": "your-tenant.auth0.com",
    "scopes": ["read:tasks", "write:tasks", "delete:tasks"]
  }
}
```

## Security Checklist

- [ ] Token verification uses JWKS (not hardcoded secret)
- [ ] All sensitive tools require authentication
- [ ] Scopes are granular (read vs write vs delete)
- [ ] User data is isolated by user_id
- [ ] Audit logs include user identity
