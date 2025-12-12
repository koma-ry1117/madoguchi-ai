# Security Guidelines

多層防御（Defense in Depth）に基づくセキュリティ設計。

## Philosophy

- Defense in depth; least privilege; secure by default; fail closed
- RLSでDBレベルの権限制御、アプリケーション層でバイパス不可
- Validate at boundaries; sanitize for context; never trust input

## Defense in Depth（多層防御）

セキュリティは単一の防御ではなく、複数レイヤーで実装：

```
┌─────────────────────────────────────────┐
│  1. 入力バリデーション（Zod）           │
├─────────────────────────────────────────┤
│  2. 認証チェック（Supabase Auth）       │
├─────────────────────────────────────────┤
│  3. 認可チェック（アプリ層ロジック）     │
├─────────────────────────────────────────┤
│  4. Row Level Security（DB層）          │
└─────────────────────────────────────────┘
```

## Server Actions Security

Server Actionsはクライアントから呼び出し可能な**パブリックAPIエンドポイント**と同等。

### 必須チェック項目

```typescript
'use server';

export async function updateProfile(formData: FormData) {
  // 1. 認証チェック（必須）
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return { error: 'ログインしてください' };
  }

  // 2. 入力バリデーション（必須）
  const parsed = profileSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }

  // 3. 認可チェック（リソースの所有者確認）
  const { data: existing } = await supabase
    .from('profiles')
    .select('user_id')
    .eq('id', parsed.data.id)
    .single();

  if (existing?.user_id !== user.id) {
    return { error: '権限がありません' };
  }

  // 4. DB操作（RLSも有効）
  const { error } = await supabase
    .from('profiles')
    .update(parsed.data)
    .eq('id', parsed.data.id);

  if (error) throw error;

  revalidatePath('/profile');
  return { success: true };
}
```

### getUser vs getSession

```typescript
// ❌ getSession()の使用（改ざん可能）
const { data: { session } } = await supabase.auth.getSession();
if (session) { /* 危険 - JWTが改ざんされている可能性 */ }

// ✅ getUser()を使用（Supabase Authサーバーで検証）
const { data: { user } } = await supabase.auth.getUser();
if (user) { /* 安全 - サーバーで検証済み */ }
```

**重要**: サーバーサイド（Server Components, Server Actions, Middleware）では必ず `getUser()` を使用。

## Authorization (RLS)

### ロール定義

| ロール | 権限 |
|--------|------|
| kiosk_user | AI会話、物件検索（会話内）※認証不要 |
| operator | 自社物件管理、自社顧客管理、営業メール配信 |
| admin | オペレーター管理、全データ閲覧、システム設定 |

### RLSポリシーパターン

```sql
-- RLSを有効化
ALTER TABLE properties ENABLE ROW LEVEL SECURITY;

-- オペレーターは自社の物件のみアクセス可能
CREATE POLICY "Operators can view own properties"
  ON properties FOR SELECT
  USING (operator_id = auth.uid());

CREATE POLICY "Operators can insert own properties"
  ON properties FOR INSERT
  WITH CHECK (operator_id = auth.uid());

CREATE POLICY "Operators can update own properties"
  ON properties FOR UPDATE
  USING (operator_id = auth.uid())
  WITH CHECK (operator_id = auth.uid());

CREATE POLICY "Operators can delete own properties"
  ON properties FOR DELETE
  USING (operator_id = auth.uid());
```

### マルチテナント用パターン

```sql
-- オペレーターテーブルとの結合
CREATE POLICY "Users access own operator data"
  ON customers FOR SELECT
  USING (
    operator_id IN (
      SELECT id FROM operators WHERE user_id = auth.uid()
    )
  );
```

### Service Role の制限

`SUPABASE_SERVICE_ROLE_KEY` はRLSをバイパスするため、使用を最小限に：

| 用途 | 使用可否 |
|------|---------|
| 管理者用バッチ処理 | ○ |
| Webhook処理（外部サービス連携） | ○ |
| 通常のユーザー操作 | ✕ |

## Environment Variables

### 命名規則

| Prefix | 用途 | 露出範囲 |
|--------|------|---------|
| `NEXT_PUBLIC_` | クライアントで使用可能 | ブラウザに露出 |
| （なし） | サーバーのみ | サーバーサイドのみ |

### 機密情報の管理

```bash
# ✅ サーバーのみ（機密）
SUPABASE_SERVICE_ROLE_KEY=eyJ...
RESEND_API_KEY=re_...
GOOGLE_GENERATIVE_AI_API_KEY=...
OPENAI_API_KEY=sk-...

# ✅ クライアント可（公開可能）
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
```

### 環境変数バリデーション

起動時に存在と形式を検証：

```typescript
// env.mjs
import { z } from 'zod';

const envSchema = z.object({
  NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
  NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
  RESEND_API_KEY: z.string().startsWith('re_'),
  GOOGLE_GENERATIVE_AI_API_KEY: z.string().min(1),
});

export const env = envSchema.parse(process.env);
```

## Input Validation

すべての外部入力をZodでバリデーション：

```typescript
import { z } from 'zod';

const propertySchema = z.object({
  name: z.string().min(1).max(100),
  price: z.number().positive().max(1_000_000_000),
  address: z.string().min(1).max(200),
  description: z.string().max(5000).optional(),
});

// Server Actionで使用
export async function createProperty(formData: FormData) {
  const parsed = propertySchema.safeParse({
    name: formData.get('name'),
    price: Number(formData.get('price')),
    address: formData.get('address'),
    description: formData.get('description'),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }
  // parsed.data は型安全
}
```

## XSS Prevention

```typescript
// ✅ 安全（自動エスケープ）
<p>{userInput}</p>

// ❌ 危険（dangerouslySetInnerHTML）
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

リッチテキストが必要な場合はサニタイザーを使用：

```typescript
import DOMPurify from 'isomorphic-dompurify';
const sanitized = DOMPurify.sanitize(userHtml);
```

## Security Headers

```typescript
// next.config.js
module.exports = {
  async headers() {
    return [{
      source: '/:path*',
      headers: [
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
      ],
    }];
  },
};
```

## Rate Limiting

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),
});

export async function POST(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? '127.0.0.1';
  const { success } = await ratelimit.limit(ip);

  if (!success) {
    return new Response('Too Many Requests', { status: 429 });
  }
  // 処理続行
}
```

## Sensitive Data Handling

### ログ出力

```typescript
// ❌ 機密情報をログ出力
console.log('User data:', user);

// ✅ 必要最小限の情報のみ
console.log('User ID:', user.id);
```

### エラーメッセージ

```typescript
// ❌ 内部エラーの詳細を露出
return { error: error.message };

// ✅ ユーザー向けの一般的なメッセージ
console.error('DB Error:', error); // サーバーログには詳細
return { error: '処理中にエラーが発生しました' };
```

## Webhook Security

### Resend署名検証

```typescript
import { Webhook } from 'svix';

const wh = new Webhook(process.env.RESEND_WEBHOOK_SECRET!);
const event = wh.verify(payload, headers);
```

## Chrome Extension Security

- Manifest V3使用
- 最小限のpermissions
- Auth Token暗号化保存
- HTTPSのみ許可

---
_Security is a process, not a product. Review regularly._
