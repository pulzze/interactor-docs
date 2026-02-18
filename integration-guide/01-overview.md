# Interactor Integration Guide: Overview

**Version:** 2.1.0
**Last Updated:** 2026-02-05

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
│  │    permissions    │     │    your users (using external_user_id)    │   │
│  └───────────────────┘     └─────────────────┬─────────────────────────┘   │
│                                              │                              │
└──────────────────────────────────────────────┼──────────────────────────────┘
                                               │
                                               ▼
                                      ┌─────────────────┐
                                      │   INTERACTOR    │
                                      └─────────────────┘
```

**Key Point:** Interactor does not manage your end users. Your backend authenticates to Interactor using OAuth client credentials, then calls Interactor APIs on behalf of your users. The `external_user_id` parameter isolates each user's data.

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
| [02-setup-and-authentication.md](02-setup-and-authentication.md) | Account setup, OAuth, token management, config sync |
| [03-credential-management.md](03-credential-management.md) | Managing OAuth tokens and API keys for external services |
| [04-ai-agents.md](04-ai-agents.md) | Creating AI assistants, chat rooms, tools, and data sources |
| [05-workflows.md](05-workflows.md) | State-machine workflows with human-in-the-loop |
| [06-webhooks-and-streaming.md](06-webhooks-and-streaming.md) | Event subscriptions and real-time updates |
| [07-sdk-examples.md](07-sdk-examples.md) | Complete code examples in TypeScript and Python |
| [08-expressions.md](08-expressions.md) | JSONata expressions for data transformation and conditions |

---

## Core Concepts

### External User ID (external_user_id)

The `external_user_id` parameter isolates data within your account. Use it to separate your end users:

```
Account: Your Company
├── external_user_id: user_123     → Credentials, rooms for user 123
├── external_user_id: user_456     → Credentials, rooms for user 456
└── external_user_id: shared       → Shared resources
```

Most API calls accept an `external_user_id` parameter to scope data to a specific user within your account. This is also used for per-user billing attribution.

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
| `payment_required` | 402 | Subscription suspended or balance depleted |
| `forbidden` | 403 | Insufficient permissions |
| `not_found` | 404 | Resource not found |
| `validation_error` | 422 | Invalid request parameters |
| `rate_limited` | 429 | Too many requests |
| `limit_exceeded` | 429 | Usage limit reached (subscription or allocation) |
| `internal_error` | 500 | Server error |
| `billing_unavailable` | 503 | Billing server temporarily unavailable |

### Rate Limits

| Endpoint Category | Limit |
|-------------------|-------|
| All API endpoints | 100 requests per 60 seconds |
| Streaming | 10 concurrent connections |

When rate limited, you'll receive a `429 Too Many Requests` response.

---

## Billing Integration

Interactor integrates with the Billing Server for usage tracking, limit enforcement, and per-user quotas. Understanding this architecture helps you manage billing for your end users effectively.

### Service Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              YOUR APPLICATION                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Credential/Agent/Workflow APIs          Subscription/Allocation APIs      │
│   (with external_user_id)                 (per-user quotas and credits)     │
│                  │                                      │                   │
└──────────────────┼──────────────────────────────────────┼───────────────────┘
                   │                                      │
                   ▼                                      ▼
            ┌─────────────┐                      ┌─────────────────┐
            │ INTERACTOR  │─── Usage Reports ───►│ BILLING SERVER  │
            └─────────────┘   (per-user)         └─────────────────┘
```

| Service | Responsibility |
|---------|----------------|
| **Interactor** | Credentials, AI Agents, Workflows, OAuth flows |
| **Billing Server** | Subscriptions, allocations, usage limits, credits |

### How It Works

1. **Your application calls Interactor APIs** with `external_user_id` to identify your end user
2. **Interactor reports usage** to Billing Server with the same `external_user_id`
3. **Billing Server tracks usage** at both subscription and per-user levels
4. **Limits are enforced** before processing requests

### What You Manage Directly

**Through Interactor (this guide):**
- OAuth credentials for external services
- AI agent configurations and chat rooms
- Workflow definitions and instances

**Through Billing Server (separate service):**
- Subscriptions to payment plans
- Per-user allocations (quotas/credits for each `external_user_id`)
- Balance top-ups and credit management
- Usage summaries and billing history

### Per-User Allocations

If you want to set individual limits or credit balances for your end users, you manage allocations directly with the Billing Server:

```bash
# Create an allocation for a specific user
curl -X POST https://billing.interactor.com/api/subscriptions/{subscription_id}/allocations \
  -H "Authorization: Bearer <token>" \
  -d '{
    "external_user_id": "user_123",
    "metric_name": "api_calls",
    "allocation_type": "limit",
    "limit": 1000
  }'
```

When you then call Interactor APIs with `external_user_id=user_123`, the usage is tracked against that user's allocation.

### Related Documentation

For subscription and allocation management, see the **Billing Server Integration Guide**:
- Subscriptions and payment plans
- Per-user allocations (limits and balances)
- Balance-type metrics (prepaid credits)
- Usage reporting and summaries
- Webhook events for billing

---

## Next Steps

Continue to [Setup and Authentication](02-setup-and-authentication.md) to get started with Interactor.
