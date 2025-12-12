# Implementation Tasks

## Task 1: Supabase Storage バケット設定とライフサイクル管理 (P)

_Requirements: 3.1, 3.3_

音声キャッシュ用のStorageバケットを作成し、30日自動削除ポリシーを設定する

### Subtasks
1. audio-cacheバケットをSupabase Storageに作成
   - バケット名: `audio-cache`
   - Public: No（認証ユーザーのみアクセス）
   - 最大ファイルサイズ: 5MB
   - 許可ファイル形式: audio/mpeg
2. RLSポリシーを設定（キオスクユーザーの読み取り許可）
3. ライフサイクル設定で30日後に自動削除

### Files
- `supabase/migrations/YYYYMMDDHHMMSS_create_audio_cache_bucket.sql` (create)

---

## Task 2: 音声キャッシュユーティリティ実装 (P)

_Requirements: 3.1, 3.2, 3.4_

テキストをハッシュ化し、Supabase Storageでキャッシュを管理するユーティリティ

### Subtasks
1. SHA256ハッシュ関数を実装
   - 入力: テキスト文字列
   - 出力: ハッシュ値（キャッシュキー）
2. キャッシュ取得関数
   - Supabase Storageから `audio-cache/{operator_id}/{hash}.mp3` を取得
   - キャッシュヒット時はバイナリデータを返す
   - キャッシュミス時はnullを返す
3. キャッシュ保存関数
   - 音声データをStorageにアップロード
   - エラー時はログ出力（サイレント失敗）

### Files
- `src/lib/utils/audio-cache.ts` (create)

---

## Task 3: テキスト→音声変換APIルート（Google Cloud TTS + キャッシング）

_Requirements: 2.1, 2.3, 3.2_

Google Cloud TTS APIを使用してテキストを音声に変換し、キャッシュを活用する

### Subtasks
1. POST /api/voice/synthesize エンドポイント作成
   - リクエスト: `{ text: string, operator_id: string }`
   - レスポンス: audio/mpeg（バイナリ）
2. Google Cloud TTS API統合
   - voice: ja-JP-Standard-A（日本語女性音声）
   - audioEncoding: MP3
3. キャッシュロジック実装
   - SHA256(text) → hash
   - Storageでキャッシュ確認
   - キャッシュヒット時: Storageから返却（X-Cache-Status: HIT）
   - キャッシュミス時: TTS生成 → Storageに保存 → 返却（X-Cache-Status: MISS）
4. エラーハンドリング
   - Google Cloud TTS APIエラー → 500エラー
   - キャッシュ保存失敗 → ログ出力のみ（音声は返す）

### Files
- `app/api/voice/synthesize/route.ts` (create)

---

## Task 4: 音声→テキスト変換APIルート（OpenAI Whisper）

_Requirements: 1.3_

OpenAI Whisper APIで音声をテキストに変換する

### Subtasks
1. POST /api/voice/transcribe エンドポイント作成
   - リクエスト: multipart/form-data（audio: File, operator_id: string）
   - レスポンス: `{ success: true, data: { text, duration, language } }`
2. OpenAI Whisper API呼び出し
   - model: whisper-1
   - language: ja（日本語）
   - response_format: json
3. エラーハンドリング
   - 無音検出 → `{ success: false, error: 'NO_SPEECH' }`
   - API失敗 → `{ success: false, error: 'API_ERROR' }`
4. レート制限（operator_idごと）

### Files
- `app/api/voice/transcribe/route.ts` (create)

---

## Task 5: チャンク分割ユーティリティ実装 (P)

_Requirements: 2_

ストリーミングTTS用に日本語テキストを文単位でチャンク分割する

### Subtasks
1. splitIntoChunks関数を実装
   - 入力: 日本語テキスト
   - 出力: 文単位の配列（。！？で分割）
   - 例: "こんにちは。お探しですか？" → ["こんにちは。", "お探しですか？"]
2. 空白チャンクをフィルタリング
3. ユニットテスト追加

### Files
- `src/lib/utils/text-chunking.ts` (create)
- `src/lib/utils/__tests__/text-chunking.test.ts` (create)

---

## Task 6: VoiceWaveformコンポーネント実装 (P)

_Requirements: 1.2_

録音中の音声波形をリアルタイム表示するコンポーネント

### Subtasks
1. Web Audio APIでAnalyserNodeを使用
   - fftSize: 256
   - frequencyBinCountで波形データ取得
2. Canvas APIで波形を描画
   - アニメーションループ（requestAnimationFrame）
   - 録音中のみアニメーション
3. Props定義
   - analyser: AnalyserNode | null
   - isRecording: boolean

### Files
- `src/components/features/voice/VoiceWaveform.tsx` (create)

---

## Task 7: useVoiceRecorder Hook実装

_Requirements: 1.1, 1.5, 1.6_

音声録音を管理するカスタムフック

### Subtasks
1. MediaRecorder APIを使用
   - mimeType: audio/webm または audio/mp4（ブラウザ対応に応じて）
   - 録音開始/停止機能
2. 録音時間制限（最大60秒）
   - カウントダウン表示用のdurationステート
   - 60秒で自動停止
3. マイク許可リクエスト
   - navigator.mediaDevices.getUserMedia({ audio: true })
   - 許可拒否時のエラーハンドリング
4. AnalyserNode生成（波形表示用）
5. 戻り値
   - isRecording, audioBlob, duration, analyser
   - startRecording(), stopRecording()

### Files
- `src/lib/hooks/useVoiceRecorder.ts` (create)

---

## Task 8: VoiceInputコンポーネント実装（高齢者対応UI）

_Requirements: 1.1, 1.2, 1.4_

高齢者にも使いやすい大きな音声入力ボタンを提供

### Subtasks
1. 大きなマイクボタン（120px × 120px以上）
   - 画面中央に配置
   - ラベル: "タップしてお話しください"
   - クリックで録音開始/停止
2. 録音中UI
   - VoiceWaveformコンポーネント表示
   - カウントダウン表示（残り秒数）
   - 録音停止ボタン
3. API呼び出し
   - 録音停止時に /api/voice/transcribe にPOST
   - ローディング表示
   - テキスト変換完了時にonTranscribe(text)コールバック
4. エラーハンドリング
   - マイク許可拒否 → 許可を促すメッセージ
   - 無音検出 → "お話が聞き取れませんでした"
   - API失敗 → "もう一度お試しください"
5. Props定義
   - onTranscribe: (text: string) => void
   - disabled?: boolean
   - maxDuration?: number（デフォルト60秒）

### Files
- `src/components/features/voice/VoiceInput.tsx` (create)

---

## Task 9: useAudioQueue Hook実装

_Requirements: 2, 4.3_

ストリーミング音声の再生キューを管理するカスタムフック

### Subtasks
1. 音声キュー管理
   - chunks: AudioBuffer[]
   - currentIndex: number
   - isPlaying: boolean
2. enqueue(chunk: string)
   - /api/voice/synthesize を呼び出し
   - AudioBufferに変換してキューに追加
3. play()
   - キューの先頭から順次再生
   - 1つの音声再生終了時に次の音声を自動再生
4. skip()
   - 現在の再生を停止
   - 残りのチャンクのTTSリクエストをキャンセル
5. stop()
   - 再生停止（新しい応答時やユーザー発話時）
6. コールバック
   - onPlayStart(), onPlayEnd()

### Files
- `src/lib/hooks/useAudioQueue.ts` (create)

---

## Task 10: AudioPlayerコンポーネント実装

_Requirements: 2.2, 4.1, 4.4_

ストリーミング音声を順次再生するプレイヤーコンポーネント

### Subtasks
1. useAudioQueue Hookを使用
   - chunks配列を受け取り、順次TTS生成・再生
   - 最初のチャンク準備完了で再生開始
2. スキップボタン
   - クリックで読み上げスキップ
   - onSkip()コールバック
3. 再生/一時停止ボタン
4. 音量調整UI（キオスク設置時に設定）
5. Props定義
   - chunks: string[]（文単位）
   - autoPlay: boolean（キオスクではtrue）
   - onPlayStart?: () => void
   - onPlayEnd?: () => void
   - onInterrupt?: () => void
   - onSkip?: () => void

### Files
- `src/components/features/voice/AudioPlayer.tsx` (create)

---

## Task 11: 事前キャッシュスクリプト実装 (P)

_Requirements: 3.4_

よく使うフレーズを事前にキャッシュしてヒット率を向上

### Subtasks
1. 事前キャッシュフレーズ定義
   - 挨拶・基本応答（"いらっしゃいませ。"等）
   - 確認パターン（"ご予算はいくらくらいをお考えですか？"等）
   - 提案パターン（"こちらの物件はいかがでしょうか。"等）
   - 内見予約（"内見のご予約を承ります。"等）
2. 事前キャッシュスクリプト
   - 各フレーズに対して /api/voice/synthesize を呼び出し
   - 並列リクエスト（最大5並列）
   - キャッシュ完了ログ出力
3. 起動時実行設定（Next.js起動時に非同期実行）

### Files
- `src/lib/utils/precache-phrases.ts` (create)
- `scripts/precache-audio.ts` (create)

---

## Task 12: ChatInterfaceへの音声機能統合

_Requirements: 1.4, 2.2, 2.5, 4.4_

既存のChatInterfaceに音声入出力を統合

### Subtasks
1. VoiceInputコンポーネント追加
   - テキスト入力の上部に大きく配置
   - onTranscribe で変換テキストを入力欄に反映
   - 送信ボタンを自動トリガー
2. AudioPlayerコンポーネント追加
   - AI応答完了時にchunksを渡す
   - チャンク分割（splitIntoChunks）
   - autoPlay: true
3. アバター感情連携
   - onPlayStart → アバター感情を "speaking" に変更
   - onPlayEnd → アバター感情を "neutral" に戻す
4. 音声コントロール
   - 新しいAI応答時、前の音声を停止
   - ユーザーが録音開始時、再生中の音声を停止
5. 音声ON/OFFトグル（設定として保存）

### Files
- `src/components/features/chat/ChatInterface.tsx` (modify)

---

## Task 13: キオスクUIへの音声機能統合

_Requirements: 1, 2, 4_

KioskInterfaceに音声入出力UIを統合

### Subtasks
1. VoiceInputを画面中央に大きく配置
   - サイズ: 'large'（120px × 120px以上）
   - テキスト入力は補助的な位置（下部に小さく）
2. AudioPlayerを画面下部に配置
   - スキップボタンを大きく表示
   - 再生/一時停止ボタン
3. 音声とアバターの連動
   - 読み上げ中にアバター表情を変更（静止画方式）
4. レスポンシブ対応（タブレット・大画面向け）

### Files
- `src/components/features/kiosk/KioskInterface.tsx` (modify)

---

## Task 14: エラーハンドリングとフォールバック実装

_Requirements: 1, 2_

音声機能のエラーを適切に処理し、フォールバックを提供

### Subtasks
1. VoiceInputのエラーハンドリング
   - マイク許可拒否 → テキスト入力にフォールバック + 許可を促すメッセージ
   - 録音失敗 → 再試行ボタン表示
   - Whisper APIエラー → テキスト入力を促す
   - 無音検出 → "お話が聞き取れませんでした"
2. AudioPlayerのエラーハンドリング
   - Google Cloud TTS APIエラー → テキストのみ表示（音声なし）
   - キャッシュ保存失敗 → ログ出力（サイレント失敗）
   - 音声再生失敗 → 再生ボタン表示
3. エラーメッセージのUI
   - トースト通知
   - エラー内容に応じた具体的なメッセージ

### Files
- `src/components/features/voice/VoiceInput.tsx` (modify)
- `src/components/features/voice/AudioPlayer.tsx` (modify)

---

## Task 15: ユニットテスト実装 (P)

_Requirements: All_

音声機能の各モジュールにユニットテストを追加

### Subtasks
1. audio-cache.tsのテスト
   - ハッシュ生成
   - キャッシュ取得/保存
2. text-chunking.tsのテスト
   - 文単位分割
   - エッジケース（空文字、句読点なし等）
3. useVoiceRecorder Hookのテスト
   - 録音開始/停止
   - 時間制限
4. useAudioQueue Hookのテスト
   - キュー管理
   - 順次再生
   - スキップ機能

### Files
- `src/lib/utils/__tests__/audio-cache.test.ts` (create)
- `src/lib/utils/__tests__/text-chunking.test.ts` (create)
- `src/lib/hooks/__tests__/useVoiceRecorder.test.ts` (create)
- `src/lib/hooks/__tests__/useAudioQueue.test.ts` (create)

---

## Task 16: 統合テスト実装 (P)

_Requirements: All_

音声機能のエンドツーエンドフローをテスト

### Subtasks
1. /api/voice/transcribe APIテスト
   - 正常系: 音声ファイル送信 → テキスト返却
   - 異常系: 無音、API失敗
2. /api/voice/synthesize APIテスト
   - 正常系: テキスト送信 → 音声返却
   - キャッシュヒット/ミス
   - 異常系: Google Cloud TTS失敗
3. E2Eテスト（Playwright）
   - 録音 → 変換 → AI送信 → 音声応答
   - スキップ機能
   - エラーハンドリング

### Files
- `app/api/voice/__tests__/transcribe.test.ts` (create)
- `app/api/voice/__tests__/synthesize.test.ts` (create)
- `e2e/voice-flow.spec.ts` (create)

---

## Task 17: パフォーマンス最適化とモニタリング

_Requirements: 2, 3_

ストリーミングTTSの体感速度向上とキャッシュヒット率モニタリング

### Subtasks
1. 並列TTSリクエスト実装
   - 複数チャンクを並列で生成（最大3並列）
   - Promise.allSettledで待ち時間削減
2. キャッシュヒット率モニタリング
   - X-Cache-Statusをログ出力
   - 目標: 40%以上のヒット率
3. 体感待ち時間測定
   - AI応答完了から最初の音声再生までの時間
   - 目標: 全文一括より50%以上削減
4. パフォーマンスログ実装
   - TTSレイテンシー、キャッシュヒット率を記録

### Files
- `src/lib/hooks/useAudioQueue.ts` (modify)
- `src/lib/utils/performance-logger.ts` (create)

---

## 推奨実装順序

### Phase 1: インフラ・API（並列可能）
- Task 1: Supabase Storage設定
- Task 2: 音声キャッシュユーティリティ
- Task 3: /api/voice/synthesize
- Task 4: /api/voice/transcribe
- Task 5: チャンク分割ユーティリティ

### Phase 2: 音声入力（依存: Task 4）
- Task 6: VoiceWaveform
- Task 7: useVoiceRecorder Hook
- Task 8: VoiceInput

### Phase 3: 音声出力（依存: Task 3, 5）
- Task 9: useAudioQueue Hook
- Task 10: AudioPlayer

### Phase 4: UI統合（依存: Task 8, 10）
- Task 12: ChatInterface統合
- Task 13: KioskUI統合
- Task 14: エラーハンドリング

### Phase 5: 最適化・テスト（並列可能）
- Task 11: 事前キャッシュ
- Task 15: ユニットテスト
- Task 16: 統合テスト
- Task 17: パフォーマンス最適化
