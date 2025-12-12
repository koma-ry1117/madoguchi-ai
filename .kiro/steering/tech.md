# Technology Stack

モダンなフルスタック構成。Next.js 15 App Routerによるサーバーレスアーキテクチャ。

## Architecture

```
フロントエンド (Next.js 15 + React 19)
         ↕
   認証・認可層 (Supabase Auth + RLS)
         ↕
AI・音声処理層 (Google Gemini + OpenAI Whisper + Google Cloud TTS)
         ↕
  データベース層 (Supabase PostgreSQL)
         ↕
   メール配信層 (Resend + React Email)
```

## Core Technologies

| カテゴリ | 技術 | バージョン |
|---------|------|-----------|
| Language | TypeScript | 5.x（strict mode） |
| Framework | Next.js | 15（App Router） |
| Runtime | Node.js | 22+ |
| UI | React | 19 |
| Styling | Tailwind CSS | 4.x |
| UI Components | shadcn/ui | - |
| Animation | Framer Motion | - |

## Key Libraries

### AI・音声

| ライブラリ | 用途 |
|-----------|------|
| `ai` (Vercel AI SDK) | AI統合、ストリーミング、Function Calling |
| `@ai-sdk/google` | Gemini 2.5連携 |
| `openai` | Whisper（音声認識） |
| `@google-cloud/text-to-speech` | Google Cloud TTS（音声合成、WaveNet） |

### データ・認証

| ライブラリ | 用途 |
|-----------|------|
| `@supabase/supabase-js` | DB、認証、Storage、Realtime |
| `@supabase/ssr` | Next.js 15対応SSRヘルパー（auth-helpers-nextjsは非推奨） |

### 状態管理・データフェッチ

| ライブラリ | 用途 |
|-----------|------|
| `zustand` | Client State（UI状態） |
| `@tanstack/react-query` | Server State（データ同期） |

### バリデーション・フォーム

| ライブラリ | 用途 |
|-----------|------|
| `zod` | スキーマバリデーション、型推論 |
| `react-hook-form` | フォーム管理 |

### メール

| ライブラリ | 用途 |
|-----------|------|
| `resend` | メール配信API |
| `@react-email/components` | HTMLメールテンプレート |
| `ical-generator` | カレンダー招待(.ics)生成 |

## State Management Philosophy

### Server State vs Client State

| 種別 | ツール | 用途 |
|------|--------|------|
| Server State | TanStack Query | Supabaseデータ、API応答（非同期、共有、stale可能） |
| Client State | Zustand | UI状態（モーダル開閉、フォーム入力中、テーマ） |

**原則**:
- Supabaseから取得したデータをZustandにコピーしない
- Server Stateの鮮度管理はTanStack Queryに任せる
- Zustandは純粋なクライアント状態のみ

### TanStack Query + Next.js 15

Server Componentでプリフェッチ、Client Componentでハイドレーション：

```typescript
// Server Component
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';

export default async function Page() {
  const queryClient = new QueryClient();
  await queryClient.prefetchQuery({
    queryKey: ['properties'],
    queryFn: fetchProperties,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PropertyList />
    </HydrationBoundary>
  );
}
```

### Zustand ハイドレーション対策

`persist` ミドルウェア使用時のハイドレーションエラー対策：

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

const useThemeStore = create(
  persist(
    (set) => ({
      theme: 'light',
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'theme-storage',
      skipHydration: true, // SSR時のハイドレーションをスキップ
    }
  )
);

// Client Componentでマウント後に復元
useEffect(() => {
  useThemeStore.persist.rehydrate();
}, []);
```

## Runtime Selection

### Node.js vs Edge

| ルートタイプ | Runtime | 理由 |
|-------------|---------|------|
| AI API (`/api/chat`) | Node.js | Google AI SDKの互換性 |
| メール送信 | Node.js | React Email レンダリング |
| 認証Middleware | Edge可 | 軽量処理 |
| 静的ページ | Edge可 | 低レイテンシ |

```typescript
// Node.js Runtimeを明示
export const runtime = 'nodejs';
export const maxDuration = 60;
```

## Type Safety Pipeline

### 1. DBスキーマ → TypeScript型

```bash
supabase gen types typescript --local > src/types/database.types.ts
```

### 2. TypeScript型 → Zodスキーマ

```typescript
// 自動生成または手動同期
import { z } from 'zod';

export const propertySchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  rent: z.number().positive(),
  // ...
});

export type Property = z.infer<typeof propertySchema>;
```

### 3. Zodスキーマ → ランタイムバリデーション

```typescript
// Server Action
const parsed = propertySchema.safeParse(input);
if (!parsed.success) {
  return { error: parsed.error.flatten() };
}
```

## Development Standards

### Type Safety

- TypeScript strict mode必須
- `any`禁止、`unknown`で明示的に型ガード
- Zodでランタイムバリデーション
- DB型は自動生成、手動編集禁止

### Code Quality

- ESLint + Prettier
- Husky + lint-staged（pre-commit）

### Testing

| ツール | 用途 |
|--------|------|
| Vitest | ユニットテスト |
| Playwright | E2Eテスト |
| MSW | APIモック |

## Key Technical Decisions

| 決定事項 | 理由 |
|---------|------|
| Next.js 15 App Router | RSC・ストリーミング対応、部分レンダリング（PPR）準備 |
| Supabase | PostgreSQL + Auth + Storage + Realtime統合、RLSによるセキュリティ |
| @supabase/ssr | Next.js 15非同期API対応、auth-helpersの後継 |
| Google Gemini 2.5 | コスト効率の良いAI会話（Flash）、高精度タスク（Pro） |
| Google Cloud TTS | 高品質な日本語音声合成（WaveNet） |
| Vercel AI SDK | Next.jsとの統合、ストリーミング、Function Calling |
| TanStack Query | Server Stateのキャッシュ・同期・リフレッシュ管理 |
| Zustand | Client State専用、軽量、TypeScript完全対応 |
| Resend | Next.js/React Email統合、Webhook対応 |

---
_Document standards and patterns, not every dependency_
