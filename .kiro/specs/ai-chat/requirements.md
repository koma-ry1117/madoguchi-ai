# Requirements Document

## Introduction

不動産店舗に設置するAI接客システム。顧客はAIアバターとの会話のみで物件検索から内見予約までを完結する。営業担当者が不在でも24時間顧客対応が可能。最後に連絡先を入力してもらい、実際の従業員が後続対応を行う。

## 利用シーン

- 不動産店舗のタブレット/PCに常時表示
- 営業担当者が留守・接客中でも顧客対応可能
- 顧客は会話だけで操作（設定画面等は不要）
- 内見予約完了後、連絡先入力で終了

## Requirements

### Requirement 1: AI会話インターフェース
**Objective:** As a customer, I want to talk with an AI assistant, so that I can find properties without staff help

#### Acceptance Criteria
1. When 画面にアクセス, システムはAIアバターが挨拶して会話を開始する
2. システムは音声入力とテキスト入力の両方をサポートする
3. When AIが応答, システムは音声で読み上げる（高齢者対応）
4. システムは会話履歴を画面に表示する
5. When 一定時間操作がない場合, システムは初期画面にリセットする

### Requirement 2: 会話による物件検索
**Objective:** As a customer, I want to search properties through conversation, so that I can find homes matching my needs

#### Acceptance Criteria
1. AIは会話を通じて顧客の希望条件をヒアリングする（エリア、予算、間取り等）
2. When 条件が十分に集まった, AIは物件を検索して提案する
3. AIは物件カードを会話内に表示して説明する
4. When 顧客が物件に興味を示した, AIは詳細情報を説明する
5. When 条件に合う物件がない, AIは条件の緩和を提案する

### Requirement 3: 会話による内見予約
**Objective:** As a customer, I want to book a viewing through conversation, so that I can see properties in person

#### Acceptance Criteria
1. When 顧客が内見を希望, AIは希望日時を会話で確認する
2. AIは予約内容を確認して顧客に復唱する
3. When 予約確定, システムは連絡先入力フォームを表示する

### Requirement 4: 顧客情報収集
**Objective:** As a system, I want to collect customer contact info, so that staff can follow up

#### Acceptance Criteria
1. When 内見予約が確定, システムは連絡先入力画面を表示する
2. 入力項目: 名前、電話番号、メールアドレス
3. システムは入力バリデーションを行う（電話番号形式、メール形式）
4. When 入力完了, システムは確認メールを送信する
5. When 入力完了, システムはオペレーターに通知メールを送信する
6. システムは顧客情報と予約情報をDBに保存する

### Requirement 5: セッション完了とリセット
**Objective:** As a system, I want to reset after completion, so that the next customer can use it

#### Acceptance Criteria
1. When 予約完了, システムは「ありがとうございました」画面を表示する
2. 完了画面には予約内容サマリーを表示する
3. When 完了画面で一定時間経過 or 「終了」ボタン, システムは初期画面にリセットする
4. リセット時、会話履歴と入力情報はクリアする

### Requirement 6: アバター感情表現
**Objective:** As a customer, I want to see natural avatar expressions, so that I feel comfortable talking

#### Acceptance Criteria
1. システムは5種類の表情（neutral, happy, thinking, apologetic, excited）を切り替える
2. AIの応答内容に応じて適切な表情を自動選択する
3. 表情切り替えはアニメーションで滑らかに行う

### Requirement 7: 予約なし終了
**Objective:** As a customer, I want to leave without booking, so that I can just browse properties

#### Acceptance Criteria
1. 顧客は会話途中でも「終了」できる
2. When 予約せず終了, システムは「またのご来店をお待ちしています」と表示する
3. 予約なし終了時も会話ログはDBに保存する（分析用）

## ロール定義

| ロール | 説明 |
|--------|------|
| customer | 店舗来店客（ログイン不要） |
| operator | 不動産会社（オペレーター） |
