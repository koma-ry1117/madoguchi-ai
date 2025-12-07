# API Standards

Next.js App Router Route Handlersを使用。

## Philosophy
- Prefer predictable, resource-oriented design
- RLSによりDBレベルでセキュリティ担保
- Zodで入力バリデーション、型安全なレスポンス

## Endpoint Pattern

```
/api/{resource}[/{id}][/{sub-resource}]
```

### 例
- `GET /api/properties` - 物件一覧
- `GET /api/properties/:id` - 物件詳細
- `POST /api/properties/import` - 物件取り込み
- `POST /api/chat` - AI会話
- `POST /api/marketing/campaigns` - キャンペーン作成
- `GET /api/marketing/campaigns/:id/recipients` - 配信対象者検索
- `POST /api/webhooks/resend` - Resend Webhook

## Request/Response

### Success
```json
{
  "success": true,
  "data": { ... },
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 20
  }
}
```

### Error
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request",
    "details": [
      { "field": "email", "message": "Invalid email format" }
    ]
  }
}
```

## Status Codes
- `200` - 成功（GET, PUT, PATCH）
- `201` - 作成成功（POST）
- `204` - 削除成功（DELETE）
- `400` - バリデーションエラー
- `401` - 未認証
- `403` - 権限不足
- `404` - リソース不存在
- `429` - レート制限
- `500` - サーバーエラー

## Authentication

### Supabase Auth
```typescript
// Route Handler内
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';

export async function GET(request: Request) {
  const supabase = createRouteHandlerClient({ cookies });
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    return NextResponse.json(
      { success: false, error: { code: 'UNAUTHORIZED' } },
      { status: 401 }
    );
  }
  // ...
}
```

### ロール確認
```typescript
// オペレーター権限チェック
const { data: operator } = await supabase
  .from('operators')
  .select('id')
  .eq('user_id', user.id)
  .single();

if (!operator) {
  return NextResponse.json(
    { success: false, error: { code: 'FORBIDDEN' } },
    { status: 403 }
  );
}
```

## Validation

### Zodスキーマ
```typescript
import { z } from 'zod';

const createCampaignSchema = z.object({
  name: z.string().min(1).max(255),
  campaign_type: z.enum(['new_property', 'follow_up', 'reengagement']),
  property_ids: z.array(z.string().uuid()).optional(),
  target_criteria: z.object({
    preferred_areas: z.array(z.string()).optional(),
    budget_max: z.number().positive().optional(),
  }).optional(),
});

export async function POST(request: Request) {
  const body = await request.json();
  const result = createCampaignSchema.safeParse(body);

  if (!result.success) {
    return NextResponse.json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        details: result.error.errors,
      },
    }, { status: 400 });
  }

  const validatedData = result.data;
  // ...
}
```

## Pagination/Filtering

```typescript
// クエリパラメータ
const { searchParams } = new URL(request.url);
const page = parseInt(searchParams.get('page') || '1');
const limit = Math.min(parseInt(searchParams.get('limit') || '20'), 100);
const offset = (page - 1) * limit;

// Supabaseクエリ
const { data, count } = await supabase
  .from('properties')
  .select('*', { count: 'exact' })
  .range(offset, offset + limit - 1);
```

## Rate Limiting

```typescript
// Upstash Ratelimit
import { Ratelimit } from '@upstash/ratelimit';

const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(10, '10s'),
});

const { success } = await ratelimit.limit(ip);
if (!success) {
  return new Response('Too Many Requests', { status: 429 });
}
```

## Webhook Security

### Resend署名検証
```typescript
import { Webhook } from 'svix';

export async function POST(request: Request) {
  const payload = await request.text();
  const headers = Object.fromEntries(request.headers);

  const wh = new Webhook(process.env.RESEND_WEBHOOK_SECRET!);
  const event = wh.verify(payload, headers);
  // ...
}
```

---
_Focus on patterns and decisions, not endpoint catalogs._
