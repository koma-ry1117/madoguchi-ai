# Implementation Tasks

## Task 1: データベースマイグレーション作成

### Description
営業メール機能に必要な4つの新規テーブルとRLSポリシーを作成する。

### Files to Create/Modify
- `supabase/migrations/YYYYMMDD_add_marketing_email_tables.sql` (create)

### Implementation Details

1. **email_suppressions テーブル作成**
   - 配信停止理由（hard_bounce, soft_bounce, spam_complaint, unsubscribe, manual）
   - Resendイベント情報（event_id, bounce_type, bounce_category）
   - 有効期限（soft_bounceの一時停止用）
   - UNIQUE(email, operator_id)

2. **marketing_campaigns テーブル作成**
   - キャンペーンタイプ（new_property, follow_up, reengagement等）
   - 対象物件IDs（UUID配列）
   - 配信条件（JSONB）
   - ステータス（draft, scheduled, sending, completed, cancelled）
   - 配信統計（sent_count, opened_count, clicked_count, bounced_count）

3. **marketing_campaign_recipients テーブル作成**
   - キャンペーン-顧客の紐付け
   - 配信ステータス追跡
   - UNIQUE(campaign_id, customer_id)

4. **marketing_email_queue テーブル作成**
   - リトライ管理（retry_count, max_retries, next_retry_at）
   - 優先度制御
   - エラーログ

5. **RLSポリシー作成**
   - 各テーブルにオペレーター自社データのみアクセス可能なポリシー
   - admin用の全データアクセスポリシー

6. **インデックス作成**
   - 検索パフォーマンス最適化

### Acceptance Criteria
- [ ] 全テーブルが正常に作成される
- [ ] RLSが有効でポリシーが適用される
- [ ] ローカル環境でマイグレーションが成功する

---

## Task 2: 配信停止管理サービス実装

### Description
配信停止リストの管理ロジックを実装する。

### Files to Create/Modify
- `src/lib/services/suppression-service.ts` (create)
- `src/schemas/suppression.ts` (create)

### Implementation Details

1. **SuppressionService クラス作成**
   ```typescript
   class SuppressionService {
     async checkSuppression(email: string, operatorId: string): Promise<boolean>
     async addSuppression(data: AddSuppressionInput): Promise<void>
     async removeSuppression(id: string): Promise<void>
     async listSuppressions(operatorId: string, filters?: SuppressionFilters): Promise<Suppression[]>
     async cleanupExpiredSuppressions(): Promise<number>
   }
   ```

2. **Zodスキーマ定義**
   - AddSuppressionInput
   - SuppressionFilters
   - Suppression

### Acceptance Criteria
- [ ] hard_bounce/spam_complaintは恒久的に配信停止
- [ ] soft_bounceは7日間の一時停止
- [ ] 期限切れのsoft_bounceが自動解除される
- [ ] オペレーター別にデータが分離される

---

## Task 3: メールキューサービス実装

### Description
メール送信キューとリトライ処理を実装する。

### Files to Create/Modify
- `src/lib/services/email-queue-service.ts` (create)
- `src/schemas/email-queue.ts` (create)

### Implementation Details

1. **EmailQueueService クラス作成**
   ```typescript
   class EmailQueueService {
     async enqueue(data: EnqueueEmailInput): Promise<string>
     async processQueue(batchSize?: number): Promise<ProcessResult>
     async retry(emailLogId: string): Promise<void>
     async scheduleRetry(queueId: string, error: string): Promise<void>
     async getFailedEmails(operatorId: string): Promise<FailedEmail[]>
   }
   ```

2. **リトライロジック**
   - 最大3回リトライ
   - 指数バックオフ（30分、1時間、2時間）
   - 3回失敗で最終失敗として記録

3. **バッチ処理**
   - 100件/バッチ
   - Resendレート制限考慮

### Acceptance Criteria
- [ ] メールがキューに登録される
- [ ] 失敗時に自動リトライがスケジュールされる
- [ ] 3回失敗で最終失敗になる
- [ ] 手動リトライが可能

---

## Task 4: キャンペーンサービス実装

### Description
キャンペーン作成と配信対象者検索を実装する。

### Files to Create/Modify
- `src/lib/services/campaign-service.ts` (create)
- `src/schemas/campaign.ts` (create)

### Implementation Details

1. **CampaignService クラス作成**
   ```typescript
   class CampaignService {
     async create(data: CreateCampaignInput): Promise<Campaign>
     async findRecipients(campaignId: string, options?: RecipientOptions): Promise<RecipientsResult>
     async send(campaignId: string, recipientIds?: string[]): Promise<SendResult>
     async updateStats(campaignId: string): Promise<void>
     async getCampaignDetails(campaignId: string): Promise<CampaignWithStats>
   }
   ```

2. **配信対象者検索**
   - customer_preferencesで物件条件マッチング
   - 直近N日以内の送信済み除外（デフォルト7日）
   - 配信停止リスト除外
   - サマリー計算（マッチ数、除外数、最終対象数）

3. **RPC関数呼び出し**
   - get_marketing_recipients()

### Acceptance Criteria
- [ ] キャンペーンが作成できる
- [ ] 配信対象者が正しく検索される
- [ ] 除外ロジックが正しく動作する
- [ ] サマリーが正しく計算される

---

## Task 5: get_marketing_recipients RPC関数作成

### Description
配信対象者検索用のPostgreSQL関数を作成する。

### Files to Create/Modify
- `supabase/migrations/YYYYMMDD_add_marketing_recipients_function.sql` (create)

### Implementation Details

1. **RPC関数定義**
   - パラメータ: operator_id, property_ids, target_criteria, exclude_recent_days
   - マッチスコア計算
   - 除外条件適用
   - 結果ソート

2. **SECURITY DEFINER設定**
   - RLSバイパスで内部処理

### Acceptance Criteria
- [ ] 関数が正常に動作する
- [ ] マッチスコアが計算される
- [ ] 除外条件が適用される
- [ ] パフォーマンスが許容範囲内

---

## Task 6: Resend Webhook APIルート実装

### Description
Resendからのバウンス・苦情通知を受信するWebhookエンドポイントを実装する。

### Files to Create/Modify
- `app/api/webhooks/resend/route.ts` (create)
- `src/lib/resend/webhook-handler.ts` (create)

### Implementation Details

1. **署名検証**
   ```typescript
   import { Webhook } from 'svix';

   const wh = new Webhook(process.env.RESEND_WEBHOOK_SECRET!);
   ```

2. **イベントハンドラー**
   - email.bounced → handleBounce()
   - email.complained → handleComplaint()
   - email.delivered → updateEmailLog()
   - email.opened → updateEmailLog()

3. **配信停止追加ロジック**
   - hard_bounce: 恒久停止
   - soft_bounce: 7日間停止
   - spam_complaint: 恒久停止

### Acceptance Criteria
- [ ] 署名検証が正しく動作する
- [ ] バウンスイベントで配信停止が追加される
- [ ] 苦情イベントで配信停止が追加される
- [ ] email_logsのステータスが更新される

---

## Task 7: キャンペーンAPIルート実装

### Description
キャンペーン管理用のAPIエンドポイントを実装する。

### Files to Create/Modify
- `app/api/marketing/campaigns/route.ts` (create)
- `app/api/marketing/campaigns/[id]/route.ts` (create)
- `app/api/marketing/campaigns/[id]/recipients/route.ts` (create)
- `app/api/marketing/campaigns/[id]/send/route.ts` (create)

### Implementation Details

1. **POST /api/marketing/campaigns**
   - キャンペーン作成
   - operator認証必須

2. **GET /api/marketing/campaigns/:id**
   - キャンペーン詳細取得

3. **GET /api/marketing/campaigns/:id/recipients**
   - 配信対象者検索
   - preview=trueでプレビューのみ
   - exclude_recent_days パラメータ

4. **POST /api/marketing/campaigns/:id/send**
   - 配信実行
   - recipient_ids指定可能

### Acceptance Criteria
- [ ] 認証・認可が正しく動作する
- [ ] キャンペーンCRUDが動作する
- [ ] 配信対象者検索が動作する
- [ ] 配信実行がキューに登録される

---

## Task 8: 配信停止APIルート実装

### Description
配信停止リスト管理用のAPIエンドポイントを実装する。

### Files to Create/Modify
- `app/api/marketing/suppressions/route.ts` (create)

### Implementation Details

1. **GET /api/marketing/suppressions**
   - 配信停止リスト取得
   - 理由でフィルタリング
   - ページネーション

2. **POST /api/marketing/suppressions**
   - 手動配信停止追加

### Acceptance Criteria
- [ ] 配信停止リストが取得できる
- [ ] 理由でフィルタリングできる
- [ ] 手動で配信停止を追加できる

---

## Task 9: リトライAPIルート実装

### Description
失敗メールのリトライ用APIエンドポイントを実装する。

### Files to Create/Modify
- `app/api/marketing/retry/route.ts` (create)

### Implementation Details

1. **POST /api/marketing/retry**
   - email_log_ids配列でリトライ指定
   - delay_minutesでスケジュール

### Acceptance Criteria
- [ ] 失敗メールを手動でリトライできる
- [ ] 遅延時間を指定できる
- [ ] 配信停止アドレスはスキップされる

---

## Task 10: キューワーカー実装（Vercel Cron）

### Description
メールキューを処理するCronジョブを実装する。

### Files to Create/Modify
- `app/api/cron/process-email-queue/route.ts` (create)
- `vercel.json` (modify)

### Implementation Details

1. **Cronジョブ設定**
   ```json
   {
     "crons": [{
       "path": "/api/cron/process-email-queue",
       "schedule": "* * * * *"
     }]
   }
   ```

2. **キュー処理ロジック**
   - queuedステータスのメールを取得
   - next_retry_at <= NOW()のものを処理
   - バッチ処理（100件）

3. **soft_bounce期限切れ処理**
   - expires_at <= NOW()の配信停止を解除

### Acceptance Criteria
- [ ] Cronジョブが1分ごとに実行される
- [ ] キュー内のメールが処理される
- [ ] 期限切れの配信停止が解除される

---

## Task 11: フロントエンドコンポーネント実装

### Description
営業メール機能のUIコンポーネントを実装する。

### Files to Create/Modify
- `src/components/features/marketing/CampaignForm.tsx` (create)
- `src/components/features/marketing/RecipientsList.tsx` (create)
- `src/components/features/marketing/RecipientsSearchButton.tsx` (create)
- `src/components/features/marketing/RecipientsSummary.tsx` (create)
- `src/components/features/marketing/SuppressionList.tsx` (create)
- `src/components/features/marketing/CampaignStats.tsx` (create)

### Implementation Details

1. **RecipientsSearchButton**
   - 物件追加後に表示
   - クリックで配信対象者検索

2. **RecipientsSummary**
   - 条件マッチ数
   - 除外数（送信済み、配信停止）
   - 最終配信対象数

3. **RecipientsList**
   - チェックボックスで選択
   - 配信対象者の詳細表示

4. **CampaignForm**
   - キャンペーン名、タイプ
   - 対象物件選択
   - 配信条件設定

5. **SuppressionList**
   - 配信停止理由でフィルタ
   - 手動追加ボタン

### Acceptance Criteria
- [ ] 各コンポーネントが正しくレンダリングされる
- [ ] 検索ボタンで配信対象者が検索される
- [ ] サマリーが正しく表示される
- [ ] 配信対象を選択できる

---

## Task 12: オペレーターダッシュボードページ実装

### Description
営業メール機能のページを実装する。

### Files to Create/Modify
- `app/(operator)/marketing/campaigns/page.tsx` (create)
- `app/(operator)/marketing/campaigns/new/page.tsx` (create)
- `app/(operator)/marketing/campaigns/[id]/page.tsx` (create)
- `app/(operator)/marketing/suppressions/page.tsx` (create)

### Implementation Details

1. **キャンペーン一覧ページ**
   - ステータス別フィルタ
   - 統計表示

2. **キャンペーン作成ページ**
   - CampaignFormコンポーネント
   - プレビュー機能

3. **キャンペーン詳細ページ**
   - 配信状況
   - 統計グラフ

4. **配信停止リストページ**
   - SuppressionListコンポーネント

### Acceptance Criteria
- [ ] ページが正しく表示される
- [ ] 認証が必要
- [ ] 自社データのみ表示される

---

## Task 13: テスト実装

### Description
営業メール機能のテストを実装する。

### Files to Create/Modify
- `src/lib/services/__tests__/suppression-service.test.ts` (create)
- `src/lib/services/__tests__/email-queue-service.test.ts` (create)
- `src/lib/services/__tests__/campaign-service.test.ts` (create)
- `app/api/webhooks/resend/__tests__/route.test.ts` (create)

### Implementation Details

1. **単体テスト**
   - 各サービスのメソッド
   - エッジケース

2. **統合テスト**
   - Webhook処理
   - リトライロジック

### Acceptance Criteria
- [ ] テストがパスする
- [ ] カバレッジ80%以上

---

## Task 14: email_logs テーブルへのemail_type追加

### Description
既存のemail_logsテーブルにマーケティングメール用のemail_typeを追加する。

### Files to Create/Modify
- `supabase/migrations/YYYYMMDD_update_email_logs_types.sql` (create)

### Implementation Details

1. **CHECK制約更新**
   - 'marketing_campaign' タイプ追加
   - 'follow_up' タイプ追加

### Acceptance Criteria
- [ ] 新しいemail_typeが使用可能
- [ ] 既存データに影響なし
