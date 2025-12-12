# Project Structure

Feature-Sliced Design (FSD) をNext.js App Router向けに調整したアーキテクチャ。

## Organization Philosophy

**Feature-first + Colocation**

- `app/` はルーティングとレイアウトに特化（薄いレイヤー）
- `src/features/` にドメインロジックとUIをコロケーション
- `src/components/` は共有UIコンポーネント

## Directory Structure

```
src/
├── app/                      # ルーティング層（極力ロジックを持たせない）
│   ├── (auth)/               # Route Groups - 認証フロー
│   │   ├── login/
│   │   │   └── page.tsx      # features/auth/LoginFormを呼び出すだけ
│   │   └── signup/
│   ├── (customer)/           # 顧客向け（キオスク）
│   │   └── chat/
│   ├── (operator)/           # オペレーター向け
│   │   ├── dashboard/
│   │   └── properties/
│   ├── (admin)/              # 管理者向け
│   │   └── operators/
│   └── api/                  # APIルート
│       ├── chat/route.ts
│       └── webhooks/
│
├── features/                 # ドメインロジック層（スライス）
│   ├── auth/                 # 認証機能スライス
│   │   ├── components/       # LoginForm, SignupButton
│   │   ├── actions/          # Server Actionsの実体
│   │   ├── api/              # Supabaseクエリ
│   │   ├── hooks/            # useAuth
│   │   └── types/            # 認証関連の型定義
│   ├── chat/                 # AI会話スライス
│   │   ├── components/       # ChatInterface, MessageBubble
│   │   ├── actions/
│   │   ├── hooks/            # useChat
│   │   └── stores/           # chatStore (Zustand)
│   ├── properties/           # 物件管理スライス
│   │   ├── components/       # PropertyCard, PropertyList
│   │   ├── actions/
│   │   ├── api/
│   │   └── schemas/          # propertySchema (Zod)
│   ├── voice/                # 音声I/Oスライス
│   │   ├── components/       # VoiceInput, AudioPlayer
│   │   ├── hooks/            # useVoice, useAudioQueue
│   │   └── utils/            # audioUtils
│   ├── avatar/               # アバタースライス
│   │   ├── components/       # AvatarDisplay
│   │   └── stores/           # avatarStore (感情状態)
│   └── marketing/            # 営業メールスライス
│       ├── components/       # CampaignForm
│       ├── actions/
│       └── templates/        # React Email
│
├── components/               # 共有UI層
│   ├── ui/                   # Shadcn UIプリミティブ
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   └── card.tsx
│   └── shared/               # ドメインに依存しない汎用コンポーネント
│       ├── Header.tsx
│       └── Footer.tsx
│
├── lib/                      # インフラストラクチャ層
│   ├── supabase/
│   │   ├── client.ts         # Browser Client
│   │   ├── server.ts         # Server Client
│   │   └── middleware.ts     # Session管理
│   ├── ai/
│   │   └── gemini.ts         # AI SDK設定
│   └── email/
│       └── resend.ts         # Resendクライアント
│
├── schemas/                  # グローバルZodスキーマ（features横断）
│   └── common.ts
│
└── types/                    # 全域型定義
    └── database.types.ts     # Supabase自動生成型（編集禁止）
```

## Colocation Principle

特定の機能に関連するコードを `features/` フォルダ内に集約：

```
features/properties/
├── components/
│   ├── PropertyCard.tsx      # UI
│   ├── PropertyList.tsx
│   └── PropertyForm.tsx
├── actions/
│   ├── createProperty.ts     # Server Action
│   ├── updateProperty.ts
│   └── deleteProperty.ts
├── api/
│   └── queries.ts            # Supabaseクエリ
├── hooks/
│   └── useProperties.ts      # TanStack Query hooks
├── schemas/
│   └── property.ts           # Zodスキーマ
└── types/
    └── index.ts              # 物件関連の型
```

**利点**: 物件機能を修正する場合、`src/features/properties` フォルダを見るだけで完結。

## App Router Page Pattern

`app/` 内の `page.tsx` は「構成ファイル」として扱い、ロジックを持たせない：

```typescript
// app/(operator)/properties/page.tsx
import { PropertyList } from '@/features/properties/components/PropertyList';
import { getProperties } from '@/features/properties/api/queries';

export default async function PropertiesPage() {
  const properties = await getProperties();

  return (
    <div className="container py-8">
      <h1 className="text-2xl font-bold mb-6">物件一覧</h1>
      <PropertyList properties={properties} />
    </div>
  );
}
```

## Naming Conventions

| 種別 | 規則 | 例 |
|------|------|-----|
| Files | kebab-case または PascalCase | `property-card.tsx`, `PropertyCard.tsx` |
| Components | PascalCase | `PropertyCard` |
| Functions/Hooks | camelCase | `useChat`, `fetchProperties` |
| Server Actions | camelCase | `createProperty`, `deleteProperty` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Types/Interfaces | PascalCase | `Property`, `CustomerWithOperator` |
| Zod Schemas | camelCase + Schema suffix | `propertySchema` |

## Import Organization

```typescript
// 1. React/Next.js
import { useState } from 'react';
import { useRouter } from 'next/navigation';

// 2. External libraries
import { z } from 'zod';
import { useQuery } from '@tanstack/react-query';

// 3. Internal - lib/infrastructure
import { createClient } from '@/lib/supabase/server';

// 4. Internal - features (specific slice)
import { PropertyCard } from '@/features/properties/components/PropertyCard';
import { useProperties } from '@/features/properties/hooks/useProperties';

// 5. Internal - shared components
import { Button } from '@/components/ui/button';

// 6. Internal - relative paths (same feature)
import { PropertyForm } from './PropertyForm';
```

## Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/features/*": ["./src/features/*"],
      "@/lib/*": ["./src/lib/*"],
      "@/types/*": ["./src/types/*"]
    }
  }
}
```

## Dependency Rules

```
app/          →  features/, components/, lib/
features/     →  components/ui/, lib/, types/
components/   →  components/ui/, lib/
lib/          →  types/
```

**禁止パターン**:
- `lib/` → `features/`（インフラがドメインに依存）
- `components/ui/` → `features/`（UIプリミティブがドメインに依存）
- `features/A/` → `features/B/`（スライス間の直接依存）

スライス間でデータを共有する必要がある場合は、親コンポーネント（`app/`）経由で受け渡す。

## Use Client Boundary

### 原則

- `features/*/components/` 内の対話型コンポーネントには `'use client'` を付与
- `app/` 内の `page.tsx` は原則Server Component（`async function`）
- `components/ui/` は必要に応じて `'use client'` を付与

### パターン

```typescript
// features/chat/components/ChatInterface.tsx
'use client';

import { useChat } from 'ai/react';

export function ChatInterface() {
  const { messages, input, handleSubmit } = useChat();
  // クライアントサイドロジック
}
```

```typescript
// app/(customer)/chat/page.tsx (Server Component)
import { ChatInterface } from '@/features/chat/components/ChatInterface';

export default async function ChatPage() {
  // サーバーでの事前処理（認証チェック等）
  return <ChatInterface />;
}
```

## Barrel Exports

各featureに `index.ts` を配置して公開APIを制御：

```typescript
// features/properties/index.ts
export { PropertyCard } from './components/PropertyCard';
export { PropertyList } from './components/PropertyList';
export { useProperties } from './hooks/useProperties';
export { propertySchema } from './schemas/property';
export type { Property } from './types';

// 内部実装は非公開（直接インポート禁止）
// - components/PropertyCardSkeleton.tsx
// - api/queries.ts
```

---
_Document patterns, not file trees. New files following patterns shouldn't require updates_
