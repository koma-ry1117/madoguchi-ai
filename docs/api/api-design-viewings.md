# 内見予約API

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## POST /api/viewings

**説明**: 内見予約の作成

**認証**: 必須

**リクエスト**:
```json
{
  "property_id": "uuid",
  "viewing_date": "2025-11-20T14:00:00+09:00",
  "customer_notes": "駐車場の有無を確認したいです"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "viewing_id": "uuid",
    "property_id": "uuid",
    "viewing_date": "2025-11-20T14:00:00+09:00",
    "status": "confirmed",
    "message": "内見予約が完了しました。確認メールを送信しました。"
  }
}
```

**実装例**:
```typescript
// app/api/viewings/route.ts
export async function POST(request: Request) {
  const { property_id, viewing_date, customer_notes } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // 認証チェック
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // 顧客情報取得
  const { data: customer } = await supabase
    .from('customers')
    .select('*')
    .eq('user_id', user.id)
    .single();

  // 内見予約作成
  const { data: viewing, error } = await supabase
    .from('viewings')
    .insert({
      customer_id: customer.id,
      property_id,
      viewing_date,
      customer_notes,
      status: 'confirmed',
    })
    .select()
    .single();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  // メール送信（非同期）
  await sendViewingConfirmationEmail(viewing);

  return NextResponse.json({
    success: true,
    data: viewing,
  });
}
```

---

## GET /api/viewings

**説明**: 内見予約一覧取得（自分の予約のみ）

**認証**: 必須

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| status | string | ステータスフィルター |
| from_date | string | 開始日（ISO 8601） |
| to_date | string | 終了日（ISO 8601） |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "viewings": [
      {
        "id": "uuid",
        "property": {
          "id": "uuid",
          "title": "文京区本郷 2LDK",
          "address": "東京都文京区本郷1-2-3"
        },
        "viewing_date": "2025-11-20T14:00:00+09:00",
        "status": "confirmed",
        "customer_notes": "..."
      }
    ]
  }
}
```

---

## PATCH /api/viewings/[id]

**説明**: 内見予約の更新・キャンセル

**認証**: 必須

**リクエスト**:
```json
{
  "status": "cancelled",
  "cancellation_reason": "都合がつかなくなったため"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "viewing_id": "uuid",
    "status": "cancelled",
    "message": "内見予約をキャンセルしました。"
  }
}
```
