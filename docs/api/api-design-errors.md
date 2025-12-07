# レート制限・エラーコード

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## レート制限

### 実装（Upstash Redis）

```typescript
// middleware.ts
import { Ratelimit } from '@upstash/ratelimit';
import { kv } from '@vercel/kv';

const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(10, '10 s'), // 10秒で10リクエスト
});

export async function middleware(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for');

  const { success } = await ratelimit.limit(ip);

  if (!success) {
    return new Response('Too Many Requests', { status: 429 });
  }

  return NextResponse.next();
}
```

---

## エラーコード一覧

| コード | 説明 |
|--------|------|
| AUTH_FAILED | 認証失敗 |
| UNAUTHORIZED | 認証が必要 |
| FORBIDDEN | 権限不足 |
| NOT_FOUND | リソースが見つからない |
| VALIDATION_ERROR | バリデーションエラー |
| DB_ERROR | データベースエラー |
| OPENAI_ERROR | OpenAI APIエラー |
| ELEVENLABS_ERROR | ElevenLabs APIエラー |
| RATE_LIMIT_EXCEEDED | レート制限超過 |
| INTERNAL_ERROR | 内部サーバーエラー |
