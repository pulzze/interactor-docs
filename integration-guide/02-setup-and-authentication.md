# Setup and Authentication

**Last Updated:** 2026-01-28

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

If you use [Configuration Code Sync](#configuration-code-sync) to manage tools, include the secret in your sync payload:

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

## Configuration Code Sync

Config Sync allows you to define your Interactor resources (tools, assistants, workflows, data sources) as code in JSON config files and synchronize them to Interactor automatically. This enables:

- **Infrastructure as Code** - Version control your AI agent configurations
- **CI/CD Integration** - Deploy resource changes alongside your application code
- **Environment Parity** - Use the same config files across dev/staging/production with environment variable interpolation

### How It Works

The sync endpoint creates, updates, or archives resources based on your config files:

1. **New resources** are created with `managed_by="config"`
2. **Existing resources** (by name) are updated if changed
3. **Removed resources** (no longer in config) are archived (set to `active=false`)

Only resources marked as `managed_by="config"` are affected - user-created resources via the API are never modified.

### Sync Endpoint

```
POST /api/v1/config/sync
```

**Request:**
```json
{
  "callback_secret": "your-tool-callback-secret",
  "tools": [...],
  "assistants": [...],
  "workflows": [...],
  "data_sources": [...],
  "oauth_client_configs": [...]
}
```

All fields are optional. Missing resource types are skipped (not archived).

**Response:**
```json
{
  "data": {
    "tools": {
      "status": "success",
      "created": 2,
      "updated": 1,
      "unchanged": 3,
      "archived": 0,
      "errors": []
    },
    "assistants": {
      "status": "success",
      "created": 1,
      "updated": 0,
      "unchanged": 2,
      "archived": 1,
      "errors": []
    },
    "workflows": { "status": "skipped" },
    "data_sources": { "status": "skipped" },
    "oauth_client_configs": {
      "status": "success",
      "created": 2,
      "updated": 0,
      "unchanged": 0,
      "archived": 0,
      "errors": []
    }
  }
}
```

### Environment Variable Interpolation

Config files support `${VAR_NAME}` syntax for environment variables:

```json
{
  "name": "send_email",
  "callback_url": "${TOOL_CALLBACK_URL}",
  "description": "Send an email notification"
}
```

This allows the same config files to work across environments by setting different environment variables.

### Tool Configuration

```json
[
  {
    "name": "get_weather",
    "description": "Get current weather for a location",
    "execution_type": "http_callback",
    "callback_url": "${TOOL_CALLBACK_URL}",
    "callback_method": "POST",
    "parameters": {
      "type": "object",
      "properties": {
        "location": {
          "type": "string",
          "description": "City name or coordinates"
        }
      },
      "required": ["location"]
    }
  }
]
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier |
| `description` | string | Yes | What the tool does |
| `execution_type` | string | No | `http_callback` (default) or `workflow` |
| `callback_url` | string | Yes* | URL to call when tool is invoked (*required for http_callback) |
| `callback_method` | string | No | HTTP method (default: `POST`) |
| `callback_headers` | object | No | Additional headers to include |
| `callback_timeout_ms` | integer | No | Request timeout in milliseconds |
| `parameters` | object | Yes | JSON Schema defining tool parameters |
| `workflow_name` | string | No | Workflow to execute (for execution_type=workflow) |
| `required_credentials` | object | No | Service credentials required for execution |
| `active` | boolean | No | Whether the tool is enabled (default: `true`) |

### Assistant Configuration

```json
[
  {
    "name": "support-assistant",
    "description": "Helps users with support questions",
    "type": "conversational",
    "system_prompt": "You are a helpful support assistant...",
    "llm_provider": "anthropic",
    "llm_model": "claude-sonnet-4-20250514",
    "llm_config": {
      "temperature": 0.7
    },
    "default_tools": ["search_knowledge_base", "create_ticket"]
  }
]
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier |
| `description` | string | No | What the assistant does |
| `type` | string | No | `conversational` (default) |
| `system_prompt` | string | Yes | System prompt defining behavior |
| `llm_provider` | string | No | `anthropic` or `openai` |
| `llm_model` | string | No | Model identifier |
| `llm_config` | object | No | LLM settings (e.g., `{"temperature": 0.7}`) |
| `default_tools` | array | No | Tool names the assistant can use |
| `default_data_sources` | array | No | Data source names the assistant can query |
| `default_workflows` | array | No | Workflow names the assistant can trigger |
| `max_tool_calls_per_turn` | integer | No | Limit tool calls per conversation turn |
| `session_timeout_minutes` | integer | No | Auto-close rooms after inactivity |
| `active` | boolean | No | Whether the assistant is enabled (default: `true`) |

### Workflow Configuration

```json
[
  {
    "name": "approval-workflow",
    "description": "Multi-step approval process",
    "initial_state": "pending",
    "parameters": {
      "type": "object",
      "properties": {
        "request_id": { "type": "string" },
        "amount": { "type": "number" }
      },
      "required": ["request_id"]
    },
    "spec": {
      "pending": {
        "on_enter": [...],
        "transitions": [...]
      }
    }
  }
]
```

See [Workflows](05-workflows.md) for the full workflow specification format.

### Data Source Configuration

```json
[
  {
    "name": "products_db",
    "description": "Product catalog database",
    "adapter": "postgres",
    "connection_config": {
      "url": "${DATABASE_URL}"
    },
    "capabilities": ["read"],
    "allowed_tables": ["products", "categories"],
    "schema_info": {
      "products": {
        "columns": ["id", "name", "price", "category_id"]
      }
    }
  }
]
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier |
| `description` | string | No | What the data source contains |
| `adapter` | string | Yes | `postgres`, `mysql`, `http`, etc. |
| `connection_config` | object | Yes | Connection details (adapter-specific) |
| `capabilities` | array | No | `["read"]` or `["read", "write"]` |
| `allowed_tables` | array | No | Tables the assistant can query |
| `denied_tables` | array | No | Tables the assistant cannot query |
| `schema_info` | object | No | Table/column metadata for LLM context |
| `semantic_mappings` | object | No | Natural language to schema mappings |
| `max_rows` | integer | No | Maximum rows to return |
| `timeout_ms` | integer | No | Query timeout in milliseconds |
| `active` | boolean | No | Whether the data source is enabled (default: `true`) |

### OAuth Client Config Sync

OAuth client configs allow you to use your own OAuth apps (Google, Slack, etc.) instead of platform defaults. Unlike other resources, OAuth configs are built entirely from environment variables for security - no config files contain credentials.

**Environment Variable Pattern:**

```bash
# Pattern: OAUTH_<PROVIDER>_CLIENT_ID and OAUTH_<PROVIDER>_CLIENT_SECRET
OAUTH_GOOGLE_CLIENT_ID=123456789.apps.googleusercontent.com
OAUTH_GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxx

OAUTH_SLACK_CLIENT_ID=1234567890.123456789012
OAUTH_SLACK_CLIENT_SECRET=abcdef123456789

# Optional: Provider-specific or default redirect URI
OAUTH_REDIRECT_URI=https://your-app.com/oauth/callback
OAUTH_GOOGLE_REDIRECT_URI=https://your-app.com/oauth/google/callback
```

**How It Works:**

Your application scans environment variables at startup, builds the config array, and includes it in the sync payload:

```javascript
function buildOAuthConfigsFromEnv() {
  const configs = [];
  const providers = new Map();

  // Scan for OAUTH_<PROVIDER>_CLIENT_ID pattern
  for (const [key, value] of Object.entries(process.env)) {
    const match = key.match(/^OAUTH_([A-Z_]+)_(CLIENT_ID|CLIENT_SECRET|REDIRECT_URI)$/);
    if (match && value) {
      const [, providerUpper, field] = match;
      const provider = providerUpper.toLowerCase();

      if (!providers.has(provider)) providers.set(provider, {});
      providers.get(provider)[field.toLowerCase()] = value;
    }
  }

  // Build config for providers with at least client_id
  for (const [provider, config] of providers) {
    if (config.client_id) {
      configs.push({
        auth_provider: provider,
        client_id: config.client_id,
        client_secret: config.client_secret,
        redirect_uri: config.redirect_uri || process.env.OAUTH_REDIRECT_URI
      });
    }
  }

  return configs;
}
```

**Sync Payload:**

```json
{
  "oauth_client_configs": [
    {
      "auth_provider": "google",
      "client_id": "123456789.apps.googleusercontent.com",
      "client_secret": "GOCSPX-xxxxxxxxxx",
      "redirect_uri": "https://your-app.com/oauth/callback"
    },
    {
      "auth_provider": "slack",
      "client_id": "1234567890.123456789012",
      "client_secret": "abcdef123456789"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `auth_provider` | string | Yes | Provider slug (e.g., `google`, `slack`, `microsoft`) |
| `client_id` | string | Yes | OAuth client ID from the provider |
| `client_secret` | string | No | OAuth client secret (encrypted at rest) |
| `redirect_uri` | string | No | OAuth redirect URI |
| `additional_params` | object | No | Extra provider-specific parameters |
| `enabled` | boolean | No | Whether the config is enabled (default: `true`) |

**Security Notes:**

- OAuth credentials are encrypted at rest in Interactor's database
- Environment variables should be set via your deployment system (Docker secrets, Kubernetes secrets, etc.)
- Never commit actual credentials to version control
- Config-managed OAuth configs are tracked separately from manually-created ones

### Example: Startup Sync

Here's a complete example that syncs config on application startup (Node.js):

```javascript
import fs from 'fs';

// Load config files with env var interpolation
function loadConfig(filename) {
  const content = fs.readFileSync(`./config/${filename}`, 'utf-8');
  const data = JSON.parse(content);
  return interpolateEnvVars(data);
}

function interpolateEnvVars(obj) {
  if (typeof obj === 'string') {
    return obj.replace(/\$\{([^}]+)\}/g, (_, name) => process.env[name] || '');
  }
  if (Array.isArray(obj)) return obj.map(interpolateEnvVars);
  if (typeof obj === 'object' && obj !== null) {
    return Object.fromEntries(
      Object.entries(obj).map(([k, v]) => [k, interpolateEnvVars(v)])
    );
  }
  return obj;
}

// Build OAuth configs from environment variables
function buildOAuthConfigsFromEnv() {
  const providers = new Map();

  for (const [key, value] of Object.entries(process.env)) {
    const match = key.match(/^OAUTH_([A-Z_]+)_(CLIENT_ID|CLIENT_SECRET|REDIRECT_URI)$/);
    if (match && value) {
      const [, providerUpper, field] = match;
      const provider = providerUpper.toLowerCase();
      if (!providers.has(provider)) providers.set(provider, {});
      providers.get(provider)[field.toLowerCase().replace('_', '')] = value;
    }
  }

  return Array.from(providers.entries())
    .filter(([, config]) => config.clientid)
    .map(([provider, config]) => ({
      auth_provider: provider,
      client_id: config.clientid,
      client_secret: config.clientsecret,
      redirect_uri: config.redirecturi || process.env.OAUTH_REDIRECT_URI
    }));
}

async function syncConfig() {
  // Get access token
  const tokenRes = await fetch('https://auth.interactor.com/api/v1/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'client_credentials',
      client_id: process.env.CLIENT_ID,
      client_secret: process.env.CLIENT_SECRET
    })
  });
  const { data: { access_token } } = await tokenRes.json();

  // Build sync payload
  const payload = {
    callback_secret: process.env.TOOL_CALLBACK_SECRET,
    tools: loadConfig('tools.json'),
    assistants: loadConfig('assistants.json'),
    oauth_client_configs: buildOAuthConfigsFromEnv()
  };

  // Sync to Interactor
  const res = await fetch('https://core.interactor.com/api/v1/config/sync', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${access_token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(payload)
  });

  const result = await res.json();
  console.log('Config sync result:', result.data);
}

// Run on startup
syncConfig().catch(console.error);
```

### File Organization

A typical config directory structure:

```
your-app/
├── config/
│   ├── tools.json
│   ├── assistants.json
│   ├── workflows.json
│   └── data_sources.json
├── .env
└── server.js
```

### Best Practices

1. **Version control config files** - Commit them alongside your application code
2. **Use environment variables for secrets** - Never hardcode callback URLs or credentials
3. **Sync on deployment** - Run config sync as part of your CI/CD pipeline
4. **Use unique names** - Resource names must be unique within your account
5. **Test in development first** - Verify config changes work before syncing to production

---

## Next Steps

With authentication configured, proceed to:

- [Credential Management](03-credential-management.md) - Manage OAuth tokens for external services
- [AI Agents](04-ai-agents.md) - Create AI assistants and chat interfaces
- [Workflows](05-workflows.md) - Build automated state machines
