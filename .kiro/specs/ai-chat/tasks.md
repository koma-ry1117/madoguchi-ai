# Implementation Tasks

## 1. データモデルとデータベースセットアップ

- [ ] 1.1 (P) Supabaseテーブル設計と作成
  - `conversations`テーブル（session_type、operator_id、customer_id、outcome等）
  - `messages`テーブル（conversation_id、role、content等）
  - `customers`テーブル（name、phone、email等）
  - `viewings`テーブル（customer_id、property_id、viewing_date等）
  - Service Roleを使用した認証なしアクセス設定
  - _Requirements: 4.6, 5.4, 7.3_

## 2. バックエンドAPI実装

- [ ] 2.1 (P) キオスクチャットAPI実装（`/api/kiosk/chat`）
  - SSEストリーム対応（text、emotion、criteria_update、properties_add等）
  - Gemini 2.5 Flash統合（Vercel AI SDK）
  - session_idとoperator_idのハンドリング
  - レート制限の実装（IP + session_id）
  - _Requirements: 1.1, 1.2, 1.4_

- [ ] 2.2 (P) AI Function Calling実装
  - `addPropertyCriteria`: 条件追加と物件スロット追加
  - `refineCriteria`: 条件変更とスロット内物件絞り込み
  - `selectProperty`: 物件選択と詳細表示
  - `proposeViewing`: 内見予約提案
  - `confirmViewing`: 内見予約確定
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 3.1, 3.2_

- [ ] 2.3 音声処理API実装（`/api/voice/`）
  - `/api/voice/transcribe`: OpenAI Whisperで音声→テキスト変換
  - `/api/voice/synthesize`: Google Cloud TTSでテキスト→音声変換
  - ストリーミングTTS対応（文単位チャンク分割）
  - 音声キャッシュ機能（頻出フレーズの事前キャッシュ）
  - _Requirements: 1.2, 1.3_

- [ ] 2.4 予約完了API実装（`/api/kiosk/complete`）
  - 顧客情報バリデーション（電話番号形式、メール形式）
  - customersテーブルへのINSERT
  - viewingsテーブルへのINSERT
  - conversationsテーブルのUPDATE
  - Resendを使った確認メール送信（顧客向け）
  - Resendを使った通知メール送信（オペレーター向け）
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

## 3. フロントエンド状態管理とコアロジック

- [ ] 3.1 Zustandストア実装
  - KioskState（phase、sessionId、messages、propertySlots、selectedProperty等）
  - collectedCriteria（段階的に収集される条件）
  - セッション操作（reset、updatePhase等）
  - _Requirements: 1.5, 5.3, 5.4_

- [ ] 3.2 セッションタイムアウトとリセット機能
  - 10分操作なしで自動リセット
  - 完了画面30秒後自動リセット
  - リセット時の会話履歴と入力情報クリア
  - _Requirements: 1.5, 5.3, 5.4_

## 4. UIコンポーネント実装（会話インターフェース）

- [ ] 4.1 (P) KioskInterfaceコンポーネント
  - phase切り替え（welcome、conversation、contact、complete、goodbye）
  - レイアウト構成（アバター、メッセージリスト、入力エリア）
  - _Requirements: 1.1, 5.1, 5.2, 7.1, 7.2_

- [ ] 4.2 (P) AvatarDisplayコンポーネント
  - 5種類の表情（neutral、happy、thinking、apologetic、excited）
  - AI応答内容に応じた自動表情選択
  - 表情切り替えアニメーション
  - _Requirements: 6.1, 6.2, 6.3_

- [ ] 4.3 ConversationPanelコンポーネント
  - メッセージ履歴表示（user/assistant）
  - ストリーミングメッセージ表示
  - テキスト先行表示（音声は後から再生）
  - 音声再生状態の表示（preparing、playing、done、skipped）
  - スキップボタン（音声再生中のみ表示）
  - _Requirements: 1.3, 1.4_

- [ ] 4.4 PropertySlotsコンポーネント
  - 候補物件のカルーセル表示（横スクロール）
  - 物件追加/絞り込みアニメーション
  - 物件選択時のハイライト
  - 物件カード表示
  - _Requirements: 2.2, 2.3_

- [ ] 4.5 VoiceInputコンポーネント
  - 大きな音声ボタン（画面中央配置）
  - 録音開始/停止
  - `/api/voice/transcribe`への音声送信
  - テキスト入力との併用UI
  - 「終了」ボタン
  - _Requirements: 1.2, 7.1_

## 5. UIコンポーネント実装（フォームと完了画面）

- [ ] 5.1 ContactFormコンポーネント
  - 入力フィールド（name、phone、email）
  - リアルタイムバリデーション
  - エラー表示（フィールドハイライト）
  - `/api/kiosk/complete`への送信
  - リトライボタン（送信失敗時）
  - _Requirements: 4.1, 4.2, 4.3_

- [ ] 5.2 CompletionScreenコンポーネント
  - 予約内容サマリー表示
  - 「ありがとうございました」メッセージ
  - 30秒後自動リセットカウントダウン
  - 「終了」ボタン（即座にリセット）
  - _Requirements: 5.1, 5.2, 5.3_

- [ ] 5.3 GoodbyeScreenコンポーネント
  - 「またのご来店をお待ちしています」メッセージ
  - 会話ログDB保存（outcome: browsing_only）
  - 自動リセットタイマー
  - _Requirements: 7.2, 7.3_

## 6. 会話フロー統合

- [ ] 6.1 会話フロー全体統合
  - Welcome → Conversation → ContactForm → Complete → Welcome
  - 物件検索フロー（条件ヒアリング → 検索 → 提案 → 選択）
  - 内見予約フロー（日時確認 → 確認 → 連絡先入力）
  - 条件緩和提案（物件0件時）
  - _Requirements: 1.1, 2.1, 2.2, 2.3, 2.4, 2.5, 3.1, 3.2, 3.3_

- [ ] 6.2 エラーハンドリング実装
  - Gemini APIエラー時の再試行促進メッセージ
  - 物件検索0件時の条件緩和提案
  - フォーム送信失敗時のリトライ
  - タイムアウト処理
  - _Requirements: 2.5_

## 7. テストとパフォーマンス最適化

- [ ] 7.1 (P) ユニットテスト実装
  - ContactFormバリデーションテスト
  - AI Toolsパラメータ検証テスト
  - Zustandストアテスト
  - _Requirements: 4.3_

- [ ] 7.2 統合テスト実装
  - 会話フロー全体テスト
  - 予約完了フローテスト
  - エラーハンドリングテスト
  - _Requirements: 1.1, 2.1, 3.1, 4.1_

- [ ] 7.3 E2Eテスト実装
  - 初期画面 → 物件検索 → 内見予約 → 連絡先入力 → 完了 → リセット
  - 予約なし終了フロー
  - タイムアウトリセット
  - _Requirements: 1.5, 5.3, 7.1_

- [ ] 7.4 パフォーマンス最適化
  - ストリーミングTTS実装と検証
  - 音声キャッシュ実装（キャッシュヒット率40%以上）
  - テキスト先行表示の動作確認
  - スキップ機能の動作確認
  - セッションタイムアウト動作確認
  - _Requirements: 1.3_

## 8. デプロイと運用準備

- [ ] 8.1 環境変数設定
  - Gemini API Key
  - OpenAI API Key
  - Google Cloud TTS設定
  - Supabase Service Role Key
  - Resend API Key
  - operator_id設定
  - _Requirements: 1.1, 4.4, 4.5_

- [ ] 8.2 (P) 本番環境デプロイ
  - Vercelデプロイ設定
  - レート制限設定
  - エラーログ監視設定
  - _Requirements: 1.5_

- [ ] 8.3 ドキュメント作成
  - 運用マニュアル（店舗設置手順）
  - トラブルシューティングガイド
  - オペレーター向け使い方ガイド
  - _Requirements: 1.1_
