---
description: "Worktree作成・起動・停止・削除（全リポジトリ共通）"
argument-hint: "<create|up|down|remove|list> <リポジトリ名> [ブランチ名|issue番号]"
allowed-tools: ["Bash", "Read", "Glob", "Grep"]
---

# worktree: 並列開発環境管理

git worktree を使った並列開発環境の作成・起動・停止・削除を行う。
vendor/node_modules/生成ファイルのコピー、Docker volume共有、Makefile生成を自動で処理する。

**入力:** "$ARGUMENTS"

---

## スクリプトの場所

`$COMPANY_HOME/.company/scripts/worktree.sh`

`COMPANY_HOME` が未設定の場合は `~/itns-workspace/cc-company` をデフォルトとする。

---

## コマンド一覧

### create: worktree作成

```bash
bash "$SCRIPT_PATH" create <リポジトリ名> <ブランチ名>
```

**ブランチ名の決定:**
- `.company/CLAUDE.md` のブランチ命名規約に従う
  - tasks-division-cross-dev → `tasks-division-cross-dev-{Issue番号}`
  - 事業部リポジトリ → `feature/{Issue番号}`
- ブランチが存在しない場合、スクリプトが `develop` の最新から detached HEAD で作成する

**実行前チェック:**
1. 対象リポジトリが `~/itns-workspace/` に存在するか確認
2. 既にworktreeが存在しないか確認（存在する場合はスキップ）

**実行後:**
- worktreeパスをユーザーに報告する
- `{リポジトリ名}-wt-{issue番号}` が `~/itns-workspace/` に作成される

### up: worktree起動（Docker）

```bash
bash "$SCRIPT_PATH" up <リポジトリ名> <issue番号>
```

本体コンテナを停止し、worktree側のコンテナを起動する。

### down: worktree停止（Docker）

```bash
bash "$SCRIPT_PATH" down <リポジトリ名> <issue番号>
```

worktree側のコンテナを停止する。本体に戻すには `cd {本体} && make up`。

### remove: worktree削除

```bash
bash "$SCRIPT_PATH" remove <リポジトリ名> <issue番号>
```

コンテナ停止 + worktreeディレクトリ削除。

### list: worktree一覧

```bash
bash "$SCRIPT_PATH" list <リポジトリ名>
```

---

## 実行方法

入力を解析し、適切なサブコマンドを実行する。

### Step 1: スクリプトパスの解決

```bash
COMPANY_HOME="${COMPANY_HOME:-$HOME/itns-workspace/cc-company}"
SCRIPT_PATH="$COMPANY_HOME/.company/scripts/worktree.sh"
```

スクリプトが存在しない場合はエラーを報告する。

### Step 2: 入力の解析

入力パターン:
- `create <repo> <branch>` → そのまま実行
- `create <repo> <issue番号>` → ブランチ名を命名規約から生成して実行
- `up/down/remove <repo> <issue番号>` → そのまま実行
- `list <repo>` → そのまま実行
- `<repo> <issue番号>` → create として扱う（省略形）

**Issue番号からブランチ名を生成するルール:**
1. リポジトリ名が `tasks-division-cross-dev` の場合: `tasks-division-cross-dev-{Issue番号}`
2. それ以外: `feature/{Issue番号}`

### Step 3: 実行

```bash
bash "$SCRIPT_PATH" <サブコマンド> <リポジトリ名> <ブランチ名 or issue番号>
```

実行結果をユーザーに報告する。

---

## 注意事項

- Docker操作（up/down）は本体コンテナに影響する。確認なしで実行してよい
- create 時の vendor/node_modules コピーには時間がかかる場合がある
- 同時にブラウザ確認できるのは1環境のみ（ポート共有のため）
- worktree内のDBは本体と共有（volume override）
