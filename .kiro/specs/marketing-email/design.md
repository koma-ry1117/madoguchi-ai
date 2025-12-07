# Design Document

## Overview

営業メール・キャンペーン機能の技術設計。オペレーターが新着物件を既存顧客に効率的に配信し、メール重複送信防止、Resend連携による配信停止管理、失敗メールのリトライ機能を実装する。

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Operator Dashboard                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
│  │  Campaign Form  │  │ Recipients List │  │ Suppression Manager │  │
│  └────────┬────────┘  └────────┬────────┘  └──────────┬──────────┘  │
└───────────┼─────────────────────┼─────────────────────┼─────────────┘
            │                     │                     │
            ▼                     ▼                     ▼
┌───────────────────────────────────────────────────────────────────────┐
│                          Next.js API Routes                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────┐   │
│  │ /marketing/      │  │ /marketing/      │  │ /webhooks/resend   │   │
│  │ campaigns        │  │ suppressions     │  │ (Webhook Handler)  │   │
│  └────────┬─────────┘  └────────┬─────────┘  └─────────┬──────────┘   │
└───────────┼──────────────────────┼──────────────────────┼─────────────┘
            │                      │                      │
            ▼                      ▼                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│                        Business Logic Layer                           │
│  ┌────────────────┐  ┌─────────────────┐  ┌────────────────────────┐  │
│  │ CampaignService│  │SuppressionSvc   │  │ EmailQueueService      │  │
│  │ - create       │  │ - check         │  │ - enqueue              │  │
│  │ - findRecipients│ │ - add           │  │ - process              │  │
│  │ - send         │  │ - remove        │  │ - retry                │  │
│  └───────┬────────┘  └────────┬────────┘  └───────────┬────────────┘  │
└──────────┼─────────────────────┼──────────────────────┼───────────────┘
           │                     │                      │
           ▼                     ▼                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         Supabase (PostgreSQL + RLS)                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │marketing_campaigns│ │email_suppressions │  │marketing_email_queue│  │
│  │  + recipients    │  │                  │  │                     │  │
│  └──────────────────┘  └──────────────────┘  └─────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────────────────────────────────────┐
│                              Resend                                   │
│  ┌──────────────────┐  ┌──────────────────┐                           │
│  │   Send Email     │  │   Webhook Events │                           │
│  │                  │  │   (bounce, spam) │                           │
│  └──────────────────┘  └──────────────────┘                           │
└───────────────────────────────────────────────────────────────────────┘
```

### Data Flow

#### 1. 営業推奨ユーザー検索フロー

```
1. オペレーターが物件追加後「営業推奨ユーザーを検索」ボタンをクリック
2. API: GET /api/marketing/campaigns/:id/recipients?preview=true
3. get_marketing_recipients() RPC呼び出し
   - customer_preferencesで物件条件マッチング
   - email_logsで直近N日以内の送信済みを除外
   - email_suppressionsで配信停止を除外
4. 検索結果サマリーを表示
   - 条件マッチ数
   - 除外数（送信済み、配信停止）
   - 最終配信対象数
5. オペレーターが配信対象を選択・確認
```

#### 2. キャンペーン配信フロー

```
1. オペレーターがキャンペーン作成（対象物件、配信条件設定）
2. プレビュー確認 → 配信実行
3. marketing_campaign_recipientsにレコード作成
4. marketing_email_queueにメールをエンキュー
5. バックグラウンドワーカー（Vercel Cron）がキューを処理
6. Resend APIでメール送信
7. email_logsに送信結果を記録
8. campaign統計を更新
```

#### 3. Resend Webhookフロー

```
1. Resendからバウンス/苦情イベント受信
2. POST /api/webhooks/resend
3. svix署名検証
4. イベントタイプ判定
   - hard_bounce → email_suppressionsに恒久追加
   - soft_bounce → email_suppressionsに一時追加（7日間）
   - spam_complaint → email_suppressionsに恒久追加
5. email_logsのステータス更新
```

#### 4. リトライフロー

```
1. 送信失敗時、自動的にmarketing_email_queueに再登録
2. retry_count++、next_retry_at設定（指数バックオフ）
   - 1回目: 30分後
   - 2回目: 1時間後
   - 3回目: 2時間後
3. max_retries(3)到達で最終失敗として記録
4. オペレーターに通知（ダッシュボード表示）
5. 手動リトライオプション提供
```

## Database Design

### New Tables

```sql
-- 1. email_suppressions（配信停止リスト）
CREATE TABLE email_suppressions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) NOT NULL,
  operator_id UUID REFERENCES operators(id) ON DELETE CASCADE,
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  reason VARCHAR(50) NOT NULL CHECK (reason IN (
    'hard_bounce', 'soft_bounce', 'spam_complaint', 'unsubscribe', 'manual'
  )),
  resend_event_id VARCHAR(100),
  bounce_type VARCHAR(50),
  bounce_category VARCHAR(100),
  suppressed_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ,  -- soft_bounceの場合のみ
  notes TEXT,
  UNIQUE(email, operator_id)
);

-- 2. marketing_campaigns（キャンペーン）
CREATE TABLE marketing_campaigns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  campaign_type VARCHAR(50) NOT NULL CHECK (campaign_type IN (
    'new_property', 'follow_up', 'reengagement', 'seasonal', 'custom'
  )),
  property_ids UUID[],
  target_criteria JSONB DEFAULT '{}',
  status VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
    'draft', 'scheduled', 'sending', 'completed', 'cancelled'
  )),
  scheduled_at TIMESTAMPTZ,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  total_recipients INTEGER DEFAULT 0,
  sent_count INTEGER DEFAULT 0,
  opened_count INTEGER DEFAULT 0,
  clicked_count INTEGER DEFAULT 0,
  bounced_count INTEGER DEFAULT 0,
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. marketing_campaign_recipients（配信対象者）
CREATE TABLE marketing_campaign_recipients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id UUID NOT NULL REFERENCES marketing_campaigns(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
    'pending', 'sent', 'opened', 'clicked', 'bounced', 'skipped'
  )),
  skip_reason VARCHAR(100),
  email_log_id UUID REFERENCES email_logs(id),
  sent_at TIMESTAMPTZ,
  opened_at TIMESTAMPTZ,
  clicked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(campaign_id, customer_id)
);

-- 4. marketing_email_queue（メールキュー）
CREATE TABLE marketing_email_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  campaign_id UUID REFERENCES marketing_campaigns(id) ON DELETE SET NULL,
  email_type VARCHAR(50) NOT NULL,
  subject VARCHAR(255) NOT NULL,
  template_name VARCHAR(100),
  template_data JSONB DEFAULT '{}',
  status VARCHAR(20) DEFAULT 'queued' CHECK (status IN (
    'queued', 'processing', 'sent', 'failed', 'cancelled'
  )),
  retry_count INTEGER DEFAULT 0,
  max_retries INTEGER DEFAULT 3,
  next_retry_at TIMESTAMPTZ,
  last_error TEXT,
  priority INTEGER DEFAULT 100,
  scheduled_at TIMESTAMPTZ DEFAULT NOW(),
  processed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### RPC Functions

```sql
-- 営業推奨ユーザー検索（詳細マッチングロジック）
CREATE OR REPLACE FUNCTION get_marketing_recipients(
  p_operator_id UUID,
  p_property_ids UUID[],
  p_target_criteria JSONB,
  p_exclude_recent_days INTEGER DEFAULT 7
)
RETURNS TABLE (
  customer_id UUID,
  name VARCHAR,
  email VARCHAR,
  preferred_areas TEXT[],
  budget_max INTEGER,
  last_email_sent_at TIMESTAMPTZ,
  is_suppressed BOOLEAN,
  match_score FLOAT,
  match_reasons TEXT[]
) AS $$
DECLARE
  v_target_areas TEXT[];
  v_target_layouts TEXT[];
  v_target_building_types TEXT[];
  v_target_budget_max INTEGER;
BEGIN
  -- ターゲット条件の抽出
  v_target_areas := ARRAY(SELECT jsonb_array_elements_text(p_target_criteria->'preferred_areas'));
  v_target_layouts := ARRAY(SELECT jsonb_array_elements_text(p_target_criteria->'layouts'));
  v_target_building_types := ARRAY(SELECT jsonb_array_elements_text(p_target_criteria->'building_types'));
  v_target_budget_max := (p_target_criteria->>'budget_max')::INTEGER;

  RETURN QUERY
  WITH matching_customers AS (
    SELECT
      c.id,
      c.name,
      c.email,
      cp.preferred_areas,
      cp.rent_max as budget_max,
      cp.preferred_layouts,
      cp.preferred_building_types,
      -- 詳細マッチスコア計算（各条件を個別に評価）
      (
        -- エリアマッチ: 配列の重複要素数に応じてスコア付与
        CASE
          WHEN v_target_areas IS NULL OR array_length(v_target_areas, 1) IS NULL THEN 0
          WHEN cp.preferred_areas IS NULL THEN 0
          ELSE (
            SELECT COUNT(*)::FLOAT / GREATEST(array_length(v_target_areas, 1), 1)
            FROM unnest(cp.preferred_areas) AS pref_area
            WHERE pref_area = ANY(v_target_areas)
          ) * 0.4
        END +
        -- 予算マッチ: 顧客予算が物件価格以上ならマッチ
        CASE
          WHEN v_target_budget_max IS NULL THEN 0
          WHEN cp.rent_max IS NULL THEN 0.1  -- 予算未設定の場合は低スコア
          WHEN cp.rent_max >= v_target_budget_max THEN 0.3
          WHEN cp.rent_max >= v_target_budget_max * 0.8 THEN 0.15  -- 80%以上なら部分マッチ
          ELSE 0
        END +
        -- 間取りマッチ
        CASE
          WHEN v_target_layouts IS NULL OR array_length(v_target_layouts, 1) IS NULL THEN 0
          WHEN cp.preferred_layouts IS NULL THEN 0
          WHEN cp.preferred_layouts && v_target_layouts THEN 0.2
          ELSE 0
        END +
        -- 建物種別マッチ
        CASE
          WHEN v_target_building_types IS NULL OR array_length(v_target_building_types, 1) IS NULL THEN 0
          WHEN cp.preferred_building_types IS NULL THEN 0
          WHEN cp.preferred_building_types && v_target_building_types THEN 0.1
          ELSE 0
        END
      ) AS match_score,
      -- マッチ理由の配列
      ARRAY_REMOVE(ARRAY[
        CASE WHEN cp.preferred_areas && v_target_areas THEN 'area_match' END,
        CASE WHEN cp.rent_max >= v_target_budget_max THEN 'budget_match' END,
        CASE WHEN cp.preferred_layouts && v_target_layouts THEN 'layout_match' END,
        CASE WHEN cp.preferred_building_types && v_target_building_types THEN 'building_type_match' END
      ], NULL) AS match_reasons
    FROM customers c
    LEFT JOIN customer_preferences cp ON c.id = cp.customer_id
    WHERE c.operator_id = p_operator_id
      AND c.email IS NOT NULL  -- メールアドレス必須
  ),
  last_emails AS (
    SELECT
      el.customer_id,
      MAX(el.sent_at) as last_sent_at
    FROM email_logs el
    WHERE el.operator_id = p_operator_id
      AND el.sent_at > NOW() - (p_exclude_recent_days || ' days')::interval
    GROUP BY el.customer_id
  ),
  suppressions AS (
    SELECT email
    FROM email_suppressions
    WHERE operator_id = p_operator_id
      AND (expires_at IS NULL OR expires_at > NOW())
  )
  SELECT
    mc.id,
    mc.name,
    mc.email,
    mc.preferred_areas,
    mc.budget_max,
    le.last_sent_at,
    (s.email IS NOT NULL) as is_suppressed,
    mc.match_score,
    mc.match_reasons
  FROM matching_customers mc
  LEFT JOIN last_emails le ON mc.id = le.customer_id
  LEFT JOIN suppressions s ON mc.email = s.email
  WHERE le.last_sent_at IS NULL  -- 直近N日以内に送信済みを除外
    AND s.email IS NULL           -- 配信停止を除外
    AND mc.match_score > 0        -- スコア0は除外（条件にマッチしない）
  ORDER BY mc.match_score DESC;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**マッチスコア計算ロジック**
| 条件 | 最大スコア | 説明 |
|------|-----------|------|
| エリア | 0.4 | 重複エリア数 / 対象エリア数 × 0.4 |
| 予算 | 0.3 | 完全マッチ=0.3、80%以上=0.15 |
| 間取り | 0.2 | 配列の重複があれば0.2 |
| 建物種別 | 0.1 | 配列の重複があれば0.1 |
| **合計** | **1.0** | |

## API Design

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/marketing/campaigns | キャンペーン作成 |
| GET | /api/marketing/campaigns/:id | キャンペーン詳細 |
| GET | /api/marketing/campaigns/:id/recipients | 配信対象者検索 |
| POST | /api/marketing/campaigns/:id/send | 配信実行 |
| POST | /api/marketing/retry | 失敗メールリトライ |
| GET | /api/marketing/suppressions | 配信停止リスト取得 |
| POST | /api/marketing/suppressions | 手動配信停止追加 |
| POST | /api/webhooks/resend | Resend Webhook受信 |

### Request/Response Examples

#### GET /api/marketing/campaigns/:id/recipients

```typescript
// Request Query
interface RecipientsQuery {
  preview?: boolean;
  exclude_recent_days?: number;  // default: 7
}

// Response
interface RecipientsResponse {
  success: true;
  data: {
    recipients: {
      customer_id: string;
      name: string;
      email: string;
      preferred_areas: string[];
      budget_max: number;
      last_email_sent_at: string | null;
      is_suppressed: boolean;
      match_score: number;
    }[];
    summary: {
      total_matched: number;
      excluded_recent_email: number;
      excluded_suppressed: number;
      final_recipients: number;
    };
  };
}
```

#### POST /api/webhooks/resend

```typescript
// Resend Event Payload
interface ResendWebhookEvent {
  type: 'email.bounced' | 'email.complained' | 'email.delivered' | 'email.opened';
  data: {
    email_id: string;
    to: string[];
    bounce?: {
      type: 'permanent' | 'transient';
      category: string;
    };
  };
}
```

## Component Design

### Server Components

```
src/
├── actions/
│   └── marketing.ts              # Server Actions
├── lib/
│   ├── services/
│   │   ├── campaign-service.ts   # キャンペーン管理
│   │   ├── suppression-service.ts # 配信停止管理
│   │   └── email-queue-service.ts # キュー処理
│   └── resend/
│       └── webhook-handler.ts    # Webhook処理
```

### Client Components

```
src/components/features/marketing/
├── CampaignForm.tsx              # キャンペーン作成フォーム
├── RecipientsList.tsx            # 配信対象者一覧
├── RecipientsSearchButton.tsx    # 営業推奨ユーザー検索ボタン
├── RecipientsSummary.tsx         # 検索結果サマリー
├── SuppressionList.tsx           # 配信停止リスト
└── CampaignStats.tsx             # キャンペーン統計
```

## Security Considerations

### RLS Policies

```sql
-- email_suppressions: オペレーター自社データのみ
CREATE POLICY "operator_own_suppressions"
ON email_suppressions FOR ALL
TO authenticated
USING (
  operator_id IN (SELECT id FROM operators WHERE user_id = auth.uid())
);

-- marketing_campaigns: オペレーター自社データのみ
CREATE POLICY "operator_own_campaigns"
ON marketing_campaigns FOR ALL
TO authenticated
USING (
  operator_id IN (SELECT id FROM operators WHERE user_id = auth.uid())
);
```

### Webhook Security

- Resend Webhook署名検証（svix）
- IP制限（Resend IPs）
- イベントIDの重複処理防止

```typescript
import { Webhook } from 'svix';

const wh = new Webhook(process.env.RESEND_WEBHOOK_SECRET!);

export async function verifyWebhook(payload: string, headers: Headers) {
  return wh.verify(payload, {
    'svix-id': headers.get('svix-id')!,
    'svix-timestamp': headers.get('svix-timestamp')!,
    'svix-signature': headers.get('svix-signature')!,
  });
}
```

## Testing Strategy

### Unit Tests

- CampaignService: create, findRecipients, send
- SuppressionService: check, add, remove
- EmailQueueService: enqueue, process, retry

### Integration Tests

- 配信対象者検索（除外ロジック）
- Webhook処理（バウンス、苦情）
- リトライ処理（指数バックオフ）

### E2E Tests

- キャンペーン作成 → 配信 → 統計更新
- Webhook → 配信停止 → 次回配信で除外

## Performance Considerations

### Indexes

```sql
CREATE INDEX idx_email_suppressions_operator_email ON email_suppressions(operator_id, email);
CREATE INDEX idx_email_suppressions_expires_at ON email_suppressions(expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_marketing_email_queue_status ON marketing_email_queue(status, next_retry_at);
CREATE INDEX idx_email_logs_customer_sent ON email_logs(customer_id, sent_at DESC);
```

### Batch Processing

- メール送信は100件/バッチで処理
- キュー処理はVercel Cron（1分間隔）
- 大量配信時はレート制限考慮（Resend: 100/sec）

## Dependencies

- resend: ^4.0.0 - メール送信
- svix: ^1.0.0 - Webhook署名検証
- ics: ^3.0.0 - カレンダー招待生成（optional）
