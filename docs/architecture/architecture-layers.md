# アーキテクチャ - レイヤー構成

> 本ドキュメントは [アーキテクチャ設計](../architecture.md) の一部です。

## レイヤーアーキテクチャ

### 1. プレゼンテーション層（Presentation Layer）

**責務**: UI表示、ユーザー入力処理、ルーティング

```
src/app/
├── page.tsx             # ランディングページ
├── login/               # ログインページ（operator/admin用）
│   └── page.tsx
├── (operator)/          # オペレーター向けページ（認証必須）
│   ├── dashboard/
│   ├── properties/
│   ├── customers/
│   ├── viewings/
│   ├── kiosk/           # キオスク画面（オペレーター認証必須）
│   │   ├── page.tsx     # キオスクメイン画面（セッションからoperator_id取得）
│   │   └── layout.tsx   # キオスク専用レイアウト（ヘッダー/サイドバー非表示）
│   └── layout.tsx
├── (admin)/             # 管理者向けページ（認証必須）
│   ├── dashboard/
│   ├── operators/
│   └── layout.tsx
└── layout.tsx           # ルートレイアウト

src/components/
├── ui/                  # shadcn/uiコンポーネント
├── features/
│   ├── kiosk/           # キオスク専用コンポーネント
│   │   ├── KioskInterface.tsx
│   │   ├── AvatarDisplay.tsx
│   │   ├── ConversationPanel.tsx
│   │   ├── VoiceInput.tsx
│   │   ├── ContactForm.tsx
│   │   └── CompletionScreen.tsx
│   ├── properties/      # オペレーター向け物件管理
│   │   ├── PropertyCard.tsx
│   │   ├── PropertyList.tsx
│   │   └── PropertyFilter.tsx
│   └── viewings/        # オペレーター向け予約管理
│       ├── ViewingList.tsx
│       └── ViewingDetail.tsx
└── layouts/
    ├── OperatorLayout.tsx
    ├── AdminLayout.tsx
    └── KioskLayout.tsx
```

---

### 2. アプリケーション層（Application Layer）

**責務**: ビジネスロジック、API呼び出し、状態管理

```
src/lib/
├── hooks/               # カスタムReact Hooks
│   ├── useChat.ts
│   ├── useVoice.ts
│   ├── useProperties.ts
│   └── useAuth.ts
├── stores/              # Zustand状態管理
│   ├── chatStore.ts
│   ├── avatarStore.ts
│   ├── authStore.ts
│   └── uiStore.ts
└── services/            # ビジネスロジック
    ├── chatService.ts
    ├── propertyService.ts
    ├── voiceService.ts
    └── authService.ts
```

---

### 3. インフラストラクチャ層（Infrastructure Layer）

**責務**: 外部サービス連携、データアクセス

```
src/lib/
├── api/                 # API クライアント
│   ├── openai.ts
│   ├── elevenlabs.ts
│   ├── resend.ts
│   └── supabase.ts
├── db/                  # データベースアクセス
│   ├── queries/
│   │   ├── properties.ts
│   │   ├── customers.ts
│   │   ├── viewings.ts
│   │   └── conversations.ts
│   └── mutations/
│       ├── properties.ts
│       ├── viewings.ts
│       └── conversations.ts
├── email/               # メールテンプレート
│   ├── templates/
│   │   ├── ViewingConfirmationEmail.tsx
│   │   ├── InquiryConfirmationEmail.tsx
│   │   └── SystemAnnouncementEmail.tsx
│   └── utils/
│       ├── generateICS.ts
│       └── sendEmail.ts
└── utils/               # ユーティリティ関数
    ├── embedding.ts
    ├── vectorSearch.ts
    └── cache.ts
```

---

### 4. ドメイン層（Domain Layer）

**責務**: ビジネスルール、エンティティ、バリデーション

```
src/types/
├── property.ts          # 物件エンティティ
├── customer.ts          # 顧客エンティティ
├── conversation.ts      # 会話エンティティ
└── auth.ts              # 認証エンティティ

src/schemas/             # Zodスキーマ
├── propertySchema.ts
├── customerSchema.ts
└── conversationSchema.ts
```
