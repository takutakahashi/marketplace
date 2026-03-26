---
name: agentapi-proxy-api
description: |
  Interact with agentapi-proxy API using API Key authentication for session management.
  Use when you need to: (1) Create new agentapi sessions, (2) Search and list existing sessions,
  (3) Delete sessions, (4) Route requests to specific session instances, (5) Manage session sharing,
  (6) Access user settings and notifications, (7) Create and manage tasks associated with sessions,
  (8) Manage task groups for organizing tasks, (9) Create and manage memory entries for storing
  contextual information. Supports multiple authentication methods including static API keys
  (X-API-Key header) and Authorization Bearer tokens.
  Note: For schedule management, use the schedule-management skill instead. For webhook management,
  use the webhook-management skill instead. For SlackBot management, use the slackbot-management
  skill instead.
---

# agentapi-proxy API

This skill provides guidance for interacting with the agentapi-proxy API.

## CLI: `agentapi-proxy client`

`agentapi-proxy client` を使うことで全リソースへアクセスできます。以下のサブコマンドが利用可能です：

| サブコマンド | 用途 | 操作 |
|-------------|------|------|
| `task` | タスク管理 | create / list / get / update / delete |
| `memory` | メモリ管理 | create / list / get / update / delete / upsert |
| `schedule` | スケジュール管理 | create / list / get / apply / delete |
| `webhook` | Webhook 管理 | create / list / get / apply / delete / regenerate-secret |
| `slackbot` | SlackBot 管理 | create / list / get / apply / delete |
| `send` | エージェントへのメッセージ送信 | - |
| `status` | エージェントのステータス取得 | - |
| `events` | エージェントイベントの監視 | - |
| `history` | 会話履歴の取得 | - |
| `delete-session` | 現在のセッションの削除 | - |
| `send-notification` | プッシュ通知の送信 | - |
| `summarize-drafts` | ドラフトメモリの要約 | - |

> **Note:** タスクグループ (`task-group`) は CLI 未対応です。MCP ツールを使用してください。

### 接続設定

エンドポイントと認証は以下の環境変数から自動解決されます：

```bash
export AGENTAPI_PROXY_SERVICE_HOST=<host>
export AGENTAPI_PROXY_SERVICE_PORT_HTTP=<port>  # または AGENTAPI_PROXY_SERVICE_PORT
export AGENTAPI_KEY=<api-key>                    # 省略可（認証不要な場合）
```

または `--endpoint` フラグで明示的に指定：

```bash
agentapi-proxy client <command> --endpoint http://proxy:8080 --session-id SESSION_ID
```

## Core Workflows

### Managing Tasks

Tasks are units of work associated with a session. There are two types: `agent` tasks (created by the AI agent) and `user` tasks (created for human action).

**Create a task:**
```bash
# Agent task (default) - work performed by the agent
agentapi-proxy client task create \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --title "Implement feature X" \
  --task-type agent \
  --scope user

# User task - action required from a human
agentapi-proxy client task create \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --title "Review PR #123" \
  --task-type user \
  --scope user \
  --description "Please review and approve the pull request" \
  --link "https://github.com/owner/repo/pull/123|PR #123"
```

**List tasks:**
```bash
# List all tasks
agentapi-proxy client task list \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID

# Filter by status (todo / done)
agentapi-proxy client task list \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --status todo

# Filter by type
agentapi-proxy client task list \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --task-type user
```

**Update a task:**
```bash
agentapi-proxy client task update TASK_ID \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --status done
```

**Get / Delete a task:**
```bash
# Get
agentapi-proxy client task get TASK_ID \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID

# Delete
agentapi-proxy client task delete TASK_ID \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID
```

See [TASK_REFERENCE.md](references/TASK_REFERENCE.md) for complete task API documentation.

### Managing Memory Entries

Memory entries allow storing and retrieving contextual information for agents and users. Supports user-scoped (private) and team-scoped (shared) memories.

**Create a memory entry:**
```bash
# User-scoped memory
agentapi-proxy client memory create \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --title "Project conventions" \
  --content "Always use TypeScript for new files. Follow ESLint rules." \
  --scope user

# Team-scoped memory with tags
agentapi-proxy client memory create \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --title "Team knowledge" \
  --content-file /tmp/notes.md \
  --scope team --team-id myorg/myteam \
  --tag project=myapp
```

**List memory entries:**
```bash
# List all user memories
agentapi-proxy client memory list \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --scope user

# List team memories filtered by tag
agentapi-proxy client memory list \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --scope team --team-id myorg/myteam \
  --tag project=myapp

# Output in Markdown format (suitable for injection into CLAUDE.md)
agentapi-proxy client memory list \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --format markdown
```

**Get / Update / Delete a memory entry:**
```bash
# Get
agentapi-proxy client memory get MEMORY_ID \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID

# Update
agentapi-proxy client memory update MEMORY_ID \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --title "Updated conventions" \
  --content "Updated content"

# Delete
agentapi-proxy client memory delete MEMORY_ID \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID
```

**Upsert (create or update by key tags):**
```bash
# Create or update a memory identified by key tags
agentapi-proxy client memory upsert \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --title "Session summary" \
  --content-file /tmp/content.md \
  --key project=myapp --key env=prod \
  --scope user
```

`upsert` はキータグで既存のメモリを検索し、見つかれば更新、なければ作成します。定期的なメモリ更新に便利です。

**Summarize draft memories:**
```bash
# セッション終了後、ドラフトメモリを要約して本メモリに統合
agentapi-proxy client summarize-drafts \
  --endpoint http://proxy:8080 \
  --session-id SESSION_ID \
  --source-session-id SOURCE_SESSION_ID \
  --scope user \
  --key project=myapp
```

### Managing Task Groups

Task groups provide a way to organize and manage related tasks together.

> **Note:** タスクグループは CLI 未対応です。MCP ツールを使用してください。

```
# Create a task group
Use the mcp__ccplant__create_task_group tool

# List task groups
Use the mcp__ccplant__list_task_groups tool

# Delete a task group
Use the mcp__ccplant__delete_task_group tool
```

## API Reference

For complete API endpoint documentation, permissions, and authentication details, see:
- [API_REFERENCE.md](references/API_REFERENCE.md) - Complete endpoint reference
- [AUTHENTICATION.md](references/AUTHENTICATION.md) - Authentication methods and configuration
- [PERMISSIONS.md](references/PERMISSIONS.md) - Role-based access control details
- [TASK_REFERENCE.md](references/TASK_REFERENCE.md) - Task management API reference
