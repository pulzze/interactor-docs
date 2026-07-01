# AI Agents

_Last verified: 2026-06-18_

**Last Updated:** 2026-05-24

AI Agents are LLM-powered assistants that can have conversations, use tools, and access your data sources to help your users accomplish tasks.

---

## Prerequisites

> **Authentication Required**: All AI Agent API operations require a valid `access_token`. Without it, all requests will fail with `401 Unauthorized`.

Before using the AI Agents API, you **must** complete the following:

1. **Complete [Setup and Authentication](02-setup-and-authentication.md)** - Register your organization and create OAuth client credentials
2. **Obtain an access token** - Exchange your `client_id` and `client_secret` for an `access_token` (Step 4 of Setup guide)
3. **Implement token refresh** - Tokens expire after 15 minutes; implement the caching strategy from the Setup guide

### Quick Token Check

```bash
# Test your token is working
curl https://core.interactor.com/api/v1/agents/assistants \
  -H "Authorization: Bearer <your_access_token>"

# If you get 401 Unauthorized, your token is missing, invalid, or expired
```

### Common Variable Names

When integrating Interactor into your application, you may see these variable names referring to the same token:

| Variable Name | Where Used | Same Token? |
|---------------|------------|-------------|
| `access_token` | OAuth response, API docs | ✓ |
| `interactor_access_token` | Solution app code | ✓ |
| `token` | curl examples, shorthand | ✓ |
| `Bearer <token>` | HTTP Authorization header | ✓ |

All refer to the JWT access token obtained from the OAuth token endpoint.

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
    "description": "Helps users with support questions",
    "system_prompt": "You are a helpful support assistant. Be concise and friendly.",
    "llm_provider": "openai",
    "llm_model": "gpt-4o",
    "llm_config": {
      "temperature": 0.7
    },
    "default_tools": ["search_knowledge_base", "create_ticket"]
  }'
```

**Response:**
```json
{
  "data": {
    "id": "asst_abc",
    "name": "support_assistant",
    "description": "Helps users with support questions",
    "created_at": "2026-01-20T12:00:00Z"
  }
}
```

### Assistant Configuration Options

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier (lowercase, underscores) |
| `system_prompt` | string | Yes | System prompt defining behavior |
| `description` | string | No | What the assistant does |
| `llm_provider` | string | No | `openai` (default: `openai`) |
| `llm_model` | string | No | Model identifier |
| `llm_config` | object | No | LLM settings (e.g., `{"temperature": 0.7}`) |
| `default_tools` | array | No | Tool IDs the assistant can use |
| `default_data_sources` | array | No | Data source IDs the assistant can query |
| `max_tool_calls_per_turn` | integer | No | Limit tool calls per conversation turn |
| `session_timeout_minutes` | integer | No | Auto-close rooms after inactivity |
| `active` | boolean | No | Whether the assistant is enabled (default: `true`) |
| `metadata` | object | No | Custom data to store with the assistant |
| `builtin_tools` | object | No | Configure built-in tool availability (see User Profiles) |
| `supporting_assistants` | array | No | Assistants this orchestrator can delegate to (see [Supporting Assistants](#supporting-assistants)) |
| `delegation_config` | object | No | Delegation behavior configuration (see [Supporting Assistants](#supporting-assistants)) |
| `allow_as_supporting` | boolean | No | Can this assistant be delegated to (default: `true`) |

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
    "system_prompt": "Updated instructions...",
    "llm_config": {
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
    "external_user_id": "user_123",
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
| `external_user_id` | string | Filter by user |
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

Returns `204 No Content` on success.

### Interrupt Room

Interrupt an active agent processing (e.g., during a long-running tool call or delegation):

```bash
curl -X POST https://core.interactor.com/api/v1/agents/rooms/room_xyz/interrupt \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"reason": "user_requested"}'
```

**Response:**
```json
{
  "interrupted": true,
  "delegation_id": "del_abc123",
  "resumable": true,
  "resume_timeout_at": "2026-01-20T12:30:00Z"
}
```

> **Note:** This endpoint returns a flat response (not wrapped in `data`).

### Delegation Status

Check if an agent room has an active delegation to another assistant:

```bash
curl https://core.interactor.com/api/v1/agents/rooms/room_xyz/delegation \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "status": "active",
    "active": true,
    "delegation_id": "del_abc123",
    "supporting_assistant": "asst_helper",
    "started_at": "2026-01-20T12:00:00Z",
    "resumable": true
  }
}
```

---

## Messages

### Send a Message

```bash
curl -X POST https://core.interactor.com/api/v1/agents/rooms/room_xyz/messages \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "How do I update my billing information?"
  }'
```

| Param | Required | Description |
|-------|----------|-------------|
| `content` | Yes | Message text |
| `external_user_id` | No | User identifier for context |
| `profile_context` | No | Per-message profile context |
| `enabled_tools` | No | **Strict allowlist** — see note below |
| `enabled_data_sources` | No | Override available data sources for this turn |

> **`enabled_tools` is a strict allowlist, including builtins.** When
> the field is omitted, every registered tool (custom + builtin) is
> available. When it's provided, ONLY tools whose names appear in the
> list are exposed to the assistant for that turn. To keep builtins
> like `get_current_time`, `find_capability`, or `update_state`
> available while narrowing your custom-tool surface, include their
> names explicitly:
>
> ```json
> "enabled_tools": [
>   "your_custom_tool",
>   "get_current_time",
>   "find_capability"
> ]
> ```
| `metadata` | No | Arbitrary metadata (default: `{}`) |

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
| `offset` | integer | Number of messages to skip (for pagination) |

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

Or via [Configuration Code Sync](02-setup-and-authentication.md#configuration-code-sync) (recommended):
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

### Input/Output Mapping (Optional)

Tools use a unified JSONata-based mapping system for workflow data:
- **Input mapping**: Extract workflow data and transform it for the callback
- **Output mapping**: Extract tool response data and merge it into workflow state

#### Input Mapping

When tools are executed within workflows, `input_mapping` specifies which workflow data to include in the callback:

```bash
curl -X POST https://core.interactor.com/api/v1/tools \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "apply_campaign_settings",
    "description": "Apply campaign settings to ad platform",
    "input_mapping": {
      "strategy": "context.strategy",
      "tactics.channels": "channels",
      "budget": "limits.budget"
    },
    "parameters": {...},
    "callback_url": "https://yourapp.com/api/tools/apply-settings"
  }'
```

When this tool is called from a workflow with data `{"strategy": {"objective": "awareness"}, "tactics": {"channels": ["display", "video"]}, "budget": 50000}`, the callback body will include:
```json
{
  "tool_name": "apply_campaign_settings",
  "arguments": {...},
  "context": {...},
  "credentials": {},
  "context": {
    "strategy": {"objective": "awareness"}
  },
  "channels": ["display", "video"],
  "limits": {"budget": 50000}
}
```

**Note:** If `input_mapping` is omitted, no workflow data is sent. The mapping allows you to rename and restructure data as needed.

#### Output Mapping

Map tool response data into workflow state using JSONata expressions:

```bash
curl -X POST https://core.interactor.com/api/v1/tools \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "get_campaign_metrics",
    "description": "Fetch campaign performance metrics",
    "output_mapping": {
      "metrics.ctr": "baseline_ctr",
      "metrics.impressions": "total_impressions"
    },
    "parameters": {...},
    "callback_url": "https://yourapp.com/api/tools/get-metrics"
  }'
```

When this tool returns:
```json
{"metrics": {"ctr": 0.025, "impressions": 10000, "clicks": 250}}
```

The workflow data will receive:
```json
{"baseline_ctr": 0.025, "total_impressions": 10000}
```

### Retry Configuration (Optional)

Tools can be configured to automatically retry on transient failures:

```bash
curl -X POST https://core.interactor.com/api/v1/tools \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "submit_order",
    "description": "Submit order to fulfillment system",
    "parameters": {...},
    "callback_url": "https://yourapp.com/api/tools/submit-order",
    "retry_config": {
      "enabled": true,
      "max_attempts": 3,
      "backoff_ms": [1000, 2000, 4000],
      "retry_on": ["5xx", "timeout", "connection_error"]
    }
  }'
```

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | boolean | Enable/disable retry (default: false) |
| `max_attempts` | integer | Maximum retry attempts (1-10, default: 3) |
| `backoff_ms` | array | Delay between retries in ms (e.g., `[1000, 2000, 4000]`) |
| `retry_on` | array | Error types to retry: `5xx`, `4xx`, `timeout`, `connection_error` |

When retries are enabled, Interactor sends the same `x-idempotency-key` header across all retry attempts. Use this key to detect duplicate requests and ensure idempotent handling.

**Circuit Breaker Protection**

Interactor automatically tracks failures per callback URL domain. After 5 consecutive failures, the circuit opens and requests fail immediately with a `circuit_open` error. The circuit closes after 30 seconds if a test request succeeds.

### Tool Callback

When the assistant invokes your tool, Interactor POSTs to your `callback_url` with these headers:

| Header | Description |
|--------|-------------|
| `x-interactor-signature` | HMAC-SHA256 signature (hex encoded) |
| `x-interactor-timestamp` | Unix timestamp when request was signed |
| `x-interactor-tool` | Name of the tool being executed |
| `x-interactor-account` | Your account ID |
| `x-idempotency-key` | UUID for safe retries (same key used across retries) |

**Request body:**
```json
{
  "tool_name": "search_products",
  "arguments": {"query": "laptop", "category": "electronics"},
  "context": {
    "account_id": "acc_abc123",
    "external_user_id": "user_456"
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
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "force": false,
    "include_analysis": true
  }'
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `force` | boolean | `false` | Bypass 5-minute cooldown between refreshes |
| `include_analysis` | boolean | `true` | Include AI-assisted schema enrichment (descriptions, aliases) |

**Response:**
```json
{
  "data": {
    "refresh_id": "uuid",
    "status": "completed",
    "previous_snapshot_at": "2026-01-07T02:00:00Z",
    "current_snapshot_at": "2026-02-02T14:30:00Z",
    "changes": {
      "severity": "additive",
      "tables_added": ["campaign_analytics"],
      "tables_removed": [],
      "column_changes": {}
    },
    "actions_taken": [
      "Extracted schema from database",
      "Detected 1 new table(s)",
      "Generated AI-assisted descriptions"
    ],
    "duration_ms": 1250
  }
}
```

Change severity levels:
- `none` - No schema changes detected
- `additive` - New tables or columns added
- `modification` - Column types or constraints changed
- `breaking` - Tables or columns removed

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

### Adaptive Learning (Learned Mappings)

The system automatically learns which semantic mappings work well based on query success/failure. Learned mappings are scoped hierarchically: **User → Account → Data Source**.

Mappings include multi-dimensional scoring:
- **`confidence`**: Stored score (legacy, for backward compatibility)
- **`effective_score`**: Query-time computed score including source weight, feedback boost, recency, and scope weight (preferred for ranking)
- **`source`**: How the mapping was discovered (`specification`, `admin_defined`, `documentation`, `ai_inferred`, `usage_learned`)

**Get Learned Mappings:**

Retrieve all learned mappings for a user (merged from all applicable scopes):

```bash
curl "https://core.interactor.com/api/v1/data-sources/ds_abc/learned-mappings?account_id=acc_123&external_user_id=user_456" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": [
    {
      "id": "lsm_abc",
      "term": "active customers",
      "expression": "customers WHERE status = 'active'",
      "scope_type": "user",
      "source": "usage_learned",
      "confidence": 0.89,
      "effective_score": 0.9234,
      "success_count": 15,
      "failure_count": 2,
      "selection_count": 5,
      "last_used_at": "2026-02-06T10:30:00Z",
      "last_selected_at": "2026-02-06T09:15:00Z",
      "dismissed": false
    }
  ],
  "meta": {
    "count": 1,
    "scopes_included": ["data_source", "account", "user"]
  }
}
```

**Get Promotion Candidates (Admin):**

Find high-confidence mappings that multiple users have learned (candidates for promotion to account scope):

```bash
curl "https://core.interactor.com/api/v1/data-sources/ds_abc/learned-mappings/candidates?account_id=acc_123&min_confidence=0.7&min_users=2" \
  -H "Authorization: Bearer <token>"
```

**Promote a Mapping:**

Promote a user-scoped mapping to account scope (or account to data_source):

```bash
curl -X POST https://core.interactor.com/api/v1/data-sources/ds_abc/learned-mappings/lsm_abc/promote \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "to_scope": "account",
    "promoted_by": "admin_external_user_id"
  }'
```

**Dismiss a Mapping:**

Mark a mapping as dismissed (won't be suggested again for this user):

```bash
curl -X POST https://core.interactor.com/api/v1/data-sources/ds_abc/learned-mappings/lsm_abc/dismiss \
  -H "Authorization: Bearer <token>"
```

**Record Selection (Feedback):**

When a user selects/confirms a mapping is correct, record the feedback to boost its `effective_score`:

```bash
curl -X POST https://core.interactor.com/api/v1/data-sources/ds_abc/learned-mappings/lsm_abc/select \
  -H "Authorization: Bearer <token>"
```

**Create Admin Mapping:**

Create an admin-defined mapping (higher source weight than usage-learned):

```bash
curl -X POST https://core.interactor.com/api/v1/data-sources/ds_abc/learned-mappings \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "term": "quarterly revenue",
    "expression": "SUM(total) WHERE quarter = CURRENT_QUARTER",
    "scope_type": "data_source",
    "source": "admin_defined"
  }'
```

For account or user scope, include `account_id` (and `external_user_id` for user scope).

---

## Service Knowledge Base Search

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

## User Profiles

User profiles allow you to store user preferences, context, and instructions that follow users across all their interactions with assistants.

### Overview

Profiles exist at three levels with inheritance:

| Scope | Purpose | When to Use |
|-------|---------|-------------|
| **Account Default** | Organization-wide defaults | Global settings for all users |
| **Context** | Team/project/region settings | Different behavior per department or use case |
| **User** | Individual preferences | Personal customizations |

When a user sends a message, profiles are merged in order: **Default → Context → User** (later layers override earlier ones).

### Profile Structure

```json
{
  "preferences": {
    "units": "metric",
    "verbosity": "brief",
    "timezone": "America/New_York",
    "locale": "en-US",
    "date_format": "ISO",
    "currency": "USD"
  },
  "context": {
    "department": "Engineering",
    "role": "Senior Developer",
    "team": "Platform"
  },
  "instructions": [
    "Always be concise and technical",
    "Prefer code examples over prose"
  ],
  "memory": {
    "facts": [
      {"key": "project", "value": "Project Atlas", "learned_at": "2026-01-15"},
      {"key": "preference", "value": "prefers dark mode"}
    ]
  }
}
```

**Validated Preference Values:**

| Field | Valid Values |
|-------|--------------|
| `units` | `metric`, `imperial` |
| `verbosity` | `brief`, `normal`, `detailed` |
| `date_format` | `ISO`, `US`, `EU` |

Other preferences (`timezone`, `locale`, `currency`) are freeform strings.

**Size Limit:** 64KB per profile

### Account Default Profile

```bash
# Get account default
curl https://core.interactor.com/api/v1/profiles/default \
  -H "Authorization: Bearer <token>"

# Set account default
curl -X PUT https://core.interactor.com/api/v1/profiles/default \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "preferences": {
        "verbosity": "normal",
        "units": "metric"
      },
      "instructions": [
        "Be helpful and professional"
      ]
    }
  }'
```

**Response:**
```json
{
  "data": {
    "profile": {
      "preferences": {"verbosity": "normal", "units": "metric"},
      "instructions": ["Be helpful and professional"]
    },
    "metadata": {
      "scope": "default",
      "scope_id": null,
      "updated_at": "2026-01-20T12:00:00Z"
    }
  }
}
```

### Context Profiles

Context profiles provide an optional middle layer for teams, projects, or regions.

```bash
# List all context profiles
curl https://core.interactor.com/api/v1/profiles/context \
  -H "Authorization: Bearer <token>"

# Get specific context profile
curl https://core.interactor.com/api/v1/profiles/context/team-engineering \
  -H "Authorization: Bearer <token>"

# Create/update context profile
curl -X PUT https://core.interactor.com/api/v1/profiles/context/team-engineering \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "context": {
        "team": "Engineering",
        "department": "Product"
      },
      "instructions": [
        "Focus on technical accuracy",
        "Reference internal documentation when applicable"
      ]
    }
  }'

# Delete context profile
curl -X DELETE https://core.interactor.com/api/v1/profiles/context/team-engineering \
  -H "Authorization: Bearer <token>"
```

### User Profiles

```bash
# List user profiles (paginated)
curl "https://core.interactor.com/api/v1/profiles/user?limit=20&offset=0" \
  -H "Authorization: Bearer <token>"

# Get specific user profile
curl https://core.interactor.com/api/v1/profiles/user/user_123 \
  -H "Authorization: Bearer <token>"

# Create/update user profile
curl -X PUT https://core.interactor.com/api/v1/profiles/user/user_123 \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "preferences": {
        "verbosity": "brief",
        "units": "imperial"
      },
      "memory": {
        "facts": [
          {"key": "role", "value": "Product Manager"},
          {"key": "project", "value": "Q1 Launch"}
        ]
      }
    }
  }'

# Delete user profile
curl -X DELETE https://core.interactor.com/api/v1/profiles/user/user_123 \
  -H "Authorization: Bearer <token>"
```

**Pagination Response:**
```json
{
  "data": [
    {"profile": {...}, "metadata": {...}},
    {"profile": {...}, "metadata": {...}}
  ],
  "pagination": {
    "total": 150,
    "limit": 20,
    "offset": 0
  }
}
```

### Get Effective (Merged) Profile

Get the fully merged profile for a user, combining all applicable layers:

```bash
# Without context
curl https://core.interactor.com/api/v1/profiles/effective/user_123 \
  -H "Authorization: Bearer <token>"

# With context layer
curl "https://core.interactor.com/api/v1/profiles/effective/user_123?context=team-engineering" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "profile": {
      "preferences": {"verbosity": "brief", "units": "imperial"},
      "context": {"team": "Engineering", "department": "Product"},
      "instructions": [
        "Be helpful and professional",
        "Focus on technical accuracy"
      ],
      "memory": {
        "facts": [{"key": "role", "value": "Product Manager"}]
      }
    },
    "metadata": {
      "user_id": "user_123",
      "context_id": "team-engineering",
      "merged_at": "2026-01-20T12:05:00Z"
    }
  }
}
```

### Using Profiles with Messages

Pass profile context when sending messages to load the appropriate profile:

```bash
curl -X POST https://core.interactor.com/api/v1/agents/rooms/room_xyz/messages \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "What time is our standup?",
    "external_user_id": "user_123",
    "profile_context": "team-engineering"
  }'
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `external_user_id` | string | User identifier for profile lookup |
| `profile_context` | string | Optional context_id for 3-layer merge |

When `external_user_id` is provided:
1. User profile is auto-created if it doesn't exist
2. Effective profile is loaded (merging default → context → user)
3. Profile data is included in the assistant's system prompt

### Built-in Profile Tools

Assistants have access to profile management tools that let users view and update their preferences during conversations.

| Tool | Description |
|------|-------------|
| `get_my_profile` | Returns the user's profile data |
| `update_my_profile` | Updates specific fields in the user's profile |
| `get_effective_profile` | Returns the fully merged profile |
| `suggest_memory_update` | Suggests a memory fact for the solution app to review |

**Disabling Profile Tools:**

You can disable profile tools for specific assistants:

```bash
curl -X PUT https://core.interactor.com/api/v1/agents/assistants/asst_abc \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "builtin_tools": {
      "profile_management": false
    }
  }'
```

**Example Conversation:**

```
User: "Update my preferences to use metric units"
Assistant: [Uses update_my_profile tool]
  I have updated your preferences to use metric units for measurements.

User: "What are my current settings?"
Assistant: [Uses get_my_profile tool]
  Your current profile settings are:
  - Units: metric
  - Verbosity: brief
  ...
```

---

## Supporting Assistants

Supporting assistants enable a modular orchestration pattern where a primary ("orchestrator") assistant can delegate tasks to specialized assistants.

### Overview

```
┌─────────────────────┐
│  Primary Assistant  │
│  (General Support)  │
└─────────┬───────────┘
          │ delegates specialized tasks
    ┌─────┴─────┬─────────────┐
    ▼           ▼             ▼
┌─────────┐ ┌─────────┐ ┌───────────┐
│Billing  │ │Technical│ │Shipping   │
│Assistant│ │Support  │ │Tracker    │
└─────────┘ └─────────┘ └───────────┘
```

### Configuring an Orchestrator

To configure an assistant as an orchestrator, add the `supporting_assistants` array:

```bash
curl -X PUT https://core.interactor.com/api/v1/agents/assistants/asst_primary \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "supporting_assistants": [
      {
        "assistant_id": "asst_billing",
        "description": "Handles billing questions, refunds, and payment issues"
      },
      {
        "assistant_id": "asst_technical",
        "description": "Handles technical troubleshooting and product issues"
      }
    ],
    "delegation_config": {
      "delegation_timeout_ms": 60000,
      "credential_inheritance": "scoped"
    }
  }'
```

The orchestrator's system prompt automatically receives information about available supporting assistants, and the orchestrator can choose to delegate tasks using the built-in `delegate_to_assistant` tool.

### Configuration Options

**Assistant Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `supporting_assistants` | array | List of assistants this orchestrator can delegate to |
| `delegation_config` | object | Delegation behavior settings |
| `allow_as_supporting` | boolean | Whether this assistant can be delegated to (default: `true`) |

**supporting_assistants Array:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `assistant_id` | string | Yes | ID of the supporting assistant |
| `description` | string | Yes | What this assistant handles (shown to orchestrator) |

**delegation_config Object:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `delegation_timeout_ms` | integer | 60000 | Timeout for delegation completion |
| `timeout_grace_ms` | integer | 30000 | Grace period after timeout before termination |
| `resume_timeout_ms` | integer | 300000 | Time allowed to resume a paused delegation |
| `credential_inheritance` | string | `"scoped"` | How credentials pass: `all`, `scoped`, or `none` |

**Credential Inheritance Modes:**

| Mode | Description |
|------|-------------|
| `all` | Supporting assistant inherits all user credentials |
| `scoped` | Only credentials for services the supporting assistant's tools require |
| `none` | No credential inheritance - uses assistant-level credentials only |

### Configuring a Supporting Assistant

Supporting assistants work like regular assistants, but set `allow_as_supporting: true` (the default):

```bash
curl -X POST https://core.interactor.com/api/v1/agents/assistants \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "billing_assistant",
    "description": "Specialized billing and refund assistant",
    "system_prompt": "You are a billing specialist. Help with refunds, payments, and invoices.",
    "default_tools": ["get_invoice", "process_refund", "update_payment_method"],
    "allow_as_supporting": true
  }'
```

To prevent an assistant from being used as a supporting assistant:

```bash
curl -X PUT https://core.interactor.com/api/v1/agents/assistants/asst_xyz \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"allow_as_supporting": false}'
```

### Message Attribution

Messages from delegated tasks include a `delegation_context` field indicating their origin:

```json
{
  "id": "msg_123",
  "role": "assistant",
  "content": "I've processed your refund of $50.00.",
  "delegation_context": {
    "delegation_id": "del_abc123",
    "parent_assistant_id": "asst_primary",
    "parent_assistant_name": "general_support",
    "supporting_assistant_id": "asst_billing",
    "supporting_assistant_name": "billing_assistant"
  }
}
```

Use this to display attribution in your UI, such as "Billing Assistant (via General Support)".

### Client Control API

Clients can monitor and control delegations via REST endpoints.

**Get Delegation Status:**

```bash
curl https://core.interactor.com/api/v1/agents/rooms/room_xyz/delegation \
  -H "Authorization: Bearer <token>"
```

**Response (when active):**
```json
{
  "data": {
    "status": "active",
    "active": true,
    "delegation_id": "del_abc123",
    "supporting_assistant": {
      "id": "asst_billing",
      "name": "billing_assistant"
    },
    "started_at": "2026-02-05T12:00:00Z",
    "task": "Process refund for order #123"
  }
}
```

**Response (when paused):**
```json
{
  "data": {
    "status": "paused",
    "active": false,
    "resumable": true,
    "delegation_id": "del_abc123",
    "supporting_assistant": {
      "id": "asst_billing",
      "name": "billing_assistant"
    },
    "paused_at": "2026-02-05T12:01:00Z",
    "resume_timeout_at": "2026-02-05T12:06:00Z",
    "interrupt_reason": "user_requested"
  }
}
```

**Interrupt Active Delegation:**

```bash
curl -X POST https://core.interactor.com/api/v1/agents/rooms/room_xyz/interrupt \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"reason": "user_requested"}'
```

**Response:**
```json
{
  "interrupted": true,
  "delegation_id": "del_abc123",
  "resumable": true,
  "resume_timeout_at": "2026-02-05T12:06:00Z"
}
```

The orchestrator can resume paused delegations in subsequent conversation turns.

### Delegation Events (SSE)

When subscribed to the room's SSE stream (`GET /rooms/:id/stream`), clients receive delegation lifecycle events:

| Event | Payload | Description |
|-------|---------|-------------|
| `delegation.started` | `{delegation_id, supporting_assistant}` | Delegation began |
| `delegation.completed` | `{delegation_id, status}` | Delegation finished successfully |
| `delegation.paused` | `{delegation_id, reason, resumable, resume_timeout_at}` | Delegation paused (timeout or interrupted) |
| `delegation.resumed` | `{delegation_id}` | Paused delegation resumed |
| `delegation.timeout` | `{delegation_id, reason}` | Delegation timed out |
| `delegation.expired` | `{delegation_id, reason}` | Paused delegation expired (not resumed in time) |

Use these events to update your UI in real-time:
- Show "Connecting to Billing Assistant..." on `delegation.started`
- Show "Paused - tap to resume" on `delegation.paused`
- Clear delegation indicator on `delegation.completed`

---

## Best Practices

1. **Keep instructions focused** - Clear, specific instructions produce better results
2. **Use semantic mappings** - Help assistants understand your data schema
3. **Secure tool callbacks** - Always verify signatures on tool callbacks
4. **Use read-only database users** - Limit data source connections to read-only access
5. **Monitor tool usage** - Track which tools are being called and how often
6. **Test conversations** - Verify assistant behavior before deploying to users
7. **Use profiles for personalization** - Store user preferences in profiles rather than room metadata
8. **Use supporting assistants for modularity** - Specialized assistants for specific domains improve reliability

---

## Webhook Events

Subscribe to agent events:

| Event | Description |
|-------|-------------|
| `agent.room_created` | A new room was created |
| `agent.message_received` | A user message was received in a room |
| `agent.response_sent` | The assistant sent a response |
| `agent.action_executed` | The assistant invoked a tool / executed an action |
| `agent.error` | An error occurred during agent processing |

See [Webhooks and Streaming](06-webhooks-and-streaming.md) for details on payload shapes and signature verification.

---

## Troubleshooting

### Getting 401 Unauthorized on Agent API Calls

This means your token is invalid, expired, or missing. Check:

| Issue | Solution |
|-------|----------|
| No token provided | Add `Authorization: Bearer <token>` header to all requests |
| Token expired | Tokens expire after 15 minutes - implement token refresh (see [Token Caching Strategy](02-setup-and-authentication.md#token-management)) |
| Wrong token type | Use the `access_token` from OAuth client credentials, not a user login token |
| Invalid credentials | Verify your `client_id` and `client_secret` are correct |

**Debug Steps:**

```bash
# 1. Get a fresh token
TOKEN=$(curl -s -X POST https://auth.interactor.com/api/v1/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET"
  }' | jq -r '.data.access_token')

# 2. Verify token is not empty
echo "Token: $TOKEN"

# 3. Test with the token
curl -i https://core.interactor.com/api/v1/agents/assistants \
  -H "Authorization: Bearer $TOKEN"
```

### Agent Operations Work Locally But Not in Production

Ensure your production environment has access to the token:

```typescript
// Common mistake: Token not passed through the call chain
class AgentService {
  // ❌ Bad: Token hardcoded or missing
  async createAgent(name: string) {
    return fetch('/api/v1/agents/assistants', {
      method: 'POST',
      body: JSON.stringify({ name })
      // Missing Authorization header!
    });
  }

  // ✅ Good: Token passed explicitly
  async createAgent(name: string, accessToken: string) {
    return fetch('https://core.interactor.com/api/v1/agents/assistants', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ name })
    });
  }
}
```

### Integration Checklist

Before deploying your Interactor integration, verify:

- [ ] OAuth client credentials are stored securely (environment variables, not code)
- [ ] Token exchange is working (`POST /api/v1/oauth/token` returns `access_token`)
- [ ] Token refresh logic is implemented (tokens expire after 15 minutes)
- [ ] **All API calls include the `Authorization: Bearer <token>` header**
- [ ] Error handling for 401 responses triggers token refresh
- [ ] Token is passed through your entire call chain (controllers → services → API client)

---

## Next Steps

- [Workflows](05-workflows.md) - Combine AI agents with automated workflows
- [Webhooks and Streaming](06-webhooks-and-streaming.md) - Real-time message streaming
- [SDK Examples](07-sdk-examples.md) - Complete code examples

