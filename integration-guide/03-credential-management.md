# Credential Management

**Last Updated:** 2026-05-24

Credential Management is Interactor's primary feature for securely storing and managing OAuth tokens and API keys for external services (Google, Slack, Salesforce, etc.).

---

## Overview

Interactor handles the complexity of OAuth:

- **Token Storage** - Encrypted storage of access and refresh tokens
- **Automatic Refresh** - Tokens are refreshed before expiry
- **Multi-tenant Isolation** - `external_user_id` separates different users' credentials
- **Revocation Handling** - Detects when users revoke access

---

## API Reference

### List All Credentials

```bash
curl "https://core.interactor.com/api/v1/credentials?external_user_id=user_123" \
  -H "Authorization: Bearer <token>"
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `external_user_id` | string | Yes | Your end user's identifier |

**Response:**
```json
{
  "data": [
    {
      "id": "cred_abc",
      "external_user_id": "user_123",
      "service_id": "google_calendar",
      "service_name": "Google Calendar",
      "auth_type": "oauth2_authorization_code",
      "status": "active",
      "scopes": ["https://www.googleapis.com/auth/calendar.readonly"],
      "expires_at": "2026-02-01T00:00:00Z",
      "expires_in_seconds": 3540,
      "expired": false,
      "has_refresh_token": true,
      "last_used_at": "2026-01-20T10:00:00Z",
      "created_at": "2026-01-01T00:00:00Z"
    }
  ]
}
```

---

### Get Credentials Summary

Get summary statistics for a user's credentials:

```bash
curl "https://core.interactor.com/api/v1/credentials/summary?external_user_id=user_123" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "total": 3,
    "by_service": {
      "google_calendar": 1,
      "slack": 2
    },
    "by_status": {
      "active": 2,
      "expired": 1
    }
  }
}
```

---

### Get a Specific Credential

```bash
curl https://core.interactor.com/api/v1/credentials/cred_abc \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "id": "cred_abc",
    "external_user_id": "user_123",
    "service_id": "google_calendar",
    "service_name": "Google Calendar",
    "auth_type": "oauth2_authorization_code",
    "status": "active",
    "scopes": ["https://www.googleapis.com/auth/calendar.readonly"],
    "expires_at": "2026-02-01T00:00:00Z",
    "expires_in_seconds": 3540,
    "expired": false,
    "has_refresh_token": true,
    "last_used_at": "2026-01-20T10:00:00Z",
    "created_at": "2026-01-01T00:00:00Z"
  }
}
```

---

### Get Access Token

Retrieve the current access token for a credential.

```bash
curl https://core.interactor.com/api/v1/credentials/cred_abc/token \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "access_token": "ya29.a0AfH6SM...",
    "token_type": "Bearer",
    "expires_at": "2026-01-20T12:00:00Z",
    "expires_in_seconds": 3540,
    "expired": false,
    "has_refresh_token": true
  }
}
```

Use this token to call the external service's API directly.

---

### Force Token Refresh

Manually trigger a token refresh (schedules a background job):

```bash
curl -X POST https://core.interactor.com/api/v1/credentials/cred_abc/refresh \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "message": "Refresh scheduled",
    "credential_id": "cred_abc"
  }
}
```

> **Note:** The refresh happens asynchronously. Wait a few seconds then call `/credentials/{id}/token` to get the updated token.

---

### Delete a Credential

Delete a credential (revokes OAuth tokens if applicable):

```bash
curl -X DELETE https://core.interactor.com/api/v1/credentials/cred_abc \
  -H "Authorization: Bearer <token>"
```

**Response:** `204 No Content`

---

## OAuth Flow

### Initiate OAuth Authorization

Start an OAuth flow to connect an external service:

```bash
curl -X POST https://core.interactor.com/api/v1/oauth/initiate \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "service_id": "google_calendar",
    "external_user_id": "user_123",
    "scopes": ["https://www.googleapis.com/auth/calendar.readonly"],
    "success_redirect_url": "https://yourapp.com/oauth/success"
  }'
```

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `service_id` | string | Yes | Service ID from Service Knowledge Base |
| `external_user_id` | string | Yes | Your end user's identifier |
| `scopes` | array | No | OAuth scopes (defaults to service's default scopes) |
| `redirect_uri` | string | No | Override Interactor's OAuth callback URL (advanced - rarely needed) |
| `success_redirect_url` | string | No | Where to redirect your user after successful authorization |
| `metadata` | object | No | Custom data to store with the flow |

> **Note:** Most integrations only need `success_redirect_url`. The `redirect_uri` parameter is for advanced cases where you need to receive the OAuth callback directly instead of letting Interactor handle it.

**Response (201 Created):**
```json
{
  "data": {
    "flow_id": "flow_xyz",
    "authorization_url": "https://accounts.google.com/o/oauth2/auth?...",
    "state": "state_abc123"
  }
}
```

### Integration Steps

1. Call `/oauth/initiate` to get the authorization URL
2. Redirect your user to `authorization_url`
3. User authorizes on the external service (Google, Slack, etc.)
4. External service redirects to Interactor's callback endpoint
5. Interactor exchanges the code for tokens and stores the credential
6. User is redirected to your `success_redirect_url` with `credential_id` and `service_id` query params

**Success Redirect Example:**
```
https://yourapp.com/oauth/success?credential_id=cred_abc&service_id=google_calendar
```

---

### Check OAuth Flow Status

```bash
curl https://core.interactor.com/api/v1/oauth/status/flow_xyz \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "data": {
    "flow_id": "flow_xyz",
    "status": "awaiting_callback"
  }
}
```

**Status values:**

| Status | Description |
|--------|-------------|
| `initialized` | Flow created, authorization URL being generated |
| `awaiting_callback` | User redirected to provider, waiting for callback |

> **Note:** When the OAuth flow completes, the flow process terminates. Subsequent status checks will return `404 Not Found`. The recommended way to detect completion is via the redirect to your `success_redirect_url` or by subscribing to webhook events.

---

## Custom OAuth Apps

By default, Interactor uses platform OAuth credentials for common services. You can configure your own OAuth app credentials for greater control.

### List OAuth Client Configs

```bash
curl https://core.interactor.com/api/v1/oauth-client-configs \
  -H "Authorization: Bearer <token>"
```

### Create Custom OAuth Config

Use your own Google/Slack/etc OAuth app:

```bash
curl -X POST https://core.interactor.com/api/v1/oauth-client-configs \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "auth_provider": "google",
    "client_id": "your-google-client-id.apps.googleusercontent.com",
    "client_secret": "your-google-client-secret"
  }'
```

### Get Config by Provider

```bash
curl https://core.interactor.com/api/v1/oauth-client-configs/provider/google \
  -H "Authorization: Bearer <token>"
```

### Update Config

```bash
curl -X PUT https://core.interactor.com/api/v1/oauth-client-configs/config_123 \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "client_secret": "new-client-secret"
  }'
```

### Toggle Config (Enable/Disable)

```bash
curl -X POST https://core.interactor.com/api/v1/oauth-client-configs/config_123/toggle \
  -H "Authorization: Bearer <token>"
```

### Delete Config

```bash
curl -X DELETE https://core.interactor.com/api/v1/oauth-client-configs/config_123 \
  -H "Authorization: Bearer <token>"
```

---

## Credential Status Lifecycle

```
                    ┌─────────────┐
                    │   pending   │──────────────┐
                    └──────┬──────┘              │
                           │ OAuth completes     │ OAuth fails
                           ▼                     ▼
                    ┌─────────────┐        ┌─────────────┐
           ┌───────>│   active    │────┐   │   failed    │
           │        └──────┬──────┘    │   └─────────────┘
           │               │           │
   refresh │    token      │    user revokes
  succeeds │   expires or  │           │
           │ refresh fails │           ▼
           │               │    ┌─────────────┐
           │        ┌──────┴──┐ │   revoked   │
           └────────│ expired │ └─────────────┘
                    └─────────┘    (re-auth to recover)
```

Both `expired` and `revoked` are recoverable by re-initiating the OAuth flow, which creates a new `active` credential.

**Status values:**

| Status | Description |
|--------|-------------|
| `pending` | OAuth flow initiated, waiting for user authorization |
| `active` | Credential is valid and ready to use |
| `expired` | Token has expired (may be auto-refreshed) |
| `revoked` | User revoked access or credential was deleted |
| `failed` | OAuth flow or token refresh failed |

---

## Error Handling

| Error Code | Description | Resolution |
|------------|-------------|------------|
| `credential_expired` | OAuth token expired and refresh failed | Re-initiate OAuth flow |
| `credential_revoked` | User revoked access | Re-initiate OAuth flow |
| `invalid_scopes` | Requested scopes not available | Check service documentation |
| `oauth_flow_expired` | OAuth flow timed out | Start new OAuth flow |

---

## Webhook Events

Subscribe to credential events via webhooks:

| Event | Description |
|-------|-------------|
| `credential.ready` | New credential is active and ready to use (fired after OAuth completes) |
| `credential.expired` | Token has expired |
| `credential.revoked` | User revoked access |
| `credential.refresh_failed` | Automatic token refresh failed (re-authorization required) |

See [Webhooks and Streaming](06-webhooks-and-streaming.md) for setup details.

---

## Scope Formats

OAuth providers return scopes in different formats. Interactor normalizes scopes automatically, so you can use either format when requesting credentials:

| Format | Example |
|--------|---------|
| **Short form** | `calendar.readonly` |
| **Full URL** | `https://www.googleapis.com/auth/calendar.readonly` |

Both formats are equivalent. Use whichever is more convenient - Interactor handles the normalization internally when validating scopes for tool execution.

---

## Best Practices

1. **Use external_user_id per user** - Isolate each of your users' credentials
2. **Handle revocation gracefully** - Prompt users to re-authorize when credentials are revoked
3. **Request minimal scopes** - Only request the permissions you need
4. **Use custom OAuth apps for production** - Provides better branding and higher rate limits
5. **Monitor credential health** - Subscribe to webhook events

---

## Next Steps

- [AI Agents](04-ai-agents.md) - AI assistants can use credentials to access external services
- [Workflows](05-workflows.md) - Automate tasks using stored credentials
- [Webhooks](06-webhooks-and-streaming.md) - Get notified of credential status changes
