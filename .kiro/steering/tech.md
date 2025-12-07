# Technology Stack

## Architecture

モダンなフルスタック構成。Next.js App Routerによるサーバーレスアーキテクチャ。

```
フロントエンド (Next.js + React)
         ↕
   認証・認可層 (Supabase Auth + RLS)
         ↕
AI・音声処理層 (OpenAI + ElevenLabs)
         ↕
  データベース層 (Supabase PostgreSQL)
         ↕
   メール配信層 (Resend + React Email)
```

## Core Technologies

- **Language**: TypeScript 5.x（strict mode）
- **Framework**: Next.js 15（App Router）
- **Runtime**: Node.js 20+
- **UI**: React 19 + Tailwind CSS + shadcn/ui
- **Animation**: Framer Motion（アバター表情切り替え）

## Key Libraries

### AI・音声
- `ai` (Vercel AI SDK) - OpenAI統合、ストリーミング、Function Calling
- `openai` - GPT-4o（会話・構造化抽出）、Whisper（音声認識）
- `elevenlabs` - 音声合成

### データ・認証
- `@supabase/supabase-js` - DB、認証、Storage、Realtime
- `@supabase/auth-helpers-nextjs` - Next.js用認証ヘルパー

### 状態管理・データフェッチ
- `zustand` - グローバル状態（会話履歴、アバター感情）
- `@tanstack/react-query` - サーバーステート管理

### バリデーション・フォーム
- `zod` - スキーマバリデーション
- `react-hook-form` - フォーム管理

### メール
- `resend` - メール配信API
- `@react-email/components` - HTMLメールテンプレート
- `ical-generator` - カレンダー招待(.ics)生成

## Development Standards

### Type Safety
- TypeScript strict mode必須
- `any`禁止、`unknown`で明示的に型ガード
- Zodでランタイムバリデーション

### Code Quality
- ESLint + Prettier
- Husky + lint-staged（pre-commit）

### Testing
- Vitest（ユニットテスト）
- Playwright（E2Eテスト）
- MSW（APIモック）

## Development Environment

### Required Tools
- Node.js 20+
- pnpm 8+
- Docker（ローカルSupabase）

### Common Commands
```bash
# Dev: pnpm dev
# Build: pnpm build
# Test: pnpm test
# Lint: pnpm lint
# Type check: pnpm type-check
# Supabase: pnpm supabase:start
```

## Key Technical Decisions

| 決定事項 | 理由 |
|---------|------|
| Next.js App Router | RSC・ストリーミング対応、Vercelとの統合 |
| Supabase | PostgreSQL + Auth + Storage + Realtime統合、RLSによるセキュリティ |
| OpenAI統一 | GPT-4o + Whisperでコスト管理が容易 |
| Vercel AI SDK | Next.jsとの完璧な統合、ストリーミング対応 |
| Resend | Next.jsとの相性、React Email対応 |
| Zustand | Redux より軽量、TypeScript完全対応 |

---
_Document standards and patterns, not every dependency_
