---
name: session-manager-setup
description: |
  Set up and manage an External Session Manager (ESM) for agentapi-proxy on a native
  Linux/macOS host or a self-hosted Kubernetes server. Use when you need to install,
  register, inspect, update, rotate credentials for, or remove an ESM; route sessions
  with allocator labels; or diagnose ESM connectivity and heartbeat failures.
---

# External Session Manager (ESM)

An ESM lets a parent agentapi-proxy route sessions to another machine. Native ESMs use
outbound allocation polling, so the parent does not need inbound access for allocation.
The parent must still be able to reach the ESM's `public_url` for session traffic.

```text
user -> parent /start -> allocation queue
                         ^ outbound polling by ESM
user -> parent /:sessionId/* -> HMAC-signed proxy -> ESM public_url
```

## Native Linux/macOS installation (recommended)

Use the one-command installer. It registers the host, installs and starts the daemon,
then verifies local health and the parent heartbeat.

```bash
export AGENTAPI_KEY="<parent-proxy-api-key>"

sudo --preserve-env=AGENTAPI_KEY agentapi-proxy native install \
  --upstream https://parent-proxy.example.com \
  --public-url http://10.0.0.10:8080 \
  --name native-builder-01 \
  --label pool=native \
  --label machine=native-builder-01
```

On macOS, omit `sudo`; the command installs a per-user LaunchAgent. Prefer
`--api-key-stdin` or `--api-key-file` if preserving the environment is undesirable.
The API key is used only for registration and is not saved in daemon configuration.

Installation is idempotent: a stable instance ID is kept, registration and service
configuration are updated on subsequent runs, and the connection token is preserved.
The installer automatically detects `os`, `arch`, and `hostname` labels.

### Lifecycle commands

```bash
agentapi-proxy native status
agentapi-proxy native doctor
agentapi-proxy native restart
agentapi-proxy native rotate-token
agentapi-proxy native uninstall
```

Linux stores configuration under `/etc/agentapi-native`, state under
`/var/lib/agentapi-native`, and the managed executable under
`/usr/local/libexec/agentapi-proxy`. macOS uses
`~/Library/Application Support/agentapi-native` and
`~/Library/LaunchAgents/com.agentapi.native.plist`.

Native sessions are not sandboxed. Use a dedicated host for one user or a mutually
trusted team. Each session has an isolated directory beneath
`<state-dir>/sessions/<session-id>`; deleting the public session terminates its process
group and removes that directory.

## Registration API

The CLI is preferred for installation, but the parent API supports complete ESM
management. These endpoints currently require direct REST calls; the
`agentapi-proxy client` has no ESM subcommand.

```bash
PARENT_PROXY_URL="https://parent-proxy.example.com"
API_KEY="<parent-proxy-api-key>"

# Idempotently register by stable instance_id. connection_token is returned only
# on creation or explicit rotation.
curl -X POST "$PARENT_PROXY_URL/external-session-managers" \
  -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "instance_id": "host-stable-id",
    "name": "native-builder-01",
    "scope": "user",
    "labels": {"pool":"native", "os":"linux"},
    "default": false,
    "public_url": "http://10.0.0.10:8080"
  }'

curl -H "X-API-Key: $API_KEY" \
  "$PARENT_PROXY_URL/external-session-managers?scope=user"

curl -X PATCH "$PARENT_PROXY_URL/external-session-managers/<manager-id>" \
  -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"instance_id":"host-stable-id","name":"native-builder-01","labels":{"pool":"build"}}'

curl -X POST -H "X-API-Key: $API_KEY" \
  "$PARENT_PROXY_URL/external-session-managers/<manager-id>/rotate-token"

curl -X DELETE -H "X-API-Key: $API_KEY" \
  "$PARENT_PROXY_URL/external-session-managers/<manager-id>"
```

For team ownership, set `scope=team` and include `team_id` in registration and list
requests. Treat `connection_token` as a secret; it is returned once on creation or
rotation. `GET /external-session-managers/{id}` exposes only
`has_connection_token`, not the token value.

The daemon sends `POST /external-session-managers/{id}/heartbeat` with its connection
token. The parent verifies that `public_url/healthz` is reachable. HTTP `401` means the
token is invalid; HTTP `424` means the public URL is unreachable.

## Routing sessions

Manager labels are selected with `allocator.*` session tags. Multiple tags use AND
semantics, and `allocator.id` selects one manager exactly.

```bash
curl -X POST "$PARENT_PROXY_URL/start" \
  -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "tags": {"allocator.pool":"native", "allocator.os":"linux"},
    "params": {"message":"Run the build", "agent_type":"codex-acp"}
  }'
```

If a manager has `default: true`, sessions without allocator tags may route to it.

## Kubernetes ESM (advanced)

For a Kubernetes-backed ESM, run the normal `agentapi-proxy server` with Kubernetes
provisioning enabled. Register it first to obtain the connection token, then configure:

```bash
export SESSION_MANAGER_ENABLED=true
export SESSION_MANAGER_UPSTREAM_URL="https://parent-proxy.example.com"
export SESSION_MANAGER_CONNECTION_TOKEN="<connection-token>"
export SESSION_MANAGER_HMAC_SECRET="<connection-token>"
export SESSION_MANAGER_PUBLIC_URL="https://esm.example.com"
export AGENTAPI_K8S_SESSION_PROVISIONER_PROXY_URL="https://esm.example.com"
```

The same token authenticates outbound allocator polling and verifies HMAC-signed proxy
requests. `SESSION_MANAGER_PUBLIC_URL` must be reachable from the parent, and provisioned
session pods must call the ESM through `AGENTAPI_K8S_SESSION_PROVISIONER_PROXY_URL`.

## Verification and troubleshooting

After `/start`, query the returned session through the parent:

```bash
curl -H "X-API-Key: $API_KEY" "$PARENT_PROXY_URL/<session-id>/status"
```

- Run `agentapi-proxy native doctor` first for native installations.
- `503 External session manager has not reported a routable session yet`: verify the
  ESM public URL and heartbeat.
- `404 Session not found`: verify that the ESM reports the concrete local session ID.
- `invalid signature`: rotate/reinstall credentials and confirm the connection token is
  also used as `SESSION_MANAGER_HMAC_SECRET`.
- Heartbeat `424`: ensure the parent can reach `<public_url>/healthz`.
- Native ESM logs and status should show successful outbound polling to the parent.
