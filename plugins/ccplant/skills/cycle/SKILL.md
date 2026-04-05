---
name: cycle
description: |
  agentapi-proxy の cycle CLI を使って、エージェントセッションを条件が満たされるまで自動的に繰り返し実行する。
  以下のような言葉が出たときに起動する:
  「繰り返して」「繰り返し実行して」「ループさせて」「ループして」「自動ループ」
  「成功するまで」「完了するまで」「条件を満たすまで」
  「何度も試して」「リトライして」「自動で回して」
  「cycle させて」「cycle して」
  Use when you need to: (1) セッションを指定条件が満たされるまで繰り返し実行する,
  (2) 最大繰り返し回数を指定してループを制御する,
  (3) AI エージェントが作業を自律的に繰り返し試行するよう設定する,
  (4) Stop hook を使って Claude Code のサイクル実行を自動化する,
  (5) CYCLE_ENABLED ファイルによるループの有効化・無効化を制御する.
---

# cycle — 繰り返し実行スキル

このスキルは `agentapi-proxy client cycle` コマンドを使い、エージェントセッションを条件が満たされるまで自動的に繰り返し実行する方法を提供します。

## 概要

`cycle` コマンドは、Claude Code の **Stop hook** に組み込まれ、エージェントが停止するたびに自動的にメッセージを再送信します。

### 基本抽象

- **`/tmp/check/CYCLE_ENABLED`** ファイルが**存在するとキック**、その**ファイル内容がメッセージとして送信**されます
- **ファイルが存在しない場合**、cycle は何もしない（ノーオペレーション）
- **ループを終了する**には `CYCLE_ENABLED` を削除する（エージェント自身も実行できる）
- Stop hook は**永続的に登録**され、`CYCLE_ENABLED` の有無によって動作を制御

### 自動付加される完了確認指示

cycle が送信するメッセージには、以下の指示が **自動的に追記** されます。エージェントは達成条件を確認し、満たしていれば `CYCLE_ENABLED` を削除してループを自律的に終了します:

```
---
⚠️ [cycle セッション] このメッセージを受け取ったら、まず上記の達成条件を確認してください。
条件をすべて満たしている場合は、追加の作業は行わずに直ちに以下のコマンドを実行してサイクルを終了させてください:

```bash
rm -f /tmp/check/CYCLE_ENABLED
```

条件を満たしていない場合は、引き続き作業を行ってください。
```

---

## 基本的な使い方

### コマンド構文

```bash
agentapi-proxy client cycle [flags]
```

> **重要**: メッセージは CLI 引数ではなく、`/tmp/check/CYCLE_ENABLED` のファイル内容から読み取ります。

### フラグ一覧

| フラグ | 説明 | デフォルト |
|--------|------|------------|
| `--max-count N` | 最大サイクル回数（0 = 無制限） | **10** |
| `--session-id` | セッション ID（環境変数 `AGENTAPI_SESSION_ID` でも指定可） | - |
| `--endpoint` | プロキシのエンドポイント URL | 環境変数から自動解決 |

### 手動でエーブル化する例

```bash
# CYCLE_ENABLED にメッセージを書き込んでキック開始
mkdir -p /tmp/check
echo "タスクを続けてください" > /tmp/check/CYCLE_ENABLED

# ループを止めたいときは削除する
rm -f /tmp/check/CYCLE_ENABLED

# 最大 5 回まで制限
agentapi-proxy client cycle --max-count 5

# セッション ID とエンドポイントを明示指定
agentapi-proxy client cycle \
  --session-id my-session \
  --endpoint http://localhost:8080
```

---

## セッション作成時に cycle を設定する方法

セッションを作成する際に `cycle_message` パラメータを指定すると、以下が自動実行されます:

1. `/tmp/check/CYCLE_ENABLED` に `cycle_message` の内容が書き込まれる（セッション起動時に作成）
2. Stop hook に `agentapi-proxy client cycle` が登録される（メッセージ引数なし）
3. エージェントが停止するたびに hook が発火→ `CYCLE_ENABLED` を読んで送信

### API パラメータ

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `cycle_message` | string | `/tmp/check/CYCLE_ENABLED` に書き込まれるメッセージ。空の場合は cycle 無効 |
| `cycle_max_count` | int | 最大繰り返し回数（0 = 無制限）。デフォルト **10**。`cycle_message` が必要 |

### セッション作成例（API）

```json
{
  "params": {
    "message": "テストが全て通るまでコードを修正してください",
    "cycle_message": "続きの作業を進めてください",
    "cycle_max_count": 10
  }
}
```

---

## 動作フロー

```
1. セッション開始（cycle_message 付き）
   ↓
2. agentapi-proxy が /tmp/check/CYCLE_ENABLED にメッセージを書き込む
   agentapi-proxy が Stop hook (引数なし) を Claude Code 設定に注入
   ↓
3. Claude Code がタスクを処理して停止
   ↓
4. Stop hook が非同期で実行:
   nohup agentapi-proxy client cycle [--max-count N] >> /tmp/cycle.log 2>&1 &
   ↓
5. cycle コマンドが以下を順番に実行:
   a. /tmp/check/CYCLE_ENABLED が存在しない → "CYCLE_ENABLED not found" と出力して即時終了
   b. CYCLE_ENABLED の内容をメッセージとして読み込む
   c. /tmp/check/CYCLE_COUNT が max-count に達した → CYCLE_ENABLED を削除して即時終了
   d. セッションが stable 状態になるまで最大 120 秒待機（2 秒間隔でポーリング）
   e. メッセージ + cycleConditionCheckSuffix を自動付加して送信
   ↓
6. エージェントが作業を継続
   ・達成条件を満たした → rm -f /tmp/check/CYCLE_ENABLED してループ終了
   ・まだ未達成 → 作業を続けて再度停止 → ステップ 3 へ
```

---

## ループの制御

### ループを有効化する（エーブル）

```bash
mkdir -p /tmp/check
echo "続きの作業を進めてください" > /tmp/check/CYCLE_ENABLED
```

### ループを無効化する（ディスエーブル）

```bash
rm -f /tmp/check/CYCLE_ENABLED
```

### エージェントが自律的にループを終了する（自律的完了）

エージェントは cycleConditionCheckSuffix の指示に従い、達成条件を満たしたら以下を実行します:

```bash
rm -f /tmp/check/CYCLE_ENABLED
```

---

## 管理ファイル

| ファイルパス | 用途 |
|-------------|------|
| `/tmp/check/CYCLE_ENABLED` | 存在すると cycle が有効。内容が送信メッセージ |
| `/tmp/check/CYCLE_COUNT` | 現在の繰り返し回数を記録するカウンター |
| `/tmp/cycle.log` | cycle コマンドの実行ログ |

### カウンターの動作

- カウンターはメッセージ送信**前**にインクリメントされます
- `/tmp/check/CYCLE_COUNT` が存在しない場合は 0 として扱います
- `--max-count` が 0 の場合、カウンターチェックはスキップされます
- **上限に達した場合、`CYCLE_ENABLED` が自動的に削除**されます

---

## ユースケース

### 1. テストが通るまで修正を繰り返す

```json
{
  "params": {
    "message": "make test を実行してテストが全て通るまでコードを修正してください。全テストが通ったらループを終了してください。",
    "cycle_message": "テストの修正を続けてください",
    "cycle_max_count": 20
  }
}
```

### 2. lint/test が通るまで CI を改善する

```json
{
  "params": {
    "message": "lint と test を実行して全て通るまで修正してください。完了したらループを終了してください。",
    "cycle_message": "引き続き lint/test を修正してください",
    "cycle_max_count": 15
  }
}
```

### 3. 無制限にタスクを処理し続ける

```json
{
  "params": {
    "message": "Issue リストのタスクを順番に処理してください",
    "cycle_message": "次のタスクを処理してください",
    "cycle_max_count": 0
  }
}
```

### 4. 手動で CYCLE_ENABLED を使ってループを制御する

```bash
# cycle をエーブル化（メッセージも書き込む）
mkdir -p /tmp/check
echo "作業を続けてください" > /tmp/check/CYCLE_ENABLED

# cycle を無効化（いつでも実行可能）
rm -f /tmp/check/CYCLE_ENABLED
```

---

## 注意事項

- `cycle_message` が空の場合、Stop hook は注入されません（cycle 無効）
- `cycle_max_count` のデフォルトは **10**（0 の場合のみ無制限）
- `--max-count` の上限に達したとき、`CYCLE_ENABLED` が自動的に削除されます
- cycle コマンドは Stop hook から `nohup` で非同期実行されるため、hook の完了をブロックしません
- セッションが stable 状態になるまで最大 **120 秒** 待機します（2 秒間隔でポーリング）
- カウンターはメッセージ送信**前**にインクリメントされます
- Stop hook にメッセージ引数は無い（`CYCLE_ENABLED` の内容を直接読む）
