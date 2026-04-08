---
description: "設計→レビュー→実装→レビュー→テスト→レビューの3フェーズ自動開発"
argument-hint: "<Issue番号 or タスク説明>"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Write", "Edit", "Agent", "Task", "WebSearch", "WebFetch"]
---

# autodev: 3フェーズ自動開発

Issue番号またはタスク説明を受け取り、設計→レビュー→実装→レビュー→テスト→レビューを自動ループして品質を担保する。

**入力:** "$ARGUMENTS"

---

## ワークフロー概要

```
Phase 1: 設計  →  spec-review  →  修正（最大2回）
Phase 2: 実装  →  code-review  →  修正（最大2回）
Phase 3: テスト →  test-review  →  修正（最大2回）
→ 最終報告
```

---

## Step 0: コンテキスト収集

### 0-1. Issue/チケット情報の取得

入力がIssue番号（`#123` や `123`）の場合:
1. `.company/pm/tickets/` から該当チケットを検索
2. GitHub Issue があれば `gh issue view <番号>` で取得
3. 両方の情報をマージして要件を整理する

入力がタスク説明（自然言語）の場合:
1. そのまま要件として使用する

### 0-2. リポジトリコンテキストの取得

1. `CLAUDE.md`（リポジトリルート）を読み込む
2. `.company/CLAUDE.md` の運営ルールを確認
3. 既存のコード構造を把握（関連ディレクトリの `ls`、主要ファイルの確認）
4. ブランチ情報: `git branch --show-current`

### 0-3. Guardian / Review コンテキストの取得

以下のファイルが存在する場合は読み込み、各フェーズのエージェントに渡す:

- `.company/guardian/CLAUDE.md` — 品質チェックの6つの柱
- `.company/guardian/approach-patterns.md` — 12の具体的アプローチパターン
- `.company/review/review-principles.md` — 16のレビュー原則

### 0-4. 設計ドキュメントの出力先を決定

- `.company/` が存在する場合: `.company/secretary/notes/YYYY-MM-DD-design-<slug>.md`
- PM部署がある場合: チケットファイルにも設計概要を追記

---

## Phase 1: 設計

### 1-1. designer エージェント起動

**designer エージェント**に以下を渡して設計ドキュメントを生成させる:

- 要件（Step 0 で収集した Issue/チケット内容）
- リポジトリの CLAUDE.md
- 既存コード構造（関連ファイル一覧）
- Guardian approach-patterns.md（存在する場合）

### 1-2. 設計ドキュメントの保存

生成された設計ドキュメントを `.company/secretary/notes/YYYY-MM-DD-design-<slug>.md` に保存する。

### 1-3. 設計レビュー（ループ最大2回）

**spec-reviewer エージェント**に以下を渡してレビューさせる:

- 設計ドキュメント
- 元の Issue/チケット要件
- Guardian CLAUDE.md（存在する場合）

**ループ条件:**
- **PASS**: Phase 2 へ進む
- **WARN/FAIL**: designer に指摘事項を渡して設計修正 → 設計ドキュメントを更新 → 再レビュー
- **2回目も FAIL**: 指摘事項をまとめてユーザーに判断を仰ぐ

---

## Phase 2: 実装

### 2-1. implementer エージェント起動（phase: code）

**implementer エージェント**に以下を渡してプロダクションコードを実装させる:

- レビュー済み設計ドキュメント（実装計画セクション）
- リポジトリの CLAUDE.md
- 既存コードの関連部分
- **フェーズ指定: `code`**（テストコードは書かない）
- Guardian approach-patterns.md（存在する場合）

### 2-2. 実装レビュー（ループ最大2回）

**code-reviewer エージェント**に以下を渡してレビューさせる:

- `git diff`（実装による変更分）
- 設計ドキュメント（仕様との照合用）
- review-principles.md（存在する場合）

**ループ条件:**
- **PASS**: Phase 3 へ進む
- **Critical あり**: implementer（phase: code）に指摘事項を渡して修正 → 再レビュー
- **2回目も Critical**: 指摘事項をまとめてユーザーに判断を仰ぐ
- **Important のみ**: 一覧をまとめて Phase 3 へ進む（最終報告に含める）

---

## Phase 3: テスト

### 3-1. implementer エージェント起動（phase: test）

**implementer エージェント**に以下を渡してテストコードを実装させる:

- レビュー済み設計ドキュメント（テスト計画セクション）
- Phase 2 で実装したコード（`git diff` で確認）
- リポジトリの CLAUDE.md
- **フェーズ指定: `test`**（プロダクションコードは変更しない）

### 3-2. テスト実行

implementer がテストを実行し、全テストがパスすることを確認する。

### 3-3. テストレビュー（ループ最大2回）

**test-reviewer エージェント**に以下を渡してレビューさせる:

- テストコードの `git diff`
- 設計ドキュメント（テスト計画との照合用）
- 実装コード（カバレッジ確認用）
- Guardian CLAUDE.md（存在する場合）

**ループ条件:**
- **PASS**: 最終報告へ進む
- **Critical あり**: implementer（phase: test）に指摘事項を渡して修正 → テスト再実行 → 再レビュー
- **2回目も Critical**: 指摘事項をまとめてユーザーに判断を仰ぐ
- **Important のみ**: 一覧をまとめて最終報告に含める

---

## 最終報告

全フェーズ完了後、以下の形式でユーザーに報告する:

```markdown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  autodev 完了レポート
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 対象
- Issue/タスク: [内容]
- ブランチ: [ブランチ名]

## Phase 1: 設計
- ドキュメント: [ファイルパス]
- 設計レビュー: [PASS（N回目）/ ユーザー判断]

## Phase 2: 実装
- 変更ファイル: X件
- 新規ファイル: Y件
- コードレビュー: [PASS（N回目）/ ユーザー判断]

## Phase 3: テスト
- テストファイル: Z件
- テスト結果: [全件パス / N件失敗]
- テストレビュー: [PASS（N回目）/ ユーザー判断]

## 残課題（Important）
1. [Phase][指摘内容]（対応推奨）

## 次のアクション
- [ ] 変更内容を確認（git diff）
- [ ] コミット
- [ ] PR作成

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 重要な原則

- **コミットしない**: 全フェーズ終了後もコミットはユーザー判断
- **ループ上限**: 各レビューフェーズは最大2回まで。超えたらユーザーに相談
- **中断可能**: ユーザーがいつでも介入・方向転換できる
- **設計を残す**: 設計ドキュメントは必ず `.company/` に保存（ナレッジ蓄積）
- **フェーズ分離**: 実装（code）とテスト（test）は別フェーズ。混ぜない
- **グレースフル**: `.company/` や guardian ファイルがなくても動作する
