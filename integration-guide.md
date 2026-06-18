# Building AI Agents on Interactor

_Last verified: 2026-06-18_

**Version:** 3.0.0
**Last Updated:** 2026-05-25

A guide for developers who want to build their own AI agents on top of the Interactor platform.

---

## Who this guide is for

You're building an AI agent — a product where a user's intent gets translated into actions against real services (their calendar, their CRM, their internal data, their tools). You want the agent to:

- Connect to whatever services your users already use, without you hand-coding every integration
- Get better over time as it encounters new services and edge cases
- Stay safe and predictable when it acts on a user's behalf
- Stream responses, recover from partial failures, and keep humans in the loop where it matters

Interactor is the platform layer that gives you those primitives so you can focus on your agent's product behavior — its persona, its prompts, its UX — instead of the integration plumbing.

---

## Why build on Interactor

Most teams building AI agents end up writing the same scaffolding: OAuth flows for dozens of providers, a tool-call dispatcher, a retry layer that knows the quirks of every third-party API, a way to pause for human approval, a streaming protocol for the UI. Then they discover their agent guesses wrong on parameter types, hits a 400, and has no way to learn from it.

Interactor is designed around three operating principles that map directly onto how production AI agents actually fail and recover.

### Self-learning: services discovered, not pre-configured

The platform sits on top of a **Service Knowledge Base** — a vectorized catalog of external services and their capabilities. When your agent says *"book a meeting with Alice"*, Interactor doesn't look that up in a hand-written switch statement. It does semantic matching against the catalog: which connected services expose a "create calendar event" capability, with what parameters, against what OAuth scopes.

For you, this means:

- You don't enumerate the services your agent supports. It grows with the catalog.
- New services come online without you shipping a release.
- Your agent's prompts can reference user intent (*"send this to the team channel"*) rather than vendor names (*"call Slack `chat.postMessage` with channel_id…"*).

### Closed-loop: the executor learns from every call

When the executor invokes a downstream API on a user's behalf and gets back a `400` or `422` — the API said *"this parameter is the wrong shape"* — the platform doesn't just propagate the error. It tries to self-heal:

1. **Diagnose** — read the error against the declared parameter schema. Did the agent pass an ISO string where the API wants epoch millis? A bare ID where the API wants a wrapped object?
2. **Coerce** — apply a fix derived from the schema.
3. **Retry** — once, with the corrected payload.
4. **Report back** — on success, the learned coercion gets pushed back to the Service Knowledge Base. The next agent that calls the same capability gets the fix applied proactively, before the failure happens.

This is the closed loop: discovery populates the schema → runtime catches the gaps that discovery missed → corrections close the gaps → the catalog gets richer. Your agent benefits without you doing anything.

### Self-validation: guardrails that keep the agent honest

An agent that acts on a user's behalf needs to recognize when it's about to do something it shouldn't — and when it's stuck. Interactor gives you three layers of validation that you compose, not write from scratch:

- **Approval gates** on tool calls. You declare *"actions that move money or send external messages require user confirmation"* and Interactor halts the agent at that boundary, surfaces the action to your UI, and resumes when the user approves.
- **Loop detection** at the agent runtime. If the agent makes the same tool call repeatedly, hits the same error in succession, or burns through its iteration budget, the platform breaks the loop and emits an `agent.loop_detected` event — you decide how to recover.
- **Pre-flight validation endpoints** for expressions (`/api/v1/expressions/validate`) and workflows (`/api/v1/workflows/validate`) so your agent's plans get checked before they execute, not after.

The goal: an agent that fails fast and visibly, never silently.

---

## The building blocks

Concretely, you'll use four Interactor primitives.

| Primitive | What it is | What it solves |
|---|---|---|
| **Credentials** | OAuth + API-key storage with auto-refresh, multi-tenant isolation, and per-user namespacing via `external_user_id` | Lets your users connect their accounts; your agent calls those services on their behalf without you handling token lifecycle |
| **AI Agents** | Configured assistants with prompts, model choice, tool/data-source bindings, supporting-agent delegation, and approval gates | The conversation surface and orchestration runtime — you bring the persona, Interactor brings the loop |
| **Tools** | Callable functions registered with Interactor, with parameter schemas and HTTP callbacks back to your service | Lets the agent take actions in *your* systems, with signature verification and replay protection |
| **Workflows** | Declarative state-machine automations with human-in-the-loop, parallel threads, JSONata expressions, and a validator | When the agent needs to coordinate multi-step work that outlives a single LLM turn |

You don't have to adopt all four. A simple agent starts with Credentials + AI Agents. As the product gets more ambitious you layer in Tools (for write actions in your own systems) and Workflows (for long-running processes).

---

## How a request flows

```
   USER                YOUR APP             INTERACTOR              SKB / EXTERNAL SVCS
    │                     │                     │                          │
    │  "Schedule lunch    │                     │                          │
    │   with Alice Fri"   │                     │                          │
    │ ───────────────────>│                     │                          │
    │                     │  send message       │                          │
    │                     │  external_user_id   │                          │
    │                     │ ───────────────────>│                          │
    │                     │                     │  semantic search:        │
    │                     │                     │  "create calendar event" │
    │                     │                     │ ────────────────────────>│
    │                     │                     │<──────────── capability  │
    │                     │                     │                          │
    │                     │                     │  invoke w/ user's        │
    │                     │                     │  Google Calendar creds   │
    │                     │                     │ ────────────────────────>│
    │                     │                     │   400: bad time format   │
    │                     │                     │<──────────────────────── │
    │                     │                     │  self-heal: coerce time, │
    │                     │                     │  retry, report to SKB    │
    │                     │                     │ ────────────────────────>│
    │                     │                     │<──────────── event_id    │
    │                     │  SSE: response_sent │                          │
    │                     │<─────────────────── │                          │
    │   "Booked Friday    │                     │                          │
    │   12pm with Alice"  │                     │                          │
    │<─────────────────── │                     │                          │
```

You wrote: the prompt, the UI, the user's identity mapping (`external_user_id`). Interactor handled: capability lookup, OAuth token retrieval, parameter coercion, retry on the format error, and event streaming.

---

## Build order

Most teams shipping their first agent on Interactor move through this order. None of it is dogma; the headings link to the section where you'll spend the time.

1. **[Setup and Authentication](integration-guide/02-setup-and-authentication.md)** — Register, get a client_id / client_secret, get an access token. ~30 minutes.
2. **[Credential Management](integration-guide/03-credential-management.md)** — Connect at least one external service (your own Google account is fine) so you have a real user-scoped credential to drive an agent against. ~1 hour.
3. **[AI Agents](integration-guide/04-ai-agents.md)** — Create an assistant, open a room, send a message, get a streamed response. This is the "hello world" of your agent. ~1–2 hours.
4. **[Webhooks and Streaming](integration-guide/06-webhooks-and-streaming.md)** — Wire SSE into your UI so responses arrive incrementally and your front-end stays in sync with the agent's state. ~half a day.
5. **[Workflows](integration-guide/05-workflows.md)** — When you have a multi-step process the agent should orchestrate (approval flows, batch jobs, long-running tasks), define it as a workflow rather than scripting it in your app.
6. **[Expressions](integration-guide/08-expressions.md)** — Reference whenever you need to transform data between workflow steps or condition transitions. JSONata, with a validation endpoint.
7. **[Overview](integration-guide/01-overview.md)** and **[SDK Examples](integration-guide/07-sdk-examples.md)** — The overview covers deployment modes and platform-wide concepts; the SDK examples are working TypeScript and Python clients you can fork.

---

## Quick start

This is the shortest path from zero to a token you can call APIs with.

### 1. Register

```bash
curl -X POST https://auth.interactor.com/api/v1/admin/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "you@company.com",
    "password": "SecureP@ss!",
    "org_name": "your-company"
  }'
```

Check your email and verify. `org_name` is a slug — lowercase, hyphens.

### 2. Create an OAuth client

```bash
# Log in to get an admin token
curl -X POST https://auth.interactor.com/api/v1/admin/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@company.com", "password": "SecureP@ss!"}'

# Use the access_token from the response
curl -X POST https://auth.interactor.com/api/v1/admin/orgs/your-company/applications \
  -H "Authorization: Bearer <admin_access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Agent Backend",
    "description": "Backend service that drives our AI agent",
    "scopes": ["interactor:read", "interactor:write"]
  }'
```

Save the `client_secret` from the response — it's shown once.

### 3. Get an access token

```bash
curl -X POST https://auth.interactor.com/api/v1/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "<client_id>",
    "client_secret": "<client_secret>"
  }'
```

### 4. Call the platform

```bash
curl https://core.interactor.com/api/v1/credentials/summary \
  -H "Authorization: Bearer <access_token>"
```

From here, the [Setup and Authentication guide](integration-guide/02-setup-and-authentication.md) covers token caching, rotation, and the patterns you'll use in production. The [AI Agents guide](integration-guide/04-ai-agents.md) takes you from here to a working assistant.

---

## Base URLs

| Service | URL | Purpose |
|---------|-----|---------|
| Account Server | `https://auth.interactor.com/api/v1` | Auth, org/app management, token issuance |
| Interactor API | `https://core.interactor.com/api/v1` | Credentials, AI agents, workflows, webhooks |

If you're on a single-tenant deployment, both URLs change. Call `https://<your-host>/info` to discover the deployment's configuration (whether auth is required, what callback URL to register with external OAuth providers).

---

## Where things live

| Section | Description |
|---|---|
| [01. Overview](integration-guide/01-overview.md) | Architecture, deployment modes, base URLs, core concepts |
| [02. Setup and Authentication](integration-guide/02-setup-and-authentication.md) | Registration, OAuth clients, token management, config sync |
| [03. Credential Management](integration-guide/03-credential-management.md) | Connecting users' external accounts; OAuth flows |
| [04. AI Agents](integration-guide/04-ai-agents.md) | Assistants, rooms, tools, data sources, approval gates, delegation |
| [05. Workflows](integration-guide/05-workflows.md) | State machines, human-in-the-loop, parallel threads |
| [06. Webhooks and Streaming](integration-guide/06-webhooks-and-streaming.md) | Event subscriptions, SSE, signature verification |
| [07. SDK Examples](integration-guide/07-sdk-examples.md) | TypeScript and Python clients you can fork |
| [08. Expressions](integration-guide/08-expressions.md) | JSONata reference for transforms and conditions |

---

## Getting help

- Read the section that covers what you're stuck on (linked above)
- The [SDK Examples](integration-guide/07-sdk-examples.md) include working code for every major flow
- Email `support@interactor.com`
