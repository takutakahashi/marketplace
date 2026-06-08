# agentapi-proxy API Reference

## Table of Contents

- [Session Management Endpoints](#session-management-endpoints)
- [Session Status Monitoring Endpoints](#session-status-monitoring-endpoints)
- [Session Sharing Endpoints](#session-sharing-endpoints)
- [Schedule Management Endpoints](#schedule-management-endpoints)
- [Webhook Management Endpoints](#webhook-management-endpoints)
- [Task Management Endpoints](#task-management-endpoints)
- [Task Group Management Endpoints](#task-group-management-endpoints)
- [Memory Management Endpoints](#memory-management-endpoints)
- [SlackBot Management Endpoints](#slackbot-management-endpoints)
- [Files Management Endpoints](#files-management-endpoints)
- [Credentials Management Endpoints](#credentials-management-endpoints)
- [User & Settings Endpoints](#user--settings-endpoints)
- [Notification Endpoints](#notification-endpoints)
- [Settings Sync (GitHub) Endpoints](#settings-sync-github-endpoints)
- [Session Profiles Endpoints](#session-profiles-endpoints)
- [Sandbox Policies Endpoints](#sandbox-policies-endpoints)
- [Codex Device Auth Endpoints](#codex-device-auth-endpoints)
- [Authentication Endpoints](#authentication-endpoints)

## Session Management Endpoints

### POST /start

Create a new agentapi session.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "environment": {
    "GITHUB_TOKEN": "ghp_token",
    "CUSTOM_VAR": "value"
  },
  "tags": {
    "repository": "agentapi-proxy",
    "branch": "main",
    "env": "production"
  },
  "scope": "user",
  "team_id": "org/team-slug",
  "memory_key": {
    "project": "myapp",
    "env": "production"
  },
  "params": {
    "message": "Initial message to agent",
    "agent_type": "claude-agentapi",
    "oneshot": false,
    "sandbox": {
      "enabled": true,
      "policy_id": "policy-uuid",
      "allowed_domains": ["github.com", "*.example.com"],
      "denied_domains": []
    }
  }
}
```

**Note:** `user_id` is automatically assigned from the authenticated user's token.

**SessionParams fields (`params`):**
- `message`: Initial message to send to the agent after session starts
- `agent_type`: Agent type (`claude-agentapi` is the default)
- `oneshot`: When true, the session auto-deletes after Claude stops responding
- `cycle_message`: Message to send after each Claude stop event (for recurring execution)
- `cycle_max_count`: Maximum number of cycles (requires `cycle_message`)
- `initial_message_wait_second`: Seconds to wait before sending the initial message
- `sandbox`: Network sandbox configuration (see [Sandbox Policies Endpoints](#sandbox-policies-endpoints))

**Response:**
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/start \
  -H "X-API-Key: ap_user_alice_987654321fedcba" \
  -H "Content-Type: application/json" \
  -d '{
    "environment": {
      "GITHUB_TOKEN": "ghp_..."
    },
    "tags": {
      "repository": "my-project"
    }
  }'
```

### GET /search

Search and filter existing sessions.

**Permissions Required:** `session:list`

**Query Parameters:**
- `status`: Filter by session status (e.g., `active`)
- `tag.{key}`: Filter by tag key-value pairs (e.g., `tag.repository=my-repo`)

**Response:**
```json
{
  "sessions": [
    {
      "session_id": "abc123",
      "user_id": "alice",
      "status": "active",
      "started_at": "2024-01-01T12:00:00Z",
      "port": 9000,
      "tags": {
        "repository": "agentapi-proxy",
        "branch": "main"
      }
    }
  ]
}
```

**Examples:**
```bash
# List all sessions
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/search

# Filter by status
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/search?status=active"

# Filter by multiple tags
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/search?tag.repository=my-repo&tag.env=production"
```

**Access Control:**
- Non-admin users can only see their own sessions
- Admin users can see all sessions

### DELETE /sessions/:sessionId

Delete a specific session.

**Permissions Required:** `session:delete`

**Response:**
```json
{
  "ok": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/sessions/550e8400-e29b-41d4-a716-446655440000 \
  -H "X-API-Key: ap_user_alice_987654321fedcba"
```

### GET/POST /:sessionId/:path

Proxy requests to the agentapi instance for the specified session.

**Permissions Required:** `session:access`

**Parameters:**
- `sessionId` (path): Session ID
- `path` (path): Path to forward to agentapi

**Description:**
- All requests to `/{sessionId}/*` are proxied to the corresponding agentapi server instance
- Supports GET and POST methods
- Users can only access their own sessions

**Response Codes:**
- `200`: Response from agentapi server
- `403`: Forbidden - can only access own sessions
- `404`: Session not found
- `502`: Bad Gateway - agentapi server unavailable

**Examples:**
```bash
# Send a message to the agent (POST)
curl -X POST https://api.example.com/550e8400-e29b-41d4-a716-446655440000/message \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello", "type": "user"}'

# Get session status (GET)
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/550e8400-e29b-41d4-a716-446655440000/status

# Get conversation history (GET)
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/550e8400-e29b-41d4-a716-446655440000/messages
```

### GET /sessions/:sessionId/sandbox-domains

Get the list of domains accessed by a sandboxed session.

**Permissions Required:** `session:access`

**Path Parameters:**
- `sessionId` (required): Session ID

**Description:**
- Returns the list of domains that the session's network filter proxy has seen, split into allowed (accessed) and denied (blocked) sets
- Only available for Kubernetes sessions with a sandbox sidecar
- Returns `503` when the network filter is not running

**Response:**
```json
{
  "allowed": ["github.com", "api.example.com"],
  "denied": ["blocked.example.com"]
}
```

**Response Codes:**
- `200`: Domain lists from the network filter
- `401`: Unauthorized
- `403`: Forbidden - can only access own sessions
- `404`: Session not found
- `501`: Not implemented for this session type
- `503`: Network filter not available for this session

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/sessions/550e8400-e29b-41d4-a716-446655440000/sandbox-domains
```

## Session Status Monitoring Endpoints

These endpoints enable real-time monitoring of session status changes and message updates. They provide both Server-Sent Events (SSE) streaming and long-polling mechanisms for different use cases.

### GET /sessions/status/stream

Stream all session status changes via Server-Sent Events (SSE).

**Permissions Required:** `session:read`

**Description:**
- Opens a Server-Sent Events stream that pushes a `SessionStatusEvent` whenever any session accessible to the authenticated user changes status
- A heartbeat comment (`: heartbeat`) is sent every 30 seconds to keep the connection alive
- The stream stays open until the client disconnects

**Response:**
Server-Sent Events stream. Each event line:
```
data: {"session_id":"abc123","status":"active","timestamp":"2026-05-02T12:00:00Z"}

```

**Example:**
```bash
curl -N -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/sessions/status/stream
```

**Response Codes:**
- `200`: SSE stream of SessionStatusEvent objects
- `401`: Unauthorized
- `501`: Not implemented for this session manager type

### GET /sessions/status/wait

Long-poll for next session status change.

**Permissions Required:** `session:read`

**Description:**
Blocks until any session accessible to the authenticated user changes status, or until the timeout expires. Returns a `SessionStatusEvent` on change, or `{"events": []}` on timeout.

**Query Parameters:**
- `timeout`: Maximum wait time in seconds (default: 30, max: 60)

**Response:**
```json
{
  "session_id": "abc123",
  "status": "active",
  "timestamp": "2026-05-02T12:00:00Z"
}
```

Or on timeout:
```json
{
  "events": []
}
```

**Example:**
```bash
# Wait up to 30 seconds for status change
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/sessions/status/wait?timeout=30"
```

**Response Codes:**
- `200`: SessionStatusEvent or empty events array
- `401`: Unauthorized
- `501`: Not implemented for this session manager type

### GET /sessions/:sessionId/messages/wait

Long-poll for message updates in a specific session.

**Permissions Required:** `session:access`

**Description:**
Blocks until a `message_update` event is received from the agentapi backend for the specified session, or until the timeout expires. Returns `{"updated": true, "session_id": "...", "timestamp": "..."}` on update, or `{"updated": false}` on timeout. Clients should immediately re-issue the request after each response to maintain continuous notification.

**Path Parameters:**
- `sessionId` (required): Session ID

**Query Parameters:**
- `timeout`: Maximum wait time in seconds (default: 30, max: 60)
- `since`: Timestamp of the last known message update. If the session has received a message_update after this time, the response is returned immediately without waiting. Accepts Unix timestamp in milliseconds (integer) or RFC3339 string. Use the `timestamp` value from the previous response.

**Response:**

Message update received:
```json
{
  "updated": true,
  "session_id": "abc123",
  "timestamp": "2026-05-02T12:00:00Z"
}
```

Timeout elapsed with no update:
```json
{
  "updated": false
}
```

**Example:**
```bash
# Wait for message updates
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/sessions/abc123/messages/wait?timeout=30"

# Wait with since parameter for catch-up
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/sessions/abc123/messages/wait?timeout=30&since=2026-05-02T11:00:00Z"
```

**Response Codes:**
- `200`: Message update status
- `401`: Unauthorized
- `403`: Forbidden - can only access own sessions
- `404`: Session not found
- `501`: Not implemented for this session manager type

**Access Control:**
- Users can only monitor their own sessions

## Session Sharing Endpoints

### POST /sessions/:sessionId/share

Create a share token for read-only access to a session.

**Permissions Required:** `session:access`

**Response:**
```json
{
  "share_token": "sh_abc123def456"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/sessions/550e8400-e29b-41d4-a716-446655440000/share \
  -H "X-API-Key: YOUR_API_KEY"
```

### GET /sessions/:sessionId/share

Get the share token for a session.

**Permissions Required:** `session:access`

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/sessions/550e8400-e29b-41d4-a716-446655440000/share
```

### DELETE /sessions/:sessionId/share

Revoke the share token for a session.

**Permissions Required:** `session:access`

**Example:**
```bash
curl -X DELETE https://api.example.com/sessions/550e8400-e29b-41d4-a716-446655440000/share \
  -H "X-API-Key: YOUR_API_KEY"
```

### GET/HEAD /s/:shareToken/:path

Access a shared session in read-only mode (no authentication required).

**Security:** No authentication required

**Parameters:**
- `shareToken` (path): Share token (32-character hex string)
- `path` (path): Path to forward to agentapi

**Description:**
- All GET/HEAD requests to `/s/{shareToken}/*` are proxied to the corresponding agentapi server instance in read-only mode
- Only GET and HEAD methods are allowed for shared sessions
- POST/PUT/DELETE methods are not allowed and will return 403 Forbidden

**Response Codes:**
- `200`: Response from agentapi server
- `403`: Shared sessions are read-only (POST/PUT/DELETE not allowed)
- `404`: Invalid share token or session not found
- `410`: Share link has expired
- `502`: Bad Gateway - agentapi server unavailable

**Example:**
```bash
curl https://api.example.com/s/sh_abc123def456/messages
```

## Schedule Management Endpoints

### POST /schedules

Create a new schedule for delayed or recurring session execution.

**Permissions Required:** `session:create`

**Fields:**
- `name` (required): Schedule name
- `scheduled_at`: ISO 8601 timestamp for one-time execution (either this or `cron_expr` must be set)
- `cron_expr`: Cron expression for recurring execution (either this or `scheduled_at` must be set)
- `timezone`: Timezone for the schedule (default: `Asia/Tokyo`)
- `scope`: `user` (default) or `team`
- `team_id`: Required when `scope` is `team`
- `session_config`: Configuration for created sessions

**Request Body:**
```json
{
  "name": "Daily Standup",
  "cron_expr": "0 9 * * 1-5",
  "session_config": {
    "tags": {
      "repository": "org/standup-bot"
    },
    "params": {
      "message": "Generate daily standup"
    }
  }
}
```

**Response:**
```json
{
  "id": "schedule-abc123",
  "name": "Daily Standup",
  "status": "active",
  "cron_expr": "0 9 * * 1-5",
  "created_at": "2024-01-01T12:00:00Z"
}
```

**Examples:**

One-time delayed execution:
```bash
curl -X POST https://api.example.com/schedules \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Code Review",
    "scheduled_at": "2025-01-15T14:00:00Z",
    "session_config": {
      "tags": {
        "repository": "org/repo"
      },
      "params": {
        "message": "Review PRs"
      }
    }
  }'
```

Recurring execution (cron):
```bash
curl -X POST https://api.example.com/schedules \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Daily Standup",
    "cron_expr": "0 9 * * 1-5",
    "session_config": {
      "tags": {
        "repository": "org/standup-bot"
      },
      "params": {
        "message": "Generate daily standup"
      }
    }
  }'
```

### GET /schedules

List schedules. Non-admin users can see their own schedules and team-scoped schedules they have access to.

**Permissions Required:** `session:list`

**Query Parameters:**
- `status`: Filter by schedule status (e.g., `active`, `paused`)
- `scope`: Filter by resource scope (e.g., `user`, `team`)
- `team_id`: Filter by team ID (e.g., `org/team-slug`)

**Response:**
```json
{
  "schedules": [
    {
      "id": "schedule-abc123",
      "name": "Daily Standup",
      "status": "active",
      "cron_expr": "0 9 * * 1-5",
      "created_at": "2024-01-01T12:00:00Z",
      "next_run": "2024-01-02T09:00:00Z"
    }
  ]
}
```

**Examples:**
```bash
# List all schedules
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/schedules

# Filter by status
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/schedules?status=active"

# Filter by team
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/schedules?team_id=org/my-team"
```

### GET /schedules/:id

Get a specific schedule by ID.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "id": "schedule-abc123",
  "name": "Daily Standup",
  "status": "active",
  "cron_expr": "0 9 * * 1-5",
  "session_config": {
    "tags": {
      "repository": "org/standup-bot"
    },
    "params": {
      "message": "Generate daily standup"
    }
  },
  "created_at": "2024-01-01T12:00:00Z",
  "next_run": "2024-01-02T09:00:00Z"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/schedules/schedule-abc123
```

**Access Control:**
- Users can only access their own schedules
- Admin users can access all schedules

### PUT /schedules/:id

Update an existing schedule.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "Updated Schedule Name",
  "status": "paused",
  "cron_expr": "0 10 * * 1-5"
}
```

**Response:**
```json
{
  "id": "schedule-abc123",
  "name": "Updated Schedule Name",
  "status": "paused",
  "cron_expr": "0 10 * * 1-5",
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/schedules/schedule-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Schedule Name",
    "status": "paused"
  }'
```

**Access Control:**
- Users can only update their own schedules

### DELETE /schedules/:id

Delete a schedule by ID.

**Permissions Required:** `session:delete`

**Response:**
```json
{
  "message": "Schedule deleted successfully",
  "id": "schedule-abc123"
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/schedules/schedule-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- Users can only delete their own schedules

### POST /schedules/:id/trigger

Manually trigger a schedule to immediately create a new session.

**Permissions Required:** `session:create`

**Response:**
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "triggered_at": "2024-01-02T15:30:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/schedules/schedule-abc123/trigger \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- Users can only trigger their own schedules

## Webhook Management Endpoints

Webhooks enable automatic session creation when events occur in external systems (GitHub, Slack, Datadog, custom services).

### POST /webhooks

Create a new webhook for GitHub or custom events.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "Pull Request Reviewer",
  "type": "github",
  "scope": "user",
  "github": {
    "allowed_events": ["pull_request"],
    "allowed_repositories": ["owner/repo"]
  },
  "triggers": [
    {
      "name": "PR opened",
      "enabled": true,
      "conditions": {
        "github": {
          "events": ["pull_request"],
          "actions": ["opened", "synchronize"],
          "base_branches": ["main"],
          "draft": false
        }
      },
      "session_config": {
        "initial_message_template": "Review PR #{{.pull_request.number}}: {{.pull_request.title}}",
        "tags": {
          "repository": "{{.repository.full_name}}",
          "pr": "{{.pull_request.number}}"
        }
      }
    }
  ]
}
```

**Fields:**
- `name` (required): Webhook name
- `type` (required): `github` or `custom`
- `scope`: `user` (default) or `team`
- `team_id`: Required when `scope` is `team`
- `secret`: Custom secret for HMAC signature verification (optional, auto-generated if not provided)
- `github`: GitHub-specific configuration (for type=github)
  - `allowed_events`: List of allowed GitHub events
  - `allowed_repositories`: Repository patterns (e.g., `myorg/*`)
- `triggers` (required): Array of trigger configurations
  - `name`: Trigger name
  - `enabled`: Whether trigger is active
  - `conditions`: Conditions for matching events
  - `session_config`: Configuration for created sessions
- `signature_header`: HTTP header containing the signature (default: `X-Signature`)
- `signature_type`: Signature verification type - `hmac` (default) or `static`
- `signature_prefix`: Prefix to strip from signature header before verification (auto-detected if empty)
- `max_sessions`: Maximum concurrent sessions for this webhook (default: 10, min: 1, max: 100)

**Response:**
```json
{
  "id": "webhook-123",
  "name": "Pull Request Reviewer",
  "type": "github",
  "status": "active",
  "webhook_url": "https://api.example.com/hooks/github/webhook-123",
  "secret": "generated-secret-key",
  "created_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/webhooks \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "PR Review Bot",
    "type": "github",
    "github": {
      "allowed_events": ["pull_request"],
      "allowed_repositories": ["myorg/*"]
    },
    "triggers": [{
      "name": "Review PRs",
      "conditions": {
        "github": {
          "events": ["pull_request"],
          "actions": ["opened"]
        }
      },
      "session_config": {
        "initial_message_template": "Review PR #{{.pull_request.number}}"
      }
    }]
  }'
```

### GET /webhooks

List webhooks accessible to the authenticated user.

**Permissions Required:** `session:list`

**Query Parameters:**
- `type`: Filter by webhook type (`github`, `custom`)
- `status`: Filter by status (`active`, `paused`, `disabled`)
- `scope`: Filter by resource scope (`user`, `team`)
- `team_id`: Filter by team ID

**Response:**
```json
{
  "webhooks": [
    {
      "id": "webhook-123",
      "name": "PR Review Bot",
      "type": "github",
      "status": "active",
      "webhook_url": "https://api.example.com/hooks/github/webhook-123",
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-02T10:00:00Z"
    }
  ]
}
```

**Example:**
```bash
# List all webhooks
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/webhooks

# Filter by type
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/webhooks?type=github"
```

**Access Control:**
- Non-admin users can see their own webhooks and team-scoped webhooks they have access to
- Admin users can see all webhooks

### GET /webhooks/:id

Get a specific webhook by ID.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "id": "webhook-123",
  "name": "PR Review Bot",
  "type": "github",
  "status": "active",
  "scope": "user",
  "owner_id": "alice",
  "webhook_url": "https://api.example.com/hooks/github/webhook-123",
  "github": {
    "allowed_events": ["pull_request"],
    "allowed_repositories": ["myorg/*"]
  },
  "triggers": [...],
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-02T10:00:00Z"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/webhooks/webhook-123
```

**Access Control:**
- Users can only access their own webhooks

### PUT /webhooks/:id

Update an existing webhook.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "Updated Webhook Name",
  "status": "paused",
  "triggers": [...]
}
```

**Response:**
```json
{
  "id": "webhook-123",
  "name": "Updated Webhook Name",
  "status": "paused",
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/webhooks/webhook-123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Name", "status": "paused"}'
```

**Access Control:**
- Users can only update their own webhooks

### DELETE /webhooks/:id

Delete a webhook by ID.

**Permissions Required:** `session:delete`

**Response:**
```json
{
  "message": "Webhook deleted successfully",
  "id": "webhook-123"
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/webhooks/webhook-123 \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- Users can only delete their own webhooks

### POST /webhooks/:id/regenerate-secret

Generate a new secret for webhook signature verification.

**Permissions Required:** `session:create`

**Response:**
```json
{
  "id": "webhook-123",
  "secret": "new-generated-secret-key"
}
```

**Note:** The new secret is shown only once. Update your webhook configuration in the external service immediately.

**Example:**
```bash
curl -X POST https://api.example.com/webhooks/webhook-123/regenerate-secret \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- Users can only regenerate secrets for their own webhooks

### POST /webhooks/:id/trigger

Manually trigger a webhook with a test payload.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "payload": {
    "event": "test_event",
    "data": "test_data"
  },
  "dry_run": true
}
```

**Fields:**
- `payload` (required): Test payload to evaluate
- `dry_run`: If true, only evaluates triggers without creating a session (default: false)

**Response:**
```json
{
  "matched": true,
  "trigger_id": "trigger-456",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "message": "Trigger matched and session created"
}
```

**Example:**
```bash
# Dry run test
curl -X POST https://api.example.com/webhooks/webhook-123/trigger \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"payload": {"test": "data"}, "dry_run": true}'

# Actual trigger
curl -X POST https://api.example.com/webhooks/webhook-123/trigger \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"payload": {"test": "data"}, "dry_run": false}'
```

**Access Control:**
- Users can only trigger their own webhooks

### POST /hooks/github/:id

Receive webhook payloads from GitHub (webhook endpoint).

**Security:** No authentication required (uses webhook signature verification)

**Note:** This is the webhook URL that should be configured in GitHub. The ID is the webhook ID returned when creating the webhook.

**Example GitHub webhook configuration:**
- Payload URL: `https://api.example.com/hooks/github/webhook-123`
- Content type: `application/json`
- Secret: Use the secret provided when creating the webhook

### POST /hooks/custom/:id

Receive webhook payloads from custom services (Slack, Datadog, PagerDuty, etc.).

**Security:** No authentication required (uses webhook signature verification)

**Note:** This is the webhook URL that should be configured in external services. The ID is the webhook ID returned when creating the webhook.

### POST /hooks/slack/:id

Receive Slack event payloads via the Slack Events API.

**Security:** No authentication required (uses Slack signature verification)

**Parameters:**
- `id` (path): SlackBot ID (UUID) or 'default' for server-configured default bot

**Request Body:**
```json
{
  "type": "event_callback",
  "event": {
    "type": "message",
    "channel": "C01234567",
    "user": "U01234567",
    "text": "Hello, world!",
    "ts": "1628000000.000000",
    "thread_ts": "1628000000.000000"
  }
}
```

**Response:**
```json
{
  "challenge": "challenge-token-for-url-verification"
}
```

**Description:**
- Receives Slack Events API payloads
- Verifies Slack v0 HMAC-SHA256 signature
- Handles URL verification challenges
- Creates or reuses a session per Slack thread
- Triggers the claude-posts sidecar for Slack thread replies

**Note:** This is the webhook URL that should be configured in Slack Events API. The ID is either a registered SlackBot UUID or 'default' to use the server startup configuration.

## Task Management Endpoints

Tasks are units of work associated with a session. They can be used to track work items for both agents and users.

### Task Object

```json
{
  "id": "uuid",
  "title": "string",
  "description": "string",
  "status": "todo | done",
  "task_type": "agent | user",
  "scope": "user | team",
  "owner_id": "string",
  "session_id": "string",
  "group_id": "string",
  "team_id": "string",
  "links": [
    {
      "id": "uuid",
      "url": "https://example.com",
      "title": "optional title"
    }
  ],
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

### POST /tasks

Create a new task.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "title": "Review PR #123",
  "description": "Please review and approve the pull request",
  "task_type": "user",
  "scope": "user",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "group_id": "optional-group-id",
  "team_id": "optional-team-id",
  "links": [
    {
      "url": "https://github.com/owner/repo/pull/123",
      "title": "PR #123"
    }
  ]
}
```

**Fields:**
- `title` (required): Task title
- `task_type` (required): `agent` (default) or `user`
- `scope` (required): `user` (default) or `team`
- `session_id`: ID of the session to associate with (optional)
- `description`: Optional description
- `group_id`: Optional group ID for grouping tasks
- `team_id`: Required when `scope` is `team`
- `links`: Optional array of associated URLs

**Response:**
```json
{
  "id": "1b81fae1-a266-4538-a66c-2b0b0e274a81",
  "title": "Review PR #123",
  "description": "Please review and approve the pull request",
  "status": "todo",
  "task_type": "user",
  "scope": "user",
  "owner_id": "alice",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "links": [
    {
      "id": "6c7a10ec-feb5-4d2f-9b84-9df8f50f0b1d",
      "url": "https://github.com/owner/repo/pull/123",
      "title": "PR #123"
    }
  ],
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/tasks \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Review PR #123",
    "task_type": "user",
    "scope": "user",
    "session_id": "550e8400-e29b-41d4-a716-446655440000",
    "links": [{"url": "https://github.com/owner/repo/pull/123", "title": "PR #123"}]
  }'
```

**CLI equivalent:**
```bash
agentapi-proxy client task create \
  --endpoint https://api.example.com \
  --session-id SESSION_ID \
  --title "Review PR #123" \
  --task-type user \
  --scope user \
  --link "https://github.com/owner/repo/pull/123|PR #123"
```

### GET /tasks

List tasks with optional filters.

**Permissions Required:** `session:list`

**Query Parameters:**
- `status`: Filter by status (`todo`, `done`)
- `task_type`: Filter by type (`agent`, `user`)
- `scope`: Filter by scope (`user`, `team`)
- `team_id`: Filter by team ID
- `group_id`: Filter by group ID

**Response:**
```json
{
  "tasks": [
    {
      "id": "1b81fae1-a266-4538-a66c-2b0b0e274a81",
      "title": "Review PR #123",
      "status": "todo",
      "task_type": "user",
      "scope": "user",
      "owner_id": "alice",
      "session_id": "550e8400-e29b-41d4-a716-446655440000",
      "links": [],
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ],
  "total": 1
}
```

**Examples:**
```bash
# List all tasks
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/tasks

# Filter pending tasks
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/tasks?status=todo"

# Filter by type
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/tasks?task_type=user"
```

**CLI equivalent:**
```bash
agentapi-proxy client task list \
  --endpoint https://api.example.com \
  --session-id SESSION_ID \
  --status todo \
  --task-type user
```

**Access Control:**
- Non-admin users can only see their own tasks
- Admin users can see all tasks

### GET /tasks/:taskId

Get a specific task by ID.

**Permissions Required:** `session:read`

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/tasks/1b81fae1-a266-4538-a66c-2b0b0e274a81
```

**CLI equivalent:**
```bash
agentapi-proxy client task get TASK_ID \
  --endpoint https://api.example.com \
  --session-id SESSION_ID
```

### PUT /tasks/:taskId

Update an existing task. Use the CLI for this operation.

**Permissions Required:** `session:create`

**Updatable Fields:**
- `title`: New title
- `description`: New description
- `status`: New status (`todo` or `done`)
- `group_id`: New group ID (set to empty string to remove from group)
- `session_id`: New session ID to associate with
- `links`: Replaces all existing links when present

**CLI (recommended):**
```bash
# Mark a task as done
agentapi-proxy client task update TASK_ID \
  --endpoint https://api.example.com \
  --session-id SESSION_ID \
  --status done

# Update title and description
agentapi-proxy client task update TASK_ID \
  --endpoint https://api.example.com \
  --session-id SESSION_ID \
  --title "New title" \
  --description "New description"
```

### DELETE /tasks/:taskId

Delete a task by ID.

**Permissions Required:** `session:delete`

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/tasks/1b81fae1-a266-4538-a66c-2b0b0e274a81 \
  -H "X-API-Key: YOUR_API_KEY"
```

**CLI equivalent:**
```bash
agentapi-proxy client task delete TASK_ID \
  --endpoint https://api.example.com \
  --session-id SESSION_ID
```

## Task Group Management Endpoints

Task groups provide a way to organize and manage related tasks together.

### POST /task-groups

Create a new task group.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "Sprint 23 Tasks",
  "description": "Tasks for the current sprint",
  "scope": "user",
  "team_id": "optional-team-id"
}
```

**Fields:**
- `name` (required): Group name
- `description`: Optional description
- `scope`: `user` (default) or `team`
- `team_id`: Required when `scope` is `team`

**Response:**
```json
{
  "id": "group-abc123",
  "name": "Sprint 23 Tasks",
  "description": "Tasks for the current sprint",
  "scope": "user",
  "owner_id": "alice",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/task-groups \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sprint 23 Tasks",
    "description": "Tasks for the current sprint",
    "scope": "user"
  }'
```

### GET /task-groups

List task groups accessible to the authenticated user.

**Permissions Required:** `session:list`

**Query Parameters:**
- `scope`: Filter by scope (`user`, `team`)
- `team_id`: Filter by team ID (required when scope=team)

**Response:**
```json
{
  "task_groups": [
    {
      "id": "group-abc123",
      "name": "Sprint 23 Tasks",
      "description": "Tasks for the current sprint",
      "scope": "user",
      "owner_id": "alice",
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ],
  "total": 1
}
```

**Examples:**
```bash
# List all task groups
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/task-groups

# Filter by scope
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/task-groups?scope=user"
```

**Access Control:**
- Non-admin users can only see their own task groups
- Admin users can see all task groups

### GET /task-groups/:groupId

Get a specific task group by ID.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "id": "group-abc123",
  "name": "Sprint 23 Tasks",
  "description": "Tasks for the current sprint",
  "scope": "user",
  "owner_id": "alice",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/task-groups/group-abc123
```

### PUT /task-groups/:groupId

Update an existing task group.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "Updated Group Name",
  "description": "Updated description"
}
```

**Note:** Omitted fields are not changed.

**Response:**
```json
{
  "id": "group-abc123",
  "name": "Updated Group Name",
  "description": "Updated description",
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/task-groups/group-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Group Name"}'
```

### DELETE /task-groups/:groupId

Delete a task group by ID.

**Permissions Required:** `session:delete`

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/task-groups/group-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

## Memory Management Endpoints

Memory entries allow storing and retrieving contextual information for agents and users. Supports user-scoped (private) and team-scoped (shared) memories.

### POST /memories

Create a new memory entry.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "title": "Project conventions",
  "content": "Always use TypeScript for new files. Follow ESLint rules.",
  "scope": "user",
  "team_id": "optional-team-id"
}
```

**Fields:**
- `title` (required): Memory title
- `content` (required): Memory content
- `scope`: `user` (default) or `team`
- `team_id`: Required when `scope` is `team`

**Response:**
```json
{
  "id": "mem-abc123",
  "title": "Project conventions",
  "content": "Always use TypeScript for new files. Follow ESLint rules.",
  "scope": "user",
  "owner_id": "alice",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/memories \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Project conventions",
    "content": "Always use TypeScript for new files.",
    "scope": "user"
  }'
```

### GET /memories

List memory entries accessible to the authenticated user.

**Permissions Required:** `session:list`

**Query Parameters:**
- `scope`: Filter by scope (`user`, `team`)
- `team_id`: Filter by team ID (required when scope=team)
- `q`: Full-text search query (searches title and content, case-insensitive)

**Response:**
```json
{
  "memories": [
    {
      "id": "mem-abc123",
      "title": "Project conventions",
      "content": "Always use TypeScript for new files.",
      "scope": "user",
      "owner_id": "alice",
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ],
  "total": 1
}
```

**Examples:**
```bash
# List all memories
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/memories

# Search memories
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/memories?q=typescript"

# Filter by team
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/memories?scope=team&team_id=myorg/team-slug"
```

**Access Control:**
- User-scoped: Only the owner can see their memories
- Team-scoped: All team members can see the memories
- Admin privilege does NOT bypass these restrictions

### GET /memories/:memoryId

Get a specific memory entry by ID.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "id": "mem-abc123",
  "title": "Project conventions",
  "content": "Always use TypeScript for new files.",
  "scope": "user",
  "owner_id": "alice",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/memories/mem-abc123
```

**Access Control:**
- User-scoped: Only the owner can access
- Team-scoped: Only team members can access
- Admin privilege does NOT bypass these restrictions

### PUT /memories/:memoryId

Update an existing memory entry.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "title": "Updated conventions",
  "content": "Updated content"
}
```

**Note:** Omitted fields are not changed.

**Response:**
```json
{
  "id": "mem-abc123",
  "title": "Updated conventions",
  "content": "Updated content",
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/memories/mem-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated conventions"}'
```

**Access Control:**
- User-scoped: Only the owner can update
- Team-scoped: Only team members can update

### DELETE /memories/:memoryId

Delete a memory entry by ID.

**Permissions Required:** `session:delete`

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/memories/mem-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- User-scoped: Only the owner can delete
- Team-scoped: Only team members can delete

## SlackBot Management Endpoints

SlackBot configurations enable automated session creation in response to Slack events. Each SlackBot corresponds to a Slack App installation and receives events via Socket Mode (WebSocket).

### POST /slackbots

Create a new SlackBot configuration.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "My Slack Bot",
  "scope": "user",
  "team_id": "optional-team-id",
  "bot_token_secret_name": "my-slack-bot-token",
  "bot_token_secret_key": "bot-token",
  "allowed_channel_names": ["dev", "backend"],
  "session_config": {
    "initial_message_template": "New Slack message from {{.event.user}} in <#{{.event.channel}}>: {{.event.text}}",
    "tags": {
      "channel": "{{.event.channel}}"
    }
  }
}
```

**Fields:**
- `name` (required): SlackBot name
- `scope`: `user` (default) or `team`
- `team_id`: Required when `scope` is `team`
- `bot_token_secret_name`: Kubernetes secret name containing the bot token
- `bot_token_secret_key`: Key within the secret containing the bot token
- `allowed_channel_names`: Optional list of allowed channel names (without #)
- `session_config`: Configuration for created sessions
  - `initial_message_template`: Go template for initial message
  - `tags`: Tags to apply to created sessions

**Response:**
```json
{
  "id": "slackbot-abc123",
  "name": "My Slack Bot",
  "status": "active",
  "scope": "user",
  "owner_id": "alice",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/slackbots \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Team Bot",
    "scope": "team",
    "team_id": "myorg/backend",
    "bot_token_secret_name": "my-slack-bot-token",
    "bot_token_secret_key": "bot-token",
    "allowed_channel_names": ["dev", "backend"],
    "session_config": {
      "initial_message_template": "{{.event.text}}",
      "tags": {
        "channel": "{{.event.channel}}"
      }
    }
  }'
```

### GET /slackbots

List SlackBots accessible to the authenticated user.

**Permissions Required:** `session:list`

**Query Parameters:**
- `status`: Filter by SlackBot status (e.g., `active`, `inactive`)
- `scope`: Filter by resource scope (`user`, `team`)
- `team_id`: Filter by team ID

**Response:**
```json
{
  "slackbots": [
    {
      "id": "slackbot-abc123",
      "name": "My Slack Bot",
      "status": "active",
      "scope": "user",
      "owner_id": "alice",
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ]
}
```

**Examples:**
```bash
# List all SlackBots
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/slackbots

# Filter by scope
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/slackbots?scope=team"

# Filter by team
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/slackbots?team_id=myorg/backend"
```

**Access Control:**
- Non-admin users can see their own SlackBots and team-scoped SlackBots they have access to
- Admin users can see all SlackBots

### GET /slackbots/:id

Get a specific SlackBot by ID.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "id": "slackbot-abc123",
  "name": "My Slack Bot",
  "status": "active",
  "scope": "user",
  "owner_id": "alice",
  "bot_token_secret_name": "my-slack-bot-token",
  "bot_token_secret_key": "bot-token",
  "allowed_channel_names": ["dev", "backend"],
  "session_config": {
    "initial_message_template": "{{.event.text}}",
    "tags": {
      "channel": "{{.event.channel}}"
    }
  },
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/slackbots/slackbot-abc123
```

**Access Control:**
- Users can only access their own SlackBots and team-scoped SlackBots they have access to

### PUT /slackbots/:id

Update an existing SlackBot.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "Updated Bot Name",
  "status": "inactive",
  "allowed_channel_names": ["dev", "backend", "general"]
}
```

**Note:** Omitted fields are not changed.

**Response:**
```json
{
  "id": "slackbot-abc123",
  "name": "Updated Bot Name",
  "status": "inactive",
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/slackbots/slackbot-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Bot Name"}'
```

**Access Control:**
- Users can only update their own SlackBots
- Team members can update team-scoped SlackBots

### DELETE /slackbots/:id

Delete a SlackBot by ID.

**Permissions Required:** `session:delete`

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/slackbots/slackbot-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- Users can only delete their own SlackBots
- Team members can delete team-scoped SlackBots

## Files Management Endpoints

The Files endpoints allow users to register arbitrary files (e.g., SSH keys, configuration files) that will be placed inside agent sessions at startup. Files are stored securely and automatically mounted in agent containers.

### POST /files

Create a new user file that will be placed in agent sessions.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "SSH Private Key",
  "path": "/home/agentapi/.ssh/id_rsa",
  "content": "-----BEGIN OPENSSH PRIVATE KEY-----\n...",
  "permissions": "0600"
}
```

**Fields:**
- `name`: Human-readable display name (optional)
- `path` (required): Destination path inside the agent container (e.g., `/home/agentapi/.ssh/id_rsa`)
- `content`: File content (plain text or base64-encoded)
- `permissions`: Permissions hint (informational, e.g., `"0600"`)

**Response:**
```json
{
  "id": "file-abc123",
  "name": "SSH Private Key",
  "path": "/home/agentapi/.ssh/id_rsa",
  "permissions": "0600",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/files \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "SSH Private Key",
    "path": "/home/agentapi/.ssh/id_rsa",
    "content": "-----BEGIN OPENSSH PRIVATE KEY-----\n...",
    "permissions": "0600"
  }'
```

### GET /files

List all files registered for the authenticated user.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "files": [
    {
      "id": "file-abc123",
      "name": "SSH Private Key",
      "path": "/home/agentapi/.ssh/id_rsa",
      "permissions": "0600",
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ]
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/files
```

**Note:** The file `content` is not included in the list response for security reasons. Use the individual file GET endpoint to retrieve content.

### GET /files/:fileId

Get a specific file by ID, including its content.

**Permissions Required:** `session:read`

**Path Parameters:**
- `fileId` (required): File identifier (UUID)

**Response:**
```json
{
  "id": "file-abc123",
  "name": "SSH Private Key",
  "path": "/home/agentapi/.ssh/id_rsa",
  "content": "-----BEGIN OPENSSH PRIVATE KEY-----\n...",
  "permissions": "0600",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/files/file-abc123
```

**Access Control:**
- Users can only access their own files

### PUT /files/:fileId

Update an existing file.

**Permissions Required:** `session:create`

**Path Parameters:**
- `fileId` (required): File identifier (UUID)

**Request Body:**
```json
{
  "name": "Updated SSH Key",
  "path": "/home/agentapi/.ssh/id_ed25519",
  "content": "-----BEGIN OPENSSH PRIVATE KEY-----\n...",
  "permissions": "0600"
}
```

**Note:** All fields are optional. Omitted fields will not be modified.

**Response:**
```json
{
  "id": "file-abc123",
  "name": "Updated SSH Key",
  "path": "/home/agentapi/.ssh/id_ed25519",
  "permissions": "0600",
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/files/file-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated SSH Key",
    "content": "-----BEGIN OPENSSH PRIVATE KEY-----\n..."
  }'
```

**Access Control:**
- Users can only update their own files

### DELETE /files/:fileId

Delete a file by ID.

**Permissions Required:** `session:create`

**Path Parameters:**
- `fileId` (required): File identifier (UUID)

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/files/file-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- Users can only delete their own files

**Usage Notes:**
- Files are automatically mounted in agent sessions at startup
- The provisioner writes files with mode `0600` by default for security
- Files persist across sessions until explicitly deleted
- Common use cases: SSH keys, configuration files, API tokens
- File paths should be absolute and within the agent's home directory

## Credentials Management Endpoints

The Credentials endpoints allow users to securely upload, manage, and delete authentication credentials (e.g., `auth.json` for Claude Code OAuth tokens). Credentials are stored securely in Kubernetes secrets and are accessible in agent sessions.

### GET /credentials/:name

Get metadata for a named credential. The actual credential data is never exposed via the API - only whether it exists.

**Permissions Required:** `session:read`

**Path Parameters:**
- `name` (required): Credential name (e.g., `auth` for `auth.json`). Users can access their own credentials or credentials of teams they belong to.

**Response:**
```json
{
  "name": "auth",
  "has_data": true,
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-02T12:30:00Z"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/credentials/auth
```

### PUT /credentials/:name

Upload or replace a named credential file. The request body must be valid JSON.

**Permissions Required:** `session:create`

**Path Parameters:**
- `name` (required): Credential name (e.g., `auth`)

**Request Body:**
Raw JSON credential data (e.g., contents of `auth.json`):
```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-...",
    "refreshToken": "...",
    "expiresAt": 1234567890
  }
}
```

**Response:**
```json
{
  "name": "auth",
  "has_data": true,
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-02T15:00:00Z"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/credentials/auth \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "claudeAiOauth": {
      "accessToken": "sk-ant-...",
      "refreshToken": "...",
      "expiresAt": 1234567890
    }
  }'
```

### DELETE /credentials/:name

Remove a named credential.

**Permissions Required:** `session:create`

**Path Parameters:**
- `name` (required): Credential name

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/credentials/auth \
  -H "X-API-Key: YOUR_API_KEY"
```

**Access Control:**
- Users can access and modify their own credentials (user-scoped)
- Team members can access and modify credentials for teams they belong to (team-scoped)
- Credentials are automatically mounted in agent sessions at `/credentials/{name}.json`

## User & Settings Endpoints

### GET /user/info

Get authenticated user information.

**Permissions Required:** `session:read`

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/user/info
```

### GET /settings/:name

Get settings for a user or team. Settings include Bedrock configuration, MCP servers, plugin marketplaces, enabled plugins, and custom environment variables.

**Permissions Required:** `session:read`

**Path Parameters:**
- `name` (required): Settings name - either a user ID or team name in `org/team-slug` format

**Response:**
```json
{
  "name": "myorg-backend",
  "bedrock": {
    "enabled": true,
    "model": "anthropic.claude-3-sonnet-20240229-v1:0",
    "role_arn": "arn:aws:iam::123456789012:role/bedrock-role",
    "profile": "default"
  },
  "mcp_servers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env_keys": ["GITHUB_TOKEN"]
    },
    "slack": {
      "type": "http",
      "url": "https://mcp.example.com/slack",
      "header_keys": ["Authorization"]
    }
  },
  "marketplaces": {
    "my-marketplace": {
      "url": "https://github.com/example/my-marketplace.git"
    }
  },
  "enabled_plugins": [
    "commit@claude-plugins-official",
    "plugin1@my-marketplace"
  ],
  "env_var_keys": ["CUSTOM_VAR", "API_KEY"],
  "has_claude_code_oauth_token": true,
  "auth_mode": "oauth",
  "preferred_team_id": "myorg/backend",
  "slack_user_id": "U1234567890",
  "git_sync": {
    "enabled": true,
    "repo_full_name": "myorg/agentapi-settings",
    "branch": "main",
    "root_path": "agentapi-config/",
    "auto_push": false,
    "has_github_token": true,
    "encryption": {
      "dek_version": 1,
      "dek_ready": true
    }
  },
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-02T00:00:00Z"
}
```

**Fields:**
- `bedrock`: AWS Bedrock settings (credentials not returned)
  - `enabled`: Whether Bedrock is enabled
  - `model`: Bedrock model ID
  - `role_arn`: IAM role ARN for Bedrock access
  - `profile`: AWS profile name
- `mcp_servers`: MCP server configurations (keyed by server name)
  - `type`: Server type (`stdio`, `http`, or `sse`)
  - `command`/`args`: For stdio servers
  - `url`: For HTTP/SSE servers
  - `env_keys`: Environment variable keys (values not returned)
  - `header_keys`: HTTP header keys (values not returned)
- `marketplaces`: Claude Code plugin marketplace configurations
- `enabled_plugins`: List of enabled plugins in `plugin@marketplace` format
- `env_var_keys`: Custom environment variable keys (values not returned for security)
- `has_claude_code_oauth_token`: Whether Claude Code OAuth token is configured
- `auth_mode`: Authentication mode (`oauth`, `bedrock`, or empty)
- `preferred_team_id`: Team whose settings to use exclusively (empty = merge all teams)
- `slack_user_id`: Slack user ID for DM notifications
- `git_sync`: GitHub sync configuration (null if not configured)
  - `enabled`: Whether sync is enabled
  - `repo_full_name`: GitHub repository (`owner/repo` format)
  - `branch`: Target branch
  - `root_path`: Base path within the repository
  - `auto_push`: Whether automatic push on changes is enabled
  - `has_github_token`: Whether a GitHub token is configured
  - `encryption.dek_ready`: Whether encryption key is ready

**Example:**
```bash
# Get user settings
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/settings/alice

# Get team settings
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/settings/myorg-backend
```

### PUT /settings/:name

Create or update settings for a user or team. All fields are optional - omitted fields will not be modified.

**Permissions Required:** `session:create` (users can modify their own settings; team admins/maintainers can modify team settings)

**Path Parameters:**
- `name` (required): Settings name - either a user ID or team name in `org/team-slug` format

**Request Body:**
```json
{
  "bedrock": {
    "enabled": true,
    "model": "anthropic.claude-3-sonnet-20240229-v1:0",
    "access_key_id": "AKIAIOSFODNN7EXAMPLE",
    "secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "role_arn": "arn:aws:iam::123456789012:role/bedrock-role",
    "profile": "default"
  },
  "mcp_servers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxx"
      }
    },
    "slack": {
      "type": "http",
      "url": "https://mcp.example.com/slack",
      "headers": {
        "Authorization": "Bearer xoxb-xxxx"
      }
    }
  },
  "marketplaces": {
    "my-marketplace": {
      "url": "https://github.com/example/my-marketplace.git"
    }
  },
  "enabled_plugins": [
    "plugin1@my-marketplace",
    "plugin2@my-marketplace"
  ],
  "env_vars": {
    "CUSTOM_VAR": "value",
    "API_KEY": "secret-key"
  },
  "claude_code_oauth_token": "sk-ant-...",
  "auth_mode": "oauth",
  "preferred_team_id": "myorg/backend",
  "slack_user_id": "U1234567890"
}
```

**Fields:**
- All fields are optional
- `bedrock`: AWS Bedrock configuration
- `mcp_servers`: MCP server configurations (use `env`/`headers` to set credentials)
- `marketplaces`: Plugin marketplace Git URLs
- `enabled_plugins`: List of plugins to enable
- `env_vars`: Custom environment variables (empty string preserves existing value; keys not included are preserved)
- `claude_code_oauth_token`: Claude Code OAuth token (empty string removes it)
- `auth_mode`: Preferred authentication mode
- `preferred_team_id`: Team to use exclusively (empty string = merge all teams)
- `slack_user_id`: Slack user ID for notifications (empty string removes it)
- `git_sync`: GitHub sync configuration (omitted = preserve existing config)
  - `enabled`: Enable/disable sync
  - `repo_full_name`: GitHub repository in `owner/repo` format
  - `branch`: Target branch (default: `main`)
  - `root_path`: Base path within the repository (default: `agentapi-config/`)
  - `auto_push`: Automatically push on resource changes
  - `github_token`: GitHub PAT with repo write permissions (write-only, not returned)

**Example:**
```bash
# Update Bedrock settings
curl -X PUT https://api.example.com/settings/alice \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "bedrock": {
      "enabled": true,
      "model": "anthropic.claude-3-sonnet-20240229-v1:0"
    }
  }'

# Configure MCP servers
curl -X PUT https://api.example.com/settings/myorg-backend \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "mcp_servers": {
      "github": {
        "type": "stdio",
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {
          "GITHUB_TOKEN": "ghp_xxxx"
        }
      }
    }
  }'

# Enable plugins from marketplace
curl -X PUT https://api.example.com/settings/alice \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "marketplaces": {
      "my-marketplace": {
        "url": "https://github.com/example/my-marketplace.git"
      }
    },
    "enabled_plugins": [
      "plugin1@my-marketplace"
    ]
  }'
```

### DELETE /settings/:name

Delete settings for a user or team. Also deletes associated credentials and MCP server secrets.

**Permissions Required:** `session:create`

**Path Parameters:**
- `name` (required): Settings name

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/settings/myorg-backend \
  -H "X-API-Key: YOUR_API_KEY"
```

### GET /users/me/api-key

Get or create a personal API key.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "api_key": "ap_personal_xyz123"
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/users/me/api-key
```

### POST /users/me/api-key

Generate a new personal API key (rotates the existing key).

**Permissions Required:** `session:read`

**Example:**
```bash
curl -X POST https://api.example.com/users/me/api-key \
  -H "X-API-Key: YOUR_API_KEY"
```

## Notification Endpoints

### POST /notification/subscribe

Subscribe to push notifications.

**Permissions Required:** `session:read`

**Request Body:**
```json
{
  "endpoint": "https://...",
  "keys": {
    "p256dh": "...",
    "auth": "..."
  }
}
```

**Example:**
```bash
curl -X POST https://api.example.com/notification/subscribe \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

### GET /notification/subscribe

Get current subscription information.

**Permissions Required:** `session:read`

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/notification/subscribe
```

### DELETE /notification/subscribe

Unsubscribe from notifications.

**Permissions Required:** `session:read`

**Example:**
```bash
curl -X DELETE https://api.example.com/notification/subscribe \
  -H "X-API-Key: YOUR_API_KEY"
```

### GET /notifications/history

Get notification history for the current user.

**Permissions Required:** `session:read`

**Response:**
```json
{
  "notifications": [
    {
      "id": "notif-123",
      "title": "Session completed",
      "body": "Your session has finished successfully",
      "url": "https://example.com/sessions/abc123",
      "created_at": "2024-01-02T15:30:00Z",
      "read": false
    }
  ]
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/notifications/history
```

### POST /notifications/send

Send a push notification to subscribers of a session or to a specific user.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Task completed",
  "body": "Your requested task has been completed",
  "url": "https://example.com/sessions/abc123"
}
```

**Fields:**
- `session_id` or `user_id` (required): Either session ID to notify subscribers or user ID to notify a specific user
- `title` (required): Notification title
- `body` (required): Notification body text
- `url` (optional): URL to open when notification is clicked

**Response:**
```json
{
  "success": true,
  "sent_count": 3
}
```

**Example:**
```bash
# Notify subscribers of a session
curl -X POST https://api.example.com/notifications/send \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "Task completed",
    "body": "Your requested task has been completed",
    "url": "https://example.com/sessions/abc123"
  }'

# Notify a specific user
curl -X POST https://api.example.com/notifications/send \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "alice",
    "title": "Important update",
    "body": "Please check your tasks"
  }'
```

## Settings Sync (GitHub) Endpoints

These endpoints enable synchronization of agentapi-proxy resources with a GitHub repository. Resources are serialized as YAML files and can be version-controlled, enabling GitOps workflows. Sensitive data is encrypted with AES-256-GCM using AWS KMS.

**Sync configuration** is managed via the `git_sync` field in `PUT /settings/{name}` (see [User & Settings Endpoints](#user--settings-endpoints)).

### DELETE /settings/:name/sync

Delete GitHub sync configuration for a settings tenant.

**Permissions Required:** `session:delete`

**Path Parameters:**
- `name` (required): Settings name (user ID or team name in `org/team-slug` format)

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X DELETE https://api.example.com/settings/alice/sync \
  -H "X-API-Key: YOUR_API_KEY"
```

### POST /settings/:name/sync/push

Push resources to GitHub.

**Permissions Required:** `session:create`

**Path Parameters:**
- `name` (required): Settings name

**Request Body:**
```json
{
  "commit_message": "Update agentapi settings"
}
```

**Fields:**
- `commit_message`: Custom commit message (optional)

**Response:**
```json
{
  "commit_sha": "abc123def456",
  "pushed_at": "2026-05-02T12:00:00Z",
  "summary": {}
}
```

**Example:**
```bash
curl -X POST https://api.example.com/settings/alice/sync/push \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"commit_message": "Update settings"}'
```

**Notes:**
- Push is idempotent: no commit is created when file contents are unchanged
- Sensitive data (tokens, secrets) is encrypted before pushing

### POST /settings/:name/sync/pull

Pull resources from GitHub and apply them locally.

**Permissions Required:** `session:create`

**Path Parameters:**
- `name` (required): Settings name

**Request Body:**
```json
{
  "delete_orphans": true
}
```

**Fields:**
- `delete_orphans`: Remove local resources that no longer exist in GitHub (default: false)

**Response:**
```json
{
  "pulled_at": "2026-05-02T12:00:00Z",
  "summary": {}
}
```

**Example:**
```bash
curl -X POST https://api.example.com/settings/alice/sync/pull \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"delete_orphans": true}'
```

**Notes:**
- Encrypted data is decrypted during pull
- Local changes will be overwritten by GitHub state

### POST /settings/:name/sync/rotate-key

Rotate the encryption key used for syncing sensitive data.

**Permissions Required:** `session:create`

**Path Parameters:**
- `name` (required): Settings name

**Response:**
```json
{
  "success": true
}
```

**Example:**
```bash
curl -X POST https://api.example.com/settings/alice/sync/rotate-key \
  -H "X-API-Key: YOUR_API_KEY"
```

**Notes:**
- Generates a new DEK via AWS KMS and re-encrypts all values in the GitHub repository
- After rotation, push resources again to re-encrypt with the new key

### POST /settings/sync/all

Sync all tenants with their configured GitHub repositories.

**Permissions Required:** Admin only

**Description:**
Syncs GitHub for every settings tenant that has sync enabled. The direction (push/pull) is determined automatically per tenant by comparing the remote `.sync-meta.yaml` `syncedAt` timestamp against the local `LastPushedAt`.

**Request Body:**
```json
{
  "delete_orphans": true,
  "commit_message": "Automated sync"
}
```

**Fields:**
- `delete_orphans`: Remove local resources that no longer exist in GitHub (applies when pull is selected)
- `commit_message`: Override the default commit message when push is selected

**Response:**
```json
{
  "success": true,
  "results": [
    {
      "name": "alice",
      "action": "push",
      "success": true,
      "commit_sha": "abc123"
    },
    {
      "name": "myorg/backend",
      "action": "pull",
      "success": true
    }
  ],
  "total": 2,
  "succeeded": 2,
  "failed": 0
}
```

**Example:**
```bash
curl -X POST https://api.example.com/settings/sync/all \
  -H "X-API-Key: YOUR_ADMIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "delete_orphans": true,
    "commit_message": "Daily automated sync"
  }'
```

**Notes:**
- Only accessible by admin users
- Individual tenant errors are captured in `results[].error`
- Push is idempotent: no commit is created when file contents are unchanged

## Session Profiles Endpoints

Session profiles are named, reusable session configurations. Instead of specifying session parameters on every call, you can define a profile once and reference it by ID.

### POST /session-profiles

Create a new session profile.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "PR Review Bot",
  "description": "Profile for automated PR reviews",
  "scope": "user",
  "team_id": "optional-team-id",
  "is_default": false,
  "config": {
    "environment": {
      "CUSTOM_VAR": "value"
    },
    "tags": {
      "type": "pr-review"
    },
    "initial_message_template": "Review PR #{{.pull_request.number}}: {{.pull_request.title}}",
    "reuse_session": false,
    "params": {
      "agent_type": "claude-agentapi",
      "oneshot": true
    }
  }
}
```

**Fields:**
- `name` (required): Profile name
- `description`: Human-readable description
- `scope`: `user` (default) or `team`
- `team_id`: Required when `scope` is `team`
- `is_default`: Whether this is the default profile for the tenant
- `config`: Session configuration
  - `environment`: Environment variables to inject
  - `tags`: Tags to attach to sessions
  - `initial_message_template`: Go template for the initial message
  - `reuse_message_template`: Go template for reuse message when reusing a session
  - `reuse_session`: Whether to reuse an existing session
  - `memory_key`: Memory key mapping for session context
  - `params`: SessionParams (agent type, sandbox, oneshot, etc.)

**Response:**
```json
{
  "id": "profile-abc123",
  "name": "PR Review Bot",
  "description": "Profile for automated PR reviews",
  "user_id": "alice",
  "scope": "user",
  "is_default": false,
  "config": {...},
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/session-profiles \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "PR Review Bot",
    "config": {
      "initial_message_template": "Review PR #{{.pull_request.number}}",
      "params": {"oneshot": true}
    }
  }'
```

### GET /session-profiles

List session profiles accessible to the authenticated user.

**Permissions Required:** `session:list`

**Query Parameters:**
- `scope`: Filter by scope (`user`, `team`)
- `team_id`: Filter by team ID

**Response:**
```json
{
  "session_profiles": [
    {
      "id": "profile-abc123",
      "name": "PR Review Bot",
      "scope": "user",
      "is_default": false,
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ]
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/session-profiles
```

### GET /session-profiles/:id

Get a specific session profile by ID.

**Permissions Required:** `session:read`

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/session-profiles/profile-abc123
```

### PUT /session-profiles/:id

Update an existing session profile.

**Permissions Required:** `session:create`

**Example:**
```bash
curl -X PUT https://api.example.com/session-profiles/profile-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Name", "is_default": true}'
```

### DELETE /session-profiles/:id

Delete a session profile by ID.

**Permissions Required:** `session:delete`

**Example:**
```bash
curl -X DELETE https://api.example.com/session-profiles/profile-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

## Sandbox Policies Endpoints

Sandbox policies define named, reusable network filter rule sets. Sessions can reference a policy by ID to restrict outbound network access. Policies support allowlist mode (only listed domains permitted) and denylist mode (listed domains blocked).

### POST /sandbox-policies

Create a new sandbox policy.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "GitHub Only",
  "description": "Restrict to GitHub domains",
  "scope": "user",
  "team_id": "optional-team-id",
  "allowed_domains": ["github.com", "*.github.com", "*.githubusercontent.com"],
  "denied_domains": []
}
```

**Fields:**
- `name` (required): Human-readable name
- `description`: Optional description
- `scope`: `user` (default) or `team`
- `team_id`: Required when `scope` is `team`
- `allowed_domains`: Allowlist mode — only these domains are permitted. Supports wildcard prefixes (e.g., `*.example.com`). When set, `denied_domains` is ignored.
- `denied_domains`: Denylist mode — these domains are blocked, all others allowed. Used only when `allowed_domains` is empty.

**Response:**
```json
{
  "id": "policy-abc123",
  "name": "GitHub Only",
  "description": "Restrict to GitHub domains",
  "allowed_domains": ["github.com", "*.github.com", "*.githubusercontent.com"],
  "denied_domains": [],
  "scope": "user",
  "owner_id": "alice",
  "created_at": "2024-01-01T12:00:00Z",
  "updated_at": "2024-01-01T12:00:00Z"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/sandbox-policies \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "GitHub Only",
    "allowed_domains": ["github.com", "*.github.com", "*.githubusercontent.com"]
  }'
```

### GET /sandbox-policies

List sandbox policies accessible to the authenticated user.

**Permissions Required:** `session:list`

**Query Parameters:**
- `scope`: Filter by scope (`user`, `team`)
- `team_id`: Filter by team ID

**Response:**
```json
{
  "sandbox_policies": [
    {
      "id": "policy-abc123",
      "name": "GitHub Only",
      "scope": "user",
      "owner_id": "alice",
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ],
  "total": 1
}
```

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/sandbox-policies
```

### GET /sandbox-policies/:id

Get a specific sandbox policy by ID.

**Permissions Required:** `session:read`

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/sandbox-policies/policy-abc123
```

### PUT /sandbox-policies/:id

Update an existing sandbox policy.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "name": "Updated Policy Name",
  "allowed_domains": ["github.com", "*.npmjs.com"]
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/sandbox-policies/policy-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"allowed_domains": ["github.com", "*.npmjs.com"]}'
```

### DELETE /sandbox-policies/:id

Delete a sandbox policy by ID.

**Permissions Required:** `session:delete`

**Example:**
```bash
curl -X DELETE https://api.example.com/sandbox-policies/policy-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

### GET /sandbox-policies/:id/domains

Get aggregated domain lists collected from all sessions using a sandbox policy.

**Permissions Required:** `session:read`

**Description:**
- Returns the allowed/denied/ignored domain lists collected by the background domain collector from all sessions using this policy
- Data is refreshed every 60 seconds
- Returns empty lists when no data has been collected yet

**Response:**
```json
{
  "allowed": ["github.com", "api.example.com"],
  "denied": ["blocked.example.com"],
  "ignored": ["tracking.example.com"],
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Fields:**
- `allowed`: Domains allowed through the filter
- `denied`: Domains blocked by the filter
- `ignored`: Domains the user has chosen to ignore (not shown as suggestions)
- `updated_at`: When the data was last collected

**Response Codes:**
- `200`: Aggregated domain lists
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Sandbox policy not found
- `501`: Domain collection not available

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/sandbox-policies/policy-abc123/domains
```

### PUT /sandbox-policies/:id/domains/ignored

Update the ignored domain list for a sandbox policy.

**Permissions Required:** `session:create`

**Description:**
Replaces the ignored domain list stored in the collected domain data for the given policy. Ignored domains are suppressed from the import suggestion UI so users are not repeatedly prompted to act on them.

**Request Body:**
```json
{
  "ignored": ["tracking.example.com", "ads.example.com"]
}
```

**Fields:**
- `ignored` (required): Full list of domains to ignore (replaces existing list)

**Response:**
```json
{
  "allowed": ["github.com", "api.example.com"],
  "denied": ["blocked.example.com"],
  "ignored": ["tracking.example.com", "ads.example.com"],
  "updated_at": "2024-01-02T12:00:00Z"
}
```

**Response Codes:**
- `200`: Updated domain lists
- `400`: Invalid request body
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Sandbox policy not found
- `501`: Domain collection not available

**Example:**
```bash
curl -X PUT https://api.example.com/sandbox-policies/policy-abc123/domains/ignored \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ignored": ["tracking.example.com"]}'
```

**Using Sandbox in Sessions:**

Reference a policy when creating a session via the `sandbox` field in `params`:
```json
{
  "params": {
    "sandbox": {
      "enabled": true,
      "policy_id": "policy-abc123",
      "allowed_domains": ["extra-domain.com"]
    }
  }
}
```
The `policy_id` domains are merged with any inline `allowed_domains`/`denied_domains`.

## Codex Device Auth Endpoints

These endpoints support device authorization flow for Codex authentication. Users complete the authentication in their browser using a user code and verification URI returned by the API.

### GET /codex/device-auth/config

Check whether Codex device authentication is configured on this proxy instance.

**Permissions Required:** `session:read` (Bearer or API Key)

**Response:**
```json
{
  "configured": true
}
```

**Example:**
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
  https://api.example.com/codex/device-auth/config
```

### POST /codex/device-auth

Start the Codex device authorization flow.

**Permissions Required:** `session:read` (Bearer or API Key)

**Description:**
Runs `codex login --device-auth` and returns the user code and verification URI for the user to complete authentication in their browser.

**Response:**
```json
{
  "user_code": "ABCD-1234",
  "verification_uri": "https://auth.example.com/device"
}
```

**Example:**
```bash
curl -X POST https://api.example.com/codex/device-auth \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### POST /codex/device-auth/token

Poll the status of an in-progress Codex device auth session.

**Permissions Required:** `session:read` (Bearer or API Key)

**Description:**
Returns the current status of the caller's in-progress device auth session. The session is identified by the caller's authentication token. Poll this endpoint until the status indicates completion.

**Response:**
```json
{
  "status": "pending"
}
```

Status values: `pending` (waiting for user), `complete` (authenticated), `expired` (timed out).

**Example:**
```bash
curl -X POST https://api.example.com/codex/device-auth/token \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## Authentication Endpoints

These endpoints are typically used for OAuth flows and do not require API key authentication.

### GET /health

Health check endpoint (no authentication required).

**Example:**
```bash
curl https://api.example.com/health
```

### GET /auth/status

Check authentication status.

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/auth/status
```

### POST /oauth/authorize

Initiate OAuth authorization flow (no authentication required).

### GET /oauth/callback

OAuth callback endpoint (no authentication required).

### POST /oauth/logout

Logout from OAuth session.

### POST /oauth/refresh

Refresh OAuth access token.

## Error Responses

### 401 Unauthorized

Invalid or missing API key:
```json
{
  "error": "Invalid API key"
}
```

Expired API key:
```json
{
  "error": "API key expired"
}
```

### 403 Forbidden

Insufficient permissions:
```json
{
  "error": "Insufficient permissions"
}
```

Session access denied (non-owner):
```json
{
  "error": "Access denied"
}
```

### 404 Not Found

Session not found:
```json
{
  "error": "Session not found"
}
```
