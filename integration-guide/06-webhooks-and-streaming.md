# Webhooks and Streaming

_Last verified: 2026-06-18_

**Last Updated:** 2026-05-24

Receive real-time updates from Interactor via webhooks (push) or Server-Sent Events (pull).

---

## Webhooks

Webhooks push events to your server when things happen in Interactor.

### Available Event Types

```bash
curl https://core.interactor.com/api/v1/webhooks/event-types \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "credential": [
      "credential.ready",
      "credential.expired",
      "credential.revoked",
      "credential.refresh_failed"
    ],
    "workflow": [
      "workflow.started",
      "workflow.state_changed",
      "workflow.halted",
      "workflow.completed",
      "workflow.failed"
    ],
    "agent": [
      "agent.room_created",
      "agent.message_received",
      "agent.response_sent",
      "agent.action_executed",
      "agent.error"
    ]
  }
}
```

Event types are grouped by category (the prefix before the first dot). You can subscribe to individual events or use wildcards:

| Pattern | Matches |
|---------|---------|
| `credential.*` | All credential events |
| `workflow.*` | All workflow events |
| `agent.*` | All agent events |
| `*` | All events |

**Examples:**
```json
{
  "event_types": ["credential.*", "workflow.completed"]
}
```

### Create a Webhook

```bash
curl -X POST https://core.interactor.com/api/v1/webhooks \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Webhook",
    "url": "https://yourapp.com/webhooks/interactor",
    "event_types": ["workflow.completed", "agent.response_sent"],
    "enabled": true,
    "headers": {
      "X-Custom-Header": "my-value"
    },
    "description": "Production webhook for workflow notifications",
    "metadata": {"environment": "production"}
  }'
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Display name for the webhook |
| `url` | string | Yes | Destination URL (HTTP or HTTPS) |
| `event_types` | array | Yes | Events to subscribe to (supports wildcards like `workflow.*`) |
| `enabled` | boolean | No | Whether webhook is active (default: true) |
| `headers` | object | No | Custom headers to include in webhook requests |
| `description` | string | No | Description for documentation purposes |
| `metadata` | object | No | Arbitrary metadata for your use |

**Response:**
```json
{
  "data": {
    "id": "wh_abc",
    "name": "My Webhook",
    "url": "https://yourapp.com/webhooks/interactor",
    "secret": "whsec_xyz_SAVE_THIS",
    "event_types": ["workflow.completed", "agent.response_sent"],
    "headers": {"X-Custom-Header": "[REDACTED]"},
    "enabled": true,
    "description": "Production webhook for workflow notifications",
    "metadata": {"environment": "production"},
    "created_at": "2026-02-05T12:00:00Z",
    "updated_at": "2026-02-05T12:00:00Z"
  }
}
```

> **Important:** The `secret` is only shown once at creation time. Save it immediately—you cannot retrieve it later. You'll need it to verify webhook signatures.

### List Webhooks

```bash
curl https://core.interactor.com/api/v1/webhooks \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": [
    {
      "id": "wh_abc",
      "name": "My Webhook",
      "url": "https://yourapp.com/webhooks/interactor",
      "event_types": ["workflow.completed", "agent.response_sent"],
      "headers": {"X-Custom-Header": "[REDACTED]"},
      "enabled": true,
      "description": "Production webhook for workflow notifications",
      "metadata": {"environment": "production"},
      "created_at": "2026-02-05T12:00:00Z",
      "updated_at": "2026-02-05T12:00:00Z"
    }
  ]
}
```

Note: The `secret` field is never included in list or get responses.

### Get Webhook

```bash
curl https://core.interactor.com/api/v1/webhooks/wh_abc \
  -H "Authorization: Bearer <token>"
```

### Update Webhook

```bash
curl -X PUT https://core.interactor.com/api/v1/webhooks/wh_abc \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Webhook Name",
    "event_types": ["credential.ready", "credential.expired"],
    "url": "https://yourapp.com/webhooks/v2/interactor"
  }'
```

All fields are optional—only include fields you want to change.

### Toggle Webhook (Enable/Disable)

```bash
curl -X POST https://core.interactor.com/api/v1/webhooks/wh_abc/toggle \
  -H "Authorization: Bearer <token>"
```

### Delete Webhook

```bash
curl -X DELETE https://core.interactor.com/api/v1/webhooks/wh_abc \
  -H "Authorization: Bearer <token>"
```

### Regenerate Secret

If your secret is compromised, generate a new one:

```bash
curl -X POST https://core.interactor.com/api/v1/webhooks/wh_abc/regenerate-secret \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "id": "wh_abc",
    "secret": "whsec_new_secret_SAVE_THIS",
    ...
  }
}
```

The old secret becomes invalid immediately. Save the new secret—it won't be shown again.

### View Recent Events

```bash
curl "https://core.interactor.com/api/v1/webhooks/wh_abc/events?limit=20" \
  -H "Authorization: Bearer <token>"
```

**Query Parameters:**

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `limit` | integer | 50 | 100 | Number of events to return |
| `offset` | integer | 0 | — | Pagination offset |

**Response:**
```json
{
  "data": [
    {
      "id": "evt_123",
      "event_type": "workflow.completed",
      "status": "delivered",
      "attempts": 1,
      "last_response_status": 200,
      "last_attempt_at": "2026-02-05T12:00:00Z",
      "delivered_at": "2026-02-05T12:00:00Z",
      "created_at": "2026-02-05T12:00:00Z"
    },
    {
      "id": "evt_124",
      "event_type": "workflow.failed",
      "status": "failed",
      "attempts": 3,
      "last_response_status": 500,
      "last_error": "Connection timeout",
      "last_attempt_at": "2026-02-05T12:30:00Z",
      "created_at": "2026-02-05T12:00:00Z"
    }
  ]
}
```

**Event Status Values:**
- `pending` - Scheduled for delivery or awaiting retry
- `delivered` - Successfully delivered (2xx response)
- `failed` - All retry attempts exhausted

### Test Webhook

Send a test event to verify your endpoint:

```bash
curl -X POST https://core.interactor.com/api/v1/webhooks/wh_abc/test \
  -H "Authorization: Bearer <token>"
```

Test events are sent with `max_attempts: 1` (no retries) and use a `test` event type.

---

## Webhook Payload Format

All webhook events follow this structure:

```json
{
  "id": "evt_abc123",
  "type": "workflow.completed",
  "created_at": "2026-01-20T12:00:00Z",
  "data": {
    "instance_id": "inst_xyz",
    "workflow_name": "approval_workflow",
    "status": "completed",
    "output": {...}
  }
}
```

The delivery timestamp used for signature verification is in the `X-Interactor-Timestamp` header (Unix seconds), not in the body.

### Event-Specific Data

**credential.ready:**
```json
{
  "credential_id": "cred_abc",
  "service_id": "google_calendar",
  "external_user_id": "user_123"
}
```

**credential.refresh_failed:**
```json
{
  "credential_id": "cred_abc",
  "service_id": "google_calendar",
  "external_user_id": "user_123",
  "error": "refresh_token_expired"
}
```

**workflow.halted:**
```json
{
  "instance_id": "inst_xyz",
  "workflow_name": "approval_workflow",
  "current_state": "await_approval",
  "halted_options": [...]
}
```

**agent.response_sent:**
```json
{
  "room_id": "room_xyz",
  "message_id": "msg_123",
  "response": "Here's what I found..."
}
```

**agent.error:**
```json
{
  "room_id": "room_xyz",
  "error": "internal_error",
  "message": "The conversation encountered an unexpected error."
}
```

---

## Webhook Headers

Interactor sends these headers with every webhook request:

| Header | Description |
|--------|-------------|
| `Content-Type` | Always `application/json` |
| `X-Interactor-Event` | The event type (e.g., `workflow.completed`) |
| `X-Interactor-Delivery` | Unique delivery ID for this attempt |
| `X-Interactor-Timestamp` | Unix timestamp (seconds) when signature was generated |
| `X-Interactor-Signature` | HMAC-SHA256 signature for verification |

Plus any custom headers you configured when creating the webhook.

---

## Verifying Webhook Signatures

Always verify that webhooks came from Interactor by checking the signature.

### Signature Algorithm

The signature is computed as:
1. Concatenate: `{timestamp}.{raw_json_body}`
2. Compute HMAC-SHA256 using your webhook secret
3. Hex-encode the result (lowercase)

### Verification Code

**TypeScript/Node.js:**
```typescript
import crypto from 'crypto';

function verifyWebhook(
  payload: string,
  signature: string,
  timestamp: string,
  secret: string
): boolean {
  // Verify timestamp is recent (within 5 minutes)
  const now = Math.floor(Date.now() / 1000);
  const ts = parseInt(timestamp, 10);
  if (Math.abs(now - ts) > 300) {
    return false; // Replay attack protection
  }

  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

// Express middleware example
app.post('/webhooks/interactor', express.raw({type: 'application/json'}), (req, res) => {
  const signature = req.headers['x-interactor-signature'] as string;
  const timestamp = req.headers['x-interactor-timestamp'] as string;
  const payload = req.body.toString();

  if (!verifyWebhook(payload, signature, timestamp, process.env.WEBHOOK_SECRET!)) {
    return res.status(401).send('Invalid signature');
  }

  const event = JSON.parse(payload);
  console.log('Received event:', event.type);

  // Handle event asynchronously - respond quickly
  handleEventAsync(event);

  res.status(200).send('OK');
});
```

**Python:**
```python
import hmac
import hashlib
import time

def verify_webhook(payload: bytes, signature: str, timestamp: str, secret: str) -> bool:
    # Verify timestamp is recent (within 5 minutes)
    now = int(time.time())
    ts = int(timestamp)
    if abs(now - ts) > 300:
        return False  # Replay attack protection

    message = f"{timestamp}.{payload.decode()}"
    expected = hmac.new(
        secret.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature, expected)

# Flask example
@app.route('/webhooks/interactor', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-Interactor-Signature')
    timestamp = request.headers.get('X-Interactor-Timestamp')
    payload = request.get_data()

    if not verify_webhook(payload, signature, timestamp, os.environ['WEBHOOK_SECRET']):
        return 'Invalid signature', 401

    event = request.get_json()
    print(f"Received event: {event['type']}")

    # Handle event asynchronously - respond quickly
    handle_event_async(event)

    return 'OK', 200
```

---

## Server-Sent Events (SSE)

For real-time streaming, use SSE endpoints instead of webhooks.

### Authentication

SSE endpoints support two authentication methods:

1. **Authorization Header** (preferred for server-side):
   ```bash
   curl -N https://core.interactor.com/api/v1/workflows/instances/inst_xyz/stream \
     -H "Authorization: Bearer <token>" \
     -H "Accept: text/event-stream"
   ```

2. **Query Parameter** (required for browser EventSource):
   ```
   https://core.interactor.com/api/v1/workflows/instances/inst_xyz/stream?token=<token>
   ```

> **Note:** Browser's native `EventSource` API cannot set custom headers, so use the query parameter method for browser clients.

### Workflow Instance Stream

Stream updates for a specific workflow instance:

```bash
curl -N https://core.interactor.com/api/v1/workflows/instances/inst_xyz/stream \
  -H "Authorization: Bearer <token>" \
  -H "Accept: text/event-stream"
```

**Events:**

| Event | Description |
|-------|-------------|
| `state` | Initial state when connecting |
| `state_changed` | Workflow transitioned to a new state |
| `step_started` | A step within the current state started executing |
| `step_progress` | Progress update from a long-running step |
| `step_completed` | A step finished executing |
| `data_updated` | Workflow `data` was modified |
| `halted` | Workflow paused awaiting input |
| `completed` | Workflow finished successfully |
| `failed` | Workflow encountered an error |
| `cancelled` | Workflow was cancelled |
| `forked` | Workflow spawned parallel threads |
| `thread_completed` | A parallel thread finished |
| `thread_waiting` | A thread is waiting for others |
| `agent_handoff` | Control was handed off to an agent |
| `heartbeat` | Keepalive ping (every 30 seconds) |
| `done` | Stream is ending |

**Example event stream:**
```
event: state
data: {"instance_id":"inst_xyz","status":"running","current_state":"processing"}

event: state_changed
data: {"state":"validating","thread_id":"main"}

event: heartbeat
data: {"timestamp":"2026-02-05T12:00:30Z"}

event: completed
data: {"status":"completed","output":{"result":"success"}}

event: done
data: {"status":"completed"}
```

### Agent Room Stream

Stream messages and events in a chat room:

```bash
curl -N https://core.interactor.com/api/v1/agents/rooms/room_xyz/stream \
  -H "Authorization: Bearer <token>" \
  -H "Accept: text/event-stream"
```

**Events:**

| Event | Description |
|-------|-------------|
| `connected` | Initial connection established |
| `room.state` | Room snapshot (streaming status, queued messages) sent after `connected` |
| `agent.message_received` | A message was received in the room |
| `agent.response_sent` | Agent sent a response. May include `"replay": true` when re-emitted to a reconnecting client |
| `agent.status` | Status update (e.g., "thinking", "searching") |
| `agent.action_executed` | Agent executed a tool/action |
| `agent.error` | An error occurred |
| `agent.loop_detected` | Agent detected a potential loop |
| `workflow.*` | When a room is bound to a workflow, workflow SSE events are forwarded with the `workflow.` prefix |
| `ping` | Keepalive ping (every 30 seconds) |
| `done` | Stream is ending (room closed) |

**Example event stream:**
```
event: connected
data: {"room_id":"room_xyz","status":"active"}

event: agent.message_received
data: {"room_id":"room_xyz","message_id":"msg_1","role":"user"}

event: agent.status
data: {"room_id":"room_xyz","message":"Searching for products..."}

event: agent.action_executed
data: {"room_id":"room_xyz","action":"search_products","parameters":{"query":"laptop"}}

event: agent.response_sent
data: {"room_id":"room_xyz","message_id":"msg_2","response":"I found 5 laptops matching your criteria..."}

event: ping
data: {"timestamp":"2026-02-05T12:00:30Z"}
```

> **Note:** Agent responses are sent as complete messages via `agent.response_sent`, not streamed token-by-token.

---

## JavaScript SSE Client

### Browser

```typescript
const token = 'your_access_token';
const roomId = 'room_xyz';

// Browser EventSource requires token in URL (can't set headers)
const eventSource = new EventSource(
  `https://core.interactor.com/api/v1/agents/rooms/${roomId}/stream?token=${token}`
);

eventSource.addEventListener('connected', (event) => {
  const data = JSON.parse(event.data);
  console.log('Connected to room:', data.room_id);
});

eventSource.addEventListener('agent.response_sent', (event) => {
  const data = JSON.parse(event.data);
  displayMessage(data.response);
});

eventSource.addEventListener('agent.status', (event) => {
  const data = JSON.parse(event.data);
  showStatusIndicator(data.message);
});

eventSource.addEventListener('agent.error', (event) => {
  const data = JSON.parse(event.data);
  showError(data.message);
});

eventSource.addEventListener('ping', () => {
  // Keepalive received - connection is healthy
});

eventSource.addEventListener('done', (event) => {
  const data = JSON.parse(event.data);
  console.log('Stream ended:', data.reason);
  eventSource.close();
});

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
  // EventSource will auto-reconnect by default
};
```

### Node.js (with eventsource package)

```typescript
import EventSource from 'eventsource';

const eventSource = new EventSource(
  'https://core.interactor.com/api/v1/agents/rooms/room_xyz/stream',
  {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  }
);

eventSource.addEventListener('agent.response_sent', (event) => {
  const data = JSON.parse(event.data);
  console.log('Agent:', data.response);
});

eventSource.addEventListener('heartbeat', (event) => {
  // For workflow streams, heartbeat event
  console.log('Keepalive received');
});

eventSource.addEventListener('ping', (event) => {
  // For room streams, ping event
  console.log('Keepalive received');
});

eventSource.onerror = (error) => {
  console.error('Connection error:', error);
};
```

---

## Webhook vs SSE: When to Use Each

| Use Case | Recommended |
|----------|-------------|
| Background processing notifications | Webhooks |
| Credential status changes | Webhooks |
| Workflow completion notifications | Webhooks |
| Real-time chat UI | SSE |
| Live workflow progress display | SSE |
| Monitoring workflow state changes | SSE |

**General rule:** Use webhooks for backend-to-backend communication, SSE for frontend real-time updates.

---

## Best Practices

### Webhooks

1. **Always verify signatures** - Reject requests with invalid signatures
2. **Check timestamp freshness** - Reject events older than 5 minutes (replay protection)
3. **Respond quickly** - Return 200 within 5 seconds, process asynchronously
4. **Handle duplicates** - Events may be delivered more than once
5. **Use idempotent processing** - Track event IDs to prevent double-processing
6. **Monitor delivery** - Check webhook events list for failures

### SSE

1. **Handle reconnection** - SSE connections may drop; implement auto-reconnect
2. **Watch for keepalives** - If you don't receive `heartbeat`/`ping` within 60 seconds, reconnect
3. **Close when done** - Close connections when leaving pages/screens
4. **Handle errors gracefully** - Listen for `agent.error` events

---

## Retry Policy

Interactor retries failed webhook deliveries with exponential backoff:

| Attempt | Delay before this attempt |
|---------|---------------------------|
| 1 | Immediate |
| 2 | 1 second |
| 3 | 2 seconds |
| 4 | 4 seconds |
| 5 | 8 seconds |

Maximum **5 attempts**. The backoff doubles with each retry (1s, 2s, 4s, 8s, 16s) and is capped at 60 seconds. After all attempts fail, the event status is set to `failed`.

You can monitor failed deliveries via the [View Recent Events](#view-recent-events) endpoint and manually retry by creating a test event or re-triggering the original action.

---

## Rate Limits

API endpoints (including SSE connections) are rate-limited to **100 requests per minute** per account.

Rate limit headers are included in responses:
- `X-RateLimit-Limit`: Maximum requests allowed
- `X-RateLimit-Remaining`: Requests remaining in current window
- `X-RateLimit-Reset`: Unix timestamp when the window resets
- `Retry-After`: Seconds to wait (only when limit exceeded)

---

## URL Requirements

Webhook destination URLs must:
- Use HTTP or HTTPS protocol
- Have a valid hostname

There are no port restrictions. Requests have a 30-second timeout.

---

## Next Steps

- [SDK Examples](07-sdk-examples.md) - Complete code for webhooks and streaming
