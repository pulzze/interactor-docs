# Workflows

**Last Updated:** 2026-01-27

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

## Halting Presentations

When a workflow halts, it can specify how to present the required input to users.

### Form Presentation

```json
{
  "type": "form",
  "fields": [
    {"name": "approved", "type": "boolean", "label": "Approve this request?"},
    {"name": "amount", "type": "number", "label": "Approved Amount"},
    {"name": "notes", "type": "string", "label": "Notes", "multiline": true}
  ]
}
```

### Choice Presentation

```json
{
  "type": "choice",
  "message": "Select the next action",
  "options": [
    {"value": "approve", "label": "Approve"},
    {"value": "reject", "label": "Reject"},
    {"value": "escalate", "label": "Escalate to Manager"}
  ]
}
```

### Message Presentation

```json
{
  "type": "message",
  "message": "Waiting for external system response..."
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
