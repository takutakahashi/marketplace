---
name: webhook-management
description: |
  Manage agentapi-proxy webhooks for automated session creation.
  Use when you need to: (1) Create GitHub or custom webhooks, (2) Update webhook configurations,
  (3) List existing webhooks, (4) Delete webhooks, (5) Regenerate webhook secrets,
  (6) Configure webhook triggers and conditions. Supports GitHub webhooks and custom webhooks
  with JSONPath-based filtering.
---

# Webhook Management

This skill provides guidance for managing agentapi-proxy webhooks that automatically create sessions in response to external events.

## ⚠️ Important: Always Use This Skill for Webhook Operations

**When performing any webhook-related operations (creating, updating, listing, deleting, or modifying webhook configurations), always invoke this skill first rather than directly using curl or API calls.**

This ensures:
- Proper authentication and API endpoint configuration
- Correct payload structure and validation
- Access to up-to-date examples and best practices
- Consistent error handling and guidance

## Overview

Webhooks enable automatic session creation when events occur in external systems (GitHub, Slack, Datadog, custom services). Each webhook has:
- **Triggers**: Rules that determine when to create a session
- **Conditions**: Filters based on event properties
- **Session Config**: Environment variables, tags, and initial messages for created sessions

## Core Workflows

### Creating a Webhook

#### GitHub Webhook

```bash
curl -X POST https://api.example.com/webhooks \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pull Request Reviewer",
    "type": "github",
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
            "actions": ["opened"],
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
  }'
```

#### Custom Webhook (Slack, Datadog, etc.)

```bash
curl -X POST https://api.example.com/webhooks \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Slack Incident Alerts",
    "type": "custom",
    "triggers": [
      {
        "name": "Critical incident",
        "conditions": {
          "jsonpath": [
            {
              "path": "$.event.type",
              "operator": "eq",
              "value": "incident"
            },
            {
              "path": "$.event.severity",
              "operator": "eq",
              "value": "critical"
            }
          ]
        },
        "session_config": {
          "initial_message_template": "Incident: {{.event.title}}",
          "tags": {
            "source": "slack",
            "severity": "{{.event.severity}}"
          }
        }
      }
    ]
  }'
```

**Response:**
```json
{
  "id": "webhook-123",
  "webhook_url": "https://api.example.com/hooks/github/webhook-123",
  "secret": "generated-secret-key"
}
```

### Listing Webhooks

```bash
curl -H "X-API-Key: YOUR_API_KEY" \
  https://api.example.com/webhooks
```

### Updating a Webhook

```bash
curl -X PUT https://api.example.com/webhooks/WEBHOOK_ID \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Webhook Name",
    "status": "active",
    "triggers": [...]
  }'
```

### Deleting a Webhook

```bash
curl -X DELETE https://api.example.com/webhooks/WEBHOOK_ID \
  -H "X-API-Key: YOUR_API_KEY"
```

### Regenerating Webhook Secret

```bash
curl -X POST https://api.example.com/webhooks/WEBHOOK_ID/regenerate-secret \
  -H "X-API-Key: YOUR_API_KEY"
```

## Reference Documentation

For detailed information, see:
- [WEBHOOK_REFERENCE.md](references/WEBHOOK_REFERENCE.md) - Complete webhook API and configuration
- [WEBHOOK_TRIGGERS.md](references/WEBHOOK_TRIGGERS.md) - Trigger conditions and filtering
- [WEBHOOK_EXAMPLES.md](references/WEBHOOK_EXAMPLES.md) - Integration examples for various services
