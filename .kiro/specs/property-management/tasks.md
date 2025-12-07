# Implementation Tasks

## Task 1: PropertyService実装

### Description
物件データアクセスのサービスレイヤー

### Files to Create/Modify
- `src/lib/services/property-service.ts` (create)
- `src/schemas/property.ts` (create)

### Implementation Details
1. getProperties: 一覧取得（フィルタリング対応）
2. getPropertyById: 詳細取得
3. updateProperty: 更新（オペレーター用）
4. deleteProperty: 削除（オペレーター用）
5. Zodスキーマ定義

### Acceptance Criteria
- [ ] CRUD操作が機能する
- [ ] RLSが適用される

---

## Task 2: GET /api/properties APIルート

### Description
物件一覧取得API（フィルタリング・ページネーション対応）

### Files to Create/Modify
- `app/api/properties/route.ts` (create)

### Implementation Details
1. クエリパラメータ解析
2. Supabaseクエリ構築
3. ページネーション計算
4. レスポンス整形

### Acceptance Criteria
- [ ] フィルターが機能する
- [ ] ページネーションが機能する

---

## Task 3: GET /api/properties/[id] APIルート

### Description
物件詳細取得API

### Files to Create/Modify
- `app/api/properties/[id]/route.ts` (create)

### Implementation Details
1. IDで物件取得
2. 画像情報を含めて取得
3. RLSでアクセス制御

### Acceptance Criteria
- [ ] 詳細が取得できる
- [ ] 画像が含まれる

---

## Task 4: PATCH /api/properties/[id] APIルート

### Description
物件更新API（オペレーター用）

### Files to Create/Modify
- `app/api/properties/[id]/route.ts` (modify)

### Implementation Details
1. 認証・認可チェック
2. バリデーション
3. 更新実行

### Acceptance Criteria
- [ ] オペレーターが更新できる
- [ ] 他社の物件は更新不可

---

## Task 5: useProperties Hook

### Description
物件データ取得のカスタムフック

### Files to Create/Modify
- `src/lib/hooks/useProperties.ts` (create)
- `src/lib/hooks/useProperty.ts` (create)

### Implementation Details
1. TanStack Query使用
2. キャッシュ設定（5分）
3. フィルター変更時の再取得

### Acceptance Criteria
- [ ] データが取得できる
- [ ] キャッシュが機能する

---

## Task 6: PropertyCardコンポーネント

### Description
物件カード表示コンポーネント

### Files to Create/Modify
- `src/components/features/properties/PropertyCard.tsx` (create)

### Implementation Details
1. サムネイル画像表示
2. タイトル、価格、間取り表示
3. エリア、駅徒歩表示
4. クリックハンドラー

### Acceptance Criteria
- [ ] 情報が正しく表示される
- [ ] クリックで詳細に遷移

---

## Task 7: PropertyListコンポーネント

### Description
物件一覧表示コンポーネント

### Files to Create/Modify
- `src/components/features/properties/PropertyList.tsx` (create)

### Implementation Details
1. グリッドレイアウト
2. ローディング状態
3. 空状態
4. ページネーションUI

### Acceptance Criteria
- [ ] 一覧が表示される
- [ ] レスポンシブ対応

---

## Task 8: PropertyFilterコンポーネント

### Description
検索フィルターコンポーネント

### Files to Create/Modify
- `src/components/features/properties/PropertyFilter.tsx` (create)

### Implementation Details
1. エリア選択（セレクト）
2. 予算範囲（スライダー）
3. 間取り選択（チェックボックス）
4. 建物種別選択

### Acceptance Criteria
- [ ] フィルターが変更できる
- [ ] 即座に反映される

---

## Task 9: PropertyDetailコンポーネント

### Description
物件詳細表示コンポーネント

### Files to Create/Modify
- `src/components/features/properties/PropertyDetail.tsx` (create)
- `src/components/features/properties/PropertyGallery.tsx` (create)
- `src/components/features/properties/PropertyInfo.tsx` (create)

### Implementation Details
1. 画像ギャラリー（スライダー）
2. 詳細情報表示
3. 設備一覧
4. アクセス情報
5. アクションボタン（内見予約、問い合わせ）

### Acceptance Criteria
- [ ] 詳細が表示される
- [ ] ギャラリーが動作する

---

## Task 10: 顧客向け物件ページ

### Description
顧客向けの物件一覧・詳細ページ

### Files to Create/Modify
- `app/(customer)/properties/page.tsx` (create)
- `app/(customer)/properties/[id]/page.tsx` (create)

### Implementation Details
1. PropertyList + PropertyFilter配置
2. 詳細ページレイアウト
3. 認証ガード

### Acceptance Criteria
- [ ] ページが表示される
- [ ] 認証が必要

---

## Task 11: オペレーター向け物件管理ページ

### Description
オペレーター向けの物件管理ダッシュボード

### Files to Create/Modify
- `app/(operator)/properties/page.tsx` (create)
- `app/(operator)/properties/[id]/edit/page.tsx` (create)
- `src/components/features/properties/PropertyTable.tsx` (create)
- `src/components/features/properties/PropertyEditor.tsx` (create)

### Implementation Details
1. テーブル形式の一覧
2. 編集フォーム
3. 公開/非公開切り替え
4. 削除機能

### Acceptance Criteria
- [ ] 一覧が表示される
- [ ] 編集ができる
- [ ] 削除ができる

---

## Task 12: テスト実装

### Description
物件管理機能のテスト

### Files to Create/Modify
- `src/lib/services/__tests__/property-service.test.ts` (create)
- `app/api/properties/__tests__/route.test.ts` (create)

### Acceptance Criteria
- [ ] サービスのテストがパスする
- [ ] APIのテストがパスする
