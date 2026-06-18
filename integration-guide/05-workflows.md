# Workflows

_Last verified: 2026-06-18_

**Last Updated:** 2026-05-24

Workflows are state-machine based automations with human-in-the-loop support. Use them to model multi-step business processes that may require approvals, user input, or external integrations.

---

## Overview

Workflows consist of:

- **States** - Steps in your process (action, halting, terminal)
- **Transitions** - Rules for moving between states
- **Instances** - Running executions of a workflow
- **Threads** - Parallel execution paths within an instance

---

## Data Model

Workflows have a clear separation between input and runtime data:

| Concept | Access Pattern | Description |
|---------|----------------|-------------|
| `input` | `{{ input.order_id }}` | Immutable values passed at instance creation |
| `data` | `{{ data.status }}` | Mutable runtime data accumulated during execution |

### Input vs Data

- **`input`** - Values provided when creating an instance. These are immutable and cannot be modified by workflow steps.
- **`data`** - Runtime state that accumulates as the workflow executes. Steps can read and write to `data`.

### Input Schema

Workflows can define an `input_schema` (JSON Schema) to validate input at instance creation:

```json
{
  "name": "approval_workflow",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string" },
      "amount": { "type": "number", "minimum": 0 }
    },
    "required": ["order_id", "amount"]
  },
  "initial_state": "review",
  "states": { ... }
}
```

### Setting Data

Use the `set` step with explicit `data.` prefix:

```json
{
  "type": "set",
  "values": {
    "data.status": "processing",
    "data.requires_approval": "{{ input.amount > 10000 }}"
  }
}
```

Attempting to write to `input` returns an `INPUT_IMMUTABLE` error.

---

## Workflow Definition

### Create a Workflow

```bash
curl -X POST https://core.interactor.com/api/v1/workflows \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "approval_workflow",
    "initial_state": "request",
    "ai_guidance": "This workflow handles approval requests",
    "input_schema": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "amount": { "type": "number" },
        "requester": { "type": "string", "format": "email" }
      },
      "required": ["id", "amount"]
    },
    "states": {
      "request": {
        "type": "action",
        "logic": [
          {"type": "set", "values": {"data.request_id": "{{ input.id }}", "data.status": "pending"}}
        ],
        "transitions": [{"to": "await_approval"}]
      },
      "await_approval": {
        "type": "halting",
        "presentation": {
          "type": "form",
          "fields": [
            {"name": "approved", "type": "boolean", "label": "Approve?"},
            {"name": "comment", "type": "string", "label": "Comment"}
          ]
        },
        "transitions": [
          {"to": "approved", "condition": "data.approved"},
          {"to": "rejected"}
        ]
      },
      "approved": {
        "type": "terminal"
      },
      "rejected": {
        "type": "terminal"
      }
    }
  }'
```

**Response:**
```json
{
  "data": {
    "name": "approval_workflow",
    "version_id": "v_abc123",
    "status": "draft",
    "created_at": "2026-01-20T12:00:00Z"
  }
}
```

### State Types

| Type | Description |
|------|-------------|
| `action` | Executes logic automatically, then transitions |
| `halting` | Pauses execution, waits for external input |
| `terminal` | End state, workflow completes |

### Validate Without Saving

Test a workflow definition before creating it:

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/validate \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my_workflow",
    "initial_state": "start",
    "states": {
      "start": {
        "type": "action",
        "logic": [{"type": "set", "values": {"data.message": "Hello"}}],
        "transitions": [{"to": "end"}]
      },
      "end": {
        "type": "terminal"
      }
    }
  }'
```

### List Workflows

```bash
curl https://core.interactor.com/api/v1/workflows \
  -H "Authorization: Bearer <token>"
```

### List Versions

```bash
curl https://core.interactor.com/api/v1/workflows/approval_workflow/versions \
  -H "Authorization: Bearer <token>"
```

### Publish a Version

Workflows must be published before they can be executed:

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/approval_workflow/versions/v_abc123/publish \
  -H "Authorization: Bearer <token>"
```

---

## Workflow Instances

Instances are running executions of a workflow.

### Create Instance

Start a new workflow execution:

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/approval_workflow/instances \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "external_user_id": "user_123",
    "input": {
      "id": "req_456",
      "amount": 5000,
      "requester": "john@example.com"
    }
  }'
```

**Response:**
```json
{
  "data": {
    "id": "inst_xyz",
    "workflow_name": "approval_workflow",
    "status": "running",
    "current_state": "await_approval",
    "created_at": "2026-01-20T12:00:00Z"
  }
}
```

### List Instances

```bash
curl https://core.interactor.com/api/v1/workflows/instances \
  -H "Authorization: Bearer <token>"
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `external_user_id` | string | Filter by user |
| `workflow_name` | string | Filter by workflow |
| `status` | string | `running`, `halted`, `completed`, `failed`, `cancelled` |

### Get Instance

```bash
curl https://core.interactor.com/api/v1/workflows/instances/inst_xyz \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "id": "inst_xyz",
    "workflow_name": "approval_workflow",
    "status": "halted",
    "current_state": "await_approval",
    "input": {
      "id": "req_456",
      "amount": 5000,
      "requester": "john@example.com"
    },
    "data": {
      "request_id": "req_456",
      "status": "pending"
    },
    "halting_presentation": {
      "type": "form",
      "fields": [
        {"name": "approved", "type": "boolean", "label": "Approve?"},
        {"name": "comment", "type": "string", "label": "Comment"}
      ]
    },
    "threads": [...],
    "history": [...]
  }
}
```

### Instance Status Values

| Status | Description |
|--------|-------------|
| `running` | Actively executing |
| `halted` | Paused, waiting for input |
| `completed` | Finished successfully |
| `failed` | Terminated due to error |
| `cancelled` | Manually cancelled |

---

## Resuming Workflows

When a workflow reaches a halting state, it waits for external input.

### Resume with Selected Option

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/instances/inst_xyz/resume \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "selected_option": "approve"
  }'
```

The `selected_option` field is required and must match one of the options presented in the halting state's `halting_presentation`. The workflow continues execution based on the selected option and transition conditions.

### Cancel Instance

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/instances/inst_xyz/cancel \
  -H "Authorization: Bearer <token>"
```

---

## Threads

Workflows can have parallel execution paths (threads).

### List Threads

```bash
curl https://core.interactor.com/api/v1/workflows/instances/inst_xyz/threads \
  -H "Authorization: Bearer <token>"
```

### Resume Specific Thread

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/instances/inst_xyz/threads/thread_1/resume \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "selected_option": "approve",
    "user_input": {}
  }'
```

---

## History API

Track and debug workflow execution with the History API. Every state transition, step execution, and significant event is recorded.

### List History Events

```bash
curl https://core.interactor.com/api/v1/workflows/instances/inst_xyz/history \
  -H "Authorization: Bearer <token>"
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Max events to return (default: 100, max: 1000) |
| `cursor` | string | Pagination cursor from previous response |
| `types` | string | Filter by event type (comma-separated) |
| `since` | ISO8601 | Only events after this timestamp |
| `until` | ISO8601 | Only events before this timestamp |
| `thread` | string | Filter to specific thread |
| `state` | string | Filter to events in a specific state |
| `include_data` | boolean | Include workflow_data snapshots (default: false) |

**Response:**
```json
{
  "data": {
    "instance_id": "inst_xyz",
    "workflow_id": "wf_abc",
    "events": [
      {
        "id": "evt_01HX...",
        "type": "lifecycle",
        "subtype": "created",
        "timestamp": "2026-01-20T12:00:00Z",
        "initial_state": "request",
        "started_by": "user_123"
      },
      {
        "id": "evt_01HX...",
        "type": "transition",
        "subtype": "state_change",
        "timestamp": "2026-01-20T12:00:01Z",
        "from_state": "request",
        "to_state": "await_approval",
        "trigger": "automatic"
      },
      {
        "id": "evt_01HX...",
        "type": "halt",
        "subtype": "waiting",
        "timestamp": "2026-01-20T12:00:01Z",
        "state": "await_approval",
        "message": "Waiting for manager approval"
      }
    ],
    "pagination": {
      "has_more": false,
      "next_cursor": null
    }
  }
}
```

### Get Single Event

```bash
curl https://core.interactor.com/api/v1/workflows/instances/inst_xyz/events/evt_01HX... \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "id": "evt_01HX...",
    "type": "step",
    "subtype": "completed",
    "timestamp": "2026-01-20T12:00:01Z",
    "state": "processing",
    "step_name": "api_call",
    "step_type": "http",
    "duration_ms": 250,
    "changes": {
      "added": {"api_response": {"status": "ok"}},
      "updated": {"status": {"from": "pending", "to": "processed"}}
    }
  }
}
```

### Event Types

| Type | Subtypes | Description |
|------|----------|-------------|
| `lifecycle` | `created`, `completed`, `failed`, `cancelled` | Instance lifecycle events |
| `transition` | `state_change`, `fork`, `join` | State transitions and thread operations |
| `step` | `completed`, `failed` | Step execution results |
| `halt` | `waiting` | Workflow paused for input |
| `error` | `step_failed`, `instance_failed` | Non-retryable failures |

### Thread Status

For workflows with parallel execution:

```bash
curl https://core.interactor.com/api/v1/workflows/instances/inst_xyz/threads \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "instance_id": "inst_xyz",
    "threads": [
      {
        "thread_id": "main",
        "status": "completed",
        "current_state": "merge",
        "event_count": 5
      },
      {
        "thread_id": "payment",
        "status": "completed",
        "current_state": "payment_done",
        "parent_thread_id": "main",
        "event_count": 3
      }
    ],
    "summary": {
      "total": 2,
      "by_status": {"completed": 2}
    }
  }
}
```

### Error Aggregation

Query errors across all workflows:

```bash
curl "https://core.interactor.com/api/v1/workflows/errors?since=2026-01-20T00:00:00Z" \
  -H "Authorization: Bearer <token>"
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `since` | ISO8601 | **Required.** Start of time range |
| `until` | ISO8601 | End of time range (default: now) |
| `workflow_name` | string | Filter by workflow |
| `error_code` | string | Filter by error code |

**Response:**
```json
{
  "data": {
    "errors": [
      {
        "error_id": "err_abc",
        "instance_id": "inst_xyz",
        "workflow_name": "approval_workflow",
        "error_code": "HTTP_TIMEOUT",
        "message": "Request timed out",
        "timestamp": "2026-01-20T12:00:00Z"
      }
    ],
    "summary": {
      "total": 1,
      "by_error_code": {"HTTP_TIMEOUT": 1},
      "by_workflow": {"approval_workflow": 1}
    }
  }
}
```

---

## Halting Instructions

When a workflow halts, you can configure how the halting message is generated and presented to users using `halting_instructions`.

### AI-Generated Instructions

Use AI to dynamically generate contextual messages based on workflow data:

```json
{
  "await_approval": {
    "type": "halting",
    "halting_instructions": {
      "type": "ai",
      "config": {
        "prompt": "Summarize this order and ask the user to approve or reject it.",
        "model": "gpt-4o-mini",
        "include_data_paths": ["order", "customer", "risk_score"]
      }
    },
    "transition_mode": "selection",
    "transitions": [
      {"key": "approve", "to": "approved", "description": "Approve the order"},
      {"key": "reject", "to": "rejected", "description": "Reject the order"}
    ]
  }
}
```

**Simple AI format** - treats `instruction` as an AI prompt:

```json
{
  "halting_instructions": {
    "instruction": "Tell the user the strategy is ready for review. Highlight key metrics.",
    "include_data": ["strategy", "benchmarks"]
  }
}
```

### Static Message

For simple static messages without AI generation:

```json
{
  "halting_instructions": {
    "type": "message",
    "config": {
      "title": "Approval Required",
      "message": "This order exceeds the automatic approval threshold."
    }
  }
}
```

### Halted Response Format

When an instance is halted, the API response includes `halted_options`:

```json
{
  "status": "halted",
  "halted_at_state": "await_approval",
  "halted_options": {
    "instruction": "Order #123 for $150.00 from Acme Corp is ready for approval. Risk score: Low.",
    "include_data": ["order", "customer"],
    "transition_mode": "selection",
    "choices": [
      {"key": "approve", "description": "Approve the order", "to": "approved"},
      {"key": "reject", "description": "Reject the order", "to": "rejected"}
    ],
    "generated": true
  }
}
```

| Field | Description |
|-------|-------------|
| `instruction` | Message to display (AI-generated or static) |
| `include_data` | Workflow data paths included in context |
| `choices` | Available transitions for selection mode |
| `generated` | `true` if AI-generated, `false` if static |

### Legacy Presentation Format

For backward compatibility, form-style presentations are still supported:

```json
{
  "presentation": {
    "type": "form",
    "fields": [
      {"name": "approved", "type": "boolean", "label": "Approve this request?"},
      {"name": "notes", "type": "string", "label": "Notes", "multiline": true}
    ]
  }
}
```

---

## Workflow Logic

### Set Logic

Set values in workflow data using the `data.` prefix:

```json
{
  "type": "set",
  "values": {
    "data.total": "{{ $sum(data.items.price) }}",
    "data.approved": "{{ $sum(data.items.price) < 1000 }}"
  }
}
```

**Note:** All set paths must start with `data.`. Attempting to set `input.*` returns `INPUT_IMMUTABLE` error.

### HTTP Logic

Make external API calls:

```json
{
  "type": "http",
  "method": "POST",
  "url": "https://api.yourservice.com/orders/{{ input.order_id }}",
  "headers": {
    "Authorization": "Bearer {{ input.api_key }}",
    "Content-Type": "application/json"
  },
  "body": {
    "status": "{{ data.status }}",
    "processed_at": "{{ data.processed_at }}"
  }
}
```

Template expressions use `{{ }}` delimiters to reference `input` (immutable) and `data` (runtime state). See [Expressions](08-expressions.md) for the full expression language.

### Transition Conditions

Conditions use JSONata expressions to control state transitions. You can reference both `input` (immutable) and `data` (runtime):

```json
{
  "transitions": [
    {
      "to": "high_value_approval",
      "condition": "input.amount > 10000"
    },
    {
      "to": "auto_approve",
      "condition": "input.amount <= 10000"
    }
  ]
}
```

Transitions are evaluated in order. The first matching condition wins. A transition without a `condition` always matches (use as a default/fallback).

See [Expressions](08-expressions.md) for the full expression language.

---

## Step Output Configuration

Control how step results are stored in workflow data.

### Basic Output

```json
{
  "logic": {"type": "http", "config": {...}},
  "output": "api_result"
}
```

The entire step result is stored at `data.api_result`.

### Nested Path

```json
{
  "output": {"path": "order.validation", "mode": "merge"}
}
```

Result is stored at `data.order.validation`. With `mode: "merge"`, existing fields are preserved.

### Field Extraction

Extract specific fields from the step result:

```json
{
  "output": {
    "path": "review_result",
    "extract": {
      "decision": "outputs.approval_status",
      "total": "body.data.total"
    }
  }
}
```

### JSONata Expressions

For complex data extraction and transformation, use JSONata expressions:

```json
{
  "output": {
    "path": "filtered_data",
    "extract": {
      "all_ids": "items.id",
      "active_users": "users[active = true].name",
      "first_item": "items[0]",
      "total_price": "items.price ~> $sum()"
    }
  }
}
```

**JSONata Syntax:**
- `field` - Access a field
- `array[0]` - First element (`array[-1]` for last)
- `array.field` - Field from all elements (implicit map)
- `array[active = true]` - Filter by condition
- `items.price ~> $sum()` - Transform with functions

See [JSONata documentation](https://jsonata.org) for the full expression language.

---

## Example: Multi-Level Approval

```json
{
  "name": "purchase_approval",
  "initial_state": "submit",
  "input_schema": {
    "type": "object",
    "properties": {
      "purchase_id": { "type": "string" },
      "amount": { "type": "number", "minimum": 0 },
      "description": { "type": "string" }
    },
    "required": ["purchase_id", "amount"]
  },
  "states": {
    "submit": {
      "type": "action",
      "logic": [
        {"type": "set", "values": {"data.submitted_at": "{{ $now() }}"}}
      ],
      "transitions": [
        {"to": "manager_approval", "condition": "input.amount > 1000"},
        {"to": "approved"}
      ]
    },
    "manager_approval": {
      "type": "halting",
      "presentation": {
        "type": "form",
        "fields": [
          {"name": "approved", "type": "boolean", "label": "Approve?"},
          {"name": "comment", "type": "string", "label": "Comment"}
        ]
      },
      "transitions": [
        {"to": "vp_approval", "condition": "data.approved and input.amount > 10000"},
        {"to": "approved", "condition": "data.approved"},
        {"to": "rejected"}
      ]
    },
    "vp_approval": {
      "type": "halting",
      "presentation": {
        "type": "form",
        "fields": [
          {"name": "approved", "type": "boolean", "label": "VP Approval"},
          {"name": "comment", "type": "string", "label": "Comment"}
        ]
      },
      "transitions": [
        {"to": "approved", "condition": "data.approved"},
        {"to": "rejected"}
      ]
    },
    "approved": {
      "type": "terminal"
    },
    "rejected": {
      "type": "terminal"
    }
  }
}
```

---

## Error Handling

| Error Code | Description |
|------------|-------------|
| `workflow_not_found` | Workflow definition doesn't exist |
| `workflow_not_published` | Workflow version not published |
| `instance_not_halted` | Cannot resume - instance not in halted state |
| `invalid_transition` | Input doesn't match any transition condition |
| `step_error` | Error executing workflow step |
| `instance_not_found` | Workflow instance doesn't exist |
| `invalid_cursor` | Pagination cursor is malformed or expired |
| `validation_error` | Input doesn't match `input_schema` |
| `INPUT_IMMUTABLE` | Cannot modify input values via set operation |
| `INVALID_SET_PATH` | Set path must start with `data.` |
| `forbidden` | Insufficient permissions for this state/action |

---

## Webhook Events

Subscribe to workflow events:

| Event | Description |
|-------|-------------|
| `workflow.instance.created` | New instance started |
| `workflow.instance.completed` | Instance finished successfully |
| `workflow.instance.failed` | Instance terminated with error |
| `workflow.instance.halted` | Instance waiting for input |

See [Webhooks and Streaming](06-webhooks-and-streaming.md) for setup.

---

## State-Level Authorization

Control who can view, resume, or cancel workflow instances based on the current state. Authorization is checked per-state, allowing different approval levels at different stages.

### Tag-Based Authorization

Match user permissions against required tags:

```json
{
  "manager_approval": {
    "type": "halting",
    "authorization": {
      "requires_any": ["manager", "admin"],
      "requires_all": ["finance"],
      "actions": ["resume"]
    },
    "presentation": {...}
  }
}
```

| Property | Description |
|----------|-------------|
| `requires_any` | User must have **at least one** of these tags |
| `requires_all` | User must have **all** of these tags |
| `actions` | Which actions require auth: `view`, `resume`, `cancel` (default: all) |

Pass permissions when resuming:

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/instances/inst_xyz/resume \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "selected_option": "approve",
    "permissions": ["manager", "finance"]
  }'
```

### Callback-Based Authorization

Delegate authorization to your backend:

```json
{
  "executive_approval": {
    "type": "halting",
    "authorization": {
      "callback_url": "https://yourapp.com/api/workflow/authorize",
      "cache_ttl_seconds": 300
    }
  }
}
```

Interactor calls your endpoint with a signed POST request:

```http
POST https://yourapp.com/api/workflow/authorize
Content-Type: application/json
X-Interactor-Signature: <hmac-sha256-hex>
X-Interactor-Timestamp: <unix-seconds>

{
  "action": "resume",
  "instance_id": "inst_xyz",
  "workflow_id": "wf_abc",
  "workflow_name": "purchase_approval",
  "state": "executive_approval",
  "external_user_id": "user_123",
  "context": {
    "account_id": "acc_456",
    "workflow_data": {...}
  }
}
```

Return your authorization decision:

```json
{"authorized": true}
```

Or deny with a reason:

```json
{"authorized": false, "reason": "User not in executive group"}
```

### Signature Verification

Verify callback signatures using your `callback_secret`:

```python
import hmac
import hashlib

def verify_signature(body: bytes, signature: str, timestamp: str, secret: str) -> bool:
    payload = f"{timestamp}.{body.decode()}"
    expected = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()
    return hmac.compare_digest(signature, expected)
```

### Error Responses

Unauthorized requests return 403:

```json
{
  "error": {
    "code": "forbidden",
    "message": "requires one of: manager, admin"
  }
}
```

### Fail-Closed Behavior

Authorization fails closed for security:

| Scenario | Result |
|----------|--------|
| No authorization config | Allow (backwards compatible) |
| Callback timeout (5s) | Deny |
| Callback error (5xx) | Deny |
| Missing permissions | Deny |

### List Filtering

When listing instances, unauthorized instances are automatically filtered out. Users only see instances where they have `view` permission for the current state. This prevents exposing instance IDs that users cannot access.

---

## Best Practices

1. **Start simple** - Begin with linear workflows, add complexity as needed
2. **Use meaningful state names** - `await_manager_approval` over `state_3`
3. **Validate early** - Use `/validate` endpoint during development
4. **Version carefully** - Publish new versions rather than modifying existing ones
5. **Handle all paths** - Ensure every state has a valid transition or is terminal
6. **Use external_user_id** - Isolate workflow instances per user

---

## Next Steps

- [Webhooks and Streaming](06-webhooks-and-streaming.md) - Real-time workflow updates
- [SDK Examples](07-sdk-examples.md) - Complete code examples
