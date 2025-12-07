# Requirements Document

## Introduction

内見予約の管理機能。キオスクでの会話を通じて作成された内見予約をオペレーターが管理・対応する。予約作成時にはメール通知とカレンダー招待を送信する。

**重要**: 顧客は会話（AI）を通じてのみ予約を作成する。顧客向けの予約管理画面は存在しない。

## 利用シーン

- キオスクでAIと会話した顧客が内見を予約
- 予約完了時に顧客の連絡先を収集
- オペレーターが予約を確認・管理
- 実際の内見対応はオペレーター（人間）が行う

## Requirements

### Requirement 1: 予約自動作成（キオスク連携）
**Objective:** As a system, I want to create viewing reservations from kiosk conversations, so that customer bookings are recorded

#### Acceptance Criteria
1. When キオスク会話で内見予約が確定, システムは予約を「confirmed」ステータスで作成する
2. システムは顧客情報（名前、電話番号、メールアドレス）を保存する
3. システムは会話で選択された物件と希望日時を保存する
4. システムは会話履歴（conversation_id）と予約を紐付ける

### Requirement 2: 予約確定メール送信
**Objective:** As a customer, I want to receive confirmation, so that I know my viewing is scheduled

#### Acceptance Criteria
1. When 予約が作成, システムは顧客に確認メールを送信する
2. システムはオペレーターに予約通知メールを送信する
3. メールにはカレンダー招待（.icsファイル）を添付する
4. メールはReact Emailでデザインされる

### Requirement 3: 内見予約管理（オペレーター向け）
**Objective:** As an operator, I want to manage viewings, so that I can organize property showings

#### Acceptance Criteria
1. オペレーターは自社の予約一覧を確認できる
2. オペレーターは予約ステータス（confirmed, completed, cancelled, no_show）を更新できる
3. オペレーターは担当者をアサインできる
4. システムは日付・ステータスでフィルタリングできる
5. オペレーターは予約詳細で会話履歴を確認できる

### Requirement 4: 予約キャンセル（オペレーター操作）
**Objective:** As an operator, I want to cancel viewings, so that I can handle schedule changes

#### Acceptance Criteria
1. オペレーターは予約をキャンセルできる
2. When キャンセル, システムはキャンセル理由を記録する
3. システムは顧客にキャンセル通知メールを送信する

### Requirement 5: 内見リマインダー
**Objective:** As a customer, I want to receive reminders, so that I don't forget my viewing

#### Acceptance Criteria
1. システムは内見日の前日にリマインダーメールを送信する
2. リマインダーには物件情報と日時を含む
3. リマインダーにはオペレーターの連絡先を含む

## ロール定義

| ロール | 説明 |
|--------|------|
| kiosk_user | キオスク端末を利用する店舗来店客（認証不要、会話のみで予約） |
| operator | 不動産会社（オペレーター）、予約管理を行う |
