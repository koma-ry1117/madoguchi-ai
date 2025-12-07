# 管理者API

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## GET /api/admin/operators

**説明**: オペレーター（不動産会社）一覧取得（管理者のみ）

**認証**: 必須（admin）

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| subscription_status | string | サブスク状態フィルター |
| is_active | boolean | 有効状態フィルター |
| page | number | ページ番号 |
| limit | number | 取得件数 |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "operators": [
      {
        "id": "uuid",
        "user_id": "uuid",
        "company_name": "〇〇不動産株式会社",
        "email": "info@example-fudosan.com",
        "phone": "03-1234-5678",
        "subscription_status": "active",
        "subscription_plan": "standard",
        "contract_start_date": "2025-01-01",
        "is_active": true,
        "created_at": "2025-01-01T00:00:00Z",
        "stats": {
          "property_count": 150,
          "customer_count": 45,
          "viewing_count": 23
        }
      }
    ],
    "pagination": {
      "total": 50,
      "page": 1,
      "limit": 20
    }
  }
}
```

---

## POST /api/admin/operators

**説明**: 新規オペレーター（不動産会社）登録（管理者のみ）

**認証**: 必須（admin）

**リクエスト**:
```json
{
  "email": "operator@example.com",
  "password": "securepassword123",
  "company_name": "〇〇不動産株式会社",
  "company_address": "東京都文京区...",
  "phone": "03-1234-5678",
  "subscription_plan": "standard",
  "contract_start_date": "2025-11-15"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "operator_id": "uuid",
    "user_id": "uuid",
    "company_name": "〇〇不動産株式会社",
    "message": "オペレーターを登録しました"
  }
}
```

**実装例**:
```typescript
// app/api/admin/operators/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const {
    email,
    password,
    company_name,
    company_address,
    phone,
    subscription_plan,
    contract_start_date,
  } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // 認証チェック
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // 管理者権限チェック
  const { data: userData } = await supabase
    .from('auth.users')
    .select('role')
    .eq('id', user.id)
    .single();

  if (userData.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  // オペレーターユーザーアカウント作成
  const { data: newUser, error: authError } = await supabase.auth.admin.createUser({
    email,
    password,
    email_confirm: true,
    user_metadata: {
      role: 'operator',
    },
  });

  if (authError) {
    return NextResponse.json(
      { success: false, error: { code: 'AUTH_ERROR', message: authError.message } },
      { status: 500 }
    );
  }

  // auth.users テーブルの role を更新
  await supabase
    .from('auth.users')
    .update({ role: 'operator' })
    .eq('id', newUser.user.id);

  // operators テーブルに企業情報を登録
  const { data: operator, error: operatorError } = await supabase
    .from('operators')
    .insert({
      user_id: newUser.user.id,
      company_name,
      company_address,
      phone,
      email,
      subscription_plan,
      subscription_status: 'active',
      contract_start_date,
      is_active: true,
    })
    .select()
    .single();

  if (operatorError) {
    // ロールバック：作成したユーザーを削除
    await supabase.auth.admin.deleteUser(newUser.user.id);
    return NextResponse.json(
      { success: false, error: { code: 'DB_ERROR', message: operatorError.message } },
      { status: 500 }
    );
  }

  return NextResponse.json({
    success: true,
    data: {
      operator_id: operator.id,
      user_id: newUser.user.id,
      company_name: operator.company_name,
      message: 'オペレーターを登録しました',
    },
  });
}
```

---

## GET /api/admin/operators/[id]

**説明**: オペレーター詳細取得（管理者のみ）

**認証**: 必須（admin）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "user_id": "uuid",
    "company_name": "〇〇不動産株式会社",
    "company_address": "東京都文京区...",
    "email": "info@example-fudosan.com",
    "phone": "03-1234-5678",
    "subscription_status": "active",
    "subscription_plan": "standard",
    "contract_start_date": "2025-01-01",
    "contract_end_date": null,
    "is_active": true,
    "created_at": "2025-01-01T00:00:00Z",
    "updated_at": "2025-11-15T00:00:00Z",
    "stats": {
      "property_count": 150,
      "customer_count": 45,
      "viewing_count": 23,
      "inquiry_count": 67,
      "conversation_count": 89
    }
  }
}
```

---

## PATCH /api/admin/operators/[id]

**説明**: オペレーター情報更新（管理者のみ）

**認証**: 必須（admin）

**リクエスト**:
```json
{
  "company_name": "〇〇不動産株式会社",
  "subscription_status": "suspended",
  "subscription_plan": "premium",
  "is_active": false
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "operator_id": "uuid",
    "message": "オペレーター情報を更新しました"
  }
}
```

---

## DELETE /api/admin/operators/[id]

**説明**: オペレーター削除（管理者のみ）

**認証**: 必須（admin）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "message": "オペレーターを削除しました"
  }
}
```

**注意**: オペレーター削除時は、関連する物件・顧客・会話データも CASCADE で削除されます。

---

## GET /api/admin/stats

**説明**: システム全体の統計情報取得（管理者のみ）

**認証**: 必須（admin）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "operators": {
      "total": 50,
      "active": 45,
      "suspended": 3,
      "cancelled": 2
    },
    "properties": {
      "total": 5000,
      "public": 4500,
      "by_operator": [
        { "operator_id": "uuid", "company_name": "〇〇不動産", "count": 150 }
      ]
    },
    "customers": {
      "total": 1200,
      "by_operator": [
        { "operator_id": "uuid", "company_name": "〇〇不動産", "count": 45 }
      ]
    },
    "conversations": {
      "total": 3500,
      "last_30_days": 450
    }
  }
}
```

---

## GET /api/admin/users

**説明**: ユーザー一覧取得（管理者のみ）

**認証**: 必須（admin）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": "uuid",
        "email": "user@example.com",
        "role": "customer",
        "created_at": "2025-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

## GET /api/admin/logs

**説明**: 取り込みログ一覧（管理者のみ）

**認証**: 必須（admin）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "logs": [
      {
        "id": "uuid",
        "property_id": "uuid",
        "source_url": "https://...",
        "imported_by": "uuid",
        "imported_at": "2025-01-01T00:00:00Z"
      }
    ]
  }
}
```
