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
Phase 1: 設計  →  spec-review  →  修正（最大2回） →  Guardian独立チェック
Phase 2: 実装  →  code-review  →  修正（最大2回） →  Guardian独立チェック
Phase 3: テスト →  test-review  →  修正（最大2回） →  Guardian独立チェック
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

### 0-1.5. 要件の十分性チェック

収集した情報から、設計に着手できるだけの前提条件が揃っているか判断する。

**必須情報（1つでも不明なら対話でヒアリング）:**
- **何を作る/変えるか**: 対象の機能・画面・APIが特定できるか
- **なぜやるか**: 目的・背景が分かるか（ないと設計判断ができない）
- **対象リポジトリ**: どのリポジトリで作業するか
- **影響範囲**: 既存機能への影響が想像できるか

**ヒアリングの進め方:**
1. 不足している情報を洗い出す
2. `AskUserQuestion` で具体的に質問する（1回で複数質問OK）
3. 回答を得たら要件を再整理する
4. まだ不足があれば再度質問する（最大2往復）
5. 2往復しても不明点が残る場合は、判明している範囲で設計に進み、設計ドキュメントに「要確認事項」として明記する

**例:**
> Issueの内容が「〇〇を修正する」のみの場合:
> - 「現在どういう動作で、どう変わるべきかを教えてください」
> - 「対象の画面・APIはどこですか？」
> - 「関連するリポジトリはどれですか？」

### 0-2. リポジトリコンテキストの取得

1. `CLAUDE.md`（リポジトリルート）を読み込む
2. `.company/CLAUDE.md` の運営ルールを確認
3. 既存のコード構造を把握（関連ディレクトリの `ls`、主要ファイルの確認）
4. ブランチ情報: `git branch --show-current`

### 0-3. Guardian / Review コンテキストの取得

以下のファイルが存在する場合は読み込む:

- `.company/guardian/CLAUDE.md` — 品質チェックの6つの柱（Guardianチェックで使用）
- `.company/guardian/approach-patterns.md` — 12の具体的アプローチパターン（designer/implementerに渡す）
- `.company/review/review-principles.md` — 16のレビュー原則（code-reviewerに渡す）

**Guardianファイルの用途:**
- `approach-patterns.md`, `review-principles.md` → 各エージェントへのコンテキストとして渡す
- `guardian/CLAUDE.md` → 各フェーズ後のGuardian独立チェックで使用（レビューエージェントには渡さない）

Guardian独立チェックはレビューエージェントとは別の視点を保つため、`guardian/CLAUDE.md` の6つの柱はGuardianエージェント専用とする。

### 0-4. Worktree セットアップ

`/company:worktree` コマンドの `create` を使ってworktreeを準備する。

**入力がIssue番号の場合:**
1. **worktree確認**: `{WORKSPACE}/{REPO}-wt-{Issue番号}` が存在するか確認
2. **なければ作成**: `.company/scripts/worktree.sh create <リポジトリ名> <ブランチ名>` を実行
   - ブランチ名は `.company/CLAUDE.md` のブランチ命名規約に従う
     - tasks-division-cross-dev → `tasks-division-cross-dev-{Issue番号}`
     - 事業部リポジトリ → `feature/{Issue番号}`
   - vendor/node_modules/生成ファイルのコピー、Docker volume共有が自動処理される
3. **以降のPhase 2（実装）・Phase 3（テスト）はworktreeディレクトリで実行する**
4. Phase 1（設計）はIssue追記なのでworktree不要

**入力がタスク説明（Issue番号なし）の場合:**
- ユーザーに対象リポジトリとブランチを確認してからworktreeを作成する

**重要**: implementer エージェントには必ずworktreeのパスを渡すこと。メインリポジトリに直接書き込まない。

### 0-5. 成果物の出力先

autodevの成果物（設計・レビュー結果・Guardianフィードバック）は**GitHub Issueの詳細に追記**する。

- **入力がIssue番号の場合**: `gh issue edit <番号> --body "..."` で Issue 本文に追記
- **入力がタスク説明の場合**: `.company/secretary/notes/YYYY-MM-DD-design-<slug>.md` にフォールバック

**Issue追記のフォーマット:**
```markdown
（既存のIssue本文）

---

## autodev: 設計
（Phase 1 完了後に追記）

## autodev: 設計レビュー
（spec-review + Guardian チェック結果を追記）

## autodev: 実装レビュー
（code-review + Guardian チェック結果を追記）

## autodev: テストレビュー
（test-review + Guardian チェック結果を追記）
```

**注意**: 既存のIssue本文は絶対に消さない。`---` 区切りの下に追記する。

---

## Phase 1: 設計

### 1-1. designer エージェント起動

**designer エージェント**に以下を渡して設計ドキュメントを生成させる:

- 要件（Step 0 で収集した Issue/チケット内容）
- リポジトリの CLAUDE.md
- 既存コード構造（関連ファイル一覧）
- Guardian approach-patterns.md（存在する場合）

### 1-2. 設計ドキュメントの保存

- **Issue番号あり**: GitHub Issueの詳細に `## autodev: 設計` セクションとして追記する
- **Issue番号なし**: `.company/secretary/notes/YYYY-MM-DD-design-<slug>.md` に保存する

### 1-3. 設計レビュー（ループ最大2回）

**spec-reviewer エージェント**に以下を渡してレビューさせる:

- 設計ドキュメント
- 元の Issue/チケット要件

**ループ条件:**
- **PASS**: 1-4 Guardian チェックへ進む
- **WARN/FAIL**: designer に指摘事項を渡して設計修正 → 設計ドキュメントを更新 → 再レビュー
- **2回目も FAIL**: 指摘事項をまとめてユーザーに判断を仰ぐ

### 1-4. Guardian 独立チェック

spec-review 通過後、**Guardian エージェント**が独立した視点でチェックする。

渡すもの:
- 設計ドキュメント
- `guardian/CLAUDE.md`（6つの柱）

**Guardianが見る観点（spec-reviewerとの違い）:**
- **横断的視野**: 他事業部・共通リポジトリへの影響は考慮されているか？
- **再利用性**: この設計は次に似た案件が来たとき再利用できるか？
- **防御的思考**: 失敗時のリカバリが設計に含まれているか？
- **構造思考**: 責務の配置は適切か？

**判定:**
- **✅ 通過**: Phase 2 へ進む
- **🔴 MUST あり**: designer に指摘を渡して修正 → 修正後、Guardian 再チェック（1回のみ）
- **🟡 SHOULD のみ**: 一覧をまとめて Phase 2 へ進む（最終報告に含める）

---

## Phase 2: 実装

### 2-1. implementer エージェント起動（phase: code）

**implementer エージェント**に以下を渡してプロダクションコードを実装させる:

- **作業ディレクトリ: worktreeパス**（Step 0-4 で作成）
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
- **PASS**: 2-3 Guardian チェックへ進む
- **Critical あり**: implementer（phase: code）に指摘事項を渡して修正 → 再レビュー
- **2回目も Critical**: 指摘事項をまとめてユーザーに判断を仰ぐ
- **Important のみ**: 一覧をまとめて 2-3 Guardian チェックへ進む（最終報告に含める）

### 2-3. Guardian 独立チェック

code-review 通過後、**Guardian エージェント**が独立した視点でチェックする。

渡すもの:
- `git diff`（実装による変更分）
- 設計ドキュメント
- `guardian/CLAUDE.md`（6つの柱）

**Guardianが見る観点（code-reviewerとの違い）:**
- **横断的視野**: 共通リポジトリ（itns-admin/itns-library）への影響は？他事業部への波及は？
- **再利用性**: Service/Trait/Enumは再利用前提で書かれているか？コピペはないか？
- **防御的思考**: null上書き、トランザクション漏れ、Job失敗時のリカバリは？
- **構造思考**: Actionにロジックが肥大化していないか？責務の配置は適切か？

**判定:**
- **✅ 通過**: Phase 3 へ進む
- **🔴 MUST あり**: implementer（phase: code）に指摘を渡して修正 → 修正後、Guardian 再チェック（1回のみ）
- **🟡 SHOULD のみ**: 一覧をまとめて Phase 3 へ進む（最終報告に含める）

---

## Phase 3: テスト

### 3-1. implementer エージェント起動（phase: test）

**implementer エージェント**に以下を渡してテストコードを実装させる:

- **作業ディレクトリ: worktreeパス**（Step 0-4 で作成、Phase 2 と同じ）
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

**ループ条件:**
- **PASS**: 3-4 Guardian チェックへ進む
- **Critical あり**: implementer（phase: test）に指摘事項を渡して修正 → テスト再実行 → 再レビュー
- **2回目も Critical**: 指摘事項をまとめてユーザーに判断を仰ぐ
- **Important のみ**: 一覧をまとめて 3-4 Guardian チェックへ進む（最終報告に含める）

### 3-4. Guardian 独立チェック

test-review 通過後、**Guardian エージェント**が独立した視点でチェックする。

渡すもの:
- テストコードの `git diff`
- 実装コード
- 設計ドキュメント（テスト計画）
- `guardian/CLAUDE.md`（6つの柱）

**Guardianが見る観点（test-reviewerとの違い）:**
- **防御的思考**: 異常系・境界値・失敗パスのテストは十分か？
- **横断的視野**: 共通側の変更がある場合、他事業部のテストに影響しないか？
- **構造思考**: テストの構造は保守しやすいか？テストヘルパーの再利用は適切か？
- **実利主義**: 過剰なテストはないか？重要なパスに集中しているか？

**判定:**
- **✅ 通過**: 最終報告へ進む
- **🔴 MUST あり**: implementer（phase: test）に指摘を渡して修正 → テスト再実行 → Guardian 再チェック（1回のみ）
- **🟡 SHOULD のみ**: 一覧をまとめて最終報告に含める

---

## 最終報告

全フェーズ完了後、以下の形式でユーザーに報告する:

```markdown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  autodev 完了レポート
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 対象
- Issue/タスク: [内容]
- Worktree: [パス]
- ブランチ: [ブランチ名]

## Phase 1: 設計
- ドキュメント: [ファイルパス]
- 設計レビュー: [PASS（N回目）/ ユーザー判断]
- Guardian: [✅通過 / 🟡SHOULD N件 / 🔴修正あり]

## Phase 2: 実装
- 変更ファイル: X件
- 新規ファイル: Y件
- コードレビュー: [PASS（N回目）/ ユーザー判断]
- Guardian: [✅通過 / 🟡SHOULD N件 / 🔴修正あり]

## Phase 3: テスト
- テストファイル: Z件
- テスト結果: [全件パス / N件失敗]
- テストレビュー: [PASS（N回目）/ ユーザー判断]
- Guardian: [✅通過 / 🟡SHOULD N件 / 🔴修正あり]

## 残課題
### レビュー指摘（Important）
1. [Phase][指摘内容]（対応推奨）

### Guardian指摘（SHOULD）
1. [Phase][観点][指摘内容]（対応推奨）

## 次のアクション
- [ ] 変更内容を確認（git diff）
- [ ] コミット
- [ ] PR作成

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 重要な原則

- **Worktree必須**: 実装・テストは必ずworktreeで行う。メインリポジトリに直接書き込まない
- **コミットしない**: 全フェーズ終了後もコミットはユーザー判断
- **ループ上限**: 各レビューフェーズは最大2回まで。超えたらユーザーに相談
- **中断可能**: ユーザーがいつでも介入・方向転換できる
- **設計を残す**: 設計・レビュー結果はGitHub Issueに追記する（Issue番号なしの場合は `.company/` にフォールバック）
- **フェーズ分離**: 実装（code）とテスト（test）は別フェーズ。混ぜない
- **グレースフル**: `.company/` や guardian ファイルがなくても動作する
