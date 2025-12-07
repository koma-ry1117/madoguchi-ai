# アーキテクチャ - フロントエンド構成

> 本ドキュメントは [アーキテクチャ設計](../architecture.md) の一部です。

## コンポーネント構成

### Atomic Design パターン

```
src/components/
├── ui/ (Atoms)
│   ├── button.tsx
│   ├── input.tsx
│   ├── card.tsx
│   ├── avatar.tsx
│   ├── badge.tsx
│   └── ...
│
├── features/ (Molecules)
│   ├── chat/
│   │   ├── MessageBubble.tsx
│   │   ├── VoiceButton.tsx
│   │   └── TypingIndicator.tsx
│   ├── avatar/
│   │   ├── AvatarDisplay.tsx
│   │   └── EmotionIndicator.tsx
│   └── properties/
│       ├── PropertyCard.tsx
│       ├── PropertyImage.tsx
│       └── PropertyPrice.tsx
│
└── layouts/ (Organisms)
    ├── ChatInterface.tsx
    ├── PropertyList.tsx
    ├── DashboardLayout.tsx
    └── Header.tsx
```

---

## ページ構成（App Router）

```
app/
├── layout.tsx                    # ルートレイアウト
├── page.tsx                      # ランディングページ
├── login/
│   └── page.tsx                  # ログイン（operator/admin用）
├── error.tsx                     # エラーページ
│
├── (operator)/                   # オペレーターグループ（認証必須）
│   ├── layout.tsx
│   ├── dashboard/
│   │   └── page.tsx
│   ├── properties/
│   │   ├── page.tsx              # 物件一覧
│   │   ├── [id]/
│   │   │   └── page.tsx          # 物件詳細・編集
│   │   └── create/
│   │       └── page.tsx          # 物件登録
│   ├── customers/
│   │   ├── page.tsx              # 顧客一覧
│   │   └── [id]/
│   │       └── page.tsx          # 顧客詳細
│   ├── viewings/
│   │   ├── page.tsx              # 内見予約一覧
│   │   └── [id]/
│   │       └── page.tsx          # 内見予約詳細
│   ├── conversations/
│   │   ├── page.tsx              # 会話一覧
│   │   └── [id]/
│   │       └── page.tsx          # 会話詳細
│   ├── marketing/
│   │   └── campaigns/
│   │       ├── page.tsx          # キャンペーン一覧
│   │       └── [id]/
│   │           └── page.tsx      # キャンペーン詳細
│   └── kiosk/                    # キオスク画面（オペレーター認証必須）
│       ├── page.tsx              # キオスクメイン（セッションからoperator_id取得）
│       └── layout.tsx            # キオスク専用レイアウト（サイドバー非表示）
│
├── (admin)/                      # 管理者グループ（認証必須）
│   ├── layout.tsx
│   ├── dashboard/
│   │   └── page.tsx              # システム全体のダッシュボード
│   ├── operators/
│   │   ├── page.tsx              # オペレーター一覧
│   │   ├── [id]/
│   │   │   └── page.tsx          # オペレーター詳細・編集
│   │   └── create/
│   │       └── page.tsx          # 新規オペレーター登録
│   ├── stats/
│   │   └── page.tsx              # 統計ダッシュボード
│   ├── logs/
│   │   └── page.tsx              # 監査ログ
│   └── settings/
│       └── page.tsx              # システム設定
│
└── api/                          # API Routes
    ├── kiosk/                   # キオスク用API（オペレーター認証必須）
    │   ├── chat/
    │   │   └── route.ts         # POST (キオスク会話)
    │   └── complete/
    │       └── route.ts         # POST (予約完了、顧客情報登録)
    ├── voice/
    │   ├── transcribe/
    │   │   └── route.ts
    │   └── synthesize/
    │       └── route.ts
    ├── properties/
    │   ├── route.ts             # GET (一覧), POST (作成)
    │   ├── [id]/
    │   │   └── route.ts         # GET/PATCH/DELETE
    │   └── import/
    │       └── route.ts         # POST (Chrome拡張からの取り込み)
    ├── viewings/
    │   ├── route.ts             # GET (一覧)
    │   └── [id]/
    │       ├── route.ts         # GET/PATCH/DELETE
    │       └── conversation/
    │           └── route.ts     # GET (会話履歴)
    ├── marketing/
    │   └── campaigns/
    │       └── route.ts
    └── admin/
        ├── operators/
        │   ├── route.ts
        │   └── [id]/
        │       └── route.ts
        └── stats/
            └── route.ts
```

**重要**:
- `(customer)/` グループは存在しない。顧客向け機能は `/(operator)/kiosk` のみ。
- キオスク画面はオペレーターがログインした状態で開く。セッションからoperator_idを取得するため、URLにoperator_idを露出しない。
- キオスク利用者（顧客）は認証アカウントを持たない。会話終了時に名前・電話番号・メールアドレスのみを入力する。

---

## 状態管理戦略

### Zustand（クライアント状態）

```typescript
// src/lib/stores/chatStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
}

interface ChatStore {
  messages: Message[];
  isLoading: boolean;
  addMessage: (message: Message) => void;
  setLoading: (loading: boolean) => void;
  clearMessages: () => void;
}

export const useChatStore = create<ChatStore>()(
  persist(
    (set) => ({
      messages: [],
      isLoading: false,
      addMessage: (message) =>
        set((state) => ({ messages: [...state.messages, message] })),
      setLoading: (loading) => set({ isLoading: loading }),
      clearMessages: () => set({ messages: [] }),
    }),
    {
      name: 'chat-storage',
      partialize: (state) => ({ messages: state.messages }),
    }
  )
);
```

### TanStack Query（サーバー状態）

```typescript
// src/lib/hooks/useProperties.ts
import { useQuery } from '@tanstack/react-query';
import { searchProperties } from '@/lib/services/propertyService';

export const useProperties = (filters: PropertyFilters) => {
  return useQuery({
    queryKey: ['properties', filters],
    queryFn: () => searchProperties(filters),
    staleTime: 5 * 60 * 1000, // 5分間キャッシュ
    cacheTime: 10 * 60 * 1000, // 10分間保持
    refetchOnWindowFocus: false,
  });
};
```
