# SDK Examples

**Last Updated:** 2026-02-05

Complete code examples for integrating with Interactor in TypeScript/Node.js and Python.

---

## TypeScript / Node.js

### Full Client Implementation

```typescript
import axios, { AxiosInstance } from 'axios';

interface TokenResponse {
  access_token: string;
  token_type: string;
  expires_in: number;
}

interface ApiResponse<T> {
  data: T;
}

export class InteractorClient {
  private clientId: string;
  private clientSecret: string;
  private accessToken: string | null = null;
  private tokenExpiry: Date | null = null;
  private httpClient: AxiosInstance;

  constructor(clientId: string, clientSecret: string) {
    this.clientId = clientId;
    this.clientSecret = clientSecret;
    this.httpClient = axios.create({
      baseURL: 'https://core.interactor.com/api/v1',
      headers: { 'Content-Type': 'application/json' }
    });
  }

  private async getToken(): Promise<string> {
    if (this.accessToken && this.tokenExpiry && this.tokenExpiry > new Date()) {
      return this.accessToken;
    }

    const response = await axios.post<ApiResponse<TokenResponse>>(
      'https://auth.interactor.com/api/v1/oauth/token',
      {
        grant_type: 'client_credentials',
        client_id: this.clientId,
        client_secret: this.clientSecret
      }
    );

    this.accessToken = response.data.data.access_token;
    this.tokenExpiry = new Date(Date.now() + (response.data.data.expires_in - 60) * 1000);
    return this.accessToken;
  }

  async request<T>(method: string, path: string, data?: any): Promise<T> {
    const token = await this.getToken();
    const response = await this.httpClient.request<ApiResponse<T>>({
      method,
      url: path,
      headers: { Authorization: `Bearer ${token}` },
      data
    });
    return response.data.data;
  }

  // ============ Credentials ============

  async listCredentials(userRef?: string) {
    const params = userRef ? `?user_ref=${userRef}` : '';
    return this.request<any>('GET', `/credentials${params}`);
  }

  async getCredential(id: string) {
    return this.request<any>('GET', `/credentials/${id}`);
  }

  async getCredentialToken(id: string) {
    return this.request<{ access_token: string; expires_in: number }>(
      'GET',
      `/credentials/${id}/token`
    );
  }

  async initiateOAuth(serviceId: string, userRef: string, redirectUri: string, scopes?: string[]) {
    return this.request<{ flow_id: string; authorization_url: string }>(
      'POST',
      '/oauth/initiate',
      { service_id: serviceId, user_ref: userRef, redirect_uri: redirectUri, scopes }
    );
  }

  async getOAuthStatus(flowId: string) {
    return this.request<{ status: string; credential_id?: string }>(
      'GET',
      `/oauth/status/${flowId}`
    );
  }

  async deleteCredential(id: string) {
    return this.request<void>('DELETE', `/credentials/${id}`);
  }

  // ============ Workflows ============

  async createWorkflow(definition: any) {
    return this.request<{ name: string; version_id: string }>('POST', '/workflows', definition);
  }

  async listWorkflows() {
    return this.request<any[]>('GET', '/workflows');
  }

  async publishWorkflow(name: string, versionId: string) {
    return this.request<void>('POST', `/workflows/${name}/versions/${versionId}/publish`);
  }

  async createWorkflowInstance(name: string, userRef: string, input: any) {
    return this.request<{ id: string; status: string }>(
      'POST',
      `/workflows/${name}/instances`,
      { user_ref: userRef, input }
    );
  }

  async getWorkflowInstance(id: string) {
    return this.request<any>('GET', `/workflows/instances/${id}`);
  }

  async listWorkflowInstances(filters?: { user_ref?: string; workflow_name?: string; status?: string }) {
    const params = new URLSearchParams(filters as any).toString();
    return this.request<any[]>('GET', `/workflows/instances${params ? '?' + params : ''}`);
  }

  async resumeWorkflow(id: string, input: any) {
    return this.request<void>('POST', `/workflows/instances/${id}/resume`, { input });
  }

  async cancelWorkflow(id: string) {
    return this.request<void>('POST', `/workflows/instances/${id}/cancel`);
  }

  // ============ AI Agents ============

  async createAssistant(config: {
    name: string;
    system_prompt: string;
    description?: string;
    llm_provider?: string;
    llm_model?: string;
    llm_config?: { temperature?: number };
    default_tools?: string[];
  }) {
    return this.request<{ id: string; name: string }>('POST', '/agents/assistants', config);
  }

  async listAssistants() {
    return this.request<any[]>('GET', '/agents/assistants');
  }

  async getAssistant(id: string) {
    return this.request<any>('GET', `/agents/assistants/${id}`);
  }

  async updateAssistant(id: string, updates: any) {
    return this.request<any>('PUT', `/agents/assistants/${id}`, updates);
  }

  async createRoom(assistantId: string, userRef: string, metadata?: any) {
    return this.request<{ id: string; status: string }>(
      'POST',
      `/agents/${assistantId}/rooms`,
      { user_ref: userRef, metadata }
    );
  }

  async listRooms(filters?: { user_ref?: string; assistant_id?: string; status?: string }) {
    const params = new URLSearchParams(filters as any).toString();
    return this.request<any[]>('GET', `/agents/rooms${params ? '?' + params : ''}`);
  }

  async getRoom(id: string) {
    return this.request<any>('GET', `/agents/rooms/${id}`);
  }

  async sendMessage(roomId: string, content: string) {
    return this.request<{ id: string }>('POST', `/agents/rooms/${roomId}/messages`, {
      content,
      role: 'user'
    });
  }

  async getMessages(roomId: string, limit?: number) {
    const params = limit ? `?limit=${limit}` : '';
    return this.request<any[]>('GET', `/agents/rooms/${roomId}/messages${params}`);
  }

  async closeRoom(id: string) {
    return this.request<void>('POST', `/agents/rooms/${id}/close`);
  }

  // ============ Tools ============

  // Note: callback_secret is configured at account level, not per-tool
  // See: POST /api/v1/account/callback-secret or config sync
  async registerTool(tool: {
    name: string;
    description: string;
    parameters: any;
    callback_url: string;
  }) {
    return this.request<{ id: string }>('POST', '/tools', tool);
  }

  async listTools() {
    return this.request<any[]>('GET', '/tools');
  }

  async deleteTool(id: string) {
    return this.request<void>('DELETE', `/tools/${id}`);
  }

  // ============ Webhooks ============

  async createWebhook(url: string, eventTypes: string[]) {
    return this.request<{ id: string; secret: string }>(
      'POST',
      '/webhooks',
      { url, event_types: eventTypes, enabled: true }
    );
  }

  async listWebhooks() {
    return this.request<any[]>('GET', '/webhooks');
  }

  async deleteWebhook(id: string) {
    return this.request<void>('DELETE', `/webhooks/${id}`);
  }
}
```

### Usage Examples

```typescript
const client = new InteractorClient(
  process.env.INTERACTOR_CLIENT_ID!,
  process.env.INTERACTOR_CLIENT_SECRET!
);

// OAuth Flow
async function connectGoogleCalendar(userId: string) {
  const oauth = await client.initiateOAuth(
    'google_calendar',
    `user_${userId}`,
    'https://myapp.com/oauth/callback',
    ['calendar.readonly', 'calendar.events']
  );

  // Redirect user to oauth.authorization_url
  // After callback, check status:
  const status = await client.getOAuthStatus(oauth.flow_id);
  if (status.status === 'completed') {
    console.log('Credential created:', status.credential_id);
  }
}

// Workflow Example
async function runApprovalWorkflow(userId: string, request: any) {
  const instance = await client.createWorkflowInstance(
    'approval_workflow',
    `user_${userId}`,
    request
  );

  // Poll for halted state
  let status = await client.getWorkflowInstance(instance.id);
  while (status.status === 'running') {
    await new Promise(resolve => setTimeout(resolve, 1000));
    status = await client.getWorkflowInstance(instance.id);
  }

  if (status.status === 'halted') {
    // Show halting_presentation to user, get their input
    // Then resume:
    await client.resumeWorkflow(instance.id, { approved: true });
  }
}

// Chat with AI Assistant
async function chat(userId: string, message: string) {
  // Create or get existing room
  const rooms = await client.listRooms({ user_ref: `user_${userId}`, status: 'active' });
  let room = rooms[0];

  if (!room) {
    room = await client.createRoom('asst_support', `user_${userId}`);
  }

  // Send message
  await client.sendMessage(room.id, message);

  // Get response (in real app, use streaming)
  await new Promise(resolve => setTimeout(resolve, 2000));
  const messages = await client.getMessages(room.id);
  return messages[messages.length - 1];
}
```

---

## Python

### Full Client Implementation

```python
import os
import requests
from datetime import datetime, timedelta
from typing import Optional, Dict, Any, List

class InteractorClient:
    def __init__(self, client_id: str, client_secret: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.access_token: Optional[str] = None
        self.token_expiry: Optional[datetime] = None
        self.base_url = 'https://core.interactor.com/api/v1'
        self.auth_url = 'https://auth.interactor.com/api/v1'

    def _get_token(self) -> str:
        if self.access_token and self.token_expiry and self.token_expiry > datetime.now():
            return self.access_token

        response = requests.post(
            f'{self.auth_url}/oauth/token',
            json={
                'grant_type': 'client_credentials',
                'client_id': self.client_id,
                'client_secret': self.client_secret
            }
        )
        response.raise_for_status()
        data = response.json()['data']

        self.access_token = data['access_token']
        self.token_expiry = datetime.now() + timedelta(seconds=data['expires_in'] - 60)
        return self.access_token

    def _request(self, method: str, path: str, data: Optional[Dict] = None) -> Any:
        token = self._get_token()
        response = requests.request(
            method,
            f'{self.base_url}{path}',
            headers={'Authorization': f'Bearer {token}'},
            json=data
        )
        response.raise_for_status()
        return response.json().get('data')

    # ============ Credentials ============

    def list_credentials(self, user_ref: Optional[str] = None) -> List[Dict]:
        params = f'?user_ref={user_ref}' if user_ref else ''
        return self._request('GET', f'/credentials{params}')

    def get_credential(self, id: str) -> Dict:
        return self._request('GET', f'/credentials/{id}')

    def get_credential_token(self, id: str) -> Dict:
        return self._request('GET', f'/credentials/{id}/token')

    def initiate_oauth(
        self,
        service_id: str,
        user_ref: str,
        redirect_uri: str,
        scopes: Optional[List[str]] = None
    ) -> Dict:
        return self._request('POST', '/oauth/initiate', {
            'service_id': service_id,
            'user_ref': user_ref,
            'redirect_uri': redirect_uri,
            'scopes': scopes
        })

    def get_oauth_status(self, flow_id: str) -> Dict:
        return self._request('GET', f'/oauth/status/{flow_id}')

    def delete_credential(self, id: str) -> None:
        self._request('DELETE', f'/credentials/{id}')

    # ============ Workflows ============

    def create_workflow(self, definition: Dict) -> Dict:
        return self._request('POST', '/workflows', definition)

    def list_workflows(self) -> List[Dict]:
        return self._request('GET', '/workflows')

    def publish_workflow(self, name: str, version_id: str) -> None:
        self._request('POST', f'/workflows/{name}/versions/{version_id}/publish')

    def create_workflow_instance(self, name: str, user_ref: str, input_data: Dict) -> Dict:
        return self._request('POST', f'/workflows/{name}/instances', {
            'user_ref': user_ref,
            'input': input_data
        })

    def get_workflow_instance(self, id: str) -> Dict:
        return self._request('GET', f'/workflows/instances/{id}')

    def list_workflow_instances(
        self,
        user_ref: Optional[str] = None,
        workflow_name: Optional[str] = None,
        status: Optional[str] = None
    ) -> List[Dict]:
        params = []
        if user_ref:
            params.append(f'user_ref={user_ref}')
        if workflow_name:
            params.append(f'workflow_name={workflow_name}')
        if status:
            params.append(f'status={status}')
        query = '?' + '&'.join(params) if params else ''
        return self._request('GET', f'/workflows/instances{query}')

    def resume_workflow(self, id: str, input_data: Dict) -> None:
        self._request('POST', f'/workflows/instances/{id}/resume', {'input': input_data})

    def cancel_workflow(self, id: str) -> None:
        self._request('POST', f'/workflows/instances/{id}/cancel')

    # ============ AI Agents ============

    def create_assistant(
        self,
        name: str,
        system_prompt: str,
        description: Optional[str] = None,
        llm_provider: Optional[str] = None,
        llm_model: Optional[str] = None,
        llm_config: Optional[Dict] = None,
        default_tools: Optional[List[str]] = None
    ) -> Dict:
        return self._request('POST', '/agents/assistants', {
            'name': name,
            'system_prompt': system_prompt,
            'description': description,
            'llm_provider': llm_provider,
            'llm_model': llm_model,
            'llm_config': llm_config or {},
            'default_tools': default_tools or []
        })

    def list_assistants(self) -> List[Dict]:
        return self._request('GET', '/agents/assistants')

    def get_assistant(self, id: str) -> Dict:
        return self._request('GET', f'/agents/assistants/{id}')

    def update_assistant(self, id: str, updates: Dict) -> Dict:
        return self._request('PUT', f'/agents/assistants/{id}', updates)

    def create_room(
        self,
        assistant_id: str,
        user_ref: str,
        metadata: Optional[Dict] = None
    ) -> Dict:
        return self._request('POST', f'/agents/{assistant_id}/rooms', {
            'user_ref': user_ref,
            'metadata': metadata or {}
        })

    def list_rooms(
        self,
        user_ref: Optional[str] = None,
        assistant_id: Optional[str] = None,
        status: Optional[str] = None
    ) -> List[Dict]:
        params = []
        if user_ref:
            params.append(f'user_ref={user_ref}')
        if assistant_id:
            params.append(f'assistant_id={assistant_id}')
        if status:
            params.append(f'status={status}')
        query = '?' + '&'.join(params) if params else ''
        return self._request('GET', f'/agents/rooms{query}')

    def get_room(self, id: str) -> Dict:
        return self._request('GET', f'/agents/rooms/{id}')

    def send_message(self, room_id: str, content: str) -> Dict:
        return self._request('POST', f'/agents/rooms/{room_id}/messages', {
            'content': content,
            'role': 'user'
        })

    def get_messages(self, room_id: str, limit: Optional[int] = None) -> List[Dict]:
        params = f'?limit={limit}' if limit else ''
        return self._request('GET', f'/agents/rooms/{room_id}/messages{params}')

    def close_room(self, id: str) -> None:
        self._request('POST', f'/agents/rooms/{id}/close')

    # ============ Tools ============

    # Note: callback_secret is configured at account level, not per-tool
    # See: POST /api/v1/account/callback-secret or config sync
    def register_tool(
        self,
        name: str,
        description: str,
        parameters: Dict,
        callback_url: str
    ) -> Dict:
        return self._request('POST', '/tools', {
            'name': name,
            'description': description,
            'parameters': parameters,
            'callback_url': callback_url
        })

    def list_tools(self) -> List[Dict]:
        return self._request('GET', '/tools')

    def delete_tool(self, id: str) -> None:
        self._request('DELETE', f'/tools/{id}')

    # ============ Webhooks ============

    def create_webhook(self, url: str, event_types: List[str]) -> Dict:
        return self._request('POST', '/webhooks', {
            'url': url,
            'event_types': event_types,
            'enabled': True
        })

    def list_webhooks(self) -> List[Dict]:
        return self._request('GET', '/webhooks')

    def delete_webhook(self, id: str) -> None:
        self._request('DELETE', f'/webhooks/{id}')
```

### Usage Examples

```python
import os
import time

client = InteractorClient(
    os.environ['INTERACTOR_CLIENT_ID'],
    os.environ['INTERACTOR_CLIENT_SECRET']
)

# OAuth Flow
def connect_google_calendar(user_id: str):
    oauth = client.initiate_oauth(
        'google_calendar',
        f'user_{user_id}',
        'https://myapp.com/oauth/callback',
        ['calendar.readonly', 'calendar.events']
    )

    print(f"Redirect user to: {oauth['authorization_url']}")

    # After callback, check status:
    status = client.get_oauth_status(oauth['flow_id'])
    if status['status'] == 'completed':
        print(f"Credential created: {status['credential_id']}")

# Workflow Example
def run_approval_workflow(user_id: str, request_data: dict):
    instance = client.create_workflow_instance(
        'approval_workflow',
        f'user_{user_id}',
        request_data
    )

    # Poll for halted state
    status = client.get_workflow_instance(instance['id'])
    while status['status'] == 'running':
        time.sleep(1)
        status = client.get_workflow_instance(instance['id'])

    if status['status'] == 'halted':
        print(f"Workflow halted at: {status['current_state']}")
        print(f"Presentation: {status['halting_presentation']}")

        # Resume with user input
        client.resume_workflow(instance['id'], {'approved': True})

# Chat with AI Assistant
def chat(user_id: str, message: str):
    # Create or get existing room
    rooms = client.list_rooms(user_ref=f'user_{user_id}', status='active')
    room = rooms[0] if rooms else client.create_room('asst_support', f'user_{user_id}')

    # Send message
    client.send_message(room['id'], message)

    # Get response (in real app, use streaming)
    time.sleep(2)
    messages = client.get_messages(room['id'])
    return messages[-1]


# Example usage
if __name__ == '__main__':
    # List credentials
    credentials = client.list_credentials()
    print(f"Found {len(credentials)} credentials")

    # Create a chat session
    response = chat('123', 'How do I update my billing information?')
    print(f"Assistant: {response['content']}")
```

---

## Webhook Handler Examples

### Express.js (Node.js)

```typescript
import express from 'express';
import crypto from 'crypto';

const app = express();

app.post(
  '/webhooks/interactor',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const signature = req.headers['x-interactor-signature'] as string;
    const timestamp = req.headers['x-interactor-timestamp'] as string;
    const payload = req.body.toString();

    // Verify signature: HMAC-SHA256 of "timestamp.payload"
    const expected = crypto
      .createHmac('sha256', process.env.WEBHOOK_SECRET!)
      .update(`${timestamp}.${payload}`)
      .digest('hex');

    if (!crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expected)
    )) {
      return res.status(401).send('Invalid signature');
    }

    const event = JSON.parse(payload);

    // Handle different event types
    switch (event.type) {
      case 'credential.ready':
        console.log(`New credential: ${event.data.credential_id}`);
        break;

      case 'workflow.completed':
        console.log(`Workflow completed: ${event.data.instance_id}`);
        break;

      case 'agent.message_received':
        console.log(`New message in room ${event.data.room_id}`);
        break;
    }

    res.status(200).send('OK');
  }
);

app.listen(3000);
```

### Flask (Python)

```python
from flask import Flask, request
import hmac
import hashlib
import os

app = Flask(__name__)

@app.route('/webhooks/interactor', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-Interactor-Signature', '')
    timestamp = request.headers.get('X-Interactor-Timestamp', '')
    payload = request.get_data()

    # Verify signature: HMAC-SHA256 of "timestamp.payload"
    message = f"{timestamp}.{payload.decode()}"
    expected = hmac.new(
        os.environ['WEBHOOK_SECRET'].encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        return 'Invalid signature', 401

    event = request.get_json()

    # Handle different event types
    if event['type'] == 'credential.ready':
        print(f"New credential: {event['data']['credential_id']}")

    elif event['type'] == 'workflow.completed':
        print(f"Workflow completed: {event['data']['instance_id']}")

    elif event['type'] == 'agent.message_received':
        print(f"New message in room {event['data']['room_id']}")

    return 'OK', 200

if __name__ == '__main__':
    app.run(port=3000)
```

---

## SSE Streaming Example

### React Component

```tsx
import { useEffect, useState, useRef } from 'react';

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
}

function ChatRoom({ roomId, token }: { roomId: string; token: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [status, setStatus] = useState<string>('');
  const eventSourceRef = useRef<EventSource | null>(null);

  useEffect(() => {
    const eventSource = new EventSource(
      `https://core.interactor.com/api/v1/agents/rooms/${roomId}/stream?token=${token}`
    );
    eventSourceRef.current = eventSource;

    // Connection established
    eventSource.addEventListener('connected', (event) => {
      const data = JSON.parse(event.data);
      console.log('Connected to room:', data.room_id);
    });

    // User message received (for multi-client sync)
    eventSource.addEventListener('agent.message_received', (event) => {
      const data = JSON.parse(event.data);
      if (data.role === 'user') {
        // Optionally refresh messages list
      }
    });

    // Status updates (e.g., "Thinking...", "Searching...")
    eventSource.addEventListener('agent.status', (event) => {
      const data = JSON.parse(event.data);
      setStatus(data.message);
    });

    // Assistant response ready (complete message, not streamed)
    eventSource.addEventListener('agent.response_sent', (event) => {
      const data = JSON.parse(event.data);
      setMessages((prev) => [...prev, {
        id: data.message_id,
        role: 'assistant',
        content: data.response
      }]);
      setStatus('');
    });

    // Error occurred
    eventSource.addEventListener('agent.error', (event) => {
      const data = JSON.parse(event.data);
      console.error('Agent error:', data.message);
      setStatus('');
    });

    // Keepalive ping
    eventSource.addEventListener('ping', () => {
      // Connection healthy
    });

    // Stream ending
    eventSource.addEventListener('done', (event) => {
      const data = JSON.parse(event.data);
      console.log('Stream ended:', data.reason);
      eventSource.close();
    });

    eventSource.onerror = () => {
      eventSource.close();
    };

    return () => {
      eventSource.close();
    };
  }, [roomId, token]);

  return (
    <div>
      {messages.map((msg) => (
        <div key={msg.id} className={msg.role}>
          {msg.content}
        </div>
      ))}
      {status && (
        <div className="status">
          {status}
        </div>
      )}
    </div>
  );
}
```

---

## Environment Variables

Required environment variables for all examples:

```bash
INTERACTOR_CLIENT_ID=your_client_id
INTERACTOR_CLIENT_SECRET=your_client_secret
WEBHOOK_SECRET=your_webhook_secret
```
