# Architecture Patterns

Next.js 15 App Router と React 19 における設計パターンと注意点。

## Next.js 15 Breaking Changes

### 非同期リクエストAPI

Next.js 15で `cookies()`, `headers()`, `params` などのリクエストAPIが非同期化。

```typescript
// ❌ Next.js 14以前（動作しない）
const cookieStore = cookies();
const token = cookieStore.get('token');

// ✅ Next.js 15以降
const cookieStore = await cookies();
const token = cookieStore.get('token');
```

**影響**: Supabaseクライアント生成、認証チェック、ミドルウェア処理すべてに影響。

### キャッシュ戦略の変更

Next.js 15では `fetch` のデフォルトが **キャッシュなし**（`cache: 'no-store'` 相当）に変更。

| データ種別 | 戦略 | 実装 |
|-----------|------|------|
| 動的データ（ユーザー固有） | キャッシュなし | デフォルトのまま |
| 静的データ（マスタ） | 明示的キャッシュ | `unstable_cache` または `next: { revalidate }` |

```typescript
// 静的データのキャッシュ例
import { unstable_cache } from 'next/cache';

const getPrefectures = unstable_cache(
  async () => {
    const { data } = await supabase.from('prefectures').select('*');
    return data;
  },
  ['prefectures'],
  { revalidate: 3600 } // 1時間
);
```

### Request Memoization

同一リクエスト内で同じデータを複数回取得する場合、Reactの `cache` でメモ化：

```typescript
import { cache } from 'react';

export const getCurrentUser = cache(async () => {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  return user;
});

// Layout と Page の両方で呼んでも1回のみ実行
```

## React 19 Strict Mode

### ハイドレーション不一致

サーバーとクライアントでDOMが異なると、ハイドレーションエラーが発生しクライアントレンダリングにフォールバック（パフォーマンス劣化）。

**主な原因と対策**:

| 原因 | 対策 |
|------|------|
| 不正なHTMLネスト（`<p>`内の`<div>`等） | HTML仕様に準拠したマークアップ |
| `window`/`localStorage`の早期アクセス | `useEffect`内または`useSyncExternalStore`を使用 |
| 日時表示（タイムゾーン差） | サーバーでフォーマット、またはClient Component化 |
| ブラウザ拡張機能によるDOM変更 | 開発時はシークレットウィンドウで検証 |

```typescript
// ❌ ハイドレーションエラーの原因
function Component() {
  const width = window.innerWidth; // SSR時にエラー
  return <div>{width > 768 ? 'Desktop' : 'Mobile'}</div>;
}

// ✅ 正しいパターン
function Component() {
  const [width, setWidth] = useState<number | null>(null);

  useEffect(() => {
    setWidth(window.innerWidth);
  }, []);

  if (width === null) return <div>Loading...</div>;
  return <div>{width > 768 ? 'Desktop' : 'Mobile'}</div>;
}
```

### Effect二重実行

Strict Modeでは開発環境でEffectが2回実行される。クリーンアップ関数を必ず返す：

```typescript
// ✅ 正しいクリーンアップ
useEffect(() => {
  const { data: { subscription } } = supabase.auth.onAuthStateChange(
    (event, session) => {
      // 処理
    }
  );

  return () => {
    subscription.unsubscribe(); // 必須
  };
}, []);
```

## Component Boundaries

### Server vs Client Components

| 種別 | 用途 | マーカー |
|------|------|---------|
| Server Component | データ取得、静的レンダリング | デフォルト |
| Client Component | インタラクション、ブラウザAPI | `'use client'` |

**原則**:
- できる限りServer Componentを使用（バンドルサイズ削減）
- `useState`, `useEffect`, イベントハンドラが必要な場合のみClient Component
- Server ComponentからClient Componentをインポートして子として渡すのはOK

```typescript
// app/dashboard/page.tsx (Server Component)
import { ClientSidebar } from '@/components/ClientSidebar';

export default async function DashboardPage() {
  const data = await fetchData(); // サーバーでデータ取得

  return (
    <div>
      <ClientSidebar /> {/* Client Componentを配置 */}
      <main>{/* data を使用 */}</main>
    </div>
  );
}
```

### Composition Pattern

Server ComponentのchildrenとしてClient Componentにデータを渡す：

```typescript
// ServerWrapper.tsx (Server Component)
export default async function ServerWrapper() {
  const data = await fetchData();
  return <ClientComponent data={data} />;
}

// ClientComponent.tsx
'use client';
export function ClientComponent({ data }: { data: Data }) {
  const [state, setState] = useState(data);
  // インタラクティブな処理
}
```

## Routing Patterns

### Route Groups

URLに影響を与えずにレイアウトをグループ化：

```
app/
├── (auth)/           # 認証フロー用レイアウト
│   ├── login/
│   └── signup/
├── (customer)/       # 顧客向けレイアウト（キオスク）
│   └── chat/
├── (operator)/       # オペレーター向けレイアウト
│   └── dashboard/
└── (admin)/          # 管理者向けレイアウト
    └── operators/
```

### Parallel Routes

同時に複数のページセグメントをレンダリング：

```
app/
└── dashboard/
    ├── @stats/page.tsx
    ├── @activity/page.tsx
    └── layout.tsx
```

### Loading & Error States

```
app/
└── properties/
    ├── page.tsx
    ├── loading.tsx    # Suspense boundary
    └── error.tsx      # Error boundary
```

## Data Fetching Patterns

### Server Component内でのフェッチ

```typescript
// app/properties/page.tsx
export default async function PropertiesPage() {
  const supabase = await createClient();
  const { data: properties } = await supabase
    .from('properties')
    .select('*')
    .eq('is_active', true);

  return <PropertyList properties={properties} />;
}
```

### Server Actions

データ変更（mutation）は Server Actions を使用：

```typescript
// actions/properties.ts
'use server';

export async function createProperty(formData: FormData) {
  const supabase = await createClient();

  // 認証チェック
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  // バリデーション
  const parsed = propertySchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }

  // DB操作
  const { error } = await supabase.from('properties').insert(parsed.data);
  if (error) throw error;

  revalidatePath('/properties');
  return { success: true };
}
```

## Performance Optimization

### 動的レンダリングの制御

```typescript
// 静的生成を強制
export const dynamic = 'force-static';

// 動的レンダリングを強制
export const dynamic = 'force-dynamic';

// ISR（Incremental Static Regeneration）
export const revalidate = 3600; // 1時間
```

### Streaming

`loading.tsx` または `<Suspense>` でストリーミングを有効化：

```typescript
import { Suspense } from 'react';

export default function Page() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<StatsSkeleton />}>
        <Stats /> {/* 非同期コンポーネント */}
      </Suspense>
    </div>
  );
}
```

---
_Document patterns that prevent common pitfalls in Next.js 15 + React 19_
