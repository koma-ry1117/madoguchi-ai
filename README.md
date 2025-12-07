# madoguchi-ai

不動産店舗向けAI接客キオスクシステム

## 概要

**madoguchi-ai** は、不動産店舗に設置するAI接客キオスクシステムです。AIアバターが顧客対応を行い、24時間365日の物件検索・内見予約を実現します。高齢者にも使いやすい音声インターフェースを提供します。

## 主な機能

| 機能 | 説明 |
|------|------|
| **AI会話システム** | GPT-4oによる自然な会話、物件検索・提案、内見予約、感情表現アバター |
| **音声入出力** | Whisper（音声認識）+ ElevenLabs（音声合成）、高齢者対応 |
| **Chrome拡張機能** | アットホーム等からワンクリック物件取り込み、GPT-4o構造化抽出 |
| **物件管理** | オペレーター向け物件CRUD、セマンティック検索（pgvector） |
| **内見予約管理** | キオスク連携予約、カレンダー招待(.ics)、リマインダー |
| **営業メール配信** | 新着物件通知、フォローアップ、Resend連携、配信停止管理 |
| **オペレーター管理** | マルチテナントSaaS、サブスクリプション管理、システム統計 |

## 技術スタック

### フロントエンド
- Next.js 15（App Router）
- React 19 + TypeScript 5.x（strict mode）
- Tailwind CSS + shadcn/ui
- Framer Motion（アバターアニメーション）
- Zustand（状態管理）
- TanStack Query（サーバーステート）

### バックエンド・データベース
- Supabase（PostgreSQL + Auth + Storage + Realtime）
- pgvector（セマンティック検索）
- Row Level Security（RLS）によるマルチテナント分離

### AI・音声
- OpenAI GPT-4o（会話・構造化抽出）
- OpenAI Whisper（音声認識）
- ElevenLabs（音声合成）
- Vercel AI SDK

### メール・インフラ
- Resend + React Email
- Vercel（ホスティング・Cron）
- ical-generator（カレンダー招待）

## ユーザーロール

| ロール | 権限 |
|--------|------|
| **admin** | オペレーター管理、全データ閲覧、システム設定 |
| **operator** | 自社物件取り込み・編集、自社顧客管理、営業メール配信 |
| **kiosk_user** | AI会話、物件検索（会話内）、内見予約（会話内）※認証不要 |

## 画面構成（26ページ）

| カテゴリ | 画面数 | 内容 |
|----------|--------|------|
| 共通 | 3 | LP、ログイン、エラー |
| キオスク（顧客向け） | 1 | AI接客画面（5フェーズ遷移） |
| オペレーター管理 | 12 | ダッシュボード、物件・顧客・予約・会話・マーケティング管理 |
| システム管理者 | 10 | オペレーター・サブスクリプション・統計・監査ログ管理 |

## プロジェクト構成

```
madoguchi-ai/
├── .kiro/
│   ├── steering/           # プロジェクト全体のコンテキスト
│   │   ├── product.md      # プロダクト概要
│   │   ├── tech.md         # 技術スタック
│   │   ├── structure.md    # プロジェクト構造
│   │   ├── database.md     # DB設計
│   │   ├── api-standards.md # API標準
│   │   └── security.md     # セキュリティ
│   │
│   └── specs/              # 各機能の詳細仕様
│       ├── ai-chat/        # AI会話システム
│       ├── chrome-extension/ # Chrome拡張機能
│       ├── marketing-email/ # 営業メール配信
│       ├── operator-management/ # オペレーター管理
│       ├── property-management/ # 物件管理
│       ├── viewing-reservation/ # 内見予約管理
│       └── voice-io/       # 音声入出力
│
├── docs/
│   └── 御見積書_成功報酬型.md
│
├── app/                    # Next.js App Router
├── src/
│   ├── components/         # UIコンポーネント
│   ├── lib/               # ユーティリティ・サービス
│   ├── hooks/             # カスタムフック
│   ├── stores/            # Zustandストア
│   └── schemas/           # Zodスキーマ
│
└── chrome-extension/       # Chrome拡張機能
```

## 開発環境

### 必須要件
- Node.js 20+
- pnpm 8+
- Docker（ローカルSupabase）

### セットアップ

```bash
# クローン
git clone https://github.com/your-repo/madoguchi-ai.git
cd madoguchi-ai

# 依存関係インストール
pnpm install

# 環境変数設定
cp .env.example .env.local

# ローカルSupabase起動
pnpm supabase:start

# 開発サーバー起動
pnpm dev
```

### コマンド

```bash
pnpm dev          # 開発サーバー
pnpm build        # ビルド
pnpm test         # テスト実行
pnpm lint         # Lint
pnpm type-check   # 型チェック
```

## 仕様駆動開発（Spec-Driven Development）

このプロジェクトはKiro-style仕様駆動開発を採用しています。

### ワークフロー

```
Phase 1: 仕様策定
  /kiro:spec-init → /kiro:spec-requirements → /kiro:spec-design → /kiro:spec-tasks

Phase 2: 実装
  /kiro:spec-impl {feature}

進捗確認
  /kiro:spec-status {feature}
```

### 仕様の場所

- **Steering**: `.kiro/steering/` - プロジェクト全体のルールとコンテキスト
- **Specs**: `.kiro/specs/` - 各機能の要件・設計・タスク

## ライセンス

プライベートプロジェクト

## 開発者

- **クライアント**: giyam様
- **開発者**: koma
