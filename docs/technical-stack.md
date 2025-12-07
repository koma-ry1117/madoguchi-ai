# 技術スタック詳細

## 全体構成

このプロジェクトは**モダンなフルスタック構成**で、以下の技術を採用しています。

```
フロントエンド (Next.js + React)
         ↕
   認証・認可層 (Supabase Auth)
         ↕
AI・音声処理層 (OpenAI + ElevenLabs)
         ↕
  データベース層 (Supabase PostgreSQL)
         ↕
Chrome拡張機能 (物件取り込み)
```

---

## 技術スタック概要

| カテゴリ | 技術 |
|---------|------|
| **フロントエンド** | Next.js 15, React 19, TypeScript, Tailwind CSS, shadcn/ui |
| **状態管理** | Zustand, TanStack Query |
| **バックエンド** | Supabase (PostgreSQL + Auth + Storage) |
| **ホスティング** | Vercel |
| **AI/LLM** | OpenAI GPT-4o, Vercel AI SDK |
| **音声処理** | OpenAI Whisper, ElevenLabs |
| **ベクトル検索** | pgvector, OpenAI Embeddings |
| **メール送信** | Resend, React Email |
| **Chrome拡張** | Manifest V3, TypeScript |
| **テスト** | Vitest, Playwright, Testing Library |
| **モニタリング** | Sentry, Vercel Analytics |

---

## ドキュメント一覧

本ドキュメントは以下のサブドキュメントに分割されています：

| ドキュメント | 説明 |
|-------------|------|
| [フロントエンド](./technical-stack/technical-stack-frontend.md) | Next.js, React, UI/UXライブラリ, 状態管理 |
| [バックエンド・インフラ](./technical-stack/technical-stack-backend.md) | Supabase, Vercel, 認証, メール送信 |
| [AI・LLM・音声処理](./technical-stack/technical-stack-ai.md) | OpenAI, Vercel AI SDK, 音声処理, Chrome拡張 |
| [開発ツール・テスト・セキュリティ](./technical-stack/technical-stack-dev.md) | テスト, モニタリング, CI/CD, セキュリティ |

---

次のドキュメント: [アーキテクチャ設計](./architecture.md)
