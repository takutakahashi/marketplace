# Webhook Triggers and Conditions

## Overview

Triggers define when a webhook should create a session. Each trigger has:
- **Name**: Human-readable identifier
- **Priority**: Evaluation order (higher = evaluated first)
- **Enabled**: Whether the trigger is active
- **Conditions**: Rules for matching events
- **Session Config**: Configuration for created sessions
- **Stop on Match**: Stop evaluating triggers after this one matches

## Trigger Structure

```json
{
  "name": "Pull Request Opened",
  "priority": 10,
  "enabled": true,
  "conditions": {
    "github": {...},
    "jsonpath": [...],
    "gotemplate": "..."
  },
  "session_config": {...},
  "stop_on_match": true
}
```

## Condition Types

### GitHub Conditions

For GitHub webhooks only.

```json
{
  "github": {
    "events": ["pull_request", "push"],
    "actions": ["opened", "synchronize"],
    "branches": ["main", "develop"],
    "repositories": ["owner/repo"],
    "labels": ["bug", "feature"],
    "paths": ["src/**", "docs/**"],
    "base_branches": ["main"],
    "draft": false,
    "sender": ["user1", "user2"]
  }
}
```

**Fields:**
- `events`: GitHub event types (e.g., `pull_request`, `push`, `issues`)
- `actions`: Event actions (e.g., `opened`, `closed`, `synchronize`)
- `branches`: Branch names (for push events)
- `repositories`: Repository names (`owner/repo` format)
- `labels`: PR/Issue labels
- `paths`: File paths (glob patterns)
- `base_branches`: Base branch for PRs
- `draft`: Filter draft PRs (true/false/null)
- `sender`: GitHub usernames

### JSONPath Conditions

For custom webhooks and advanced GitHub filtering.

```json
{
  "jsonpath": [
    {
      "path": "$.event.type",
      "operator": "eq",
      "value": "deployment"
    },
    {
      "path": "$.event.severity",
      "operator": "in",
      "value": ["critical", "high"]
    },
    {
      "path": "$.tags",
      "operator": "contains",
      "value": "production"
    }
  ]
}
```

**Operators:**

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equals | `$.status == "success"` |
| `ne` | Not equals | `$.status != "failed"` |
| `contains` | String/array contains | `$.tags contains "prod"` |
| `matches` | Regex match | `$.name matches "^api-.*"` |
| `in` | Value in array | `$.env in ["prod", "stage"]` |
| `exists` | Path exists | `$.deployment.id exists` |
| `gt` | Greater than | `$.cpu > 90` |
| `lt` | Less than | `$.cpu < 50` |
| `gte` | Greater than or equal | `$.memory >= 80` |
| `lte` | Less than or equal | `$.disk <= 20` |

### Go Template Conditions

Most flexible option, supports both GitHub and custom webhooks.

```json
{
  "gotemplate": "{{ and (eq .action \"opened\") (not .pull_request.draft) }}"
}
```

**Available Functions:**
- `eq`, `ne`: Equality comparison
- `and`, `or`, `not`: Logical operators
- `contains`, `hasPrefix`, `hasSuffix`: String functions
- `toLower`, `toUpper`, `trimSpace`: String manipulation
- `len`: Length of string/array
- `in`: Check if value in array
- `matches`: Regex matching

**Examples:**

```go
// PR opened and not draft
{{ and (eq .action "opened") (not .pull_request.draft) }}

// Title contains "feat:" or "fix:"
{{ or (contains .pull_request.title "feat:") (contains .pull_request.title "fix:") }}

// Multiple conditions
{{ and
  (eq .action "opened")
  (not .pull_request.draft)
  (in .pull_request.base.ref (list "main" "develop"))
}}
```

## Priority and Evaluation

Triggers are evaluated in priority order (highest first). If `stop_on_match` is true, evaluation stops after the first match.

```json
{
  "triggers": [
    {
      "name": "Urgent PR",
      "priority": 100,
      "conditions": {
        "github": {
          "labels": ["urgent", "critical"]
        }
      },
      "stop_on_match": true
    },
    {
      "name": "Regular PR",
      "priority": 10,
      "conditions": {
        "github": {
          "events": ["pull_request"],
          "actions": ["opened"]
        }
      }
    }
  ]
}
```

## Examples

### GitHub: PR Review Request

```json
{
  "name": "PR needs review",
  "conditions": {
    "github": {
      "events": ["pull_request"],
      "actions": ["opened", "synchronize"],
      "draft": false,
      "base_branches": ["main"]
    }
  },
  "session_config": {
    "initial_message_template": "Review PR #{{.pull_request.number}}: {{.pull_request.title}}\n\nAuthor: {{.pull_request.user.login}}\nFiles changed: {{.pull_request.changed_files}}",
    "tags": {
      "repository": "{{.repository.full_name}}",
      "pr": "{{.pull_request.number}}"
    }
  }
}
```

### Custom: Slack Incident

```json
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
      },
      {
        "path": "$.event.environment",
        "operator": "eq",
        "value": "production"
      }
    ]
  },
  "session_config": {
    "initial_message_template": "🚨 Critical Incident\n\nTitle: {{.event.title}}\nEnvironment: {{.event.environment}}\nReported by: {{.user.name}}\n\nInvestigate and respond.",
    "tags": {
      "source": "slack",
      "severity": "critical",
      "incident_id": "{{.event.id}}"
    }
  }
}
```

### Custom: Datadog Alert

```json
{
  "name": "High CPU alert",
  "conditions": {
    "gotemplate": "{{ and (eq .alert_type \"metric_alert\") (gt .current_value 90) }}"
  },
  "session_config": {
    "initial_message_template": "⚠️ High CPU Alert\n\nHost: {{.host}}\nCurrent: {{.current_value}}%\nThreshold: {{.threshold}}%\n\nScale or investigate.",
    "environment": {
      "DATADOG_HOST": "{{.host}}"
    }
  }
}
```

## Combining Conditions

All condition types can be combined. They are evaluated with AND logic.

```json
{
  "conditions": {
    "github": {
      "events": ["pull_request"]
    },
    "jsonpath": [
      {
        "path": "$.pull_request.changed_files",
        "operator": "gt",
        "value": 10
      }
    ],
    "gotemplate": "{{ contains .pull_request.title \"refactor\" }}"
  }
}
```

This matches only when:
1. Event is `pull_request` (GitHub condition)
2. Changed files > 10 (JSONPath condition)
3. Title contains "refactor" (Go template condition)
