# Tasks Document

## タスク一覧

### 1. データベーススキーマとRLS設定 _Requirements: 1, 2, 3, 4, 5, 6_
オペレーターごとに物件データを分離し、検索性能を確保するテーブルとインデックスを構築する。

- 1.1 物件テーブル（properties）とpgvector拡張を作成する _Requirements: 5_ (P)
  - propertiesテーブル定義（id, operator_id, title, address, area, transaction_type, rent, sale_price, layout, building_type, floor_area, station, distance_from_station, description, amenities, is_public, created_at, updated_at）
  - pgvectorエクステンションを有効化
  - セマンティック検索用のembeddingカラム（vector(1536)）を追加
  - 検索性能のためのインデックスを作成（operator_id, is_public, area, rent, layout）

- 1.2 物件画像テーブル（property_images）とストレージを設定する _Requirements: 4_ (P)
  - property_imagesテーブル定義（id, property_id, storage_path, display_order）
  - Supabase Storage バケット設定（properties-images）
  - 画像アップロード用のRLSポリシー設定

- 1.3 RLSポリシーを実装する _Requirements: 6_
  - オペレーター向けポリシー: 自社物件のみCRUD可能
  - 管理者向けポリシー: 全物件CRUD可能
  - テストデータでRLS動作を検証

### 2. API Routes実装 _Requirements: 1, 2, 3, 4, 5_
オペレーターが物件を管理し、キオスクAIが物件を検索できるAPIエンドポイントを実装する。

- 2.1 GET /api/properties を実装する _Requirements: 1, 2_ (P)
  - クエリパラメータによるフィルタリング（area, rent_min, rent_max, layout, building_type, keyword）
  - ページネーション（page, limit）
  - ソート機能（新着順・更新順）
  - RLSによるoperator_idフィルタリング

- 2.2 GET /api/properties/[id] を実装する _Requirements: 3_ (P)
  - IDによる物件詳細取得
  - 画像リレーションの取得
  - RLSチェック

- 2.3 POST /api/properties を実装する _Requirements: 4_
  - 物件登録エンドポイント
  - バリデーション（必須項目チェック）
  - operator_idの自動設定
  - 画像アップロード処理
  - トランザクション管理

- 2.4 PATCH /api/properties/[id] を実装する _Requirements: 3_
  - 物件更新エンドポイント
  - 部分更新サポート（title, rent, is_public等）
  - 編集履歴の記録
  - RLSチェック

- 2.5 DELETE /api/properties/[id] を実装する _Requirements: 3_ (P)
  - 物件削除エンドポイント
  - 関連画像の削除
  - ソフトデリート検討
  - RLSチェック

- 2.6 POST /api/kiosk/search を実装する _Requirements: 5_ (P)
  - キオスクAI会話用の物件検索API
  - operator_idフィルター（必須）
  - is_public=trueのみ検索対象
  - pgvectorによるセマンティック検索
  - Service Role Keyでアクセス（認証バイパス）

### 3. フロントエンド実装（オペレーター向け物件管理） _Requirements: 1, 2, 3, 4_
オペレーターが物件を一覧・検索・編集できるUIを実装する。

- 3.1 PropertyCardコンポーネントを作成する _Requirements: 1_ (P)
  - shadcn/uiベースのカードUI
  - 物件画像、タイトル、価格、間取り、エリア、ステータス表示
  - クリックイベントハンドラー
  - レスポンシブ対応

- 3.2 PropertyFilterコンポーネントを作成する _Requirements: 2_ (P)
  - エリア、予算範囲、間取り、公開ステータスのフィルターUI
  - キーワード検索入力フィールド
  - フィルター状態管理（useState）
  - フィルター変更時のコールバック

- 3.3 物件一覧ページ（/dashboard/properties）を実装する _Requirements: 1, 2_
  - PropertyCardを使用した一覧表示
  - PropertyFilterの統合
  - TanStack Queryによるデータフェッチ
  - ページネーションUI
  - ソート切り替え（新着順・更新順）
  - ローディング・エラー状態の処理

- 3.4 物件詳細ページ（/dashboard/properties/[id]）を実装する _Requirements: 3_
  - 物件詳細情報の表示
  - 画像ギャラリー（Next.js Image）
  - 編集ボタン・削除ボタン
  - 公開/非公開切り替えUI
  - TanStack Queryによるデータフェッチ

- 3.5 物件編集フォーム（PropertyEditor）を実装する _Requirements: 3, 4_
  - 全フィールドの編集フォーム
  - バリデーション（React Hook Form + Zod）
  - 画像アップロードUI（複数画像対応、ドラッグ&ドロップ）
  - プレビュー機能
  - 保存・キャンセルボタン
  - TanStack Queryによる楽観的更新

- 3.6 物件登録ページ（/dashboard/properties/new）を実装する _Requirements: 4_
  - PropertyEditorを再利用
  - 新規登録モード
  - 登録完了メッセージ
  - 登録後に詳細ページへリダイレクト

### 4. テスト実装 _Requirements: 1, 2, 3, 4, 5, 6_
各機能の品質を保証するテストを実装する。

- 4.1 APIルートのインテグレーションテストを作成する (P)
  - GET /api/properties: フィルタリング、ページネーション、RLS
  - POST /api/properties: バリデーション、画像アップロード
  - PATCH /api/properties/[id]: 更新、RLS
  - DELETE /api/properties/[id]: 削除、RLS
  - POST /api/kiosk/search: セマンティック検索、operator_idフィルター

- 4.2 フロントエンドコンポーネントのユニットテストを作成する (P)
  - PropertyCard: 表示内容、クリックイベント
  - PropertyFilter: フィルター変更、バリデーション
  - PropertyEditor: フォームバリデーション、画像アップロード

- 4.3 E2Eテストを作成する
  - オペレーター: 物件一覧 → フィルター適用 → 詳細表示
  - オペレーター: 物件登録 → 編集 → 削除
  - マルチテナント: オペレーターAがオペレーターBの物件を見れないことを確認
  - 管理者: 全物件閲覧可能を確認

### 5. パフォーマンス最適化 _Requirements: 1, 5_
検索性能と画像読み込みを最適化する。

- 5.1 データベースインデックスの最適化 _Requirements: 1, 2, 5_ (P)
  - 複合インデックスの作成（area, rent, layout）
  - pgvectorインデックスの作成（IVFFlatまたはHNSW）
  - EXPLAIN ANALYZEで性能検証

- 5.2 画像最適化を実装する _Requirements: 1, 4_ (P)
  - Next.js Imageコンポーネントの適用
  - WebP変換（Supabase Storage設定）
  - 遅延読み込み（lazy loading）
  - サムネイル生成（一覧用）

- 5.3 TanStack Queryキャッシュ戦略を設定する _Requirements: 1, 2, 3_
  - staleTime、cacheTimeの調整
  - 楽観的更新（Optimistic Updates）
  - プリフェッチ（ページネーション先読み）

### 6. ドキュメント・デプロイ準備
実装内容を文書化し、デプロイ準備を行う。

- 6.1 API仕様書を作成する (P)
  - OpenAPI/Swaggerドキュメント
  - リクエスト/レスポンス例
  - エラーコード一覧

- 6.2 運用ドキュメントを作成する (P)
  - オペレーター向け操作マニュアル
  - 管理者向け運用ガイド
  - トラブルシューティング

- 6.3 マイグレーションスクリプトを準備する
  - テーブル作成SQLスクリプト
  - インデックス作成スクリプト
  - RLSポリシー設定スクリプト
  - ロールバック手順

---

## タスク実行順序の推奨

### Phase 1: データベース基盤（並列可）
- 1.1, 1.2 → 1.3

### Phase 2: API実装（部分的に並列可）
- 2.1, 2.2 → 2.3, 2.4, 2.5, 2.6

### Phase 3: フロントエンド実装
- 3.1, 3.2 → 3.3 → 3.4 → 3.5 → 3.6

### Phase 4: テスト・最適化（並列可）
- 4.1, 4.2 → 4.3
- 5.1, 5.2, 5.3

### Phase 5: ドキュメント（並列可）
- 6.1, 6.2, 6.3
