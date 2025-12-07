# Requirements Document

## Introduction

キオスク端末における音声入力（Speech-to-Text）と音声出力（Text-to-Speech）機能。高齢者でも使いやすい音声インターフェースを提供し、自然な会話体験を実現する。

## 利用シーン

- 不動産店舗のキオスク端末での利用
- 高齢者や文字入力が苦手な顧客向け
- ハンズフリーでの物件検索・内見予約

## Requirements

### Requirement 1: 音声入力（Speech-to-Text）
**Objective:** As a kiosk user, I want to speak to the AI, so that I can search properties without typing

#### Acceptance Criteria
1. When マイクボタンをクリック, システムは音声録音を開始する
2. When 録音中, システムはリアルタイムで音声波形を表示する
3. When 録音停止, システムはOpenAI Whisper APIで音声をテキスト変換する
4. When 変換完了, システムはテキストをAIに送信する
5. システムはWebM/MP3形式の音声をサポートする
6. システムは録音時間制限（最大60秒）を設ける

### Requirement 2: 音声出力（Text-to-Speech）
**Objective:** As a kiosk user, I want to hear the AI's response, so that I can have a natural conversation

#### Acceptance Criteria
1. When AIが応答完了, システムはElevenLabs APIで音声を生成する
2. When 音声生成完了, システムは自動再生する
3. システムは日本語の自然な音声を生成する
4. When 同じテキストを再度読み上げる場合, システムはキャッシュから音声を取得する
5. システムは読み上げ中にアバターの口をアニメーションする

### Requirement 3: 音声キャッシング
**Objective:** As a system, I want to cache audio files, so that I can reduce API costs and latency

#### Acceptance Criteria
1. システムはテキストのハッシュ値をキーとしてSupabase Storageに音声を保存する
2. When キャッシュが存在, システムはAPIを呼び出さずにキャッシュから返す
3. システムは一定期間後にキャッシュを自動削除する（30日）
4. システムはよく使うフレーズ（挨拶等）を事前にキャッシュする

### Requirement 4: 音声コントロール
**Objective:** As a kiosk user, I want to control audio playback, so that I can pause or skip audio

#### Acceptance Criteria
1. システムは再生/一時停止ボタンを提供する
2. システムは音量調整機能を提供する（キオスク設置時に設定）
3. When 新しい応答が来た場合, 前の音声は停止する
4. When ユーザーが話し始めた場合, 再生中の音声は停止する

## ロール定義

| ロール | 説明 |
|--------|------|
| kiosk_user | キオスク端末を利用する店舗来店客（認証不要） |
