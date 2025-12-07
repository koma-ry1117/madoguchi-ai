# Requirements Document

## Introduction

システム管理者（gyam様）がオペレーター（不動産会社）を管理する機能。オペレーターの登録、契約管理、サブスクリプション管理、システム全体の統計表示を行う。

## Requirements

### Requirement 1: オペレーター登録
**Objective:** As an admin, I want to register new operators, so that real estate companies can use the system

#### Acceptance Criteria
1. 管理者は新規オペレーターを登録できる
2. 登録時に企業情報（会社名、住所、電話、メール）を入力する
3. 登録時にサブスクリプションプラン（basic, standard, premium）を選択する
4. システムはオペレーター用のユーザーアカウントを作成する

### Requirement 2: オペレーター一覧・検索
**Objective:** As an admin, I want to view and search operators, so that I can manage them efficiently

#### Acceptance Criteria
1. 管理者は全オペレーターの一覧を表示できる
2. システムはサブスクリプション状態（active, suspended, cancelled, trial）でフィルタリングできる
3. システムは会社名で検索できる
4. 一覧には物件数、顧客数などの統計を表示する

### Requirement 3: オペレーター詳細・編集
**Objective:** As an admin, I want to edit operator details, so that I can keep information up to date

#### Acceptance Criteria
1. 管理者はオペレーターの詳細情報を確認できる
2. 管理者は企業情報を編集できる
3. 管理者はサブスクリプション状態を変更できる
4. 管理者はオペレーターを無効化（is_active=false）できる

### Requirement 4: オペレーター削除
**Objective:** As an admin, I want to delete operators, so that I can remove inactive companies

#### Acceptance Criteria
1. 管理者はオペレーターを削除できる
2. 削除時に確認ダイアログを表示する
3. 削除時に関連データ（物件、顧客、会話）もCASCADE削除される

### Requirement 5: システム統計ダッシュボード
**Objective:** As an admin, I want to see system statistics, so that I can monitor overall usage

#### Acceptance Criteria
1. システムは全オペレーター数（状態別）を表示する
2. システムは全物件数を表示する
3. システムは全顧客数を表示する
4. システムは直近30日の会話数を表示する
5. システムはオペレーター別の統計を表示する

## ロール定義

| ロール | 説明 |
|--------|------|
| admin | システム管理者（gyam様） |
