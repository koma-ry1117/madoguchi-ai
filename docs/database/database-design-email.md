# データベース設計 - メール関連テーブル

> 本ドキュメントは [データベース設計](../database-design.md) の一部です。

## 12. email_logs（メール送信ログ）

```sql
CREATE TABLE email_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 送信先情報
  to_email VARCHAR(255) NOT NULL,
  to_name VARCHAR(100),

  -- メール内容
  from_email VARCHAR(255) NOT NULL DEFAULT 'noreply@madoguchi-ai.jp',
  from_name VARCHAR(100) DEFAULT 'madoguchi-ai',
  subject VARCHAR(255) NOT NULL,
  template_name VARCHAR(100), -- 使用したテンプレート名

  -- 関連情報
  operator_id UUID REFERENCES operators(id) ON DELETE CASCADE,
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
  viewing_id UUID REFERENCES viewings(id) ON DELETE SET NULL,
  inquiry_id UUID REFERENCES inquiries(id) ON DELETE SET NULL,

  -- メール種別
  email_type VARCHAR(50) NOT NULL CHECK (email_type IN (
    'viewing_confirmation',     -- 内見予約確定
    'viewing_reminder',         -- 内見リマインダー
    'viewing_cancellation',     -- 内見キャンセル
    'inquiry_confirmation',     -- 問い合わせ確認
    'inquiry_response',         -- 問い合わせ返信
    'system_announcement',      -- システムお知らせ
    'new_property_alert',       -- 新着物件通知
    'password_reset',           -- パスワードリセット
    'email_verification',       -- メール確認
    'other'                     -- その他
  )),

  -- 送信状態
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'failed', 'bounced')),
  resend_id VARCHAR(100), -- Resend APIのメッセージID
  error_message TEXT,

  -- メタ情報
  sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_email_logs_operator_id ON email_logs(operator_id);
CREATE INDEX idx_email_logs_to_email ON email_logs(to_email);
CREATE INDEX idx_email_logs_customer_id ON email_logs(customer_id);
CREATE INDEX idx_email_logs_email_type ON email_logs(email_type);
CREATE INDEX idx_email_logs_status ON email_logs(status);
CREATE INDEX idx_email_logs_sent_at ON email_logs(sent_at DESC);
CREATE INDEX idx_email_logs_created_at ON email_logs(created_at DESC);
```

---

## 13. email_suppressions（メール配信停止リスト）

Resendからのバウンス・苦情通知を受けて配信停止を管理するテーブル。

```sql
CREATE TABLE email_suppressions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 配信停止対象
  email VARCHAR(255) NOT NULL,
  operator_id UUID REFERENCES operators(id) ON DELETE CASCADE,
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,

  -- 配信停止理由
  reason VARCHAR(50) NOT NULL CHECK (reason IN (
    'hard_bounce',      -- 宛先不明（恒久的）
    'soft_bounce',      -- 一時的なエラー
    'spam_complaint',   -- スパム報告
    'unsubscribe',      -- 配信停止リクエスト
    'manual'            -- 手動で停止
  )),

  -- Resend Webhook情報
  resend_event_id VARCHAR(100),
  bounce_type VARCHAR(50),       -- permanent / transient
  bounce_category VARCHAR(100),  -- invalid_email, mailbox_full, etc.

  -- メタ情報
  suppressed_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ,        -- soft_bounceの場合、一定期間後に解除
  notes TEXT,

  UNIQUE(email, operator_id)
);

CREATE INDEX idx_email_suppressions_email ON email_suppressions(email);
CREATE INDEX idx_email_suppressions_operator_id ON email_suppressions(operator_id);
CREATE INDEX idx_email_suppressions_reason ON email_suppressions(reason);
CREATE INDEX idx_email_suppressions_suppressed_at ON email_suppressions(suppressed_at DESC);
```

---

## 14. marketing_campaigns（営業キャンペーン）

新着物件配信や営業メールのキャンペーン管理。

```sql
CREATE TABLE marketing_campaigns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 所属オペレーター
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,

  -- キャンペーン情報
  name VARCHAR(255) NOT NULL,
  description TEXT,
  campaign_type VARCHAR(50) NOT NULL CHECK (campaign_type IN (
    'new_property',         -- 新着物件通知
    'follow_up',            -- フォローアップ営業
    'reengagement',         -- 再エンゲージメント
    'seasonal',             -- 季節キャンペーン
    'custom'                -- カスタム
  )),

  -- 対象物件（新着物件通知の場合）
  property_ids UUID[],

  -- 配信条件（JSON）
  target_criteria JSONB DEFAULT '{}',
  -- 例: {"preferred_areas": ["文京区"], "budget_max": 150000, "last_active_days": 30}

  -- 配信状態
  status VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
    'draft',        -- 下書き
    'scheduled',    -- 配信予約済み
    'sending',      -- 配信中
    'completed',    -- 配信完了
    'cancelled'     -- キャンセル
  )),

  -- 配信スケジュール
  scheduled_at TIMESTAMPTZ,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,

  -- 配信統計
  total_recipients INTEGER DEFAULT 0,
  sent_count INTEGER DEFAULT 0,
  opened_count INTEGER DEFAULT 0,
  clicked_count INTEGER DEFAULT 0,
  bounced_count INTEGER DEFAULT 0,

  -- メタ情報
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_marketing_campaigns_operator_id ON marketing_campaigns(operator_id);
CREATE INDEX idx_marketing_campaigns_status ON marketing_campaigns(status);
CREATE INDEX idx_marketing_campaigns_campaign_type ON marketing_campaigns(campaign_type);
CREATE INDEX idx_marketing_campaigns_scheduled_at ON marketing_campaigns(scheduled_at);
```

---

## 15. marketing_campaign_recipients（キャンペーン配信対象）

キャンペーンごとの配信対象者と配信状態を管理。

```sql
CREATE TABLE marketing_campaign_recipients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- キャンペーン・顧客関連
  campaign_id UUID NOT NULL REFERENCES marketing_campaigns(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,

  -- 配信状態
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
    'pending',      -- 配信待ち
    'sent',         -- 送信済み
    'opened',       -- 開封済み
    'clicked',      -- クリック済み
    'bounced',      -- バウンス
    'skipped'       -- スキップ（配信停止など）
  )),
  skip_reason VARCHAR(100),  -- スキップ理由

  -- 配信情報
  email_log_id UUID REFERENCES email_logs(id),
  sent_at TIMESTAMPTZ,
  opened_at TIMESTAMPTZ,
  clicked_at TIMESTAMPTZ,

  -- メタ情報
  created_at TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE(campaign_id, customer_id)
);

CREATE INDEX idx_campaign_recipients_campaign_id ON marketing_campaign_recipients(campaign_id);
CREATE INDEX idx_campaign_recipients_customer_id ON marketing_campaign_recipients(customer_id);
CREATE INDEX idx_campaign_recipients_status ON marketing_campaign_recipients(status);
```

---

## 16. marketing_email_queue（営業メールキュー）

営業メールのリトライ管理用キュー。

```sql
CREATE TABLE marketing_email_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 関連情報
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  campaign_id UUID REFERENCES marketing_campaigns(id) ON DELETE SET NULL,

  -- メール内容
  email_type VARCHAR(50) NOT NULL,
  subject VARCHAR(255) NOT NULL,
  template_name VARCHAR(100),
  template_data JSONB DEFAULT '{}',

  -- キュー状態
  status VARCHAR(20) DEFAULT 'queued' CHECK (status IN (
    'queued',       -- キュー待ち
    'processing',   -- 処理中
    'sent',         -- 送信完了
    'failed',       -- 失敗
    'cancelled'     -- キャンセル
  )),

  -- リトライ管理
  retry_count INTEGER DEFAULT 0,
  max_retries INTEGER DEFAULT 3,
  next_retry_at TIMESTAMPTZ,
  last_error TEXT,

  -- 優先度（低い値が高優先度）
  priority INTEGER DEFAULT 100,

  -- メタ情報
  scheduled_at TIMESTAMPTZ DEFAULT NOW(),
  processed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_email_queue_operator_id ON marketing_email_queue(operator_id);
CREATE INDEX idx_email_queue_status ON marketing_email_queue(status);
CREATE INDEX idx_email_queue_next_retry_at ON marketing_email_queue(next_retry_at);
CREATE INDEX idx_email_queue_priority ON marketing_email_queue(priority, created_at);
```
