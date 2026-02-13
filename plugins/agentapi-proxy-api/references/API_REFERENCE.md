# agentapi-proxy API Reference

## Table of Contents

- [Session Management Endpoints](#session-management-endpoints)
- [Session Sharing Endpoints](#session-sharing-endpoints)
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
