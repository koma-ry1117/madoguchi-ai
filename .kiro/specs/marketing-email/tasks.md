# Implementation Tasks

## Task 1: データベーステーブル作成

_Requirements: 1.1, 2.1, 3.1, 4.1, 5.1_

### Description
営業メール機能に必要な4つのテーブルを作成する

### Files to Create/Modify
- `supabase/migrations/20250101000001_create_marketing_tables.sql` (create)

### Implementation Details

#### 1.1 email_suppressions テーブル作成（1-1.5時間）
1. カラム定義（id, email, operator_id, customer_id, reason, resend_event_id, bounce_type, bounce_category, suppressed_at, expires_at, notes）
2. reason CHECK制約（hard_bounce, soft_bounce, spam_complaint, unsubscribe, manual）
3. UNIQUE制約（email, operator_id）
4. operator_id, customer_idの外部キー設定

#### 1.2 marketing_campaigns テーブル作成（1-1.5時間）
1. カラム定義（id, operator_id, name, description, campaign_type, property_ids, target_criteria, status, scheduled_at, started_at, completed_at, total_recipients, sent_count, opened_count, clicked_count, bounced_count, created_by, created_at, updated_at）
2. campaign_type CHECK制約（new_property, follow_up, reengagement, seasonal, custom）
3. status CHECK制約（draft, scheduled, sending, completed, cancelled）
4. 外部キー設定

#### 1.3 marketing_campaign_recipients テーブル作成（1時間）
1. カラム定義（id, campaign_id, customer_id, status, skip_reason, email_log_id, sent_at, opened_at, clicked_at, created_at）
2. status CHECK制約（pending, sent, opened, clicked, bounced, skipped）
3. UNIQUE制約（campaign_id, customer_id）
4. 外部キー設定

#### 1.4 marketing_email_queue テーブル作成（1-1.5時間）
1. カラム定義（id, operator_id, customer_id, campaign_id, email_type, subject, template_name, template_data, status, retry_count, max_retries, next_retry_at, last_error, priority, scheduled_at, processed_at, created_at）
2. status CHECK制約（queued, processing, sent, failed, cancelled）
3. 外部キー設定

### Acceptance Criteria
- [ ] 4つのテーブルが作成される
- [ ] CHECK制約が機能する
- [ ] 外部キーが正しく設定される
- [ ] マイグレーションが正常に実行される

---

## Task 2: (P) インデックス作成とRLSポリシー設定

_Requirements: 1.1, 2.1, 4.1, 5.1_

### Description
パフォーマンス最適化のためのインデックスとセキュリティのためのRLSポリシーを設定する

### Files to Create/Modify
- `supabase/migrations/20250101000002_create_marketing_indexes_rls.sql` (create)

### Implementation Details

#### 2.1 パフォーマンスインデックス作成（1時間）
1. `idx_email_suppressions_operator_email` ON email_suppressions(operator_id, email)
2. `idx_email_suppressions_expires_at` ON email_suppressions(expires_at) WHERE expires_at IS NOT NULL
3. `idx_marketing_email_queue_status` ON marketing_email_queue(status, next_retry_at)
4. `idx_email_logs_customer_sent` ON email_logs(customer_id, sent_at DESC)
5. `idx_marketing_campaigns_operator_status` ON marketing_campaigns(operator_id, status)
6. `idx_marketing_campaign_recipients_status` ON marketing_campaign_recipients(campaign_id, status)

#### 2.2 RLSポリシー設定（1-1.5時間）
1. email_suppressions: オペレーター自社データのみアクセス可能
2. marketing_campaigns: オペレーター自社データのみアクセス可能
3. marketing_campaign_recipients: キャンペーンのオペレーター経由でアクセス可能
4. marketing_email_queue: オペレーター自社データのみアクセス可能

### Acceptance Criteria
- [ ] インデックスが作成される
- [ ] RLSポリシーが有効化される
- [ ] オペレーターが自社データのみアクセスできる

---

## Task 3: 営業推奨ユーザー検索RPC関数作成

_Requirements: 1.1, 1.2, 1.3, 1.4_

### Description
物件条件マッチング、送信済み除外、配信停止除外を行うRPC関数を実装する

### Files to Create/Modify
- `supabase/migrations/20250101000003_create_get_marketing_recipients.sql` (create)

### Implementation Details

#### 3.1 get_marketing_recipients RPC関数作成（2-3時間）
1. パラメータ定義（p_operator_id, p_property_ids, p_target_criteria, p_exclude_recent_days）
2. 戻り値テーブル定義（customer_id, name, email, preferred_areas, budget_max, last_email_sent_at, is_suppressed, match_score, match_reasons）
3. マッチスコア計算ロジック実装
   - エリアマッチ（最大0.4）：重複エリア数 / 対象エリア数 × 0.4
   - 予算マッチ（最大0.3）：完全マッチ=0.3、80%以上=0.15
   - 間取りマッチ（最大0.2）：配列の重複があれば0.2
   - 建物種別マッチ（最大0.1）：配列の重複があれば0.1
4. 除外ロジック実装
   - 直近N日以内に送信済みの顧客を除外
   - 配信停止リスト（有効期限チェック付き）に登録されている顧客を除外
   - マッチスコア0の顧客を除外
5. 結果のソート（match_score降順）

### Acceptance Criteria
- [ ] RPC関数が作成される
- [ ] マッチスコアが正しく計算される
- [ ] 除外ロジックが機能する
- [ ] 結果がスコア順で返る

---

## Task 4: SuppressionService実装

_Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 5.1, 5.2, 5.3, 5.4, 5.5_

### Description
配信停止リストの管理（チェック、追加、削除、自動解除）を行うサービスクラスを実装する

### Files to Create/Modify
- `src/lib/services/suppression-service.ts` (create)

### Implementation Details

#### 4.1 SuppressionServiceクラス定義（2-3時間）
1. checkSuppression: 配信停止チェック
   - email_suppressionsテーブルを検索
   - expires_atのチェック（有効期限内のみ）
   - 配信停止の場合はtrue、そうでない場合はfalseを返す
2. addSuppression: 配信停止追加
   - reason別の処理（hard_bounce: 恒久、soft_bounce: 7日間、spam_complaint: 恒久、unsubscribe: 恒久、manual: 恒久）
   - soft_bounceの場合はexpires_atを設定（NOW() + 7 days）
   - email_suppressionsにINSERT（UPSERT）
3. removeSuppression: 配信停止削除
4. handleBounce: バウンス処理（Webhook用）
   - Resendイベントデータの解析
   - bounce_type判定（permanent → hard_bounce, transient → soft_bounce）
   - addSuppressionを呼び出し
5. handleSpamComplaint: 苦情処理（Webhook用）
6. cleanupExpired: 期限切れsoft_bounce削除
   - expires_at < NOW()のレコードをDELETE

### Acceptance Criteria
- [ ] 配信停止チェックが機能する
- [ ] 配信停止追加が機能する
- [ ] soft_bounceが7日後に自動解除される
- [ ] バウンス処理が正しく動作する
- [ ] 苦情処理が正しく動作する

---

## Task 5: EmailQueueService実装

_Requirements: 2.3, 3.1, 3.2, 3.3, 3.4, 3.5_

### Description
メールキューの管理（エンキュー、処理、リトライ）を行うサービスクラスを実装する

### Files to Create/Modify
- `src/lib/services/email-queue-service.ts` (create)

### Implementation Details

#### 5.1 EmailQueueServiceクラス定義（2-3時間）
1. enqueue: メールをキューに追加
   - marketing_email_queueにINSERT
   - status: queued
   - scheduled_at: NOW() or 指定時刻
   - priority設定
2. processQueue: キューの処理（Cron job用）
   - status = 'queued' AND scheduled_at <= NOW() のレコードを取得
   - 配信停止チェック（SuppressionService使用）
   - Resend API呼び出し
   - email_logsに記録
   - marketing_campaign_recipientsのステータス更新
   - キャンペーン統計更新
3. processEmail: 個別メール送信
4. handleRetry: リトライ処理
   - 送信失敗時、retry_count++
   - 指数バックオフ計算（30分、1時間、2時間）
   - next_retry_at設定
   - status: queued
   - max_retries到達時はmarkAsFailedを呼び出し
5. markAsFailed: 最終失敗処理
   - status: failed
   - last_error記録
   - オペレーター通知（ダッシュボード表示用）

### Acceptance Criteria
- [ ] メールがキューに追加される
- [ ] キューが順次処理される
- [ ] 配信停止チェックが機能する
- [ ] 失敗時にリトライする
- [ ] 指数バックオフが機能する
- [ ] 3回失敗で最終失敗となる

---

## Task 6: CampaignService実装

_Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

### Description
キャンペーンの作成、配信対象者検索、配信実行を行うサービスクラスを実装する

### Files to Create/Modify
- `src/lib/services/campaign-service.ts` (create)

### Implementation Details

#### 6.1 CampaignServiceクラス定義（2-3時間）
1. createCampaign: キャンペーン作成（draft状態）
2. getRecipients: 営業推奨ユーザー検索（RPCを呼び出し）
   - get_marketing_recipients() RPCを呼び出し
   - 条件マッチ数、除外数、最終配信対象数を計算
   - サマリー情報を返す
3. getRecipientsSummary: 検索結果サマリー計算
4. sendCampaign: 配信実行（キュー登録）
   - marketing_campaign_recipientsにレコード作成
   - marketing_email_queueにメールをエンキュー
   - キャンペーンステータスを「sending」に更新
   - バッチ処理（100件/バッチ）
5. updateCampaignStats: キャンペーン統計更新
   - sent_count, opened_count, clicked_count, bounced_count更新
   - ステータス更新（completed）

### Acceptance Criteria
- [ ] キャンペーンが作成できる
- [ ] 配信対象者が検索できる
- [ ] サマリーが正しく表示される
- [ ] 配信実行でキューに登録される
- [ ] 統計が更新される

---

## Task 7: Resend Webhook Handler実装

_Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

### Description
Resendからのバウンス・苦情イベントを受信し、配信停止リストに追加するWebhookハンドラーを実装する

### Files to Create/Modify
- `src/lib/resend/webhook-handler.ts` (create)
- `app/api/webhooks/resend/route.ts` (create)

### Implementation Details

#### 7.1 Webhook署名検証実装（1-2時間）
1. svixライブラリを使用
2. RESEND_WEBHOOK_SECRETで署名検証
3. 無効な署名の場合は401エラー

#### 7.2 イベントタイプ別処理実装（1.5-2時間）
1. email.bounced → handleBounce
   - bounce.type判定（permanent → hard_bounce, transient → soft_bounce）
   - SuppressionService.handleBounceを呼び出し
2. email.complained → handleSpamComplaint
   - SuppressionService.handleSpamComplaintを呼び出し
3. email.delivered, email.opened → email_logs更新のみ

#### 7.3 イベントID重複処理防止（30分-1時間）
1. resend_event_idをemail_suppressionsに記録
2. 同一イベントIDの重複処理をスキップ

#### 7.4 エラーハンドリング（30分-1時間）
1. 署名検証失敗 → 401 Unauthorized
2. イベント解析失敗 → 400 Bad Request
3. DB処理失敗 → 500 Internal Server Error、イベントIDを記録してリトライ

### Acceptance Criteria
- [ ] Webhook署名検証が機能する
- [ ] hard_bounceイベントで恒久配信停止される
- [ ] soft_bounceイベントで7日間配信停止される
- [ ] spam_complaintイベントで恒久配信停止される
- [ ] イベントIDの重複処理が防止される

---

## Task 8: (P) POST /api/marketing/campaigns APIルート

_Requirements: 2.1, 2.2_

### Description
キャンペーン作成APIを実装する

### Files to Create/Modify
- `app/api/marketing/campaigns/route.ts` (create)

### Implementation Details

#### 8.1 リクエストバリデーション（30分-1時間）
1. name必須チェック
2. campaign_type必須チェック
3. property_ids配列チェック
4. target_criteria JSONBチェック

#### 8.2 認証とオペレーター取得（30分）
1. Supabase Auth認証
2. operator_idの取得（auth.uid()からoperatorsテーブル検索）

#### 8.3 CampaignService呼び出し（30分）
1. CampaignService.createCampaignを呼び出し
2. 作成されたキャンペーンを返す

### Acceptance Criteria
- [ ] 認証が必要
- [ ] キャンペーンが作成される
- [ ] バリデーションエラーが返る

---

## Task 9: (P) GET /api/marketing/campaigns/:id/recipients APIルート

_Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

### Description
営業推奨ユーザー検索APIを実装する

### Files to Create/Modify
- `app/api/marketing/campaigns/[id]/recipients/route.ts` (create)

### Implementation Details

#### 9.1 クエリパラメータ処理（30分）
1. preview: boolean（デフォルト: false）
2. exclude_recent_days: number（デフォルト: 7）

#### 9.2 CampaignService呼び出し（1時間）
1. キャンペーン情報取得
2. CampaignService.getRecipientsを呼び出し
3. CampaignService.getRecipientsSummaryを呼び出し

#### 9.3 レスポンス構築（30分）
1. recipients配列
2. summary（total_matched, excluded_recent_email, excluded_suppressed, final_recipients）

### Acceptance Criteria
- [ ] 配信対象者が検索される
- [ ] サマリーが表示される
- [ ] exclude_recent_daysが機能する

---

## Task 10: (P) POST /api/marketing/campaigns/:id/send APIルート

_Requirements: 2.3, 2.4, 2.5_

### Description
キャンペーン配信実行APIを実装する

### Files to Create/Modify
- `app/api/marketing/campaigns/[id]/send/route.ts` (create)

### Implementation Details

#### 10.1 リクエストバリデーション（30分-1時間）
1. recipient_ids配列チェック（選択された配信対象者）
2. キャンペーン存在チェック
3. キャンペーンステータスチェック（draft or scheduled）

#### 10.2 CampaignService呼び出し（1時間）
1. CampaignService.sendCampaignを呼び出し
2. marketing_campaign_recipientsに登録
3. marketing_email_queueにエンキュー

#### 10.3 レスポンス構築（30分）
1. enqueued_count
2. skipped_count
3. campaign_status

### Acceptance Criteria
- [ ] 配信が実行される
- [ ] キューに登録される
- [ ] キャンペーンステータスが更新される

---

## Task 11: (P) POST /api/marketing/retry APIルート

_Requirements: 3.5_

### Description
失敗メールの手動リトライAPIを実装する

### Files to Create/Modify
- `app/api/marketing/retry/route.ts` (create)

### Implementation Details

#### 11.1 リクエストバリデーション（30分）
1. queue_id必須チェック
2. marketing_email_queue存在チェック
3. status = 'failed'チェック

#### 11.2 EmailQueueService呼び出し（1時間）
1. retry_countをリセット
2. status: queued
3. next_retry_at: NOW()
4. EmailQueueService.processEmailを呼び出し

### Acceptance Criteria
- [ ] 失敗メールがリトライされる
- [ ] retry_countがリセットされる

---

## Task 12: (P) GET /api/marketing/suppressions APIルート

_Requirements: 5.1, 5.2, 5.4_

### Description
配信停止リスト取得APIを実装する

### Files to Create/Modify
- `app/api/marketing/suppressions/route.ts` (create)

### Implementation Details

#### 12.1 クエリパラメータ処理（30分）
1. reason: string[]（フィルタリング用）
2. page: number（ページング）
3. limit: number（デフォルト: 50）

#### 12.2 email_suppressions検索（1時間）
1. operator_id絞り込み
2. reasonフィルタリング
3. 有効期限チェック（expires_atがNULLまたは未来）
4. ページング

#### 12.3 レスポンス構築（30分）
1. suppressions配列（email, reason, suppressed_at, expires_at, bounce_type, bounce_category, notes）
2. pagination（total, page, limit）

### Acceptance Criteria
- [ ] 配信停止リストが取得される
- [ ] reasonでフィルタリングされる
- [ ] ページングが機能する

---

## Task 13: (P) POST /api/marketing/suppressions APIルート

_Requirements: 5.3_

### Description
手動配信停止追加APIを実装する

### Files to Create/Modify
- `app/api/marketing/suppressions/route.ts` (modify)

### Implementation Details

#### 13.1 リクエストバリデーション（30分）
1. email必須チェック
2. emailフォーマットチェック
3. customer_id存在チェック（optional）
4. notes（optional）

#### 13.2 SuppressionService呼び出し（30分-1時間）
1. SuppressionService.addSuppressionを呼び出し
2. reason: manual
3. expires_at: NULL（恒久）

### Acceptance Criteria
- [ ] 手動で配信停止追加される
- [ ] emailバリデーションが機能する

---

## Task 14: (P) Vercel Cron Job設定

_Requirements: 2.3, 3.1, 4.6_

### Description
メールキュー処理とsoft_bounce期限切れ削除のCron jobを設定する

### Files to Create/Modify
- `app/api/cron/process-email-queue/route.ts` (create)
- `app/api/cron/cleanup-suppressions/route.ts` (create)
- `vercel.json` (modify)

### Implementation Details

#### 14.1 メールキュー処理Cron job（1.5-2時間）
1. EmailQueueService.processQueueを呼び出し
2. 1分間隔で実行
3. 最大100件/バッチ処理
4. レート制限考慮（Resend: 100/sec）

#### 14.2 soft_bounce期限切れ削除Cron job（1時間）
1. SuppressionService.cleanupExpiredを呼び出し
2. 1日1回実行（深夜）

#### 14.3 vercel.json設定（30分）
```json
{
  "crons": [
    {
      "path": "/api/cron/process-email-queue",
      "schedule": "* * * * *"
    },
    {
      "path": "/api/cron/cleanup-suppressions",
      "schedule": "0 3 * * *"
    }
  ]
}
```

### Acceptance Criteria
- [ ] メールキューが1分間隔で処理される
- [ ] 期限切れsoft_bounceが削除される

---

## Task 15: CampaignFormコンポーネント

_Requirements: 2.1, 2.2_

### Description
キャンペーン作成フォームコンポーネントを実装する

### Files to Create/Modify
- `src/components/features/marketing/CampaignForm.tsx` (create)

### Implementation Details

#### 15.1 フォーム項目（1.5-2時間）
1. name入力
2. description入力（textarea）
3. campaign_typeセレクト
4. property_idsマルチセレクト（物件選択）
5. target_criteria入力（エリア、予算、間取り、建物種別）
6. 送信ボタン

#### 15.2 バリデーション（30分-1時間）
1. name必須
2. campaign_type必須
3. property_ids最低1件

#### 15.3 API呼び出し（30分）
1. POST /api/marketing/campaigns
2. 成功時に配信対象者検索画面に遷移

### Acceptance Criteria
- [ ] フォームが表示される
- [ ] バリデーションが機能する
- [ ] キャンペーンが作成される

---

## Task 16: (P) RecipientsSearchButtonコンポーネント

_Requirements: 1.1_

### Description
営業推奨ユーザー検索ボタンコンポーネントを実装する

### Files to Create/Modify
- `src/components/features/marketing/RecipientsSearchButton.tsx` (create)

### Implementation Details

#### 16.1 ボタン配置（30分-1時間）
1. 物件追加後のUIに配置
2. キャンペーン作成画面に配置

#### 16.2 検索実行（1時間）
1. GET /api/marketing/campaigns/:id/recipients?preview=true
2. RecipientsSummaryコンポーネントを表示

### Acceptance Criteria
- [ ] ボタンがクリックできる
- [ ] 検索結果が表示される

---

## Task 17: (P) RecipientsSummaryコンポーネント

_Requirements: 1.5_

### Description
配信対象者サマリー表示コンポーネントを実装する

### Files to Create/Modify
- `src/components/features/marketing/RecipientsSummary.tsx` (create)

### Implementation Details

#### 17.1 サマリー表示（1時間）
1. 条件マッチ数
2. 除外数（送信済み、配信停止）
3. 最終配信対象数

#### 17.2 詳細表示（1時間）
1. マッチ理由（area_match, budget_match, layout_match, building_type_match）
2. マッチスコア

### Acceptance Criteria
- [ ] サマリーが表示される
- [ ] 数値が正しい

---

## Task 18: RecipientsListコンポーネント

_Requirements: 1.6_

### Description
配信対象者一覧表示と選択機能を実装する

### Files to Create/Modify
- `src/components/features/marketing/RecipientsList.tsx` (create)

### Implementation Details

#### 18.1 一覧表示（1.5-2時間）
1. 顧客名
2. メールアドレス
3. 希望エリア
4. 予算
5. マッチスコア
6. 最終送信日時

#### 18.2 選択機能（1時間）
1. チェックボックス（全選択/個別選択）
2. 選択された顧客IDの配列を管理

#### 18.3 配信実行ボタン（30分-1時間）
1. POST /api/marketing/campaigns/:id/send
2. 選択された顧客IDを送信

### Acceptance Criteria
- [ ] 一覧が表示される
- [ ] 選択が機能する
- [ ] 配信実行される

---

## Task 19: (P) SuppressionListコンポーネント

_Requirements: 5.1, 5.2, 5.4_

### Description
配信停止リスト表示コンポーネントを実装する

### Files to Create/Modify
- `src/components/features/marketing/SuppressionList.tsx` (create)

### Implementation Details

#### 19.1 一覧表示（1.5-2時間）
1. メールアドレス
2. 配信停止理由
3. 配信停止日時
4. 有効期限（soft_bounceの場合）
5. バウンス詳細（bounce_type, bounce_category）

#### 19.2 フィルタリング（1時間）
1. reasonセレクト（全て、hard_bounce, soft_bounce, spam_complaint, unsubscribe, manual）
2. フィルタリング実行

#### 19.3 ページング（30分-1時間）
1. 50件/ページ
2. ページネーションコントロール

### Acceptance Criteria
- [ ] 配信停止リストが表示される
- [ ] フィルタリングが機能する
- [ ] ページングが機能する

---

## Task 20: (P) SuppressionAddFormコンポーネント

_Requirements: 5.3_

### Description
手動配信停止追加フォームコンポーネントを実装する

### Files to Create/Modify
- `src/components/features/marketing/SuppressionAddForm.tsx` (create)

### Implementation Details

#### 20.1 フォーム項目（1時間）
1. email入力（必須）
2. customer_idセレクト（optional）
3. notes入力（textarea, optional）

#### 20.2 バリデーション（30分）
1. emailフォーマットチェック

#### 20.3 API呼び出し（30分）
1. POST /api/marketing/suppressions
2. 成功時にSuppressionListを更新

### Acceptance Criteria
- [ ] フォームが表示される
- [ ] バリデーションが機能する
- [ ] 配信停止が追加される

---

## Task 21: (P) CampaignStatsコンポーネント

_Requirements: 2.4_

### Description
キャンペーン統計表示コンポーネントを実装する

### Files to Create/Modify
- `src/components/features/marketing/CampaignStats.tsx` (create)

### Implementation Details

#### 21.1 統計表示（1.5-2時間）
1. 送信数（sent_count）
2. 開封数（opened_count）
3. クリック数（clicked_count）
4. バウンス数（bounced_count）
5. 開封率（opened_count / sent_count × 100）
6. クリック率（clicked_count / sent_count × 100）

#### 21.2 グラフ表示（1-1.5時間）
1. Chart.jsまたはRecharts使用
2. 棒グラフ（送信数、開封数、クリック数、バウンス数）

### Acceptance Criteria
- [ ] 統計が表示される
- [ ] グラフが表示される

---

## Task 22: (P) テスト実装

_Requirements: 全要件_

### Description
営業メール機能のユニットテスト、統合テストを実装する

### Files to Create/Modify
- `src/lib/services/__tests__/campaign-service.test.ts` (create)
- `src/lib/services/__tests__/suppression-service.test.ts` (create)
- `src/lib/services/__tests__/email-queue-service.test.ts` (create)
- `app/api/marketing/__tests__/campaigns.test.ts` (create)
- `app/api/webhooks/__tests__/resend.test.ts` (create)

### Implementation Details

#### 22.1 CampaignServiceテスト（2時間）
1. createCampaignが正しく動作する
2. getRecipientsが除外ロジックを適用する
3. sendCampaignがキューに登録する
4. updateCampaignStatsが統計を更新する

#### 22.2 SuppressionServiceテスト（2時間）
1. checkSuppressionが配信停止をチェックする
2. addSuppressionがreason別に正しく追加する
3. handleBounceがbounce_type別に処理する
4. cleanupExpiredが期限切れを削除する

#### 22.3 EmailQueueServiceテスト（2時間）
1. enqueueがキューに追加する
2. processQueueが順次処理する
3. handleRetryが指数バックオフでリトライする
4. markAsFailedが最終失敗として記録する

#### 22.4 API統合テスト（2-3時間）
1. POST /api/marketing/campaignsがキャンペーンを作成する
2. GET /api/marketing/campaigns/:id/recipientsが配信対象者を検索する
3. POST /api/marketing/campaigns/:id/sendが配信を実行する
4. POST /api/webhooks/resendがバウンス・苦情を処理する

### Acceptance Criteria
- [ ] 全テストがパスする
- [ ] カバレッジ80%以上
