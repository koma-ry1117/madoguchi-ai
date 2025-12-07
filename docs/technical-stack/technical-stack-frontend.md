# 技術スタック - フロントエンド

> 本ドキュメントは [技術スタック詳細](../technical-stack.md) の一部です。

## コア技術

### Next.js v15 (App Router)
- **選定理由**:
  - React Server Componentsによる高速化
  - ファイルベースルーティング
  - サーバーレスアーキテクチャとの親和性
  - Vercelとの完璧な統合
- **使用機能**:
  - App Router（`app/` ディレクトリ）
  - Server Actions（フォーム送信・データ更新）
  - Route Handlers（APIエンドポイント）
  - Metadata API（SEO対策）
  - Image Optimization

### TypeScript
- **選定理由**:
  - 型安全性による開発効率向上
  - AIコード生成との相性が良い
  - 大規模化時のメンテナンス性
  - チーム開発時のコミュニケーションコスト削減

### React 19
- **使用機能**:
  - Hooks（useState, useEffect, useCallback等）
  - Context API（グローバル状態の軽量管理）
  - Suspense（非同期データローディング）

### Tailwind CSS
- **選定理由**:
  - ユーティリティファーストで開発速度向上
  - デザインの一貫性確保
  - バンドルサイズ最適化（PurgeCSS）
  - shadcn/uiとの完璧な統合

---

## UI/UXライブラリ

### shadcn/ui
- **選定理由**:
  - 高品質なアクセシブルコンポーネント
  - Radix UIベースで堅牢
  - カスタマイズ性が高い
  - コピー&ペーストで使える
- **使用コンポーネント**:
  - Button, Input, Select, Dialog
  - Dropdown Menu, Popover, Tooltip
  - Card, Avatar, Badge
  - Form, Table, Tabs

### Framer Motion
- **用途**: アバター表情切り替えアニメーション
- **使用機能**:
  - `motion.img` による画像アニメーション
  - `AnimatePresence` による入退場アニメーション
  - スプリングアニメーション（自然な動き）

### React Hook Form
- **用途**: フォーム管理
- **使用機能**:
  - バリデーション統合（Zod）
  - パフォーマンス最適化
  - エラーハンドリング

### Zod
- **用途**: スキーマバリデーション
- **使用例**:
  - フォーム入力検証
  - APIレスポンス検証
  - 環境変数検証

---

## アバター表示（静止画ベース）

### アバター画像管理

**静止画セット（5種類）**:
```
public/avatars/
├── neutral.png     # 通常
├── happy.png       # 喜び
├── thinking.png    # 考え中
├── apologetic.png  # 謝罪
└── excited.png     # 興奮
```

**画像仕様**:
- 形式: PNG（透過背景）
- サイズ: 512x512px 推奨
- 最適化: TinyPNGで圧縮

### 表情切り替えシステム

#### GPT-4o感情分析
```typescript
const analyzeEmotion = async (response: string) => {
  const { object } = await generateObject({
    model: openai('gpt-4o'),
    schema: z.object({
      emotion: z.enum(['neutral', 'happy', 'thinking', 'apologetic', 'excited']),
      confidence: z.number().min(0).max(1),
      reason: z.string(),
    }),
    prompt: `以下のレスポンスから感情を判定してください：${response}`,
  });

  return object;
};
```

#### React状態管理（Zustand）
```typescript
import { create } from 'zustand';

interface AvatarStore {
  emotion: EmotionType;
  setEmotion: (emotion: EmotionType) => void;
}

export const useAvatarStore = create<AvatarStore>((set) => ({
  emotion: 'neutral',
  setEmotion: (emotion) => set({ emotion }),
}));
```

#### Framer Motion アニメーション
```typescript
import { motion, AnimatePresence } from 'framer-motion';

<AnimatePresence mode="wait">
  <motion.img
    key={emotion}
    src={`/avatars/${emotion}.png`}
    initial={{ opacity: 0, scale: 0.95 }}
    animate={{ opacity: 1, scale: 1 }}
    exit={{ opacity: 0, scale: 0.95 }}
    transition={{ duration: 0.3, ease: 'easeInOut' }}
  />
</AnimatePresence>
```

---

## 状態管理・データフェッチ

### グローバル状態管理

#### Zustand
**選定理由**:
- Reduxより軽量・シンプル
- TypeScript完全対応
- Reactフックベース
- DevTools対応

**管理する状態**:
- 会話履歴
- 現在の物件フィルタ
- アバター感情状態
- 認証状態（ユーザー情報・セッション）
- UI状態（モーダル・サイドバー等）

**ストア例**:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface ChatStore {
  messages: Message[];
  addMessage: (message: Message) => void;
  clearMessages: () => void;
}

export const useChatStore = create<ChatStore>()(
  persist(
    (set) => ({
      messages: [],
      addMessage: (message) =>
        set((state) => ({ messages: [...state.messages, message] })),
      clearMessages: () => set({ messages: [] }),
    }),
    {
      name: 'chat-storage',
    }
  )
);
```

### サーバーステート管理

#### TanStack Query (React Query)
**選定理由**:
- サーバーデータのキャッシング
- 自動再取得・バックグラウンド更新
- 楽観的更新
- 無限スクロール対応
- エラーハンドリング・リトライ

**使用例**:
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// データフェッチ
const { data, isLoading } = useQuery({
  queryKey: ['properties', filters],
  queryFn: () => fetchProperties(filters),
  staleTime: 5 * 60 * 1000, // 5分間キャッシュ
});

// データ更新
const mutation = useMutation({
  mutationFn: updateProperty,
  onSuccess: () => {
    queryClient.invalidateQueries(['properties']);
  },
});
```
