---
name: schedule-management
description: |
  Manage agentapi-proxy schedules for delayed and recurring session execution.
  Use when you need to: (1) Create one-time or recurring schedules, (2) List existing schedules,
  (3) Update schedule configurations, (4) Delete schedules, (5) Manually trigger schedules,
  (6) Filter schedules by status, scope, or team. Supports cron expressions for recurring tasks
  and ISO 8601 timestamps for one-time delayed execution.
---

# Schedule Management

This skill provides guidance for managing agentapi-proxy schedules that automatically create sessions at specified times or intervals.

## Overview

Schedules enable automatic session creation at specific times or on recurring intervals. Each schedule has:
- **Name**: Descriptive name for the schedule
- **Execution Type**: One-time (scheduled_at) or recurring (cron_expr)
- **Session Config**: Environment variables, tags, and initial messages for created sessions
- **Status**: Active, paused, or completed
- **Scope**: User-level or team-level access

## Core Workflows

### Creating a Schedule

#### One-Time Delayed Execution

```bash
curl -X POST https://api.example.com/schedules \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Code Review Session",
    "scheduled_at": "2025-01-15T14:00:00Z",
    "session_config": {
      "tags": {
        "repository": "org/repo",
        "task": "code-review"
      },
      "params": {
        "message": "Review all open PRs"
      }
    }
  }'
```

#### Recurring Execution (Cron)

```bash
curl -X POST https://api.example.com/schedules \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Daily Standup Bot",
    "cron_expr": "0 9 * * 1-5",
    "session_config": {
      "tags": {
        "repository": "org/standup-bot",
        "type": "standup"
      },
      "params": {
        "message": "Generate daily standup report"
      },
      "environment": {
        "SLACK_WEBHOOK": "https://hooks.slack.com/..."
      }
    }
  }'
```

**Common Cron Expressions:**
- `0 9 * * 1-5` - Every weekday at 9:00 AM
- `0 */6 * * *` - Every 6 hours
- `0 0 * * 0` - Every Sunday at midnight
- `30 14 1 * *` - First day of every month at 2:30 PM

**Response:**
```json
{
  "id": "schedule-abc123",
  "name": "Daily Standup Bot",
  "status": "active",
  "cron_expr": "0 9 * * 1-5",
  "created_at": "2024-01-01T12:00:00Z",
  "next_run": "2024-01-02T09:00:00Z"
}
```

### Listing Schedules

```bash
# List all schedules
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/schedules

# Filter by status
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/schedules?status=active"

# Filter by scope
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/schedules?scope=user"

# Filter by team
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.example.com/schedules?team_id=org/my-team"
```

### Getting a Specific Schedule

```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/schedules/schedule-abc123
```

### Updating a Schedule

```bash
curl -X PUT https://api.example.com/schedules/schedule-abc123 \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Schedule Name",
    "status": "paused",
    "cron_expr": "0 10 * * 1-5"
  }'
```

### Deleting a Schedule

```bash
curl -X DELETE https://api.example.com/schedules/schedule-abc123 \
  -H "X-API-Key: YOUR_API_KEY"
```

### Manually Triggering a Schedule

Immediately execute a schedule without waiting for the next scheduled time:

```bash
curl -X POST https://api.example.com/schedules/schedule-abc123/trigger \
  -H "X-API-Key: YOUR_API_KEY"
```

**Response:**
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "triggered_at": "2024-01-02T15:30:00Z"
}
```

## Use Cases

### 1. Daily Reports

```json
{
  "name": "Daily Team Report",
  "cron_expr": "0 17 * * 1-5",
  "session_config": {
    "tags": {
      "type": "report",
      "team": "engineering"
    },
    "params": {
      "message": "Generate end-of-day team report"
    }
  }
}
```

### 2. Weekly Code Reviews

```json
{
  "name": "Weekly PR Review",
  "cron_expr": "0 10 * * 1",
  "session_config": {
    "tags": {
      "repository": "org/main-repo",
      "type": "code-review"
    },
    "params": {
      "message": "Review all open PRs from last week"
    }
  }
}
```

### 3. Incident Response Drills

```json
{
  "name": "Monthly Incident Drill",
  "cron_expr": "0 14 1 * *",
  "session_config": {
    "tags": {
      "type": "incident-drill",
      "team": "sre"
    },
    "params": {
      "message": "Run incident response drill"
    }
  }
}
```

### 4. Delayed Task Execution

```json
{
  "name": "Deploy After Hours",
  "scheduled_at": "2025-01-15T22:00:00Z",
  "session_config": {
    "tags": {
      "repository": "org/production",
      "type": "deployment"
    },
    "params": {
      "message": "Deploy version 2.0 to production"
    }
  }
}
```

## Schedule Status

- **active**: Schedule is enabled and will execute at the specified time
- **paused**: Schedule is temporarily disabled
- **completed**: One-time schedule that has already executed

## Access Control

- **User Scope**: Only the creating user can access and manage the schedule
- **Team Scope**: All team members can access and manage the schedule
- **Admin**: Can view and manage all schedules

## Reference Documentation

For complete API endpoint documentation and permissions, see:
- [API_REFERENCE.md](../references/API_REFERENCE.md#schedule-management-endpoints) - Complete schedule API reference
- [PERMISSIONS.md](../references/PERMISSIONS.md) - Role-based access control details
