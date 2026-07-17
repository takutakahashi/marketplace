---
name: ccplant-session
description: Create and coordinate CCPlant agent sessions for delegated or parallel work through the agentapi-proxy start endpoint. Use when Codex needs to split independent tasks across CCPlant sessions, send initial or follow-up instructions, inspect session status or conversation messages, wait for results, and combine the completed work.
---

# CCPlant Session

Delegate only concrete, independently executable work. Keep ownership of planning, dependency ordering, result review, and final integration in the parent session.

## Configure access

Resolve the proxy URL from the explicit variable first, then the Kubernetes service variables:

```bash
CCPLANT_URL="${AGENTAPI_PROXY_URL:-http://${AGENTAPI_PROXY_SERVICE_HOST}:${AGENTAPI_PROXY_SERVICE_PORT_HTTP:-${AGENTAPI_PROXY_SERVICE_PORT:-8080}}}"
```

Use `AGENTAPI_KEY` for authentication. Never print the key or place secrets in task messages. Stop and ask the user for configuration if the URL or required authentication cannot be resolved.

## Plan delegation

Before creating sessions:

1. Split work into bounded tasks with minimal overlap.
2. State the expected artifact, relevant paths, constraints, and completion checks for each task.
3. Identify dependencies. Start dependent work only after its prerequisite result is available.
4. Assign a distinct branch or non-overlapping file ownership when sessions will edit the same repository.
5. Keep the number of sessions proportional to truly parallel work; do not create a session for trivial work.

## Create a session

Build JSON with `jq` so multiline instructions are escaped safely. Prefer the configured agent type, falling back to `codex-acp`:

```bash
TASK='Inspect the authentication package. Report the root cause with file and line references. Do not modify files.'
PAYLOAD="$(jq -n \
  --arg message "$TASK" \
  --arg agent_type "${AGENTAPI_AGENT_TYPE:-codex-acp}" \
  --arg repository "${AGENTAPI_REPO_FULLNAME:-unknown}" \
  '{
    tags: {purpose: "parallel-work", repository: $repository},
    params: {
      message: $message,
      agent_type: $agent_type,
      oneshot: false,
      session_ttl: "24h"
    }
  }')"

RESPONSE="$(curl --fail-with-body --silent --show-error \
  -X POST "$CCPLANT_URL/start" \
  -H "X-API-Key: $AGENTAPI_KEY" \
  -H 'Content-Type: application/json' \
  --data "$PAYLOAD")"
SESSION_ID="$(jq -er '.session_id' <<<"$RESPONSE")"
```

Create multiple independent sessions without waiting for one to finish before starting the next. Record each session ID together with its assignment. Do not use `oneshot: true` when follow-up questions may be necessary.

Include repository and branch context in the task message when edits are requested. Explicitly require the session to preserve unrelated changes, run relevant checks, and return commit or PR details if that is the intended handoff.

## Communicate and inspect

Use the CLI for normal interaction:

```bash
agentapi-proxy client status \
  --endpoint "$CCPLANT_URL" --session-id "$SESSION_ID"

agentapi-proxy client history \
  --endpoint "$CCPLANT_URL" --session-id "$SESSION_ID"

agentapi-proxy client send 'Please also run the focused integration test and report its output.' \
  --endpoint "$CCPLANT_URL" --session-id "$SESSION_ID"
```

Use `events` for short interactive observation. For automation, prefer bounded long-polling so the parent remains responsive:

```bash
curl --fail-with-body --silent --show-error \
  -H "X-API-Key: $AGENTAPI_KEY" \
  "$CCPLANT_URL/sessions/$SESSION_ID/messages/wait?timeout=30"
```

After an update, call `history` to read the actual messages. Reuse the returned `timestamp` as the URL-encoded `since` value on the next wait when appropriate. Treat a timeout (`{"updated":false}`) as normal, not as failure.

## Review and integrate

For every delegated task:

1. Read the final history; do not infer success from status alone.
2. Verify claimed files, commits, PRs, and test results in the authoritative repository.
3. Send a focused follow-up if evidence is missing or the output does not meet the assignment.
4. Resolve conflicting recommendations in the parent session.
5. Integrate results in dependency order and run the combined verification.
6. Report which session produced each accepted result and disclose incomplete or failed tasks.

Do not delete sessions merely because their task finished. Delete one only when the user requested cleanup or retention policy requires it:

```bash
curl --fail-with-body --silent --show-error \
  -X DELETE -H "X-API-Key: $AGENTAPI_KEY" \
  "$CCPLANT_URL/sessions/$SESSION_ID"
```

## Handle failures

- On `401` or `403`, verify endpoint, key, scope, and session ownership without exposing credentials.
- On `404`, confirm the recorded session ID and whether TTL or oneshot cleanup removed it.
- On `502`, retry status after a short interval; provisioning may still be in progress.
- If a session stalls, inspect history before sending a concise unblock message.
- If repeated attempts make no progress, stop creating replacements and report the blocker.
