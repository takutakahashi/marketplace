---
name: cycle
description: |
  agentapi-proxy の cycle CLI を使って、エージェントセッションを繰り返し実行する。
  以下のような言葉が出たときに起動する:
  「繰り返して」「繰り返し実行して」「ループさせて」「ループして」「自動ループ」
  「成功するまで」「完了するまで」「条件を満たすまで」
  「何度も試して」「リトライして」「自動で回して」
  「cycle させて」「cycle して」
  Use when you need to: (1) セッションを指定条件が満たされるまで繰り返し実行する,
  (2) 最大繰り返し回数を指定してループを制御する,
  (3) AI エージェントが作業を自律的に繰り返し試行するよう設定する,
  (4) Stop hook を使って Claude Code のサイクル実行を自動化する,
  (5) 完了条件ファイル (/tmp/check/CYCLE_OK) によるループ終了を制御する.
---

# cycle — 繰り返し実行スキル

このスキルは `agentapi-proxy client cycle` コマンドを使い、エージェントセッションを条件が満たされるまで自動的に繰り返し実行する方法を提供します。

## 概要

`cycle` コマンドは、Claude Code の **Stop hook** に組み込まれ、エージェントが停止するたびに自動的に指定メッセージを再送信します。以下のいずれかの条件が成立するとループを終了します:

- `/tmp/check/CYCLE_OK` ファイルが存在する（エージェント自身が作成）
- `--max-count` で指定した最大繰り返し回数に達した

### 自動付加される完了確認指示

cycle コマンドが送信するメッセージには、以下の指示が **自動的に追記** されます。エージェントはこの指示に従い、達成条件を確認してループを自律的に終了できます:

```
---
⚠️ [cycle セッション] このメッセージを受け取ったら、まず上記の達成条件を確認してください。
条件をすべて満たしている場合は、追加の作業は行わずに直ちに以下のコマンドを実行してサイクルを終了させてください:

```bash
mkdir -p /tmp/check && touch /tmp/check/CYCLE_OK
```

条件を満たしていない場合は、引き続き作業を行ってください。
```

この仕組みにより、ユーザーは「テストが全部通るまで修正して」のような自然な指示を与えるだけで、エージェントが自律的にループを管理・終了します。

---

## 基本的な使い方

### コマンド構文

```bash
agentapi-proxy client cycle [message] [flags]
```

### フラグ一覧

| フラグ | 説明 | デフォルト |
|--------|------|------------|
| `--max-count N` | 最大サイクル回数（0 = 無制限） | 0 |
| `--session-id` | セッション ID（環境変数 `AGENTAPI_SESSION_ID` でも指定可） | - |
| `--endpoint` | プロキシのエンドポイント URL | 環境変数から自動解決 |

### 実行例

```bash
# 「タスクを続けてください」を成功するまで繰り返し送信
agentapi-proxy client cycle "タスクを続けてください"

# 最大 10 回まで繰り返す
agentapi-proxy client cycle --max-count 10 "続きを実装してください"

# セッション ID とエンドポイントを明示指定
agentapi-proxy client cycle \
  --session-id my-session \
  --endpoint http://localhost:8080 \
  "レビューを続けてください"
```

---

## セッション作成時に cycle を設定する方法

セッションを作成する際に `cycle_message` パラメータを指定すると、Stop hook が自動注入されます。

### API パラメータ

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `cycle_message` | string | Stop hook で繰り返し送信するメッセージ。空の場合は cycle 無効 |
| `cycle_max_count` | int | 最大繰り返し回数（0 = 無制限）。`cycle_message` が必要 |

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
2. agentapi-proxy が Stop hook を Claude Code 設定に自動注入
   ↓
3. Claude Code がタスクを処理して停止
   ↓
4. Stop hook が非同期で実行:
   nohup agentapi-proxy client cycle "<message>" --max-count N >> /tmp/cycle.log 2>&1 &
   ↓
5. cycle コマンドが以下を順番にチェック:
   a. /tmp/check/CYCLE_OK が存在する → 即時終了（完了）
   b. /tmp/check/CYCLE_COUNT が max-count に達した → 即時終了
   c. セッションが stable 状態になるまで最大 120 秒待機（2 秒間隔でポーリング）
   d. メッセージ + 完了確認指示（cycleConditionCheckSuffix）を自動付加
   e. エージェントにメッセージを送信
   ↓
6. エージェントが作業を継続
   ・達成条件を満たした → /tmp/check/CYCLE_OK を作成してループ終了
   ・まだ未達成 → 作業を続けて再度停止 → ステップ 3 へ
```

---

## 管理ファイル

| ファイルパス | 用途 |
|-------------|------|
| `/tmp/check/CYCLE_OK` | このファイルが存在するとループを終了するマーカー |
| `/tmp/check/CYCLE_COUNT` | 現在の繰り返し回数を記録するカウンター（整数テキスト） |
| `/tmp/cycle.log` | cycle コマンドの実行ログ |

### カウンターの動作

- カウンターはメッセージ送信**前**にインクリメントされます
- `/tmp/check/CYCLE_COUNT` が存在しない場合は 0 として扱います
- `--max-count` が 0 の場合、カウンターチェックはスキップされます

---

## ユースケース

### 1. テストが通るまで修正を繰り返す

```json
{
  "params": {
    "message": "make test を実行してテストが全て通るまでコードを修正してください。全テストが通ったら /tmp/check/CYCLE_OK を作成して終了してください。",
    "cycle_message": "テストの修正を続けてください",
    "cycle_max_count": 20
  }
}
```

### 2. lint/test が通るまで CI を改善する

```json
{
  "params": {
    "message": "lint と test を実行して全て通るまで修正してください。完了したら CYCLE_OK を作成してください。",
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

### 4. 手動で cycle を直接実行する

```bash
# 現在のセッションに対して手動で cycle を 5 回まで実行
agentapi-proxy client cycle \
  --session-id $AGENTAPI_SESSION_ID \
  --max-count 5 \
  "作業を続けてください"
```

---

## 注意事項

- `cycle_message` が空の場合、Stop hook は注入されません（cycle 無効）
- `cycle_max_count` を指定する場合は `cycle_message` も必須です
- cycle コマンドは Stop hook から `nohup` で非同期実行されるため、hook の完了をブロックしません
- セッションが stable 状態になるまで最大 **120 秒** 待機します（2 秒間隔でポーリング）
- カウンターはメッセージ送信**前**にインクリメントされます
- メッセージには常に `cycleConditionCheckSuffix`（完了確認指示）が自動付加されます。これによりエージェントが自律的に CYCLE_OK を作成してループを終了できます
