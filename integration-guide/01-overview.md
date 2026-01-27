# Interactor Integration Guide: Overview

**Version:** 2.0.0
**Last Updated:** 2026-01-27

---

## What is Interactor?

Interactor is a platform that provides three core capabilities for your applications:

| Capability | What It Does |
|------------|--------------|
| **Credential Management** | Securely store and manage OAuth tokens and API keys for external services |
| **AI Agents** | LLM-powered assistants that can use tools and access your data |
| **Workflows** | Execute state-machine based automation with human-in-the-loop support |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           YOUR APPLICATION                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────┐     ┌───────────────────────────────────────────┐   │
│  │  Your Users       │     │  Your Backend                              │   │
│  │                   │     │                                           │   │
│  │  - You manage     │────>│  - Authenticates to Interactor with       │   │
│  │    their auth     │     │    client_id / client_secret              │   │
│  │  - You decide     │     │  - Calls Interactor APIs on behalf of     │   │
│  │    permissions    │     │    your users (using user_ref)            │   │
│  └───────────────────┘     └─────────────────┬─────────────────────────┘   │
│                                              │                              │
└──────────────────────────────────────────────┼──────────────────────────────┘
                                               │
                                               ▼
                                      ┌─────────────────┐
                                      │   INTERACTOR    │
                                      └─────────────────┘
```

**Key Point:** Interactor does not manage your end users. Your backend authenticates to Interactor using OAuth client credentials, then calls Interactor APIs on behalf of your users. The `user_ref` parameter isolates each user's data.

---

## Base URLs

| Service | URL | Purpose |
|---------|-----|---------|
| Account Server | `https://auth.interactor.com/api/v1` | Authentication, user/org management |
| Interactor API | `https://core.interactor.com/api/v1` | Core platform APIs |

---

## Quick Start

**To integrate Interactor into your application:**

1. **Register** - Create an account and organization
2. **Get Credentials** - Create OAuth client credentials (client_id / client_secret)
3. **Authenticate** - Exchange credentials for access tokens
4. **Call APIs** - Use tokens to call Interactor APIs

See [Setup and Authentication](02-setup-and-authentication.md) for detailed instructions.

---

## Documentation Structure

This integration guide is organized into the following sections:

| Document | Description |
|----------|-------------|
| [01-overview.md](01-overview.md) | This document - platform overview |
| [02-setup-and-authentication.md](02-setup-and-authentication.md) | Account setup, OAuth client creation, token management |
| [03-credential-management.md](03-credential-management.md) | Managing OAuth tokens and API keys for external services |
| [04-ai-agents.md](04-ai-agents.md) | Creating AI assistants, chat rooms, tools, and data sources |
| [05-workflows.md](05-workflows.md) | State-machine workflows with human-in-the-loop |
| [06-webhooks-and-streaming.md](06-webhooks-and-streaming.md) | Event subscriptions and real-time updates |
| [07-sdk-examples.md](07-sdk-examples.md) | Complete code examples in TypeScript and Python |

---

## Core Concepts

### User References (user_ref)

The `user_ref` parameter isolates data within your account. Use it to separate your end users:

```
Account: Your Company
├── user_ref: user_123     → Credentials, rooms for user 123
├── user_ref: user_456     → Credentials, rooms for user 456
└── user_ref: shared       → Shared resources
```

Most API calls accept a `user_ref` parameter to scope data to a specific user within your account.

### Authentication Flow

```
┌─────────────────┐       ┌──────────────────┐       ┌─────────────────┐
│  Your Backend   │       │  Account Server  │       │   Interactor    │
└────────┬────────┘       └────────┬─────────┘       └────────┬────────┘
         │                         │                          │
         │ POST /oauth/token       │                          │
         │ client_id + secret      │                          │
         │────────────────────────>│                          │
         │                         │                          │
         │ { access_token: JWT }   │                          │
         │<────────────────────────│                          │
         │                         │                          │
         │ API call + Bearer JWT   │                          │
         │───────────────────────────────────────────────────>│
         │                         │                          │
         │                         │  Verify JWT via JWKS     │
         │                         │<─────────────────────────│
         │                         │                          │
         │ Response                │                          │
         │<───────────────────────────────────────────────────│
```

---

## Health Check

Verify service availability (no authentication required):

```bash
curl https://core.interactor.com/health
```

**Response:**
```json
{
  "status": "ok",
  "timestamp": "2026-01-20T12:00:00Z"
}
```

---

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "validation_error",
    "message": "Invalid request parameters",
    "details": {
      "field": "email",
      "reason": "must be a valid email address"
    }
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `unauthorized` | 401 | Invalid or expired token |
| `forbidden` | 403 | Insufficient permissions |
| `not_found` | 404 | Resource not found |
| `validation_error` | 422 | Invalid request parameters |
| `rate_limited` | 429 | Too many requests |
| `internal_error` | 500 | Server error |

### Rate Limits

| Endpoint Category | Limit |
|-------------------|-------|
| All API endpoints | 100 requests per 60 seconds |
| Streaming | 10 concurrent connections |

When rate limited, you'll receive a `429 Too Many Requests` response.

---

## Next Steps

Continue to [Setup and Authentication](02-setup-and-authentication.md) to get started with Interactor.
