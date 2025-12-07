# アーキテクチャ - セキュリティ・パフォーマンス・監視

> 本ドキュメントは [アーキテクチャ設計](../architecture.md) の一部です。

## セキュリティアーキテクチャ

### 認証フロー

```
┌────────────┐
│  ユーザー   │
└──────┬─────┘
       │ 1. ログインフォーム送信
       ▼
┌────────────────────┐
│ Next.js             │
│ Server Action       │
└──────┬─────────────┘
       │ 2. Supabase Auth呼出
       ▼
┌────────────────────┐
│ Supabase Auth      │
│                    │
│ 1. 認証情報検証    │
│ 2. JWT発行         │
│ 3. Refresh Token発行│
└──────┬─────────────┘
       │ 3. JWT + Refresh Token
       ▼
┌────────────────────┐
│ Next.js             │
│                    │
│ Cookie設定         │
│ (httpOnly, secure) │
└──────┬─────────────┘
       │ 4. リダイレクト
       ▼
┌────────────┐
│ ダッシュボード│
└────────────┘
```

### Row Level Security（RLS）

マルチテナント構造による厳格なデータ分離

```sql
-- operators テーブル
-- オペレーター：自社情報のみ閲覧・更新
-- 管理者：全オペレーター情報を管理

-- properties テーブル
-- オペレーター：自社の物件のみ管理
CREATE POLICY "operator_manage_own_properties"
ON properties FOR ALL
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべての物件を閲覧・管理
CREATE POLICY "admin_all"
ON properties FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);

-- customers テーブル
-- オペレーター：自社の顧客のみ管理
-- 管理者：全顧客を閲覧（監査用）

-- conversations / messages テーブル
-- オペレーター：自社の顧客の会話のみ閲覧
-- 管理者：全会話を閲覧（監査用）
```

詳細は [データベース設計](./database-design.md) を参照

---

## パフォーマンス最適化戦略

### 1. フロントエンド最適化

#### 画像最適化
- Next.js Image コンポーネント使用
- WebP形式への自動変換
- 遅延読み込み（Lazy Loading）

#### コード分割
```typescript
// 動的インポート
const PropertyMap = dynamic(() => import('@/components/PropertyMap'), {
  loading: () => <Skeleton />,
  ssr: false,
});
```

#### React Server Components
```typescript
// app/properties/page.tsx
import { getProperties } from '@/lib/db/queries/properties';

export default async function PropertiesPage() {
  // サーバーで直接DB取得
  const properties = await getProperties();

  return <PropertyList properties={properties} />;
}
```

### 2. バックエンド最適化

#### データベースクエリ最適化
```sql
-- インデックス作成
CREATE INDEX idx_properties_area ON properties(area);
CREATE INDEX idx_properties_rent ON properties(rent);
CREATE INDEX idx_properties_layout ON properties(layout);

-- 複合インデックス
CREATE INDEX idx_properties_search
ON properties(area, rent, layout)
WHERE is_public = true;
```

#### キャッシング戦略
```typescript
// Redis風のキャッシング（Vercel KV）
import { kv } from '@vercel/kv';

export async function getPropertiesWithCache(filters: PropertyFilters) {
  const cacheKey = `properties:${JSON.stringify(filters)}`;

  // キャッシュ確認
  const cached = await kv.get(cacheKey);
  if (cached) return cached;

  // DBクエリ
  const properties = await searchProperties(filters);

  // キャッシュ保存（5分）
  await kv.set(cacheKey, properties, { ex: 300 });

  return properties;
}
```

### 3. API最適化

#### レスポンス圧縮
```typescript
// middleware.ts
import { NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const response = NextResponse.next();
  response.headers.set('Content-Encoding', 'gzip');
  return response;
}
```

#### レート制限
```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { kv } from '@vercel/kv';

const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(10, '10 s'),
});

export async function POST(request: Request) {
  const { success } = await ratelimit.limit(request.headers.get('x-forwarded-for'));

  if (!success) {
    return new Response('Too Many Requests', { status: 429 });
  }

  // ...処理
}
```

---

## 監視・ロギングアーキテクチャ

### エラー監視（Sentry）

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0,
  environment: process.env.NODE_ENV,
  integrations: [
    new Sentry.BrowserTracing(),
    new Sentry.Replay(),
  ],
});
```

### ログ収集（Axiom）

```typescript
import { Logger } from 'next-axiom';

export async function POST(request: Request) {
  const logger = new Logger();

  try {
    // 処理
    logger.info('Property imported', { propertyId: 123 });
  } catch (error) {
    logger.error('Import failed', { error });
  } finally {
    await logger.flush();
  }
}
```
