# Implementation Tasks

## Task 1: データベーススキーマとマイグレーション実装

_Requirements: 1.1, 1.2, 1.3, 1.4, 4.2_

### 1.1 (P) viewingsテーブルマイグレーション作成

Supabase migrationで`viewings`テーブルを作成する。オペレーターごとの内見予約データを保存。

- [ ] `supabase/migrations/{timestamp}_create_viewings_table.sql`を作成
- [ ] カラム定義（id, customer_id, property_id, operator_id, conversation_id, viewing_date, duration_minutes, status, assigned_to, notes, cancelled_at, cancellation_reason, reminder_sent, created_at, updated_at）
- [ ] status列にCHECK制約（'confirmed', 'completed', 'cancelled', 'no_show'）
- [ ] FK制約（customer_id, property_id, operator_id, conversation_id, assigned_to）
- [ ] インデックス作成（operator_id, viewing_date, status）
- [ ] デフォルト値設定（status='confirmed', reminder_sent=false）

### 1.2 (P) customersテーブルsource列追加

既存の`customers`テーブルに顧客の登録経路を示す`source`列を追加。

- [ ] `supabase/migrations/{timestamp}_add_customers_source.sql`を作成
- [ ] source列追加（'kiosk', 'web', 'manual'のenum、デフォルト'manual'）
- [ ] 既存データに対してデフォルト値を設定

### 1.3 (P) RLSポリシー設定

マルチテナント分離のためのRow Level Securityを設定。

- [ ] オペレーター向けポリシー（自社の予約のみ閲覧・更新可能）
- [ ] SELECT, INSERT, UPDATE, DELETEそれぞれにポリシー設定
- [ ] `operator_id`による分離を確認

### 1.4 (P) マイグレーションのローカルテスト

作成したマイグレーションをローカル環境で実行し、動作確認。

- [ ] `supabase db reset`でマイグレーション適用
- [ ] テストデータ投入
- [ ] RLSポリシーの動作確認（異なるoperator_idでのアクセス制御）

---

## Task 2: Zodスキーマとデータ型定義

_Requirements: 1.1, 1.2, 1.3, 1.4, 4.2_

### 2.1 (P) Viewingスキーマ定義

内見予約のバリデーションスキーマをZodで定義。

- [ ] `src/schemas/viewing.ts`を作成
- [ ] `createViewingSchema`（予約作成時のバリデーション）
- [ ] `updateViewingSchema`（ステータス更新、担当者アサイン、メモ追加）
- [ ] `cancelViewingSchema`（キャンセル理由のバリデーション）
- [ ] ViewingStatus型のexport

### 2.2 (P) 型定義の生成と拡張

Supabaseの自動生成型を拡張し、予約の詳細情報を含む型を定義。

- [ ] `src/types/database.ts`を更新（Supabase CLI自動生成）
- [ ] `ViewingWithDetails`型を定義（customer, property情報をJOIN）
- [ ] `ViewingFilters`型を定義（フィルタリング用）

---

## Task 3: ICSファイル生成ユーティリティ

_Requirements: 2.3_

### 3.1 ICS Generator実装

カレンダー招待ファイル（.ics）を生成する関数を実装。

- [ ] `src/lib/email/utils/generate-ics.ts`を作成
- [ ] `ical-generator`パッケージをインストール
- [ ] `generateViewingICS()`関数実装
  - [ ] イベント情報（タイトル、説明、開始・終了時刻、場所）
  - [ ] オーガナイザー情報設定
  - [ ] 日本語エンコーディング対応
  - [ ] タイムゾーン（Asia/Tokyo）設定
- [ ] 文字列（.icsファイル内容）を返却

### 3.2 ICS Generator単体テスト

ICSファイルの生成とフォーマットが正しいことを確認。

- [ ] `src/lib/email/utils/__tests__/generate-ics.test.ts`を作成
- [ ] 基本的なイベント生成テスト
- [ ] 日本語対応テスト
- [ ] タイムゾーンテスト
- [ ] Googleカレンダーでの読み込み互換性を手動確認

---

## Task 4: React Emailテンプレート実装

_Requirements: 2.1, 2.2, 2.4, 4.3, 5.1, 5.2, 5.3_

### 4.1 (P) 予約確認メールテンプレート

顧客向けの内見予約確認メールテンプレート。

- [ ] `src/lib/email/templates/ViewingConfirmationEmail.tsx`を作成
- [ ] Props定義（customerName, propertyTitle, propertyAddress, viewingDate, viewingTime, operatorName, operatorPhone）
- [ ] レスポンシブデザイン
- [ ] 日本語フォント対応
- [ ] 物件情報・日時を見やすく表示
- [ ] オペレーター連絡先を明記
- [ ] React Emailのベストプラクティスに従う

### 4.2 (P) オペレーター通知メールテンプレート

オペレーター向けの新規予約通知メール。

- [ ] `src/lib/email/templates/ViewingNotificationEmail.tsx`を作成
- [ ] Props定義（customerName, customerPhone, customerEmail, propertyTitle, viewingDate, conversationSummary）
- [ ] 顧客情報を強調表示
- [ ] 会話サマリーを含める（optional）
- [ ] 予約管理ページへのリンク

### 4.3 (P) リマインダーメールテンプレート

内見前日に送信するリマインダーメール。

- [ ] `src/lib/email/templates/ViewingReminderEmail.tsx`を作成
- [ ] Props定義（customerName, propertyTitle, propertyAddress, viewingDate, viewingTime, operatorName, operatorPhone）
- [ ] 「明日の内見について」という件名を推奨
- [ ] 物件情報と日時を再確認できる内容
- [ ] オペレーター連絡先を含める

### 4.4 (P) キャンセル通知メールテンプレート

予約キャンセル時に顧客へ送信するメール。

- [ ] `src/lib/email/templates/ViewingCancellationEmail.tsx`を作成
- [ ] Props定義（customerName, propertyTitle, viewingDate, cancellationReason, operatorPhone）
- [ ] キャンセル理由を丁寧に説明
- [ ] 再予約を促す文言
- [ ] オペレーター連絡先を含める

### 4.5 (P) メールテンプレートのプレビュー設定

React Email開発環境でのプレビュー設定。

- [ ] `emails/`ディレクトリにテンプレートをシンボリックリンクまたはコピー
- [ ] `npm run email:dev`でプレビュー確認
- [ ] 各テンプレートのモックデータ作成

---

## Task 5: ViewingServiceの実装

_Requirements: 1.1, 1.2, 1.3, 1.4, 3.1, 3.2, 3.3, 3.4, 3.5, 4.1, 4.2, 4.3_

### 5.1 ViewingServiceクラス作成

内見予約のビジネスロジックをカプセル化。

- [ ] `src/lib/services/viewing-service.ts`を作成
- [ ] SupabaseClientをDI
- [ ] CRUD操作のメソッド定義

### 5.2 予約作成メソッド

キオスク完了APIから呼び出される予約作成ロジック。

- [ ] `createViewing()`メソッド実装
- [ ] 入力バリデーション（createViewingSchema）
- [ ] customers INSERT または既存顧客UPDATE
- [ ] viewings INSERT（status='confirmed'）
- [ ] conversations UPDATEでoutcome設定
- [ ] トランザクション処理
- [ ] エラーハンドリング

### 5.3 予約一覧取得メソッド

オペレーター向け予約一覧取得（フィルタリング対応）。

- [ ] `getViewings()`メソッド実装
- [ ] フィルター対応（status, from_date, to_date, assigned_to）
- [ ] ページネーション対応
- [ ] customer, propertyをJOIN
- [ ] operator_idによる自動フィルタリング（RLS）
- [ ] ソート順（viewing_date DESC）

### 5.4 予約詳細取得メソッド

単一予約の詳細情報を取得。

- [ ] `getViewingById()`メソッド実装
- [ ] customer, property, conversation情報をJOIN
- [ ] operator_idチェック（RLS）
- [ ] 404エラーハンドリング

### 5.5 予約更新メソッド

ステータス更新、担当者アサイン、メモ追加。

- [ ] `updateViewing()`メソッド実装
- [ ] 入力バリデーション（updateViewingSchema）
- [ ] ステータス遷移チェック（cancelled状態からは更新不可など）
- [ ] updated_atの自動更新
- [ ] エラーハンドリング

### 5.6 予約キャンセルメソッド

予約のキャンセル処理。

- [ ] `cancelViewing()`メソッド実装
- [ ] キャンセル理由のバリデーション（cancelViewingSchema）
- [ ] status='cancelled', cancelled_at設定
- [ ] cancellation_reason保存
- [ ] 既にキャンセル済みの場合は409エラー
- [ ] エラーハンドリング

### 5.7 会話履歴取得メソッド

予約に紐づく会話履歴を取得。

- [ ] `getViewingConversation()`メソッド実装
- [ ] conversation_idから会話取得
- [ ] メッセージ履歴取得
- [ ] operator_idチェック（RLS）

### 5.8 ViewingService単体テスト

ビジネスロジックの単体テスト。

- [ ] `src/lib/services/__tests__/viewing-service.test.ts`を作成
- [ ] モックSupabaseクライアント
- [ ] 各メソッドのテストケース
  - [ ] createViewing成功・失敗
  - [ ] getViewingsフィルタリング
  - [ ] updateViewing成功・失敗
  - [ ] cancelViewing成功・重複キャンセルエラー

---

## Task 6: メール送信処理の実装

_Requirements: 2.1, 2.2, 2.3, 2.4, 4.3, 5.1, 5.2, 5.3_

### 6.1 (P) 予約確認メール送信関数

顧客向け予約確認メールの送信処理。

- [ ] `src/lib/email/send-viewing-confirmation.ts`を作成
- [ ] Resend APIクライアント使用
- [ ] ICSファイルを添付
- [ ] ViewingConfirmationEmailテンプレート使用
- [ ] エラーハンドリングとログ記録
- [ ] 送信失敗時のリトライロジック検討

### 6.2 (P) オペレーター通知メール送信関数

オペレーター向け新規予約通知メールの送信処理。

- [ ] `src/lib/email/send-viewing-notification.ts`を作成
- [ ] ViewingNotificationEmailテンプレート使用
- [ ] ICSファイルを添付
- [ ] オペレーターのメールアドレス取得
- [ ] エラーハンドリング

### 6.3 (P) リマインダーメール送信関数

内見リマインダーメールの送信処理。

- [ ] `src/lib/email/send-viewing-reminder.ts`を作成
- [ ] ViewingReminderEmailテンプレート使用
- [ ] エラーハンドリング
- [ ] 送信後にreminder_sentフラグ更新

### 6.4 (P) キャンセルメール送信関数

キャンセル通知メールの送信処理。

- [ ] `src/lib/email/send-viewing-cancellation.ts`を作成
- [ ] ViewingCancellationEmailテンプレート使用
- [ ] エラーハンドリング

---

## Task 7: キオスク完了API更新

_Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 2.3, 2.4_

### 7.1 POST /api/kiosk/complete拡張

既存のキオスク完了APIに内見予約作成機能を追加。

- [ ] `app/api/kiosk/complete/route.ts`を更新
- [ ] リクエストボディに内見予約情報を含める
  - [ ] property_id
  - [ ] viewing_date
  - [ ] customer情報（name, phone, email）
- [ ] ViewingService.createViewing()を呼び出し
- [ ] 予約作成後、メール送信を並列実行
  - [ ] sendViewingConfirmation()（顧客向け）
  - [ ] sendViewingNotification()（オペレーター向け）
- [ ] ICS生成とメール添付
- [ ] エラーハンドリング（予約作成失敗、メール送信失敗）
- [ ] トランザクション処理（予約作成とconversation更新）

---

## Task 8: 予約管理API実装（オペレーター向け）

_Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 4.1, 4.2_

### 8.1 GET /api/viewings実装

予約一覧取得API。

- [ ] `app/api/viewings/route.ts`を作成
- [ ] オペレーター認証チェック
- [ ] クエリパラメータ受け取り（status, from_date, to_date, assigned_to, page, limit）
- [ ] ViewingService.getViewings()呼び出し
- [ ] ページネーション情報を含むレスポンス
- [ ] エラーハンドリング（401, 500）

### 8.2 GET /api/viewings/[id]実装

予約詳細取得API。

- [ ] `app/api/viewings/[id]/route.ts`を作成
- [ ] オペレーター認証チェック
- [ ] ViewingService.getViewingById()呼び出し
- [ ] 404エラーハンドリング
- [ ] 403エラー（他社の予約へのアクセス）

### 8.3 PATCH /api/viewings/[id]実装

予約更新API（ステータス、担当者、メモ）。

- [ ] `app/api/viewings/[id]/route.ts`を更新
- [ ] オペレーター認証チェック
- [ ] リクエストボディバリデーション（updateViewingSchema）
- [ ] ViewingService.updateViewing()呼び出し
- [ ] エラーハンドリング（400, 403, 404, 409）

### 8.4 DELETE /api/viewings/[id]実装

予約キャンセルAPI。

- [ ] `app/api/viewings/[id]/route.ts`を更新
- [ ] オペレーター認証チェック
- [ ] リクエストボディバリデーション（cancellation_reason）
- [ ] ViewingService.cancelViewing()呼び出し
- [ ] キャンセルメール送信（sendViewingCancellation）
- [ ] エラーハンドリング（409: 既にキャンセル済み）

### 8.5 GET /api/viewings/[id]/conversation実装

予約に紐づく会話履歴取得API。

- [ ] `app/api/viewings/[id]/conversation/route.ts`を作成
- [ ] オペレーター認証チェック
- [ ] ViewingService.getViewingConversation()呼び出し
- [ ] メッセージをフォーマットして返却
- [ ] エラーハンドリング（404, 403）

---

## Task 9: オペレーター向けフロントエンド実装

_Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

### 9.1 (P) ViewingFilterコンポーネント

予約一覧のフィルタリングUI。

- [ ] `src/components/features/viewings/ViewingFilter.tsx`を作成
- [ ] フィルター項目
  - [ ] ステータス選択（ドロップダウン）
  - [ ] 日付範囲（DateRangePicker）
  - [ ] 担当者選択（ドロップダウン）
- [ ] フィルター変更時にコールバック実行
- [ ] shadcn/uiコンポーネント使用（Select, Popover, Button）

### 9.2 (P) ViewingCardコンポーネント

予約一覧のカード形式表示。

- [ ] `src/components/features/viewings/ViewingCard.tsx`を作成
- [ ] 表示項目
  - [ ] 顧客名、連絡先
  - [ ] 物件タイトル、住所
  - [ ] 内見日時
  - [ ] ステータスバッジ
  - [ ] 担当者（assigned_to）
- [ ] クリックで詳細ページへ遷移
- [ ] ステータス更新ボタン（クイックアクション）

### 9.3 ViewingListコンポーネント

予約一覧の統合コンポーネント。

- [ ] `src/components/features/viewings/ViewingList.tsx`を作成
- [ ] ViewingFilterコンポーネント組み込み
- [ ] ViewingCardの一覧表示
- [ ] ページネーション（shadcn/ui Pagination）
- [ ] ローディング状態表示
- [ ] エラー状態表示
- [ ] 空状態表示（予約なし）

### 9.4 (P) ConversationHistoryコンポーネント

会話履歴のタイムライン表示。

- [ ] `src/components/features/viewings/ConversationHistory.tsx`を作成
- [ ] メッセージをタイムライン形式で表示
- [ ] ユーザーメッセージとアシスタントメッセージを区別
- [ ] タイムスタンプ表示
- [ ] スクロール可能なコンテナ

### 9.5 ViewingDetailコンポーネント

予約詳細画面の統合コンポーネント。

- [ ] `src/components/features/viewings/ViewingDetail.tsx`を作成
- [ ] 予約情報表示セクション
  - [ ] 顧客情報（名前、電話、メール）
  - [ ] 物件情報（タイトル、住所）
  - [ ] 内見日時
  - [ ] ステータス
  - [ ] 担当者
  - [ ] メモ
- [ ] ConversationHistoryコンポーネント組み込み
- [ ] アクションボタン
  - [ ] ステータス更新
  - [ ] 担当者アサイン
  - [ ] メモ編集
  - [ ] キャンセル
- [ ] 編集モードとビューモードの切り替え

---

## Task 10: オペレーター向けページ実装

_Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

### 10.1 予約一覧ページ

オペレーターの予約管理ダッシュボード。

- [ ] `app/(operator)/viewings/page.tsx`を作成
- [ ] オペレーター認証チェック（middleware）
- [ ] ViewingListコンポーネント配置
- [ ] APIから予約一覧取得（Server Component or useQuery）
- [ ] ページタイトル「内見予約管理」
- [ ] パンくずリスト設定

### 10.2 予約詳細ページ

単一予約の詳細・編集ページ。

- [ ] `app/(operator)/viewings/[id]/page.tsx`を作成
- [ ] オペレーター認証チェック
- [ ] ViewingDetailコンポーネント配置
- [ ] APIから予約詳細と会話履歴を取得
- [ ] 更新処理のハンドラー実装
- [ ] キャンセル処理のハンドラー実装
- [ ] 戻るボタン

### 10.3 ナビゲーション追加

オペレーターダッシュボードのナビゲーションに予約管理リンクを追加。

- [ ] `src/components/layout/OperatorNav.tsx`を更新
- [ ] 「内見予約」メニュー項目追加
- [ ] アイコン設定（Calendar or ClipboardList）

---

## Task 11: リマインダーCronジョブ実装

_Requirements: 5.1, 5.2, 5.3_

### 11.1 リマインダー送信API実装

翌日の内見リマインダーを送信するCronジョブ。

- [ ] `app/api/cron/viewing-reminders/route.ts`を作成
- [ ] 翌日の内見を検索
  - [ ] viewing_date = 明日
  - [ ] status = 'confirmed'
  - [ ] reminder_sent = false
- [ ] 各予約に対してリマインダーメール送信
- [ ] 送信成功後、reminder_sentフラグ更新
- [ ] エラーハンドリングとログ記録
- [ ] Vercel Cron認証ヘッダー検証

### 11.2 Vercel Cron設定

vercel.jsonにCronスケジュールを設定。

- [ ] `vercel.json`を更新
- [ ] `/api/cron/viewing-reminders`を毎日午前9時に実行
- [ ] cronスケジュール式設定（例: `0 9 * * *`）
- [ ] 本番環境でのみ実行するよう設定

### 11.3 Cronジョブのテスト

リマインダー送信ジョブの動作確認。

- [ ] ローカル環境で手動実行してテスト
- [ ] 翌日の予約データを作成
- [ ] APIを直接呼び出し
- [ ] メール送信とフラグ更新を確認
- [ ] 重複送信しないことを確認

---

## Task 12: E2Eテストとドキュメント

_Requirements: All_

### 12.1 (P) 統合テスト実装

主要フローの統合テスト。

- [ ] `app/api/viewings/__tests__/route.test.ts`を作成
- [ ] GET /api/viewings - 一覧取得テスト
- [ ] GET /api/viewings/[id] - 詳細取得テスト
- [ ] PATCH /api/viewings/[id] - 更新テスト
- [ ] DELETE /api/viewings/[id] - キャンセルテスト
- [ ] RLSポリシーのテスト（他社データへのアクセス拒否）

### 12.2 (P) E2Eテスト実装

Playwrightでのエンドツーエンドテスト。

- [ ] `e2e/viewings.spec.ts`を作成
- [ ] オペレーターログイン
- [ ] 予約一覧表示テスト
- [ ] フィルタリングテスト
- [ ] 予約詳細表示テスト
- [ ] ステータス更新テスト
- [ ] 会話履歴表示テスト
- [ ] キャンセルフローテスト

### 12.3 (P) 機能ドキュメント作成

内見予約機能の使い方とAPI仕様をドキュメント化。

- [ ] `.kiro/specs/viewing-reservation/implementation.md`を作成
- [ ] API仕様まとめ
- [ ] フロントエンドコンポーネント一覧
- [ ] データフロー図
- [ ] トラブルシューティング
