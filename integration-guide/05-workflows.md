# Workflows

**Last Updated:** 2026-02-02

Workflows are state-machine based automations with human-in-the-loop support. Use them to model multi-step business processes that may require approvals, user input, or external integrations.

---

## Overview

Workflows consist of:

- **States** - Steps in your process (action, halting, terminal)
- **Transitions** - Rules for moving between states
- **Instances** - Running executions of a workflow
- **Threads** - Parallel execution paths within an instance

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
    "states": {
      "request": {
        "type": "action",
        "logic": {
          "type": "script",
          "code": "return {request_id: input.id, status: \"pending\"}"
        },
        "transitions": [{"target": "await_approval"}]
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
          {"target": "approved", "condition": {"field": "approved", "equals": true}},
          {"target": "rejected"}
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
        "logic": {"type": "script", "code": "return {message: \"Hello\"}"},
        "transitions": [{"target": "end"}]
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
    "user_ref": "user_123",
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
| `user_ref` | string | Filter by user reference |
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
    "workflow_data": {
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

### Resume with Input

```bash
curl -X POST https://core.interactor.com/api/v1/workflows/instances/inst_xyz/resume \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "approved": true,
      "comment": "Looks good"
    }
  }'
```

The workflow continues execution based on the input and transition conditions.

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
    "input": {...}
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
        "model": "claude-3-haiku-20240307",
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

### Script Logic

Execute JavaScript code:

```json
{
  "type": "script",
  "code": "const total = input.items.reduce((sum, i) => sum + i.price, 0); return { total, approved: total < 1000 };"
}
```

### HTTP Logic

Make external API calls:

```json
{
  "type": "http",
  "method": "POST",
  "url": "https://api.yourservice.com/process",
  "headers": {
    "Authorization": "Bearer ${secrets.API_KEY}"
  },
  "body": {
    "order_id": "${workflow_data.order_id}"
  }
}
```

### Condition Logic

Transition conditions:

```json
{
  "transitions": [
    {
      "target": "high_value_approval",
      "condition": {"field": "amount", "operator": "gt", "value": 10000}
    },
    {
      "target": "auto_approve",
      "condition": {"field": "amount", "operator": "lte", "value": 10000}
    }
  ]
}
```

**Operators:** `equals`, `not_equals`, `gt`, `gte`, `lt`, `lte`, `contains`, `in`

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

### JSONPath Extraction

For complex data extraction including arrays and filtering, use JSONPath (paths starting with `$`):

```json
{
  "output": {
    "path": "filtered_data",
    "extract": {
      "all_ids": "$.items[*].id",
      "active_users": "$.users[?(@.active == true)].name",
      "first_item": "$.items[0]"
    }
  }
}
```

**JSONPath Syntax:**
- `$.field` - Root-level field
- `$.array[0]` - First element (`[-1]` for last)
- `$.array[*].field` - Field from all elements
- `$.array[?(@.active == true)]` - Filter by condition

---

## Example: Multi-Level Approval

```json
{
  "name": "purchase_approval",
  "initial_state": "submit",
  "states": {
    "submit": {
      "type": "action",
      "logic": {
        "type": "script",
        "code": "return { ...input, submitted_at: new Date().toISOString() }"
      },
      "transitions": [
        {"target": "manager_approval", "condition": {"field": "amount", "operator": "gt", "value": 1000}},
        {"target": "approved"}
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
        {"target": "vp_approval", "condition": {"and": [
          {"field": "approved", "equals": true},
          {"field": "amount", "operator": "gt", "value": 10000}
        ]}},
        {"target": "approved", "condition": {"field": "approved", "equals": true}},
        {"target": "rejected"}
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
        {"target": "approved", "condition": {"field": "approved", "equals": true}},
        {"target": "rejected"}
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
| `script_error` | Error executing workflow script |
| `instance_not_found` | Workflow instance doesn't exist |
| `invalid_cursor` | Pagination cursor is malformed or expired |
| `missing_parameter` | Required parameter not provided |

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

## Best Practices

1. **Start simple** - Begin with linear workflows, add complexity as needed
2. **Use meaningful state names** - `await_manager_approval` over `state_3`
3. **Validate early** - Use `/validate` endpoint during development
4. **Version carefully** - Publish new versions rather than modifying existing ones
5. **Handle all paths** - Ensure every state has a valid transition or is terminal
6. **Use user_ref** - Isolate workflow instances per user

---

## Next Steps

- [Webhooks and Streaming](06-webhooks-and-streaming.md) - Real-time workflow updates
- [SDK Examples](07-sdk-examples.md) - Complete code examples
