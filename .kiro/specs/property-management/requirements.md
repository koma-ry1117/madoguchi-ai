# Requirements Document

## Introduction

物件の管理機能。オペレーターが登録した物件を管理・編集する。キオスクではAIが会話内で物件を検索・提案するため、顧客向けの物件一覧ページは存在しない。

**重要**: 顧客（キオスク利用者）は物件一覧ページにアクセスしない。物件検索はAI会話内でのみ行われる。

## Requirements

### Requirement 1: オペレーター向け物件一覧
**Objective:** As an operator, I want to browse my property listings, so that I can manage my portfolio

#### Acceptance Criteria
1. When オペレーターがダッシュボードにアクセス, システムは自社物件一覧を表示する
2. システムは物件画像、タイトル、価格、間取り、エリア、ステータスを表示する
3. システムはページネーション（20件/ページ）を提供する
4. システムは新着順・更新順でソートできる

### Requirement 2: 物件フィルター・検索（オペレーター向け）
**Objective:** As an operator, I want to filter properties, so that I can quickly find specific listings

#### Acceptance Criteria
1. When エリアフィルターを選択, システムはそのエリアの物件のみ表示する
2. When 予算範囲を指定, システムはその範囲内の物件のみ表示する
3. When 間取りを選択, システムはその間取りの物件のみ表示する
4. When 公開ステータスを選択, システムはそのステータスの物件のみ表示する
5. システムはキーワード検索をサポートする

### Requirement 3: 物件詳細・編集
**Objective:** As an operator, I want to view and edit property details, so that I can keep listings up to date

#### Acceptance Criteria
1. When 物件カードをクリック, システムは詳細ページを表示する
2. オペレーターは物件情報を編集できる
3. オペレーターは物件の公開/非公開を切り替えられる
4. オペレーターは物件を削除できる
5. システムは編集履歴を記録する

### Requirement 4: 物件登録（手動）
**Objective:** As an operator, I want to manually register properties, so that I can add listings not imported via Chrome extension

#### Acceptance Criteria
1. オペレーターは新規物件を手動で登録できる
2. システムは必須項目のバリデーションを行う
3. オペレーターは複数の画像をアップロードできる
4. システムは登録完了時に確認メッセージを表示する

### Requirement 5: AI会話用物件検索（内部API）
**Objective:** As a system, I want to search properties for AI conversation, so that the kiosk can propose properties to customers

#### Acceptance Criteria
1. システムはキオスク会話内でAIが物件を検索できるAPIを提供する
2. システムはoperator_idで自社物件のみをフィルタリングする
3. システムは公開物件のみを検索対象とする
4. システムはセマンティック検索（pgvector）をサポートする

### Requirement 6: マルチテナント分離
**Objective:** As a system, I want to separate data by operator, so that each operator only sees their properties

#### Acceptance Criteria
1. システムはRLSでオペレーターごとにデータを分離する
2. オペレーターは自社の物件のみ閲覧・編集可能
3. When 管理者がアクセス, システムは全オペレーターの物件を表示する

## ロール定義

| ロール | 説明 |
|--------|------|
| operator | 不動産会社（オペレーター） - 物件の管理者 |
| admin | システム管理者 - 全物件閲覧可能 |
| kiosk_user | キオスク利用者（認証不要） - 会話内で物件を検索（直接APIアクセスなし） |
