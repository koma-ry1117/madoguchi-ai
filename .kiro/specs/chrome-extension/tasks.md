# Implementation Tasks

## Task 1: 拡張機能プロジェクト構成

### Description
Chrome拡張機能のプロジェクト構造を作成

### Files to Create/Modify
- `extension/` ディレクトリ作成
- `extension/manifest.json` (create)
- `extension/package.json` (create)
- `extension/vite.config.ts` (create)
- `extension/tsconfig.json` (create)

### Implementation Details
1. Manifest V3形式
2. Vite + React設定
3. TypeScript設定

### Acceptance Criteria
- [ ] プロジェクト構造が作成される
- [ ] ビルドが通る

---

## Task 2: Content Script実装

### Description
物件ページからHTMLを取得するContent Script

### Files to Create/Modify
- `extension/src/content/content.ts` (create)
- `extension/src/content/site-detector.ts` (create)

### Implementation Details
1. 対応サイト判定（アットホーム、SUUMO等）
2. document.outerHTML取得
3. メッセージリスナー設定

### Acceptance Criteria
- [ ] サイト判定が機能する
- [ ] HTMLが取得できる

---

## Task 3: Background Service Worker実装

### Description
認証管理とAPI通信を行うService Worker

### Files to Create/Modify
- `extension/src/background/background.ts` (create)
- `extension/src/background/auth.ts` (create)
- `extension/src/background/api.ts` (create)

### Implementation Details
1. Chrome Storage での認証状態保持
2. Supabase Auth連携
3. /api/properties/import呼び出し
4. メッセージハンドリング

### Acceptance Criteria
- [ ] 認証状態が保持される
- [ ] API呼び出しが機能する

---

## Task 4: Popup UI実装

### Description
取り込みボタンと状態表示のPopup

### Files to Create/Modify
- `extension/src/popup/Popup.tsx` (create)
- `extension/src/popup/LoginForm.tsx` (create)
- `extension/src/popup/ImportButton.tsx` (create)
- `extension/src/popup/StatusMessage.tsx` (create)
- `extension/popup.html` (create)

### Implementation Details
1. React + shadcn/ui
2. 認証状態に応じた表示切り替え
3. ローディング状態表示
4. 成功/エラーメッセージ

### Acceptance Criteria
- [ ] ログインフォームが表示される
- [ ] 取り込みボタンが機能する
- [ ] 状態が正しく表示される

---

## Task 5: POST /api/properties/import API実装

### Description
物件取り込みAPIエンドポイント

### Files to Create/Modify
- `app/api/properties/import/route.ts` (create)

### Implementation Details
1. 認証検証（operator/admin のみ）
2. HTMLサイズ検証（5MB上限）
3. GPT-4o generateObject呼び出し
4. OpenAI Embeddings生成
5. properties テーブル INSERT
6. property_import_logs INSERT
7. Supabase Storage にHTML保存

### Acceptance Criteria
- [ ] 認証が機能する
- [ ] 構造化データが抽出される
- [ ] DBに保存される
- [ ] HTMLがStorageに保存される

---

## Task 6: GPT-4o構造化抽出ロジック

### Description
HTMLから物件情報を抽出するAIロジック

### Files to Create/Modify
- `src/lib/ai/property-extractor.ts` (create)

### Implementation Details
1. Zodスキーマ定義
2. generateObject設定
3. サイト別プロンプト最適化

### Acceptance Criteria
- [ ] アットホームのHTMLから抽出できる
- [ ] SUUMOのHTMLから抽出できる

---

## Task 7: 取り込みログ管理

### Description
取り込み履歴の保存と表示

### Files to Create/Modify
- `src/lib/services/import-log-service.ts` (create)
- `app/(operator)/import/history/page.tsx` (create)

### Implementation Details
1. 取り込みログ保存
2. 履歴一覧表示
3. HTMLスナップショットダウンロード

### Acceptance Criteria
- [ ] ログが保存される
- [ ] 履歴が確認できる

---

## Task 8: 拡張機能ビルド設定

### Description
本番用ビルドとパッケージング

### Files to Create/Modify
- `extension/scripts/build.ts` (create)
- `extension/scripts/package.ts` (create)

### Implementation Details
1. Vite本番ビルド
2. manifest.json処理
3. ZIPパッケージ作成

### Acceptance Criteria
- [ ] 本番ビルドが生成される
- [ ] Chromeに読み込める

---

## Task 9: テスト実装

### Description
Chrome拡張機能のテスト

### Files to Create/Modify
- `extension/src/__tests__/site-detector.test.ts` (create)
- `app/api/properties/import/__tests__/route.test.ts` (create)
- `src/lib/ai/__tests__/property-extractor.test.ts` (create)

### Acceptance Criteria
- [ ] サイト判定のテストがパスする
- [ ] APIのテストがパスする
- [ ] 抽出ロジックのテストがパスする

---

## Task 10: ドキュメント作成

### Description
Chrome拡張機能の使用方法ドキュメント

### Files to Create/Modify
- `extension/README.md` (create)

### Implementation Details
1. インストール方法
2. 使用方法
3. トラブルシューティング

### Acceptance Criteria
- [ ] READMEが作成される
