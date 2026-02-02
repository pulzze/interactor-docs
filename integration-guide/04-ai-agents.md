# AI Agents

**Last Updated:** 2026-02-02

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
    "description": "Helps users with support questions",
    "system_prompt": "You are a helpful support assistant. Be concise and friendly.",
    "llm_provider": "anthropic",
    "llm_model": "claude-sonnet-4-20250514",
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
| `llm_provider` | string | No | `anthropic` or `openai` (default: `openai`) |
| `llm_model` | string | No | Model identifier |
| `llm_config` | object | No | LLM settings (e.g., `{"temperature": 0.7}`) |
| `default_tools` | array | No | Tool IDs the assistant can use |
| `default_data_sources` | array | No | Data source IDs the assistant can query |
| `max_tool_calls_per_turn` | integer | No | Limit tool calls per conversation turn |
| `session_timeout_minutes` | integer | No | Auto-close rooms after inactivity |
| `active` | boolean | No | Whether the assistant is enabled (default: `true`) |
| `metadata` | object | No | Custom data to store with the assistant |
| `builtin_tools` | object | No | Configure built-in tool availability (see User Profiles) |

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
    "user_ref": "user_123",
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
| `user_ref` | string | Filter by user reference |
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

Tools use a unified JSONPath-based mapping system for workflow data:
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
      "$.strategy": "$.context.strategy",
      "$.tactics.channels": "$.channels",
      "$.budget": "$.limits.budget"
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

Map tool response data into workflow state using JSONPath expressions:

```bash
curl -X POST https://core.interactor.com/api/v1/tools \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "get_campaign_metrics",
    "description": "Fetch campaign performance metrics",
    "output_mapping": {
      "$.metrics.ctr": "$.baseline_ctr",
      "$.metrics.impressions": "$.total_impressions"
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
    "role": "user",
    "user_ref": "user_123",
    "profile_context": "team-engineering"
  }'
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_ref` | string | User identifier for profile lookup |
| `profile_context` | string | Optional context_id for 3-layer merge |

When `user_ref` is provided:
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

## Best Practices

1. **Keep instructions focused** - Clear, specific instructions produce better results
2. **Use semantic mappings** - Help assistants understand your data schema
3. **Secure tool callbacks** - Always verify signatures on tool callbacks
4. **Use read-only database users** - Limit data source connections to read-only access
5. **Monitor tool usage** - Track which tools are being called and how often
6. **Test conversations** - Verify assistant behavior before deploying to users
7. **Use profiles for personalization** - Store user preferences in profiles rather than room metadata

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

