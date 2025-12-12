# Implementation Tasks

## Task 1: OperatorService実装 (P)

_Requirements: 1.1, 1.2, 1.3, 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 4.1_

### Description
オペレーター管理のビジネスロジックとデータアクセス層を実装する。招待メール方式の登録フローをサポートし、CRUD操作と統計取得を提供する。

### Files to Create/Modify
- `src/lib/services/operator-service.ts` (create)
- `src/schemas/operator.ts` (create)

### Implementation Details
1. Zodスキーマ定義（OperatorCreateSchema, OperatorUpdateSchema）
2. createOperator: Supabase Auth `admin.inviteUserByEmail()` で招待メール送信
   - roleを'operator'に設定
   - raw_user_meta_dataに会社情報を保存
   - operators テーブルに挿入（status='pending_activation'）
3. resendInvitation: 招待メールを再送信
4. getOperators: 一覧取得（フィルタリング、検索、ページネーション対応）
   - LEFT JOIN で物件数・顧客数を集計
5. getOperatorById: 詳細取得（統計付き）
6. updateOperator: 企業情報・サブスクリプション更新
7. deleteOperator: オペレーター削除（CASCADE確認）

### Acceptance Criteria
- [ ] 招待メール方式でオペレーターを登録できる
- [ ] パスワードは管理者が設定せず、オペレーター自身が設定する
- [ ] 招待メールを再送信できる
- [ ] 一覧取得時に統計（物件数、顧客数）が含まれる
- [ ] フィルタリング・検索が機能する
- [ ] CRUD操作がすべて機能する

---

## Task 2: StatsService実装 (P)

_Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

### Description
システム全体の統計情報を効率的に集計するサービス層を実装する。

### Files to Create/Modify
- `src/lib/services/stats-service.ts` (create)

### Implementation Details
1. getSystemStats: システム全体統計
   - オペレーター数（状態別: active, suspended, cancelled, trial, pending_activation）
   - 物件数（全体、公開中）
   - 顧客数（全体）
   - 会話数（全体、直近30日）
2. getOperatorStats: オペレーター別統計
   - 物件数、顧客数、会話数
3. SQL集計関数（COUNT, GROUP BY）を使用してパフォーマンス最適化

### Acceptance Criteria
- [ ] システム全体の統計が正しく集計される
- [ ] オペレーター別統計が取得できる
- [ ] クエリのパフォーマンスが許容範囲（<500ms）

---

## Task 3: POST /api/admin/operators API

_Requirements: 1.1, 1.2, 1.3, 1.4_

### Description
オペレーター登録APIを実装する。招待メール方式でオペレーターを登録し、パスワードはオペレーター自身が設定する。

### Files to Create/Modify
- `app/api/admin/operators/route.ts` (create)

### Implementation Details
1. admin権限チェック（`auth.users.role === 'admin'`）
2. リクエストボディのバリデーション（Zod）
3. `admin.inviteUserByEmail()` で招待メール送信
   - パスワードパラメータは不要
   - roleを'operator'に設定
   - raw_user_meta_dataに会社情報を設定
4. `operators` テーブルに挿入（status='pending_activation'）
5. 成功レスポンス（`invitation_sent: true`）

### Acceptance Criteria
- [ ] adminのみアクセス可能（403 Forbidden）
- [ ] 招待メールが送信される
- [ ] オペレーターレコードが作成される（status='pending_activation'）
- [ ] メールアドレス重複時に409 Conflictを返す

---

## Task 4: POST /api/admin/operators/[id]/resend-invitation API

_Requirements: 1.1_

### Description
招待メールを再送信するAPIを実装する。有効期限切れや未受信時に使用する。

### Files to Create/Modify
- `app/api/admin/operators/[id]/resend-invitation/route.ts` (create)

### Implementation Details
1. admin権限チェック
2. オペレーター存在確認（status='pending_activation'のみ）
3. `admin.inviteUserByEmail()` で招待メール再送信
4. `operators.invited_at` を更新

### Acceptance Criteria
- [ ] pending_activationステータスのオペレーターのみ再送信可能
- [ ] 招待メールが再送信される
- [ ] invited_atが更新される

---

## Task 5: GET /api/admin/operators API

_Requirements: 2.1, 2.2, 2.3, 2.4_

### Description
オペレーター一覧取得APIを実装する。フィルタリング、検索、ページネーション、統計表示をサポートする。

### Files to Create/Modify
- `app/api/admin/operators/route.ts` (modify)

### Implementation Details
1. admin権限チェック
2. クエリパラメータ解析
   - `subscription_status`: ステータスフィルター
   - `is_active`: 有効状態フィルター
   - `search`: 会社名検索（部分一致）
   - `page`, `limit`: ページネーション
3. OperatorService.getOperators()を呼び出し
4. レスポンス整形（operators配列、pagination情報）

### Acceptance Criteria
- [ ] 一覧が取得できる
- [ ] ステータスでフィルタリングできる
- [ ] 会社名で検索できる
- [ ] ページネーションが機能する
- [ ] 各オペレーターに統計（物件数、顧客数）が含まれる

---

## Task 6: GET/PATCH/DELETE /api/admin/operators/[id] API

_Requirements: 3.1, 3.2, 3.3, 3.4, 4.1, 4.2, 4.3_

### Description
オペレーター詳細・更新・削除APIを実装する。

### Files to Create/Modify
- `app/api/admin/operators/[id]/route.ts` (create)

### Implementation Details
1. GET: 詳細取得
   - admin権限チェック
   - OperatorService.getOperatorById()
   - 統計情報を含む
2. PATCH: 更新
   - admin権限チェック
   - バリデーション
   - OperatorService.updateOperator()
   - サブスクリプション状態変更ログ
3. DELETE: 削除
   - admin権限チェック
   - CASCADE削除確認
   - OperatorService.deleteOperator()
   - 監査ログ記録

### Acceptance Criteria
- [ ] 詳細が取得できる（統計含む）
- [ ] 企業情報を更新できる
- [ ] サブスクリプション状態を変更できる
- [ ] is_activeをfalseに設定できる
- [ ] 削除時に関連データもCASCADE削除される
- [ ] 存在しないIDに対して404 Not Foundを返す

---

## Task 7: GET /api/admin/stats API (P)

_Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

### Description
システム統計APIを実装する。

### Files to Create/Modify
- `app/api/admin/stats/route.ts` (create)

### Implementation Details
1. admin権限チェック
2. StatsService.getSystemStats()を呼び出し
3. レスポンス整形
   - オペレーター数（状態別）
   - 物件数（全体、公開中）
   - 顧客数
   - 会話数（全体、直近30日）

### Acceptance Criteria
- [ ] 統計が取得できる
- [ ] 状態別オペレーター数が含まれる
- [ ] 直近30日の会話数が含まれる

---

## Task 8: OperatorListコンポーネント

_Requirements: 2.1, 2.2, 2.3, 2.4_

### Description
オペレーター一覧テーブルコンポーネントを実装する。TanStack Tableを使用してソート、フィルター、検索、ページネーションをサポートする。

### Files to Create/Modify
- `src/components/features/admin/OperatorList.tsx` (create)
- `src/components/features/admin/OperatorFilter.tsx` (create)

### Subtasks
1. OperatorListコンポーネント（1-2h）
   - TanStack Table設定
   - カラム定義（会社名、ステータス、プラン、物件数、顧客数、作成日）
   - ソート機能
   - ページネーション
   - アクションボタン（詳細、編集、削除）
2. OperatorFilterコンポーネント（1-2h）
   - ステータス選択（セレクト）
   - 有効状態フィルター（チェックボックス）
   - 会社名検索（テキスト入力）
   - フィルターリセット

### Acceptance Criteria
- [ ] テーブルが表示される
- [ ] ソートが機能する（会社名、作成日、ステータス）
- [ ] ステータスでフィルタリングできる
- [ ] 会社名で検索できる
- [ ] ページネーションが機能する
- [ ] 統計（物件数、顧客数）が表示される

---

## Task 9: OperatorFormコンポーネント

_Requirements: 1.1, 1.2, 1.3, 3.2, 3.3, 3.4_

### Description
オペレーター登録・編集フォームコンポーネントを実装する。招待メール方式に対応し、パスワード入力フィールドは不要。

### Files to Create/Modify
- `src/components/features/admin/OperatorForm.tsx` (create)

### Subtasks
1. フォーム基本構造（1-2h）
   - React Hook Form + Zod
   - 新規/編集モード切り替え
   - フィールド定義（email, company_name, company_address, phone, subscription_plan, contract_start_date, is_active）
   - パスワードフィールドは不要（招待メール方式）
2. サブスクリプション管理（1h）
   - プラン選択（basic, standard, premium）
   - ステータス選択（active, suspended, cancelled, trial, pending_activation）
   - 契約期間設定
3. バリデーション・送信（1h）
   - クライアント側バリデーション
   - API呼び出し
   - 成功/エラーハンドリング
   - トーストメッセージ

### Acceptance Criteria
- [ ] フォームが表示される
- [ ] パスワードフィールドが存在しない
- [ ] バリデーションが機能する
- [ ] 新規登録時に招待メールが送信される
- [ ] 編集モードで既存データが表示される
- [ ] 送信成功時にトーストメッセージを表示する

---

## Task 10: SystemStatsコンポーネント (P)

_Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

### Description
システム統計ダッシュボードコンポーネントを実装する。カード形式の数値表示とグラフ表示を行う。

### Files to Create/Modify
- `src/components/features/admin/SystemStats.tsx` (create)
- `src/components/features/admin/StatsCard.tsx` (create)
- `src/components/features/admin/OperatorChart.tsx` (create)

### Subtasks
1. StatsCardコンポーネント（1h）
   - カード形式の数値表示
   - アイコン付き
   - 前期比較（オプション）
2. OperatorChartコンポーネント（2h）
   - Rechartsでグラフ表示
   - 状態別オペレーター数（円グラフ）
   - オペレーター別物件数ランキング（棒グラフ）
3. SystemStatsコンポーネント（1-2h）
   - StatsCard配置
   - OperatorChart配置
   - データ取得（useQuery）
   - ローディング状態

### Acceptance Criteria
- [ ] 統計カードが表示される
- [ ] グラフが表示される
- [ ] リアルタイムで更新される
- [ ] レスポンシブ対応

---

## Task 11: 管理者ダッシュボードページ

_Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

### Description
管理者向けダッシュボードページを実装する。SystemStatsと最近のオペレーター一覧を表示する。

### Files to Create/Modify
- `app/(admin)/dashboard/page.tsx` (create)
- `app/(admin)/layout.tsx` (create)

### Implementation Details
1. 認証ガード（adminロールチェック）
2. SystemStats表示
3. 最近のオペレーター表示（最大5件）
4. クイックアクション（新規オペレーター登録）

### Acceptance Criteria
- [ ] ダッシュボードが表示される
- [ ] adminのみアクセス可能（非adminは403）
- [ ] 統計が表示される
- [ ] 最近のオペレーターが表示される

---

## Task 12: オペレーター管理ページ

_Requirements: 1.1, 2.1, 3.1, 4.1_

### Description
オペレーター一覧・詳細・編集・削除ページを実装する。

### Files to Create/Modify
- `app/(admin)/operators/page.tsx` (create)
- `app/(admin)/operators/[id]/page.tsx` (create)
- `app/(admin)/operators/create/page.tsx` (create)

### Subtasks
1. 一覧ページ（1h）
   - OperatorList配置
   - OperatorFilter配置
   - 新規作成ボタン
2. 詳細・編集ページ（2h）
   - OperatorForm配置（編集モード）
   - 詳細情報表示
   - 削除ボタン
   - 招待メール再送信ボタン（pending_activationのみ）
3. 新規作成ページ（1h）
   - OperatorForm配置（新規モード）
4. 削除確認ダイアログ（1h）
   - 確認メッセージ
   - CASCADE警告
   - 削除実行

### Acceptance Criteria
- [ ] 一覧が表示される
- [ ] 詳細が表示される
- [ ] 編集ができる
- [ ] 削除ができる
- [ ] 削除時に確認ダイアログが表示される
- [ ] pending_activationステータスの場合、招待メール再送信ボタンが表示される

---

## Task 13: 招待メール受信フロー対応

_Requirements: 1.4_

### Description
オペレーターが招待メールを受信してパスワードを設定するフローを実装する。

### Files to Create/Modify
- `app/auth/accept-invitation/page.tsx` (create)

### Implementation Details
1. 招待リンクのトークンを検証
2. パスワード設定フォーム表示
3. パスワード設定後にログイン
4. `operators.status` を 'active' に更新
5. `operators.activated_at` を記録

### Acceptance Criteria
- [ ] 招待リンクからパスワード設定ページに遷移できる
- [ ] パスワードを設定できる
- [ ] 設定後に自動的にログインされる
- [ ] ステータスが 'active' に更新される

---

## Task 14: 招待メールテンプレート

_Requirements: 1.1_

### Description
招待メールのHTMLテンプレートを作成する（Resend使用）。

### Files to Create/Modify
- `src/emails/operator-invitation.tsx` (create)

### Implementation Details
1. React Email コンポーネント
2. 招待メッセージ
3. パスワード設定リンク
4. 有効期限（24時間）の表示
5. ブランディング

### Acceptance Criteria
- [ ] メールテンプレートが表示される
- [ ] パスワード設定リンクが機能する
- [ ] デザインがブランドに沿っている

---

## Task 15: useOperators Hook (P)

_Requirements: 2.1_

### Description
オペレーターデータ取得のカスタムフックを実装する。

### Files to Create/Modify
- `src/lib/hooks/useOperators.ts` (create)
- `src/lib/hooks/useOperator.ts` (create)

### Implementation Details
1. TanStack Query使用
2. キャッシュ設定（5分）
3. フィルター変更時の再取得
4. 楽観的更新（mutation）

### Acceptance Criteria
- [ ] データが取得できる
- [ ] キャッシュが機能する
- [ ] フィルター変更時に再取得される
- [ ] mutation後にキャッシュが更新される

---

## Task 16: テスト実装 (P)

_Requirements: すべて_

### Description
オペレーター管理機能のユニットテスト・統合テストを実装する。

### Files to Create/Modify
- `src/lib/services/__tests__/operator-service.test.ts` (create)
- `src/lib/services/__tests__/stats-service.test.ts` (create)
- `app/api/admin/operators/__tests__/route.test.ts` (create)
- `app/api/admin/stats/__tests__/route.test.ts` (create)

### Subtasks
1. OperatorServiceテスト（2-3h）
   - createOperator（招待メール送信確認）
   - resendInvitation
   - getOperators（フィルター・検索）
   - updateOperator
   - deleteOperator
2. StatsServiceテスト（1-2h）
   - getSystemStats
   - getOperatorStats
3. APIルートテスト（2-3h）
   - 権限チェック
   - バリデーション
   - エラーハンドリング

### Acceptance Criteria
- [ ] サービスのテストがパスする
- [ ] APIのテストがパスする
- [ ] カバレッジが80%以上
- [ ] エッジケース（存在しないID、権限不足）がテストされる
