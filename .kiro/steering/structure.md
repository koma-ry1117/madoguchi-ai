# Project Structure

## Organization Philosophy

**Feature-first + Atomic Design**のハイブリッド構成。UIコンポーネントはAtomic Design、機能単位のロジックはfeature-firstで整理。

## Directory Patterns

### App Router Pages
**Location**: `app/`
**Purpose**: ルーティング、ページコンポーネント、APIルート
**Example**:
```
app/
├── (auth)/login/page.tsx      # 認証グループ
├── (customer)/chat/page.tsx   # 顧客グループ
├── (operator)/dashboard/      # オペレーターグループ
├── (admin)/operators/         # 管理者グループ
└── api/                       # APIルート
```

### UI Components (Atoms)
**Location**: `src/components/ui/`
**Purpose**: shadcn/uiベースの基本コンポーネント
**Example**: `button.tsx`, `input.tsx`, `card.tsx`

### Feature Components (Molecules/Organisms)
**Location**: `src/components/features/`
**Purpose**: 機能単位のコンポーネント
**Example**:
```
features/
├── chat/
│   ├── MessageBubble.tsx
│   ├── VoiceButton.tsx
│   └── ChatInterface.tsx
├── avatar/
│   └── AvatarDisplay.tsx
├── properties/
│   ├── PropertyCard.tsx
│   └── PropertyList.tsx
└── marketing/           # 営業メール機能
    ├── CampaignForm.tsx
    └── RecipientsList.tsx
```

### Server Actions & API
**Location**: `src/actions/`, `app/api/`
**Purpose**: サーバーサイドロジック
**Example**:
```
src/actions/
├── chat.ts         # AI会話アクション
├── properties.ts   # 物件操作
└── marketing.ts    # 営業メール

app/api/
├── chat/route.ts
├── properties/import/route.ts
├── marketing/campaigns/route.ts
└── webhooks/resend/route.ts
```

### Hooks & Stores
**Location**: `src/hooks/`, `src/stores/`
**Purpose**: カスタムフック、Zustandストア
**Example**:
```
hooks/
├── useChat.ts
├── useVoice.ts
└── useProperties.ts

stores/
├── chatStore.ts      # 会話履歴
├── avatarStore.ts    # 感情状態
└── authStore.ts      # 認証状態
```

### Schemas & Types
**Location**: `src/schemas/`, `src/types/`
**Purpose**: Zodスキーマ、TypeScript型定義
**Example**:
```
schemas/
├── property.ts
├── customer.ts
└── campaign.ts

types/
├── database.ts    # Supabase生成型
└── api.ts         # APIレスポンス型
```

### Lib & Utils
**Location**: `src/lib/`, `src/utils/`
**Purpose**: 外部サービスクライアント、ユーティリティ
**Example**:
```
lib/
├── supabase/
│   ├── client.ts
│   └── server.ts
├── openai.ts
├── elevenlabs.ts
└── resend.ts

utils/
├── format.ts
└── validation.ts
```

## Naming Conventions

- **Files**: kebab-case（`property-card.tsx`）またはPascalCase（`PropertyCard.tsx`）
- **Components**: PascalCase（`PropertyCard`）
- **Functions/Hooks**: camelCase（`useChat`, `fetchProperties`）
- **Constants**: SCREAMING_SNAKE_CASE（`MAX_RETRY_COUNT`）
- **Types/Interfaces**: PascalCase（`Property`, `CustomerWithOperator`）

## Import Organization

```typescript
// 1. React/Next.js
import { useState } from 'react';
import { useRouter } from 'next/navigation';

// 2. External libraries
import { z } from 'zod';
import { useQuery } from '@tanstack/react-query';

// 3. Internal - absolute paths
import { Button } from '@/components/ui/button';
import { useChat } from '@/hooks/useChat';
import { propertySchema } from '@/schemas/property';

// 4. Internal - relative paths
import { MessageBubble } from './MessageBubble';
```

**Path Aliases**:
- `@/`: `src/`にマップ
- `@/components/*`: `src/components/*`
- `@/lib/*`: `src/lib/*`

## Code Organization Principles

1. **コロケーション**: 関連するファイルは近くに配置
2. **単一責任**: 1ファイル1コンポーネント/1機能
3. **依存方向**: UI → Features → Lib（逆方向の依存禁止）
4. **バレルエクスポート**: 各ディレクトリに`index.ts`で再エクスポート

---
_Document patterns, not file trees. New files following patterns shouldn't require updates_
