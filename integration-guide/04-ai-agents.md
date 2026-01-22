# AI Agents

**Last Updated:** 2026-01-22

AI Agents are LLM-powered assistants that can have conversations, use tools, and access your data sources to help your users accomplish tasks.

---

## Overview

The AI Agents system consists of:

- **Assistants** - Configured AI agents with specific behaviors and capabilities
- **Rooms** - Chat sessions between users and assistants
- **Tools** - Custom functions that assistants can invoke
- **Data Sources** - Databases and APIs that assistants can query

---

## Assistants

### Create an Assistant

```bash
curl -X POST https://core.interactor.com/api/v1/agents/assistants \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "support_assistant",
    "title": "Support Assistant",
    "description": "Helps users with support questions",
    "model_config": {
      "provider": "anthropic",
      "model": "claude-sonnet-4-20250514",
      "temperature": 0.7
    },
    "instructions": "You are a helpful support assistant. Be concise and friendly.",
    "enabled_tools": ["search_knowledge_base", "create_ticket"]
  }'
```

**Response:**
```json
{
  "data": {
    "id": "asst_abc",
    "name": "support_assistant",
    "title": "Support Assistant",
    "created_at": "2026-01-20T12:00:00Z"
  }
}
```

### Assistant Configuration Options

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique identifier (lowercase, underscores) |
| `title` | string | Display name |
| `description` | string | What the assistant does |
| `model_config.provider` | string | `anthropic` or `openai` |
| `model_config.model` | string | Model identifier |
| `model_config.temperature` | number | Response randomness (0.0 - 1.0) |
| `instructions` | string | System prompt defining behavior |
| `enabled_tools` | array | Tool names the assistant can use |

### List Assistants

```bash
curl https://core.interactor.com/api/v1/agents/assistants \
  -H "Authorization: Bearer <token>"
```

### Get Assistant

```bash
curl https://core.interactor.com/api/v1/agents/assistants/asst_abc \
  -H "Authorization: Bearer <token>"
```

### Update Assistant

```bash
curl -X PUT https://core.interactor.com/api/v1/agents/assistants/asst_abc \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "instructions": "Updated instructions...",
    "model_config": {
      "temperature": 0.5
    }
  }'
```

---

## Chat Rooms

Rooms are conversations between a user and an assistant.

### Create a Room

```bash
curl -X POST https://core.interactor.com/api/v1/agents/asst_abc/rooms \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "namespace": "user_123",
    "metadata": {
      "user_name": "John",
      "context": "billing_question"
    }
  }'
```

**Response:**
```json
{
  "data": {
    "id": "room_xyz",
    "assistant_id": "asst_abc",
    "status": "active",
    "created_at": "2026-01-20T12:00:00Z"
  }
}
```

### List Rooms

```bash
curl https://core.interactor.com/api/v1/agents/rooms \
  -H "Authorization: Bearer <token>"
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `namespace` | string | Filter by namespace |
| `assistant_id` | string | Filter by assistant |
| `status` | string | `active` or `closed` |

### Get Room

```bash
curl https://core.interactor.com/api/v1/agents/rooms/room_xyz \
  -H "Authorization: Bearer <token>"
```

### Close Room

```bash
curl -X POST https://core.interactor.com/api/v1/agents/rooms/room_xyz/close \
  -H "Authorization: Bearer <token>"
```

---

## Messages

### Send a Message

```bash
curl -X POST https://core.interactor.com/api/v1/agents/rooms/room_xyz/messages \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "How do I update my billing information?",
    "role": "user"
  }'
```

**Response:**
```json
{
  "data": {
    "id": "msg_123",
    "role": "user",
    "content": "How do I update my billing information?",
    "created_at": "2026-01-20T12:00:00Z"
  }
}
```

The assistant's response is generated asynchronously. Use streaming or webhooks to receive it.

### List Messages

```bash
curl https://core.interactor.com/api/v1/agents/rooms/room_xyz/messages \
  -H "Authorization: Bearer <token>"
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Max messages to return |
| `before` | string | Message ID for pagination |

### Streaming Responses

For real-time message streaming, see [Webhooks and Streaming](06-webhooks-and-streaming.md).

---

## Tools

Tools are custom functions that assistants can invoke during conversations.

### Configure Callback Security (Required)

Before tools can be executed, you must configure a callback secret for your account. This secret is used to sign all tool callbacks so your endpoint can verify they came from Interactor.

**Step 1: Generate a secret**

```bash
openssl rand -base64 32
# Example output: X18FUShSrb0qGVTt17sgEgV/5naDw1AV5Aqs5HWVEMg=
```

**Step 2: Configure the secret in Interactor**

Via API:
```bash
curl -X POST https://core.interactor.com/api/v1/account/callback-secret \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"secret": "X18FUShSrb0qGVTt17sgEgV/5naDw1AV5Aqs5HWVEMg="}'
```

Or via config sync (recommended):
```json
{
  "callback_secret": "X18FUShSrb0qGVTt17sgEgV/5naDw1AV5Aqs5HWVEMg=",
  "tools": [...],
  "assistants": [...]
}
```

### Register a Tool

```bash
curl -X POST https://core.interactor.com/api/v1/tools \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_products",
    "description": "Search the product catalog",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {"type": "string", "description": "Search query"},
        "category": {"type": "string", "description": "Product category"}
      },
      "required": ["query"]
    },
    "callback_url": "https://yourapp.com/api/tools/callback"
  }'
```

**Response:**
```json
{
  "data": {
    "id": "tool_abc",
    "name": "search_products",
    "created_at": "2026-01-20T12:00:00Z"
  }
}
```

### Tool Callback

When the assistant invokes your tool, Interactor POSTs to your `callback_url` with these headers:

| Header | Description |
|--------|-------------|
| `x-interactor-signature` | HMAC-SHA256 signature (hex encoded) |
| `x-interactor-timestamp` | Unix timestamp when request was signed |
| `x-interactor-tool` | Name of the tool being executed |
| `x-interactor-account` | Your account ID |

**Request body:**
```json
{
  "tool_name": "search_products",
  "arguments": {"query": "laptop", "category": "electronics"},
  "context": {
    "account_id": "acc_abc123",
    "user_ref": "user_456"
  },
  "credentials": {}
}
```

**Your response:**
```json
{
  "products": [
    {"id": "prod_1", "name": "MacBook Pro", "price": 1999}
  ]
}
```

### Verify Tool Callback Signature

You **must** verify the signature on all callbacks. The signature is computed over a JSON payload containing the tool name, arguments, and timestamp:

```typescript
import crypto from 'crypto';

const TOOL_CALLBACK_SECRET = process.env.TOOL_CALLBACK_SECRET;

function verifyCallbackSignature(req, res, next) {
  const signature = req.headers['x-interactor-signature'];
  const timestamp = req.headers['x-interactor-timestamp'];

  if (!signature || !timestamp) {
    return res.status(401).json({ error: 'Missing signature' });
  }

  // Check timestamp (5 minute window)
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - parseInt(timestamp)) > 300) {
    return res.status(401).json({ error: 'Request expired' });
  }

  // Build signature payload
  const payload = JSON.stringify({
    tool: req.body.tool_name,
    arguments: req.body.arguments,
    timestamp: timestamp
  });

  const expected = crypto
    .createHmac('sha256', TOOL_CALLBACK_SECRET)
    .update(payload)
    .digest('hex');

  // Timing-safe comparison
  const sigBuffer = Buffer.from(signature, 'hex');
  const expectedBuffer = Buffer.from(expected, 'hex');

  if (sigBuffer.length !== expectedBuffer.length ||
      !crypto.timingSafeEqual(sigBuffer, expectedBuffer)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  next();
}
```

### List Tools

```bash
curl https://core.interactor.com/api/v1/tools \
  -H "Authorization: Bearer <token>"
```

### Get Tool

```bash
curl https://core.interactor.com/api/v1/tools/tool_abc \
  -H "Authorization: Bearer <token>"
```

### Update Tool

```bash
curl -X PUT https://core.interactor.com/api/v1/tools/tool_abc \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Updated description",
    "callback_url": "https://yourapp.com/api/v2/tools/search_products"
  }'
```

### Delete Tool

```bash
curl -X DELETE https://core.interactor.com/api/v1/tools/tool_abc \
  -H "Authorization: Bearer <token>"
```

---

## Data Sources

Connect databases and APIs that assistants can query directly.

### Register a Data Source

```bash
curl -X POST https://core.interactor.com/api/v1/data-sources \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "sales_database",
    "type": "postgresql",
    "connection": {
      "host": "db.yourcompany.com",
      "port": 5432,
      "database": "sales",
      "username": "readonly_user",
      "password": "..."
    },
    "description": "Sales and customer data"
  }'
```

**Response:**
```json
{
  "data": {
    "id": "ds_abc",
    "name": "sales_database",
    "status": "connected",
    "schema_status": "extracting"
  }
}
```

Interactor automatically extracts the database schema for the assistant to understand.

### List Data Sources

```bash
curl https://core.interactor.com/api/v1/data-sources \
  -H "Authorization: Bearer <token>"
```

### Get Data Source

```bash
curl https://core.interactor.com/api/v1/data-sources/ds_abc \
  -H "Authorization: Bearer <token>"
```

### Update Data Source

```bash
curl -X PUT https://core.interactor.com/api/v1/data-sources/ds_abc \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Updated description"
  }'
```

### Delete Data Source

```bash
curl -X DELETE https://core.interactor.com/api/v1/data-sources/ds_abc \
  -H "Authorization: Bearer <token>"
```

### Refresh Schema

Re-extract the schema if your database structure changed:

```bash
curl -X POST https://core.interactor.com/api/v1/data-sources/ds_abc/refresh-schema \
  -H "Authorization: Bearer <token>"
```

### Execute Query

Run a query against the data source:

```bash
curl -X POST https://core.interactor.com/api/v1/data-sources/ds_abc/query \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT * FROM customers WHERE created_at > $1 LIMIT 10",
    "parameters": ["2026-01-01"]
  }'
```

**Response:**
```json
{
  "data": {
    "columns": ["id", "name", "email", "created_at"],
    "rows": [
      [1, "John Doe", "john@example.com", "2026-01-15"]
    ],
    "row_count": 1
  }
}
```

### Semantic Mappings

Add synonyms and descriptions to help the assistant understand your schema:

```bash
curl -X PATCH https://core.interactor.com/api/v1/data-sources/ds_abc/semantic-mappings \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "mappings": {
      "customers": {
        "description": "Customer accounts",
        "synonyms": ["users", "clients", "accounts"]
      },
      "customers.created_at": {
        "description": "When the customer signed up",
        "synonyms": ["signup date", "registration date"]
      }
    }
  }'
```

This helps the assistant translate natural language questions like "How many users signed up last month?" into the correct SQL query.

---

## Knowledge Base Search

Search for external services that assistants can connect to:

### Search Services

```bash
curl -X POST https://core.interactor.com/api/v1/knowledge-base/services/search \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "calendar scheduling",
    "limit": 5
  }'
```

**Response:**
```json
{
  "data": {
    "services": [
      {
        "id": "google_calendar",
        "name": "Google Calendar",
        "description": "Calendar and scheduling service",
        "auth_type": "oauth2",
        "capabilities": ["create_event", "list_events", "update_event"]
      }
    ]
  }
}
```

### Lookup Service

```bash
curl -X POST https://core.interactor.com/api/v1/knowledge-base/services/lookup \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"service_id": "google_calendar"}'
```

### Get Service Details

```bash
curl https://core.interactor.com/api/v1/knowledge-base/services/google_calendar \
  -H "Authorization: Bearer <token>"
```

### Get Service OAuth Config

```bash
curl https://core.interactor.com/api/v1/knowledge-base/services/google_calendar/oauth \
  -H "Authorization: Bearer <token>"
```

---

## Best Practices

1. **Keep instructions focused** - Clear, specific instructions produce better results
2. **Use semantic mappings** - Help assistants understand your data schema
3. **Secure tool callbacks** - Always verify signatures on tool callbacks
4. **Use read-only database users** - Limit data source connections to read-only access
5. **Monitor tool usage** - Track which tools are being called and how often
6. **Test conversations** - Verify assistant behavior before deploying to users

---

## Webhook Events

Subscribe to agent events:

| Event | Description |
|-------|-------------|
| `agent.room.message` | New message in a room |
| `agent.room.closed` | Room was closed |

See [Webhooks and Streaming](06-webhooks-and-streaming.md) for details.

---

## Next Steps

- [Workflows](05-workflows.md) - Combine AI agents with automated workflows
- [Webhooks and Streaming](06-webhooks-and-streaming.md) - Real-time message streaming
- [SDK Examples](07-sdk-examples.md) - Complete code examples
