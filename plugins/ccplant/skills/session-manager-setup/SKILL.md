---
name: session-manager-setup
description: |
  Set up an External Session Manager (ESM) for agentapi-proxy using a self-hosted k3s server.
  Use when you need to: (1) Deploy agentapi-proxy in session manager mode on a separate server,
  (2) Offload agent session workloads from the main cluster to a dedicated machine,
  (3) Configure an External Session Manager (ESM) in Proxy A user settings so all new sessions
  are routed to the self-hosted Proxy B, (4) Verify end-to-end HMAC-signed session creation
  between Proxy A and Proxy B.
---

# External Session Manager (ESM) セットアップ

This skill documents how to deploy a self-hosted **agentapi-proxy in session manager mode** (Proxy B)
on a bare-metal or VM server running k3s, and wire it up as an External Session Manager in the
main agentapi-proxy (Proxy A).

## アーキテクチャ概要

```
ユーザー → Proxy A (main cluster) → HMAC署名付きリクエスト → Proxy B (k3s / ESM)
                                                                   ↓
                                                          agentapi セッションPod
```

- **Proxy A**: 既存の agentapi-proxy (例: `agentapi-proxy.agentapi-ui-dev.svc.cluster.local:8080`)
- **Proxy B**: セルフホストサーバ上の agentapi-proxy (`SESSION_MANAGER_ENABLED=true`)
- **HMAC**: Proxy A → Proxy B 間の通信は `X-Hub-Signature-256` + `X-Timestamp` ヘッダで署名・検証される

---

## Step 1: サーバに k3s をインストール

対象サーバ (例: `10.20.11.65`) に SSH でログインし、k3s をインストールします。

```bash
# k3s インストール
curl -sfL https://get.k3s.io | sh -

# kubeconfig の確認
sudo kubectl get nodes
```

動作確認:
```bash
sudo kubectl get nodes
# NAME        STATUS   ROLES                  AGE   VERSION
# myserver    Ready    control-plane,master   1m    v1.x.x+k3s1
```

---

## Step 2: HMAC シークレットの生成

Proxy A と Proxy B の間で共有するシークレットを生成します。

```bash
# ランダムな64文字のシークレットを生成
openssl rand -hex 32
# 例: (出力をメモしておく — Proxy A の ESM 設定と Proxy B の Helm values 両方に設定する)
```

> **注意**: 生成したシークレットは安全な場所に保管してください。Proxy A の ESM 設定と
> Proxy B の Helm デプロイ時に同じ値を使用します。

---

## Step 3: k3s に Proxy B をデプロイ (セッションマネージャーモード)

### Helm values ファイルの作成

```yaml
# session-manager-values.yaml
replicaCount: 1

kubernetesSession:
  enabled: true
  namespace: agentapi
  replicaCount: 1
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2"
      memory: "4Gi"
  pvc:
    enabled: true
    storageSize: "10Gi"
  otelCollector:
    enabled: false

config:
  auth:
    static:
      enabled: true
      headerName: "X-API-Key"
    github:
      enabled: false

env:
  - name: SESSION_MANAGER_ENABLED
    value: "true"
  - name: SESSION_MANAGER_HMAC_SECRET
    value: "<YOUR_HMAC_SECRET>"   # Step 2 で生成したシークレット

resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "1Gi"
    cpu: "1"

scheduleWorker:
  enabled: false
slackbotCleanupWorker:
  enabled: false

service:
  type: NodePort
  port: 8080
  nodePort: 30080   # Proxy A からアクセスできるポート
```

> **メモリ要件**: セッション Pod はデフォルトで最大 4GiB のメモリを要求します。
> サーバに十分な RAM (推奨: 8GB 以上) があることを確認してください。

### Helm でデプロイ

```bash
# k3s サーバ上で実行
helm repo add agentapi-proxy oci://ghcr.io/takutakahashi/charts/agentapi-proxy || true

helm upgrade --install agentapi-proxy \
  oci://ghcr.io/takutakahashi/charts/agentapi-proxy \
  --namespace agentapi-proxy \
  --create-namespace \
  -f session-manager-values.yaml \
  --wait
```

### デプロイ確認

```bash
kubectl rollout status deployment \
  -n agentapi-proxy \
  -l app.kubernetes.io/name=agentapi-proxy \
  --timeout=120s

# NodePort が開いていることを確認
curl -s http://localhost:30080/healthz
# 期待値: 200 OK
```

---

## Step 4: Proxy A に ESM を登録

Proxy A のユーザー設定 (`/settings/:username`) に ESM を追加します。

```bash
# Proxy A のエンドポイントと API キーを設定
PROXY_A_URL="http://<proxy-a-host>:8080"
API_KEY="<YOUR_PROXY_A_API_KEY>"
USERNAME="<your-github-username>"

# 現在の設定を取得
curl -H "X-API-Key: $API_KEY" \
  "$PROXY_A_URL/settings/$USERNAME" | jq .

# ESM を追加
curl -X PUT "$PROXY_A_URL/settings/$USERNAME" \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "external_session_managers": [
      {
        "name": "my-k3s-server",
        "url": "http://<PROXY_B_IP>:30080",
        "hmac_secret": "<YOUR_HMAC_SECRET>",
        "default": true
      }
    ]
  }'
```

フィールド説明:
- `name`: ESM の識別名 (任意)
- `url`: Proxy B の URL (`http://<k3sサーバIP>:30080`)
- `hmac_secret`: Step 2 で生成したシークレット (Proxy B と同じ値)
- `default: true`: このユーザーの全新規セッションを ESM にルーティング

### ESM 設定の確認

```bash
curl -H "X-API-Key: $API_KEY" \
  "$PROXY_A_URL/settings/$USERNAME" | jq '.external_session_managers'
# 期待値: 登録した ESM の配列
```

---

## Step 5: 動作確認

### セッションを作成して ESM 経由で起動されることを確認

```bash
# セッションを開始 (Proxy A 経由)
curl -X POST "$PROXY_A_URL/start" \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hello from ESM test!"
  }'
```

### Proxy B 側で Pod が起動していることを確認 (k3s サーバ上)

```bash
# k3s サーバ上で確認
kubectl get pods -n agentapi
# agentapi-* という Pod が Running になっていれば成功
```

### Proxy A のログで HMAC 転送を確認

```bash
kubectl logs -n agentapi-ui-dev \
  -l app.kubernetes.io/name=agentapi-proxy \
  --tail=50 | grep -i "session_manager\|hmac\|esm"
```

---

## トラブルシューティング

### "authentication required" が返る

Proxy B が HMAC 署名を正しく検証できていません。以下を確認してください:

1. `SESSION_MANAGER_HMAC_SECRET` が Proxy A の `hmac_secret` と一致しているか
2. Proxy B の agentapi-proxy バージョンが HMAC 検証 (canonical message format) をサポートしているか
   - バグが修正された PR: `takutakahashi/agentapi-proxy#722`
3. k3s サーバとメインクラスターの時刻が同期されているか (最大許容スキュー: 5分)
   ```bash
   timedatectl status  # k3s サーバ上で確認
   ```

### NodePort (30080) にアクセスできない

```bash
# ファイアウォール / セキュリティグループの確認
sudo ufw status
# または
sudo iptables -L INPUT | grep 30080

# k3s サービスの確認
kubectl get svc -n agentapi-proxy
# NodePort 列に 30080 が表示されることを確認
```

Helm upgrade 時に `--reuse-values` を使うと NodePort 設定が失われることがあります。
必ず `-f session-manager-values.yaml` を使用してください。

### セッション Pod が Pending のまま

```bash
kubectl describe pod -n agentapi <pod-name>
# "Insufficient memory" / "Insufficient cpu" が表示される場合
# → サーバのリソースを確認。推奨: CPU 2コア以上、RAM 8GB 以上
```

1-CPU サーバでは Rolling Update 中に旧 Pod が残って CPU 枯渇することがあります。
旧 Pod を手動で削除してください:
```bash
kubectl delete pod -n agentapi-proxy <old-pod-name>
```

### ESM 設定が消える

`PUT /settings/:username` に `external_session_managers` を含めない場合、
既存の ESM 設定は維持されます (フィールドは `omitempty` で省略可)。
ただし Helm upgrade 後など、設定が初期化された場合は Step 4 を再実行してください。

---

## HMAC 署名の仕組み (参考)

Proxy A が Proxy B へリクエストを転送する際、以下の canonical message を
HMAC-SHA256 で署名します:

```
METHOD\nPATH?QUERY\nTIMESTAMP\nBODY
```

- `METHOD`: HTTP メソッド (大文字)
- `PATH?QUERY`: リクエスト URI (クエリ文字列含む)
- `TIMESTAMP`: Unix epoch (秒, 10 進数文字列)
- `BODY`: リクエストボディのバイト列 (空の場合は省略)

ヘッダー:
- `X-Hub-Signature-256: sha256=<hex>` — HMAC-SHA256 署名
- `X-Timestamp: <epoch>` — タイムスタンプ (リプレイ攻撃防止)

詳細な実装は `pkg/hmacutil/hmac.go` を参照してください。
