# Go Template Variables Reference

Go templates are used throughout ccplant for dynamic message generation, trigger condition evaluation, and environment/tag configuration. This document describes all available template variables in each context.

## Overview

The following configuration fields support Go template rendering:

| Field | Context |
|-------|---------|
| `initial_message_template` | Webhook, SlackBot |
| `reuse_message_template` | Webhook, SlackBot |
| `environment` (map values) | Webhook, SlackBot, Schedule |
| `tags` (map values) | Webhook, SlackBot, Schedule |
| `params.message` | Webhook, SlackBot |
| `memory_key` (map values) | Webhook, SlackBot, Schedule |
| `conditions.go_template` | Webhook trigger conditions |

## Available Template Functions

The following functions are available in all Go template contexts:

### String Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `contains` | `contains s substring` | Returns true if string contains substring |
| `hasPrefix` | `hasPrefix s prefix` | Returns true if string has given prefix |
| `hasSuffix` | `hasSuffix s suffix` | Returns true if string has given suffix |
| `toLower` | `toLower s` | Converts string to lowercase |
| `toUpper` | `toUpper s` | Converts string to uppercase |
| `trimSpace` | `trimSpace s` | Trims leading/trailing whitespace |
| `split` | `split s sep` | Splits string into a slice |
| `join` | `join slice sep` | Joins a slice into a string |
| `replace` | `replace s old new` | Replaces all occurrences of old with new |
| `toString` | `toString v` | Converts any value to string |

### Collection/Utility Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `len` | `len v` | Length of string, slice, or map |
| `in` | `in value list` | Returns true if value is in list |
| `matches` | `matches pattern value` | Returns true if value matches regex pattern |

### Built-in Go Template Functions

| Function | Description |
|----------|-------------|
| `eq`, `ne` | Equality comparison |
| `gt`, `lt`, `ge`, `le` | Numeric comparison |
| `and`, `or`, `not` | Logical operators |

---

## GitHub Webhook Template Variables

Available in `initial_message_template`, `reuse_message_template`, `environment`, `tags`, and `conditions.go_template` for GitHub webhooks.

### Top-Level Variables

| Variable | Type | Description |
|----------|------|-------------|
| `.action` | `string` | Event action (e.g., `opened`, `closed`, `synchronize`) |
| `.ref` | `string` | Git reference (push events, e.g., `refs/heads/main`) |
| `.repository` | `object` | Repository information |
| `.sender` | `object` | User who triggered the event |
| `.pull_request` | `object` | Pull request details (PR events only) |
| `.issue` | `object` | Issue details (issue events only) |
| `.commits` | `array` | Array of commits (push events only) |
| `.head_commit` | `object` | Latest commit (push events only) |

### `.repository` Fields

| Variable | Type | Description |
|----------|------|-------------|
| `.repository.full_name` | `string` | Full repository name (e.g., `owner/repo`) |
| `.repository.name` | `string` | Repository name only |
| `.repository.owner` | `object` | Repository owner (GitHubUser) |
| `.repository.owner.login` | `string` | Owner username |
| `.repository.owner.id` | `int64` | Owner user ID |
| `.repository.owner.avatar_url` | `string` | Owner avatar URL |
| `.repository.owner.html_url` | `string` | Owner profile URL |
| `.repository.default_branch` | `string` | Default branch name |
| `.repository.html_url` | `string` | Repository HTML URL |
| `.repository.clone_url` | `string` | Repository clone URL |

### `.sender` Fields

| Variable | Type | Description |
|----------|------|-------------|
| `.sender.login` | `string` | Sender's GitHub username |
| `.sender.id` | `int64` | Sender's GitHub user ID |
| `.sender.avatar_url` | `string` | Sender's avatar URL |
| `.sender.html_url` | `string` | Sender's profile URL |

### `.pull_request` Fields

Available when `event` is `pull_request`.

| Variable | Type | Description |
|----------|------|-------------|
| `.pull_request.number` | `int` | PR number |
| `.pull_request.title` | `string` | PR title |
| `.pull_request.body` | `string` | PR description body |
| `.pull_request.state` | `string` | PR state (`open`, `closed`) |
| `.pull_request.draft` | `bool` | Whether PR is a draft |
| `.pull_request.html_url` | `string` | PR URL |
| `.pull_request.merged` | `bool` | Whether PR is merged |
| `.pull_request.merged_at` | `string` | Merge timestamp (ISO 8601) |
| `.pull_request.user` | `object` | PR author (GitHubUser) |
| `.pull_request.user.login` | `string` | PR author's username |
| `.pull_request.head` | `object` | Head branch reference |
| `.pull_request.head.ref` | `string` | Head branch name |
| `.pull_request.head.sha` | `string` | Head commit SHA |
| `.pull_request.head.repo` | `object` | Head repository (GitHubRepository) |
| `.pull_request.base` | `object` | Base branch reference |
| `.pull_request.base.ref` | `string` | Base branch name |
| `.pull_request.base.sha` | `string` | Base commit SHA |
| `.pull_request.labels` | `array` | PR labels |

**Label fields** (accessed via range or index):
```
.pull_request.labels[N].name   - Label name
.pull_request.labels[N].color  - Label color (hex)
```

### `.issue` Fields

Available when `event` is `issues`.

| Variable | Type | Description |
|----------|------|-------------|
| `.issue.number` | `int` | Issue number |
| `.issue.title` | `string` | Issue title |
| `.issue.body` | `string` | Issue description body |
| `.issue.state` | `string` | Issue state (`open`, `closed`) |
| `.issue.html_url` | `string` | Issue URL |
| `.issue.user` | `object` | Issue author (GitHubUser) |
| `.issue.user.login` | `string` | Issue author's username |
| `.issue.labels` | `array` | Issue labels |

**Label fields** (accessed via range or index):
```
.issue.labels[N].name   - Label name
.issue.labels[N].color  - Label color (hex)
```

### `.head_commit` Fields

Available in push events.

| Variable | Type | Description |
|----------|------|-------------|
| `.head_commit.id` | `string` | Commit SHA |
| `.head_commit.message` | `string` | Commit message |
| `.head_commit.author` | `object` | Commit author |
| `.head_commit.author.name` | `string` | Author name |
| `.head_commit.author.email` | `string` | Author email |
| `.head_commit.author.username` | `string` | Author GitHub username |

### `.commits` Array

Available in push events. Each element has the same fields as `.head_commit`.

```
.commits[N].id           - Commit SHA
.commits[N].message      - Commit message
.commits[N].author.name  - Author name
.commits[N].author.email - Author email
.commits[N].added        - Array of added file paths
.commits[N].removed      - Array of removed file paths
.commits[N].modified     - Array of modified file paths
```

### GitHub Webhook Template Examples

```go
// initial_message_template for PR review
Review PR #{{.pull_request.number}}: {{.pull_request.title}}

Repository: {{.repository.full_name}}
Author: {{.pull_request.user.login}}
Branch: {{.pull_request.head.ref}} → {{.pull_request.base.ref}}
URL: {{.pull_request.html_url}}

Please review and provide feedback.

// tags
{
  "repository": "{{.repository.full_name}}",
  "pr_number": "{{.pull_request.number}}",
  "author": "{{.pull_request.user.login}}"
}

// environment
{
  "GITHUB_PR_NUMBER": "{{.pull_request.number}}",
  "GITHUB_REPO": "{{.repository.full_name}}"
}

// go_template condition
{{ and (eq .action "opened") (not .pull_request.draft) (eq .pull_request.base.ref "main") }}
```

---

## Slack Event Template Variables

Available in `initial_message_template`, `reuse_message_template`, `environment`, `tags`, and SlackBot trigger conditions.

### Top-Level Variables

| Variable | Type | Description |
|----------|------|-------------|
| `.team_id` | `string` | Slack workspace team ID |
| `.thread_messages` | `string` | Full thread context (all messages in thread) |
| `.bot_id` | `string` | SlackBot ID |
| `.event` | `object` | Slack event details |

### `.event` Fields

| Variable | Type | Description |
|----------|------|-------------|
| `.event.type` | `string` | Event type (e.g., `message`, `app_mention`) |
| `.event.subtype` | `string` | Event subtype (e.g., `bot_message`) |
| `.event.text` | `string` | Message text content |
| `.event.user` | `string` | Slack user ID of the sender |
| `.event.channel` | `string` | Slack channel ID |
| `.event.ts` | `string` | Message timestamp |
| `.event.thread_ts` | `string` | Thread timestamp (for threaded replies) |
| `.event.bot_id` | `string` | Bot ID (if sent by a bot) |

### Slack Event Template Examples

```go
// initial_message_template
Slack message received in <#{{.event.channel}}> from {{.event.user}}:

{{.event.text}}

{{if .thread_messages}}
Thread context:
{{.thread_messages}}
{{end}}

// tags
{
  "channel": "{{.event.channel}}",
  "user": "{{.event.user}}"
}

// environment
{
  "SLACK_CHANNEL": "{{.event.channel}}",
  "SLACK_USER": "{{.event.user}}",
  "SLACK_THREAD_TS": "{{.event.thread_ts}}"
}
```

---

## Schedule Template Variables

Available in `environment`, `tags`, `params.message`, and `memory_key` for scheduled sessions.

> **Note:** Schedule template variables are only available for `memory_key` rendering. The `initial_message_template` in schedule `session_config` does not receive schedule context variables — use static text or configure the message directly.

### Variables

| Variable | Type | Description |
|----------|------|-------------|
| `.schedule_id` | `string` | Unique identifier of the schedule |
| `.schedule_name` | `string` | Human-readable name of the schedule |
| `.timezone` | `string` | Schedule timezone (IANA format, e.g., `Asia/Tokyo`) |

### Schedule Template Examples

```go
// memory_key
{
  "schedule_ref": "{{.schedule_id}}",
  "schedule_name": "{{.schedule_name}}"
}

// tags
{
  "schedule": "{{.schedule_name}}"
}
```

---

## Custom Webhook Template Variables

Custom webhooks accept arbitrary JSON payloads. All top-level keys from the incoming JSON body are available as template variables. Nested objects are accessed using dot notation.

### How Custom Payload Variables Work

Given a JSON payload:
```json
{
  "event_type": "deployment",
  "deployment": {
    "status": "success",
    "environment": "production",
    "service": "api-gateway",
    "version": "v1.2.3"
  },
  "triggered_by": "ci-pipeline",
  "timestamp": "2026-01-11T10:30:00Z"
}
```

All keys are available as template variables:
- `{{.event_type}}` → `deployment`
- `{{.deployment.status}}` → `success`
- `{{.deployment.environment}}` → `production`
- `{{.deployment.service}}` → `api-gateway`
- `{{.triggered_by}}` → `ci-pipeline`
- `{{.timestamp}}` → `2026-01-11T10:30:00Z`

### Custom Webhook Template Examples

```go
// initial_message_template for deployment event
🚀 Deployment {{.deployment.status}}

Service: {{.deployment.service}}
Version: {{.deployment.version}}
Environment: {{.deployment.environment}}
Triggered by: {{.triggered_by}}

{{if eq .deployment.status "failure"}}
⚠️ Deployment FAILED! Please investigate immediately.
{{else}}
✅ Deployment successful. Please verify the service is healthy.
{{end}}

// go_template condition for deployment events
{{ and (eq .event_type "deployment") (eq .deployment.environment "production") }}

// go_template condition for failure
{{ and (eq .event_type "deployment") (eq .deployment.status "failure") }}

// tags
{
  "service": "{{.deployment.service}}",
  "environment": "{{.deployment.environment}}",
  "version": "{{.deployment.version}}"
}
```

---

## Advanced Template Patterns

### Conditional Rendering

```go
{{if .pull_request.draft}}
[DRAFT] {{.pull_request.title}}
{{else}}
{{.pull_request.title}}
{{end}}
```

### Iterating Over Labels

```go
Labels:
{{range .pull_request.labels}}- {{.name}}
{{end}}
```

### Iterating Over Commits (Push Events)

```go
Recent commits to {{.repository.full_name}}:
{{range .commits}}- {{.message}} ({{.author.name}})
{{end}}
```

### String Manipulation in Templates

```go
// Lowercase branch name for use in tags
Branch: {{toLower .pull_request.head.ref}}

// Check if branch is a feature branch
{{if hasPrefix .pull_request.head.ref "feature/"}}
This is a feature branch.
{{end}}
```

### Using `in` for Multi-Value Checks

```go
// Match if base branch is main or develop
{{if in .pull_request.base.ref (list "main" "develop")}}
Target branch approved.
{{end}}

// Condition: event is push to main or develop
{{ in .ref (list "refs/heads/main" "refs/heads/develop") }}
```

### Combining Multiple Conditions

```go
{{ and
  (eq .action "opened")
  (not .pull_request.draft)
  (hasPrefix .pull_request.head.ref "feature/")
  (eq .pull_request.base.ref "main")
}}
```

---

## See Also

- [Webhook Triggers and Conditions](WEBHOOK_TRIGGERS.md) - How to use Go templates in trigger conditions
- [Webhook Reference](WEBHOOK_REFERENCE.md) - Full webhook API reference
- [Webhook Examples](WEBHOOK_EXAMPLES.md) - Real-world integration examples
