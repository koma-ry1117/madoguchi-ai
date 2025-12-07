# メール送信・営業メールAPI

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## メール送信API

### POST /api/emails/send

**説明**: メール送信（内部API）

**認証**: 必須（operator or admin）

**リクエスト**:
```json
{
  "email_type": "viewing_confirmation",
  "to_email": "customer@example.com",
  "to_name": "山田太郎",
  "template_data": {
    "customer_name": "山田太郎",
    "property_title": "文京区本郷 2LDK",
    "viewing_date": "2025年11月20日 14:00",
    "property_url": "https://madoguchi-ai.jp/properties/uuid"
  },
  "viewing_id": "uuid",
  "include_calendar": true
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "email_log_id": "uuid",
    "resend_id": "...",
    "status": "sent",
    "sent_at": "2025-11-15T12:00:00Z"
  }
}
```

**実装例**:
```typescript
// app/api/emails/send/route.ts
import { Resend } from 'resend';
import { ViewingConfirmationEmail } from '@/emails/viewing-confirmation';

const resend = new Resend(process.env.RESEND_API_KEY);

export async function POST(request: Request) {
  const {
    email_type,
    to_email,
    to_name,
    template_data,
    viewing_id,
    include_calendar,
  } = await request.json();

  // 認証・権限チェック
  // ...

  let attachments = [];

  // カレンダー招待添付
  if (include_calendar && viewing_id) {
    const { data: viewing } = await supabase
      .from('viewings')
      .select('*, property:properties(*)')
      .eq('id', viewing_id)
      .single();

    const calendar = generateICS(viewing);
    attachments.push({
      filename: 'viewing.ics',
      content: calendar.toString(),
    });
  }

  // メール送信
  const { data, error } = await resend.emails.send({
    from: 'madoguchi-ai <noreply@madoguchi-ai.jp>',
    to: to_email,
    subject: getEmailSubject(email_type),
    react: ViewingConfirmationEmail(template_data),
    attachments,
  });

  if (error) {
    // エラーログ保存
    await supabase.from('email_logs').insert({
      email_type,
      to_email,
      to_name,
      status: 'failed',
      error_message: error.message,
    });

    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  // 送信ログ保存
  const { data: emailLog } = await supabase
    .from('email_logs')
    .insert({
      email_type,
      to_email,
      to_name,
      subject: getEmailSubject(email_type),
      template_name: 'viewing_confirmation',
      viewing_id,
      status: 'sent',
      resend_id: data.id,
      sent_at: new Date().toISOString(),
    })
    .select()
    .single();

  return NextResponse.json({
    success: true,
    data: emailLog,
  });
}
```

---

### GET /api/emails/logs

**説明**: メール送信ログ取得（管理者のみ）

**認証**: 必須（admin）

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| email_type | string | メール種別 |
| status | string | 送信状態 |
| from_date | string | 開始日 |
| to_date | string | 終了日 |
| limit | number | 取得件数 |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "email_logs": [
      {
        "id": "uuid",
        "email_type": "viewing_confirmation",
        "to_email": "customer@example.com",
        "subject": "内見予約が確定しました",
        "status": "sent",
        "sent_at": "2025-11-15T12:00:00Z"
      }
    ],
    "pagination": {
      "total": 150,
      "page": 1,
      "limit": 20
    }
  }
}
```

---

## 営業メール・キャンペーンAPI

### POST /api/marketing/campaigns

**説明**: 新着物件配信キャンペーン作成

**認証**: 必須（operator）

**リクエストボディ**:
```json
{
  "name": "文京区新着物件 2025年1月",
  "campaign_type": "new_property",
  "property_ids": ["uuid1", "uuid2"],
  "target_criteria": {
    "preferred_areas": ["文京区", "千代田区"],
    "budget_max": 200000,
    "last_active_days": 90
  },
  "scheduled_at": "2025-01-20T10:00:00Z"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "campaign_id": "uuid",
    "name": "文京区新着物件 2025年1月",
    "status": "draft",
    "estimated_recipients": 45
  }
}
```

---

### GET /api/marketing/campaigns/:id/recipients

**説明**: キャンペーン配信対象者の検索・プレビュー（営業推奨ユーザー検索）

**認証**: 必須（operator）

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| preview | boolean | true: 配信対象者のプレビューのみ |
| exclude_recent_days | number | 直近N日以内にメール送信済みの顧客を除外（デフォルト: 7） |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "recipients": [
      {
        "customer_id": "uuid",
        "name": "山田太郎",
        "email": "yamada@example.com",
        "preferred_areas": ["文京区"],
        "budget_max": 150000,
        "last_activity_at": "2025-01-10T12:00:00Z",
        "last_email_sent_at": "2025-01-05T10:00:00Z",
        "is_suppressed": false,
        "match_score": 0.85
      }
    ],
    "summary": {
      "total_matched": 45,
      "excluded_recent_email": 12,
      "excluded_suppressed": 3,
      "final_recipients": 30
    }
  }
}
```

---

### POST /api/marketing/campaigns/:id/send

**説明**: キャンペーン配信実行

**認証**: 必須（operator）

**リクエストボディ**:
```json
{
  "send_now": true,
  "recipient_ids": ["uuid1", "uuid2"]
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "campaign_id": "uuid",
    "status": "sending",
    "queued_count": 30
  }
}
```

---

### POST /api/marketing/retry

**説明**: 失敗したメールのリトライ

**認証**: 必須（operator）

**リクエストボディ**:
```json
{
  "email_log_ids": ["uuid1", "uuid2"],
  "delay_minutes": 30
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "queued_count": 2,
    "skipped_count": 0,
    "skipped_reasons": []
  }
}
```

---

### GET /api/marketing/suppressions

**説明**: 配信停止リスト取得

**認証**: 必須（operator）

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| reason | string | 停止理由フィルター |
| page | number | ページ番号 |
| limit | number | 取得件数 |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "suppressions": [
      {
        "id": "uuid",
        "email": "bounced@example.com",
        "customer_name": "佐藤花子",
        "reason": "hard_bounce",
        "bounce_category": "invalid_email",
        "suppressed_at": "2025-01-10T12:00:00Z"
      }
    ],
    "pagination": {
      "total": 15,
      "page": 1,
      "limit": 20
    }
  }
}
```

---

### POST /api/webhooks/resend

**説明**: Resend Webhook受信（バウンス・苦情通知）

**認証**: Resend署名検証

**Resendからのイベント例**:
```json
{
  "type": "email.bounced",
  "data": {
    "email_id": "resend_email_id",
    "to": ["bounced@example.com"],
    "bounce": {
      "type": "permanent",
      "category": "invalid_email"
    }
  }
}
```

**処理内容**:
1. 署名検証
2. イベントタイプに応じた処理
   - `email.bounced` → `email_suppressions`に追加
   - `email.complained` → `email_suppressions`に追加（spam_complaint）
3. `email_logs`のステータス更新

**実装例**:
```typescript
// app/api/webhooks/resend/route.ts
import { Webhook } from 'svix';

export async function POST(request: Request) {
  const payload = await request.text();
  const headers = Object.fromEntries(request.headers);

  // 署名検証
  const wh = new Webhook(process.env.RESEND_WEBHOOK_SECRET!);
  const event = wh.verify(payload, headers);

  switch (event.type) {
    case 'email.bounced':
      await handleBounce(event.data);
      break;
    case 'email.complained':
      await handleComplaint(event.data);
      break;
  }

  return new Response('OK', { status: 200 });
}

async function handleBounce(data: any) {
  const { to, bounce } = data;

  for (const email of to) {
    // 配信停止リストに追加
    await supabase.from('email_suppressions').upsert({
      email,
      reason: bounce.type === 'permanent' ? 'hard_bounce' : 'soft_bounce',
      bounce_type: bounce.type,
      bounce_category: bounce.category,
      resend_event_id: data.email_id,
      // soft_bounceは7日後に解除
      expires_at: bounce.type === 'transient'
        ? new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString()
        : null,
    }, {
      onConflict: 'email,operator_id',
    });

    // 対応するemail_logsを更新
    await supabase
      .from('email_logs')
      .update({ status: 'bounced' })
      .eq('resend_id', data.email_id);
  }
}
```
