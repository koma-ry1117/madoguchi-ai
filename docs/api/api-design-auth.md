# 認証API

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## POST /api/auth/login

**説明**: メール/パスワードでログイン

**リクエスト**:
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "role": "customer"
    },
    "session": {
      "access_token": "...",
      "refresh_token": "...",
      "expires_at": 1234567890
    }
  }
}
```

**実装例**:
```typescript
// app/api/auth/login/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const { email, password } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (error) {
    return NextResponse.json(
      { success: false, error: { code: 'AUTH_FAILED', message: error.message } },
      { status: 401 }
    );
  }

  return NextResponse.json({
    success: true,
    data: {
      user: data.user,
      session: data.session,
    },
  });
}
```

---

## POST /api/auth/logout

**説明**: ログアウト

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "message": "ログアウトしました"
  }
}
```

---

## POST /api/auth/refresh

**説明**: アクセストークンのリフレッシュ

**リクエスト**:
```json
{
  "refresh_token": "..."
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "access_token": "...",
    "expires_at": 1234567890
  }
}
```
