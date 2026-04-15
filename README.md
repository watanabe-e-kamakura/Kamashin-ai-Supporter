# cc-company

> Claude Code で仮想組織を構築・運営するプラグイン（v2.5.0）

`/company` を実行すると、秘書があなた専用の窓口になります。3ステップで即運用開始。部署は使い方に合わせて自然に増えていきます。

## インストール

```
/install watanabe-e-kamakura/Kamashin-ai-Supporter
```

## コンセプト

```
あなた → 秘書（窓口） → 各部署（必要に応じて追加）
```

- **秘書**: 常に窓口。TODO管理、壁打ち、メモ、何でも相談OK
- **部署**: 仕事が増えたら秘書が提案。リサーチ、PM、開発など

最初は秘書だけ。シンプルに始めて、必要になったら部署を追加。

## 使い方

### 初回セットアップ（3ステップ）

```
> /company

秘書: はじめまして！あなたの秘書になります。
      まず、事業や活動を教えてください。
あなた: フリーランスのWeb開発やってます

秘書: 今の目標や困りごとは？
あなた: SaaSを作って月10万目指してる。タスクが散らかるのが悩み

→ .company/secretary/ が自動生成される（完了！）
```

### 日常の運営

```
> /company
秘書: おはようございます！何かありますか？

> 今日やること教えて
秘書: 今日のTODOです:
  - [ ] クライアントAへ見積もり送付
  - [ ] LP設計書のレビュー

> 競合サービスについて調べて
秘書: 承知しました。調べますね。
  → secretary/notes/2026-03-16-competitor-research.md に保存

> 海外のトレンドも知りたい
秘書: リサーチの依頼が増えていますね。
      リサーチ部門を作りましょうか？
あなた: 作って
  → research/ が自動生成される
```

## スキル・コマンド

| コマンド | 説明 |
|---------|------|
| `/company:company` | 仮想カンパニー（秘書・TODO・壁打ち・相談） |
| `/company:autodev` | 設計→レビュー→実装→テストの自動開発フロー |
| `/company:review` | 第三者目線コードレビュー（仕様・品質・テスト） |
| `/company:worktree` | Worktree の作成・起動・停止・削除 |

## エージェント

タスクに応じて自動で専門エージェントが起動します。

| エージェント | 役割 |
|-------------|------|
| `designer` | 要件から技術設計・実装計画・テスト計画を生成 |
| `implementer` | 設計ドキュメントに基づくコード・テスト実装 |
| `code-reviewer` | コード品質・バグ検出・セキュリティレビュー |
| `test-reviewer` | テスト網羅性・品質レビュー |
| `spec-reviewer` | PMチケットと実装の仕様整合性チェック |
| `impact-analyzer` | クロスリポジトリ影響分析 |

## 部署（必要に応じて追加）

| 部署 | 担当領域 |
|------|---------|
| 秘書室 | TODO管理、壁打ち、メモ、Slack管理、勤怠（常設） |
| PM | プロジェクト進捗、GitHub Issue連携、チケット管理 |
| ガーディアン | 設計・実装・レビューの品質チェック |
| リサーチ | 市場調査、競合分析、技術調査 |
| マーケティング | コンテンツ企画、SNS、キャンペーン |
| 開発 | 技術ドキュメント、設計、デバッグ |
| 経理 | 請求書、経費、売上管理 |
| 営業 | クライアント管理、提案書 |
| クリエイティブ | デザインブリーフ、ブランド管理 |
| 人事 | 採用管理、チーム管理 |

## 初期構成

```
.company/
├── CLAUDE.md              ← 組織ルール
└── secretary/
    ├── CLAUDE.md           ← 秘書の振る舞い
    ├── inbox/
    ├── todos/
    │   └── YYYY-MM-DD.md   ← 今日のTODO
    └── notes/
```

## v1 からのアップグレード

既存の v1 組織（CEO部門あり）がある場合、`/company` 実行時に自動でアップグレードを提案します。

- CEO部門 → 廃止（秘書が直接振り分け）
- レビュー部門 → 廃止（秘書が管理）
- 使用中の部署 → そのまま引き継ぎ
- 空の部署 → 削除

## ファイル構成

```
cc-company/
├── plugins/
│   └── company/
│       ├── .claude-plugin/
│       │   └── plugin.json        ← プラグイン定義（v2.5.0）
│       ├── agents/                 ← 専門エージェント
│       │   ├── code-reviewer.md
│       │   ├── designer.md
│       │   ├── impact-analyzer.md
│       │   ├── implementer.md
│       │   ├── spec-reviewer.md
│       │   └── test-reviewer.md
│       ├── commands/               ← コマンド定義
│       │   ├── autodev.md
│       │   ├── review.md
│       │   └── worktree.md
│       └── skills/
│           └── company/
│               ├── SKILL.md        ← /company スキル定義
│               └── references/
│                   ├── departments.md
│                   └── claude-md-template.md
├── docs/                           ← ドキュメントサイト（VitePress）
├── README.md
└── LICENSE
```

## ドキュメント

詳細なガイドは [ドキュメントサイト](https://watanabe-e-kamakura.github.io/Kamashin-ai-Supporter/) を参照してください。

```bash
# ローカルで閲覧する場合
npm run docs:dev
```

## ライセンス

MIT
