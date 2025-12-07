# Security Standards

多層防御（Defense in Depth）に基づくセキュリティ設計。

## Philosophy
- Defense in depth; least privilege; secure by default; fail closed
- RLSでDBレベルの権限制御、アプリケーション層でバイパス不可
- Validate at boundaries; sanitize for context; never trust input

## Security Layers

```
1. ネットワーク層: HTTPS, CORS, CSP
2. 認証・認可層: Supabase Auth, JWT, RLS
3. アプリケーション層: Zod検証, レート制限
4. データベース層: RLS, 暗号化
5. 監視・監査層: ログ記録, 異常検知
```

## Authentication

### Supabase Auth
- JWT（JSON Web Token）によるステートレス認証
- Access Token（1時間）+ Refresh Token（30日）
- 自動リフレッシュ機能

### セッション管理
```typescript
// middleware.ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs';

export async function middleware(request: NextRequest) {
  const supabase = createMiddlewareClient({ req: request, res });
  const { data: { session } } = await supabase.auth.getSession();

  if (!session) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}
```

## Authorization (RLS)

### ロール
| ロール | 権限 |
|--------|------|
| customer | 物件閲覧、会話履歴閲覧（自分のみ） |
| operator | 自社物件管理、自社顧客管理 |
| admin | 全データ管理、システム設定 |

### RLSパターン
```sql
-- オペレーター：自社データのみ
CREATE POLICY "operator_own_properties"
ON properties FOR ALL
USING (
  operator_id IN (SELECT id FROM operators WHERE user_id = auth.uid())
);

-- 管理者：全データ
CREATE POLICY "admin_all"
ON properties FOR ALL
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

## Input Validation

### Zodで境界バリデーション
```typescript
const propertySearchSchema = z.object({
  area: z.string().max(100).optional(),
  rent_min: z.number().min(0).max(999999999).optional(),
  rent_max: z.number().min(0).max(999999999).optional(),
});

const result = propertySearchSchema.safeParse(body);
if (!result.success) {
  return NextResponse.json({ error: 'VALIDATION_ERROR' }, { status: 400 });
}
```

### SQLインジェクション対策
- Supabaseクライアント使用（自動パラメータ化）
- 生SQLは使用しない

### XSS対策
- React自動エスケープ
- `dangerouslySetInnerHTML`禁止
- CSPヘッダー設定

## Secrets Management

### 環境変数
```bash
# サーバーサイドのみ（NEXT_PUBLIC_なし）
OPENAI_API_KEY=sk-...
RESEND_API_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...

# クライアント公開可
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
```

- `.env.local`はGitにコミットしない
- Vercel環境変数で管理
- 起動時に必須環境変数を検証

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
// Upstash Redis
const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(10, '10s'),
});

// エンドポイント別制限
const chatRatelimit = Ratelimit.slidingWindow(5, '60s'); // 1分5回
```

## Logging (Security-aware)

### 記録すべき
- 認証試行（成功・失敗）
- 権限拒否
- 機密操作（データ削除、設定変更）

### 記録禁止
- パスワード、トークン
- 完全なリクエストボディ
- 個人情報（マスク処理）

```typescript
await logSecurityEvent('unauthorized_access_attempt', {
  endpoint: '/api/admin/users',
  userId: user.id,
  role: user.role,
  // パスワードやトークンは含めない
});
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
_Focus on patterns and principles. Link concrete configs to ops docs._
