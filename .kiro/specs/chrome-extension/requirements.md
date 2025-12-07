# Requirements Document

## Introduction

アットホーム等の不動産ポータルサイトから物件情報をワンクリックで取り込むChrome拡張機能。GPT-4oで構造化データを抽出し、法的証跡としてHTMLスナップショットも保存する。

## Requirements

### Requirement 1: 物件ページ検出
**Objective:** As an operator, I want the extension to detect property pages, so that I can import listings easily

#### Acceptance Criteria
1. When アットホームの物件詳細ページを開く, 拡張機能アイコンがアクティブになる
2. システムは対応サイト（アットホーム、SUUMO等）を検出する
3. When 非対応サイト, 拡張機能アイコンがグレーアウトする

### Requirement 2: ワンクリック取り込み
**Objective:** As an operator, I want to import properties with one click, so that I can save time

#### Acceptance Criteria
1. When 拡張機能アイコンをクリック, ポップアップが表示される
2. When 「取り込み」ボタンをクリック, システムはページHTMLを取得する
3. システムはAPIにHTML+URLを送信する
4. When 取り込み完了, システムは成功通知を表示する

### Requirement 3: 構造化データ抽出
**Objective:** As a system, I want to extract structured data from HTML, so that properties can be searchable

#### Acceptance Criteria
1. システムはGPT-4oでHTMLから物件情報を抽出する
2. 抽出情報: タイトル、住所、エリア、賃料、間取り、建物種別、駅、駅徒歩、設備等
3. システムはOpenAI Embeddingsでベクトル埋め込みを生成する

### Requirement 4: HTMLスナップショット保存
**Objective:** As a system, I want to save HTML snapshots, so that I have legal evidence of the import source

#### Acceptance Criteria
1. システムは取り込み元HTMLをSupabase Storageに保存する
2. システムは取り込みログ（URL、日時、担当者）をDBに記録する
3. オペレーターは取り込み履歴を確認できる

### Requirement 5: 認証連携
**Objective:** As an operator, I want to authenticate with the extension, so that imports are associated with my account

#### Acceptance Criteria
1. When 未認証, 拡張機能はログインを促す
2. システムはSupabase Auth JWTを使用する
3. システムは認証状態を保持する

## ロール定義

| ロール | 説明 |
|--------|------|
| operator | 不動産会社（オペレーター） |
| admin | システム管理者 |
