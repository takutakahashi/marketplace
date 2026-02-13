# agentapi-proxy API Reference

## Table of Contents

- [Session Management Endpoints](#session-management-endpoints)
- [Session Sharing Endpoints](#session-sharing-endpoints)
- [Schedule Management Endpoints](#schedule-management-endpoints)
- [User & Settings Endpoints](#user--settings-endpoints)
- [Notification Endpoints](#notification-endpoints)
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
  }
}
```

**Note:** `user_id` is automatically assigned from the authenticated user's token.

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

### ANY /:sessionId/*

Route requests to the agentapi instance for the specified session.

**Permissions Required:** `session:access`

**Example:**
```bash
# Send a message to the agent
curl -X POST https://api.example.com/550e8400-e29b-41d4-a716-446655440000/message \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello", "type": "user"}'

# Get session status
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/550e8400-e29b-41d4-a716-446655440000/status

# Get conversation history
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/550e8400-e29b-41d4-a716-446655440000/messages
```

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

### ANY /s/:shareToken/*

Access a shared session in read-only mode (no authentication required).

**Example:**
```bash
curl https://api.example.com/s/sh_abc123def456/messages
```

## Schedule Management Endpoints

### POST /schedules

Create a new schedule for delayed or recurring session execution.

**Permissions Required:** `session:create`

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

Get a specific setting value.

**Permissions Required:** `session:read`

**Example:**
```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/settings/theme
```

### PUT /settings/:name

Update a setting value.

**Permissions Required:** `session:create`

**Request Body:**
```json
{
  "value": "dark"
}
```

**Example:**
```bash
curl -X PUT https://api.example.com/settings/theme \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"value": "dark"}'
```

### DELETE /settings/:name

Delete a setting.

**Permissions Required:** `session:create`

**Example:**
```bash
curl -X DELETE https://api.example.com/settings/theme \
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
