# Interactor Public Documentation

Developer documentation for teams building AI agents on the Interactor platform.

Interactor is the integration substrate beneath your agent: it provides credential management for the external services your users connect, an agent runtime with tool calling and human-in-the-loop guardrails, declarative workflows for multi-step orchestration, and a Service Knowledge Base that learns new services and self-heals failed calls so your agent gets better without you shipping code for every edge case.

## Start here

- **[Building AI Agents on Interactor](integration-guide.md)** — the entry-point guide. Explains why the platform looks the way it does (self-learning, closed-loop, self-validation) and the recommended build order.

## Deep-dive guides

- [01. Overview](integration-guide/01-overview.md) — architecture, deployment modes, core concepts
- [02. Setup and Authentication](integration-guide/02-setup-and-authentication.md) — registration, OAuth, tokens
- [03. Credential Management](integration-guide/03-credential-management.md) — connecting users' external accounts
- [04. AI Agents](integration-guide/04-ai-agents.md) — assistants, rooms, tools, approval gates
- [05. Workflows](integration-guide/05-workflows.md) — state machines, human-in-the-loop
- [06. Webhooks and Streaming](integration-guide/06-webhooks-and-streaming.md) — events and SSE
- [07. SDK Examples](integration-guide/07-sdk-examples.md) — TypeScript and Python clients
- [08. Expressions](integration-guide/08-expressions.md) — JSONata reference

## Help

Email `support@interactor.com`.
