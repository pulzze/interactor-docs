# Setup and Authentication

**Last Updated:** 2026-01-22

This guide covers account registration, OAuth client setup, and token management.

---

## Step 1: Register Your Organization

Create an account with the Account Server:

```bash
curl -X POST https://auth.interactor.com/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "developer@yourcompany.com",
    "password": "SecureP@ssw0rd!",
    "organization_name": "Your Company Inc"
  }'
```

**Password Requirements:**
- Minimum 12 characters
- At least one uppercase, lowercase, number, and special character

Check your email and click the verification link.

---

## Step 2: Create OAuth Client Credentials

Log in to get a user token, then create OAuth credentials for your backend:

```bash
# Login
curl -X POST https://auth.interactor.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "developer@yourcompany.com",
    "password": "SecureP@ssw0rd!"
  }'

# Response includes access_token - save it
```

```bash
# Create OAuth client
curl -X POST https://auth.interactor.com/api/v1/account/oauth-clients \
  -H "Authorization: Bearer <user_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "My Production Backend"}'
```

**Response:**
```json
{
  "data": {
    "client_id": "client_abc123",
    "client_secret": "secret_xyz789_SAVE_THIS",
    "name": "My Production Backend"
  }
}
```

> **Important:** Save `client_secret` securely - it's only shown once!

---

## Step 3: Store Your Credentials

Add to your backend's environment:

```bash
INTERACTOR_CLIENT_ID=client_abc123
INTERACTOR_CLIENT_SECRET=secret_xyz789_SAVE_THIS
```

---

## Step 4: Exchange Credentials for Access Token

Your backend exchanges credentials for tokens:

```bash
curl -X POST https://auth.interactor.com/api/v1/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "client_abc123",
    "client_secret": "secret_xyz789_SAVE_THIS"
  }'
```

**Response:**
```json
{
  "data": {
    "access_token": "eyJhbGciOiJSUzI1NiIs...",
    "token_type": "Bearer",
    "expires_in": 900
  }
}
```

---

## Step 5: Call Interactor APIs

Use the access token to call any Interactor API:

```bash
curl https://core.interactor.com/api/v1/credentials/summary \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..."
```

---

## Token Types

The Account Server issues different token types based on how you authenticate:

| Token Type | When Issued | Use Case |
|------------|-------------|----------|
| **App Token** | Client credentials grant | Backend server-to-server calls |
| **User Token** | User login (authorization code) | User-facing API calls |
| **Org Token** | Organization-level auth | Admin operations |

For most integrations, you'll use **App Tokens** obtained via client credentials. These tokens include your `org_id` for multi-tenant resource isolation.

---

## Token Management

Access tokens expire after **15 minutes**. Your backend should cache and refresh tokens proactively.

### Token Caching Strategy

```typescript
class InteractorClient {
  private accessToken: string | null = null;
  private tokenExpiry: Date | null = null;

  async getToken(): Promise<string> {
    // Return cached token if still valid (with 60s buffer)
    if (this.accessToken && this.tokenExpiry && this.tokenExpiry > new Date()) {
      return this.accessToken;
    }

    const response = await fetch('https://auth.interactor.com/api/v1/oauth/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'client_credentials',
        client_id: process.env.INTERACTOR_CLIENT_ID,
        client_secret: process.env.INTERACTOR_CLIENT_SECRET
      })
    });

    const data = await response.json();
    this.accessToken = data.data.access_token;
    // Refresh 60 seconds before expiry to avoid edge cases
    this.tokenExpiry = new Date(Date.now() + (data.data.expires_in - 60) * 1000);
    return this.accessToken;
  }

  async call(method: string, path: string, body?: any) {
    const token = await this.getToken();
    return fetch(`https://core.interactor.com/api/v1${path}`, {
      method,
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: body ? JSON.stringify(body) : undefined
    });
  }
}
```

---

## Secret Rotation

Rotate your client secret without downtime:

```bash
curl -X POST https://auth.interactor.com/api/v1/account/oauth-clients/<client_id>/rotate-secret \
  -H "Authorization: Bearer <user_access_token>"
```

**Response:**
```json
{
  "data": {
    "client_id": "client_abc123",
    "client_secret": "secret_NEW_SECRET_HERE",
    "previous_secret_expires_at": "2026-01-21T12:00:00Z"
  }
}
```

Both old and new secrets work for **24 hours** during rotation. This gives you time to update your deployment without service interruption.

### Rotation Best Practices

1. Call the rotate endpoint
2. Update your backend configuration with the new secret
3. Deploy the configuration change
4. The old secret expires automatically after 24 hours

---

## Managing OAuth Clients

### List OAuth Clients

```bash
curl https://auth.interactor.com/api/v1/account/oauth-clients \
  -H "Authorization: Bearer <user_access_token>"
```

### Delete OAuth Client

```bash
curl -X DELETE https://auth.interactor.com/api/v1/account/oauth-clients/<client_id> \
  -H "Authorization: Bearer <user_access_token>"
```

---

## Authentication Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `invalid_client` | Wrong client_id or client_secret | Verify credentials |
| `unauthorized_client` | Client not authorized for grant type | Ensure client_credentials grant is enabled |
| `invalid_grant` | Credentials expired or revoked | Create new OAuth client |

---

## SSE Authentication

For Server-Sent Events endpoints (like agent streaming), the browser's EventSource API doesn't support custom headers. Use query parameter authentication instead:

```javascript
// Browser EventSource (can't use headers)
const eventSource = new EventSource(
  `https://core.interactor.com/api/v1/agents/rooms/${roomId}/stream?token=${accessToken}`
);

// Server-side (use header)
fetch(`https://core.interactor.com/api/v1/agents/rooms/${roomId}/stream`, {
  headers: { 'Authorization': `Bearer ${accessToken}` }
});
```

Both methods are supported on all SSE endpoints.

---

## Security Best Practices

1. **Never expose client_secret in frontend code** - Only use credentials in your backend
2. **Use environment variables** - Don't hardcode credentials
3. **Rotate secrets regularly** - Use the rotation endpoint quarterly
4. **Monitor for unauthorized access** - Review OAuth client usage logs
5. **Use separate clients per environment** - Create different OAuth clients for dev/staging/production

---

## Tool Callback Security Setup

If your integration uses custom tools (solution-defined functions that Interactor can invoke), you must configure a callback secret. This ensures that only Interactor can call your tool endpoints.

### Step 1: Generate a Callback Secret

```bash
openssl rand -base64 32
```

Store this in your backend's environment:

```bash
TOOL_CALLBACK_SECRET=X18FUShSrb0qGVTt17sgEgV/5naDw1AV5Aqs5HWVEMg=
```

### Step 2: Configure in Interactor

**Option A: Via Config Sync (Recommended)**

If you use config sync to manage tools, include the secret in your sync payload:

```json
{
  "callback_secret": "${TOOL_CALLBACK_SECRET}",
  "tools": [...],
  "assistants": [...]
}
```

**Option B: Via API**

```bash
curl -X POST https://core.interactor.com/api/v1/account/callback-secret \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"secret": "your-secret-here"}'
```

### Step 3: Verify Callbacks

Your tool callback endpoint must verify the signature. See [AI Agents - Tools](04-ai-agents.md#verify-tool-callback-signature) for implementation details.

**Key points:**
- Interactor signs callbacks with HMAC-SHA256
- The `x-interactor-timestamp` header provides replay protection (5-minute window)
- Tool execution fails if no callback secret is configured

---

## Next Steps

With authentication configured, proceed to:

- [Credential Management](03-credential-management.md) - Manage OAuth tokens for external services
- [AI Agents](04-ai-agents.md) - Create AI assistants and chat interfaces
- [Workflows](05-workflows.md) - Build automated state machines
