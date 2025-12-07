# Implementation Tasks

## Task 1: ViewingService実装

### Description
内見予約のビジネスロジック

### Files to Create/Modify
- `src/lib/services/viewing-service.ts` (create)
- `src/schemas/viewing.ts` (create)

### Implementation Details
1. createViewing: 予約作成（キオスク完了APIから呼び出し）
2. getViewings: 一覧取得（オペレーター向け）
3. getViewingById: 詳細取得
4. updateViewing: ステータス更新
5. cancelViewing: キャンセル処理
6. getViewingConversation: 会話履歴取得

### Acceptance Criteria
- [ ] CRUD操作が機能する
- [ ] RLSが適用される

---

## Task 2: ICS Generator実装

### Description
カレンダー招待ファイル生成

### Files to Create/Modify
- `src/lib/email/utils/generate-ics.ts` (create)

### Implementation Details
1. ical-generator使用
2. 内見イベント情報設定
3. 日本語対応
4. オーガナイザー情報設定

### Acceptance Criteria
- [ ] ICSファイルが生成される
- [ ] Googleカレンダーで読み込める

---

## Task 3: メールテンプレート実装

### Description
内見関連のメールテンプレート

### Files to Create/Modify
- `src/lib/email/templates/ViewingConfirmationEmail.tsx` (create)
- `src/lib/email/templates/ViewingNotificationEmail.tsx` (create)
- `src/lib/email/templates/ViewingReminderEmail.tsx` (create)
- `src/lib/email/templates/ViewingCancellationEmail.tsx` (create)

### Implementation Details
1. React Email使用
2. レスポンシブデザイン
3. 日本語フォント対応
4. ICS添付対応

### Acceptance Criteria
- [ ] テンプレートが正しくレンダリングされる

---

## Task 4: GET /api/viewings API実装

### Description
内見予約一覧取得API（オペレーター向け）

### Files to Create/Modify
- `app/api/viewings/route.ts` (create)

### Implementation Details
1. オペレーター認証チェック
2. ステータスフィルター
3. 日付範囲フィルター
4. 担当者フィルター
5. ページネーション
6. 顧客・物件情報JOIN

### Acceptance Criteria
- [ ] 一覧が取得できる
- [ ] フィルターが機能する
- [ ] 自社の予約のみ取得される

---

## Task 5: GET/PATCH /api/viewings/[id] API実装

### Description
内見予約詳細・更新API

### Files to Create/Modify
- `app/api/viewings/[id]/route.ts` (create)

### Implementation Details
1. GET: 詳細取得（顧客・物件情報含む）
2. PATCH: ステータス更新、担当者アサイン、メモ追加
3. オペレーター認証チェック
4. 自社の予約のみ操作可能

### Acceptance Criteria
- [ ] 詳細が取得できる
- [ ] ステータスが更新できる

---

## Task 6: DELETE /api/viewings/[id] API実装

### Description
内見予約キャンセルAPI

### Files to Create/Modify
- `app/api/viewings/[id]/route.ts` (modify)

### Implementation Details
1. キャンセル処理
2. キャンセル理由保存
3. 顧客へキャンセルメール送信
4. ステータスをcancelledに更新

### Acceptance Criteria
- [ ] キャンセルができる
- [ ] キャンセルメールが送信される

---

## Task 7: GET /api/viewings/[id]/conversation API実装

### Description
予約に紐づく会話履歴取得API

### Files to Create/Modify
- `app/api/viewings/[id]/conversation/route.ts` (create)

### Implementation Details
1. conversation_idから会話取得
2. メッセージ履歴取得
3. オペレーター認証チェック

### Acceptance Criteria
- [ ] 会話履歴が取得できる

---

## Task 8: ViewingListコンポーネント

### Description
オペレーター向け予約一覧

### Files to Create/Modify
- `src/components/features/viewings/ViewingList.tsx` (create)
- `src/components/features/viewings/ViewingCard.tsx` (create)
- `src/components/features/viewings/ViewingFilter.tsx` (create)

### Implementation Details
1. カード形式の一覧
2. ステータス更新ボタン
3. 担当者選択
4. フィルターUI
5. ページネーション

### Acceptance Criteria
- [ ] 一覧が表示される
- [ ] ステータスが更新できる

---

## Task 9: ViewingDetailコンポーネント

### Description
予約詳細・会話履歴表示

### Files to Create/Modify
- `src/components/features/viewings/ViewingDetail.tsx` (create)
- `src/components/features/viewings/ConversationHistory.tsx` (create)

### Implementation Details
1. 予約詳細表示
2. 顧客情報表示
3. 物件情報表示
4. 会話履歴表示（タイムライン形式）
5. ステータス更新
6. キャンセルボタン

### Acceptance Criteria
- [ ] 詳細が表示される
- [ ] 会話履歴が表示される

---

## Task 10: オペレーター向けページ

### Description
オペレーターの予約管理ページ

### Files to Create/Modify
- `app/(operator)/viewings/page.tsx` (create)
- `app/(operator)/viewings/[id]/page.tsx` (create)

### Implementation Details
1. 自社の予約一覧
2. フィルター機能
3. 予約詳細ページ
4. 会話履歴閲覧
5. ステータス更新
6. 担当者アサイン

### Acceptance Criteria
- [ ] 自社の予約が管理できる
- [ ] 会話履歴が確認できる

---

## Task 11: リマインダーCronジョブ

### Description
内見リマインダーの自動送信

### Files to Create/Modify
- `app/api/cron/viewing-reminders/route.ts` (create)
- `vercel.json` (modify)

### Implementation Details
1. 翌日の内見を検索（reminder_sent = false）
2. リマインダーメール送信
3. reminder_sentフラグ更新

### Acceptance Criteria
- [ ] リマインダーが送信される
- [ ] 重複送信しない

---

## Task 12: テスト実装

### Description
内見予約機能のテスト

### Files to Create/Modify
- `src/lib/services/__tests__/viewing-service.test.ts` (create)
- `src/lib/email/utils/__tests__/generate-ics.test.ts` (create)
- `app/api/viewings/__tests__/route.test.ts` (create)

### Acceptance Criteria
- [ ] サービスのテストがパスする
- [ ] ICS生成のテストがパスする
