# Implementation Tasks

## Task 1: OperatorService実装

### Description
オペレーター管理のビジネスロジック

### Files to Create/Modify
- `src/lib/services/operator-service.ts` (create)
- `src/schemas/operator.ts` (create)

### Implementation Details
1. createOperator: 新規登録（Auth + DB）
2. getOperators: 一覧取得（統計付き）
3. getOperatorById: 詳細取得
4. updateOperator: 更新
5. deleteOperator: 削除
6. Zodスキーマ定義

### Acceptance Criteria
- [ ] CRUD操作が機能する
- [ ] Supabase Auth連携が機能する

---

## Task 2: StatsService実装

### Description
システム統計の集計ロジック

### Files to Create/Modify
- `src/lib/services/stats-service.ts` (create)

### Implementation Details
1. getSystemStats: システム全体統計
2. getOperatorStats: オペレーター別統計
3. 効率的なクエリ（集計関数使用）

### Acceptance Criteria
- [ ] 統計が正しく集計される
- [ ] パフォーマンスが許容範囲

---

## Task 3: GET /api/admin/operators API

### Description
オペレーター一覧取得API

### Files to Create/Modify
- `app/api/admin/operators/route.ts` (create)

### Implementation Details
1. admin権限チェック
2. フィルタリング（status, is_active）
3. 検索（company_name）
4. ページネーション
5. 統計情報付与

### Acceptance Criteria
- [ ] 一覧が取得できる
- [ ] adminのみアクセス可能

---

## Task 4: POST /api/admin/operators API

### Description
オペレーター登録API

### Files to Create/Modify
- `app/api/admin/operators/route.ts` (modify)

### Implementation Details
1. admin権限チェック
2. バリデーション
3. Supabase Auth createUser
4. auth.users roleを'operator'に設定
5. operators INSERT

### Acceptance Criteria
- [ ] オペレーターが登録される
- [ ] ユーザーアカウントが作成される

---

## Task 5: GET/PATCH/DELETE /api/admin/operators/[id] API

### Description
オペレーター詳細・更新・削除API

### Files to Create/Modify
- `app/api/admin/operators/[id]/route.ts` (create)

### Implementation Details
1. GET: 詳細取得（統計含む）
2. PATCH: 更新（部分更新対応）
3. DELETE: 削除（CASCADE確認）

### Acceptance Criteria
- [ ] 詳細が取得できる
- [ ] 更新ができる
- [ ] 削除ができる

---

## Task 6: GET /api/admin/stats API

### Description
システム統計API

### Files to Create/Modify
- `app/api/admin/stats/route.ts` (create)

### Implementation Details
1. admin権限チェック
2. オペレーター数（状態別）
3. 物件数
4. 顧客数
5. 会話数（直近30日）

### Acceptance Criteria
- [ ] 統計が取得できる

---

## Task 7: OperatorListコンポーネント

### Description
オペレーター一覧テーブル

### Files to Create/Modify
- `src/components/features/admin/OperatorList.tsx` (create)
- `src/components/features/admin/OperatorFilter.tsx` (create)

### Implementation Details
1. TanStack Table使用
2. ソート・フィルター・検索
3. ページネーション
4. 統計表示列

### Acceptance Criteria
- [ ] テーブルが表示される
- [ ] ソート・フィルターが機能する

---

## Task 8: OperatorFormコンポーネント

### Description
オペレーター登録・編集フォーム

### Files to Create/Modify
- `src/components/features/admin/OperatorForm.tsx` (create)

### Implementation Details
1. React Hook Form + Zod
2. 新規/編集モード切り替え
3. サブスクリプション選択
4. ステータス選択

### Acceptance Criteria
- [ ] フォームが表示される
- [ ] バリデーションが機能する

---

## Task 9: SystemStatsコンポーネント

### Description
システム統計ダッシュボード

### Files to Create/Modify
- `src/components/features/admin/SystemStats.tsx` (create)
- `src/components/features/admin/StatsCard.tsx` (create)
- `src/components/features/admin/OperatorChart.tsx` (create)

### Implementation Details
1. カード形式の数値表示
2. Rechartsでグラフ表示
3. オペレーター別ランキング

### Acceptance Criteria
- [ ] 統計が表示される
- [ ] グラフが表示される

---

## Task 10: 管理者ダッシュボードページ

### Description
管理者向けダッシュボードページ

### Files to Create/Modify
- `app/(admin)/dashboard/page.tsx` (create)
- `app/(admin)/layout.tsx` (create)

### Implementation Details
1. 認証ガード（adminのみ）
2. SystemStats表示
3. 最近のオペレーター表示

### Acceptance Criteria
- [ ] ダッシュボードが表示される
- [ ] adminのみアクセス可能

---

## Task 11: オペレーター管理ページ

### Description
オペレーター一覧・詳細・編集ページ

### Files to Create/Modify
- `app/(admin)/operators/page.tsx` (create)
- `app/(admin)/operators/[id]/page.tsx` (create)
- `app/(admin)/operators/create/page.tsx` (create)

### Implementation Details
1. 一覧ページ（OperatorList）
2. 詳細・編集ページ（OperatorForm）
3. 新規作成ページ
4. 削除確認ダイアログ

### Acceptance Criteria
- [ ] 一覧が表示される
- [ ] 詳細が表示される
- [ ] 編集ができる
- [ ] 削除ができる

---

## Task 12: テスト実装

### Description
オペレーター管理機能のテスト

### Files to Create/Modify
- `src/lib/services/__tests__/operator-service.test.ts` (create)
- `src/lib/services/__tests__/stats-service.test.ts` (create)
- `app/api/admin/__tests__/operators.test.ts` (create)

### Acceptance Criteria
- [ ] サービスのテストがパスする
- [ ] APIのテストがパスする
