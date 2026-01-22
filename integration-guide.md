# Interactor Integration Guide

**Version:** 2.0.0
**Last Updated:** 2026-01-20

---

## Overview

Interactor is a platform that provides three core capabilities for your applications:

| Capability | What It Does |
|------------|--------------|
| **Credential Management** | Securely store and manage OAuth tokens and API keys for external services |
| **AI Agents** | LLM-powered assistants that can use tools and access your data |
| **Workflows** | Execute state-machine based automation with human-in-the-loop support |

---

## Documentation

The integration guide is organized into the following sections:

| Document | Description |
|----------|-------------|
| [01. Overview](integration-guide/01-overview.md) | Platform overview, architecture, and core concepts |
| [02. Setup and Authentication](integration-guide/02-setup-and-authentication.md) | Account registration, OAuth client setup, token management |
| [03. Credential Management](integration-guide/03-credential-management.md) | OAuth flows, token storage, custom OAuth apps |
| [04. AI Agents](integration-guide/04-ai-agents.md) | Assistants, chat rooms, tools, and data sources |
| [05. Workflows](integration-guide/05-workflows.md) | State machines with human-in-the-loop support |
| [06. Webhooks and Streaming](integration-guide/06-webhooks-and-streaming.md) | Event subscriptions and real-time updates (SSE) |
| [07. SDK Examples](integration-guide/07-sdk-examples.md) | Complete TypeScript and Python code examples |

---

## Quick Start

### 1. Get Credentials

```bash
# Register
curl -X POST https://auth.interactor.com/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "you@company.com", "password": "SecureP@ss!", "organization_name": "Your Co"}'

# Create OAuth client (after email verification)
curl -X POST https://auth.interactor.com/api/v1/account/oauth-clients \
  -H "Authorization: Bearer <user_token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "My Backend"}'
```

### 2. Get Access Token

```bash
curl -X POST https://auth.interactor.com/api/v1/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "<client_id>",
    "client_secret": "<client_secret>"
  }'
```

### 3. Call APIs

```bash
curl https://core.interactor.com/api/v1/credentials/summary \
  -H "Authorization: Bearer <access_token>"
```

---

## Base URLs

| Service | URL |
|---------|-----|
| Account Server | `https://auth.interactor.com/api/v1` |
| Interactor API | `https://core.interactor.com/api/v1` |

---

## Feature Priority

When integrating Interactor, we recommend this order:

1. **[Credential Management](integration-guide/03-credential-management.md)** - The foundation for connecting external services
2. **[AI Agents](integration-guide/04-ai-agents.md)** - Add conversational AI with tool use
3. **[Workflows](integration-guide/05-workflows.md)** - Automate multi-step processes

---

## Getting Help

- Check the individual guide sections for detailed API documentation
- Review the [SDK Examples](integration-guide/07-sdk-examples.md) for working code
- Contact support at support@interactor.com
