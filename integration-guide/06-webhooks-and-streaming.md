# Webhooks and Streaming

**Last Updated:** 2026-01-27

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
      "credential.created",
      "credential.refreshed",
      "credential.expired",
      "credential.revoked"
    ],
    "workflow": [
      "workflow.instance.created",
      "workflow.instance.completed",
      "workflow.instance.failed",
      "workflow.instance.halted"
    ],
    "agent": [
      "agent.room.message",
      "agent.room.closed"
    ]
  }
}
```

Event types are grouped by category (the prefix before the first dot).

### Create a Webhook

```bash
curl -X POST https://core.interactor.com/api/v1/webhooks \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://yourapp.com/webhooks/interactor",
    "event_types": ["workflow.instance.completed", "agent.room.message"],
    "enabled": true
  }'
```

**Response:**
```json
{
  "data": {
    "id": "wh_abc",
    "url": "https://yourapp.com/webhooks/interactor",
    "secret": "whsec_xyz_SAVE_THIS",
    "event_types": ["workflow.instance.completed", "agent.room.message"],
    "enabled": true
  }
}
```

> **Important:** Save the `secret` - you'll need it to verify webhook signatures.

### List Webhooks

```bash
curl https://core.interactor.com/api/v1/webhooks \
  -H "Authorization: Bearer <token>"
```

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
    "event_types": ["credential.created", "credential.expired"],
    "url": "https://yourapp.com/webhooks/v2/interactor"
  }'
```

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

```bash
curl -X POST https://core.interactor.com/api/v1/webhooks/wh_abc/regenerate-secret \
  -H "Authorization: Bearer <token>"
```

### View Recent Events

```bash
curl https://core.interactor.com/api/v1/webhooks/wh_abc/events \
  -H "Authorization: Bearer <token>"
```

### Test Webhook

Send a test event to verify your endpoint:

```bash
curl -X POST https://core.interactor.com/api/v1/webhooks/wh_abc/test \
  -H "Authorization: Bearer <token>"
```

---

## Webhook Payload Format

All webhook events follow this structure:

```json
{
  "id": "evt_abc123",
  "type": "workflow.instance.completed",
  "timestamp": "2026-01-20T12:00:00Z",
  "data": {
    "instance_id": "inst_xyz",
    "workflow_name": "approval_workflow",
    "status": "completed",
    "output": {...}
  }
}
```

### Event-Specific Data

**credential.created:**
```json
{
  "credential_id": "cred_abc",
  "service_id": "google_calendar",
  "user_ref": "user_123"
}
```

**workflow.instance.halted:**
```json
{
  "instance_id": "inst_xyz",
  "workflow_name": "approval_workflow",
  "current_state": "await_approval",
  "halting_presentation": {...}
}
```

**agent.room.message:**
```json
{
  "room_id": "room_xyz",
  "message_id": "msg_123",
  "role": "assistant",
  "content": "Here's what I found..."
}
```

---

## Verifying Webhook Signatures

Always verify that webhooks came from Interactor by checking the signature.

### Signature Header

Webhooks include an `X-Interactor-Signature` header:

```
X-Interactor-Signature: sha256=abc123...
```

### Verification Code

**TypeScript/Node.js:**
```typescript
import crypto from 'crypto';

function verifyWebhook(payload: string, signature: string, secret: string): boolean {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(`sha256=${expected}`)
  );
}

// Express middleware example
app.post('/webhooks/interactor', express.raw({type: 'application/json'}), (req, res) => {
  const signature = req.headers['x-interactor-signature'] as string;
  const payload = req.body.toString();

  if (!verifyWebhook(payload, signature, process.env.WEBHOOK_SECRET!)) {
    return res.status(401).send('Invalid signature');
  }

  const event = JSON.parse(payload);
  console.log('Received event:', event.type);

  // Handle event...

  res.status(200).send('OK');
});
```

**Python:**
```python
import hmac
import hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature, f'sha256={expected}')

# Flask example
@app.route('/webhooks/interactor', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-Interactor-Signature')
    payload = request.get_data()

    if not verify_webhook(payload, signature, os.environ['WEBHOOK_SECRET']):
        return 'Invalid signature', 401

    event = request.get_json()
    print(f"Received event: {event['type']}")

    # Handle event...

    return 'OK', 200
```

---

## Server-Sent Events (SSE)

For real-time streaming, use SSE endpoints instead of webhooks.

### Workflow Instance Stream

Stream updates for a specific workflow instance:

```bash
curl -N https://core.interactor.com/api/v1/workflows/instances/inst_xyz/stream \
  -H "Authorization: Bearer <token>" \
  -H "Accept: text/event-stream"
```

**Events:**
```
event: state_changed
data: {"state": "processing", "thread_id": "thread_1"}

event: workflow_data_updated
data: {"key": "result", "value": {"status": "success"}}

event: completed
data: {"status": "completed", "output": {...}}
```

### Chat Room Stream

Stream messages in a chat room:

```bash
curl -N https://core.interactor.com/api/v1/agents/rooms/room_xyz/stream \
  -H "Authorization: Bearer <token>" \
  -H "Accept: text/event-stream"
```

**Events:**
```
event: message
data: {"id": "msg_1", "role": "user", "content": "Hello"}

event: message_start
data: {"id": "msg_2", "role": "assistant"}

event: message_delta
data: {"id": "msg_2", "delta": "Hi there! "}

event: message_delta
data: {"id": "msg_2", "delta": "How can I help?"}

event: message_end
data: {"id": "msg_2", "role": "assistant", "content": "Hi there! How can I help?"}

event: tool_use
data: {"tool": "search_products", "parameters": {...}}

event: tool_result
data: {"tool": "search_products", "result": {...}}
```

---

## JavaScript SSE Client

### Browser

```typescript
const token = 'your_access_token';
const roomId = 'room_xyz';

const eventSource = new EventSource(
  `https://core.interactor.com/api/v1/agents/rooms/${roomId}/stream?token=${token}`
);

eventSource.addEventListener('message_delta', (event) => {
  const data = JSON.parse(event.data);
  // Append delta to UI
  appendToMessage(data.id, data.delta);
});

eventSource.addEventListener('message_end', (event) => {
  const data = JSON.parse(event.data);
  // Message complete
  finalizeMessage(data.id, data.content);
});

eventSource.addEventListener('tool_use', (event) => {
  const data = JSON.parse(event.data);
  // Show tool being used
  showToolUsage(data.tool, data.parameters);
});

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
  eventSource.close();
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

eventSource.onmessage = (event) => {
  console.log('Message:', event.data);
};

eventSource.addEventListener('message_delta', (event) => {
  const data = JSON.parse(event.data);
  process.stdout.write(data.delta);
});
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
| Streaming AI responses | SSE |

**General rule:** Use webhooks for backend-to-backend communication, SSE for frontend real-time updates.

---

## Best Practices

### Webhooks

1. **Always verify signatures** - Reject requests with invalid signatures
2. **Respond quickly** - Return 200 within 5 seconds, process asynchronously
3. **Handle duplicates** - Events may be delivered more than once
4. **Use idempotent processing** - Track event IDs to prevent double-processing
5. **Monitor delivery** - Check webhook events list for failures

### SSE

1. **Handle reconnection** - SSE connections may drop; implement auto-reconnect
2. **Use heartbeats** - Watch for heartbeat events to detect stale connections
3. **Close when done** - Close connections when leaving pages/screens
4. **Rate limit connections** - Max 10 concurrent SSE connections per account

---

## Retry Policy

Interactor retries failed webhook deliveries with exponential backoff:

| Attempt | Delay |
|---------|-------|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

After 5 failed attempts, the webhook is disabled. Re-enable via the toggle endpoint.

---

## Next Steps

- [SDK Examples](07-sdk-examples.md) - Complete code for webhooks and streaming
