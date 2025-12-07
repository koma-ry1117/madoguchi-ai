# Implementation Tasks

## Task 1: Supabase Storage バケット設定

### Description
音声キャッシュ用のStorageバケットを作成

### Files to Create/Modify
- `supabase/storage/audio-cache.sql` (create)

### Implementation Details
1. audio-cacheバケット作成
2. RLSポリシー設定（認証ユーザーのみ読み取り）
3. ファイルサイズ制限（5MB）

### Acceptance Criteria
- [ ] バケットが作成される
- [ ] 認証ユーザーがアクセスできる

---

## Task 2: /api/voice/transcribe APIルート

### Description
音声→テキスト変換API

### Files to Create/Modify
- `app/api/voice/transcribe/route.ts` (create)

### Implementation Details
1. multipart/form-data受信
2. OpenAI Whisper API呼び出し
3. 日本語設定（language: 'ja'）
4. エラーハンドリング

### Acceptance Criteria
- [ ] WebM/MP3を受け付ける
- [ ] 日本語テキストが返る
- [ ] エラー時に適切なレスポンス

---

## Task 3: /api/voice/synthesize APIルート

### Description
テキスト→音声変換API（キャッシュ付き）

### Files to Create/Modify
- `app/api/voice/synthesize/route.ts` (create)
- `src/lib/utils/audio-cache.ts` (create)

### Implementation Details
1. テキストをSHA256ハッシュ化
2. Supabase Storageでキャッシュ確認
3. キャッシュミス時にElevenLabs API呼び出し
4. キャッシュ保存
5. audio/mpegレスポンス

### Acceptance Criteria
- [ ] 音声が生成される
- [ ] キャッシュが機能する
- [ ] 2回目の呼び出しが高速

---

## Task 4: VoiceInputコンポーネント

### Description
音声録音UIコンポーネント

### Files to Create/Modify
- `src/components/features/voice/VoiceInput.tsx` (create)
- `src/components/features/voice/VoiceWaveform.tsx` (create)
- `src/lib/hooks/useVoiceRecorder.ts` (create)

### Implementation Details
1. MediaRecorder APIで録音
2. Web Audio APIで波形表示
3. 録音状態管理（useVoiceRecorder）
4. マイク許可リクエスト

### Acceptance Criteria
- [ ] 録音開始/停止ができる
- [ ] 波形が表示される
- [ ] マイク許可が処理される

---

## Task 5: AudioPlayerコンポーネント

### Description
音声再生UIコンポーネント

### Files to Create/Modify
- `src/components/features/voice/AudioPlayer.tsx` (create)
- `src/lib/hooks/useAudioPlayer.ts` (create)

### Implementation Details
1. HTML5 Audio API使用
2. 再生/一時停止ボタン
3. 音量調整
4. 自動再生オプション

### Acceptance Criteria
- [ ] 音声が再生される
- [ ] コントロールが機能する

---

## Task 6: useVoice Hook

### Description
音声機能を統合するカスタムフック

### Files to Create/Modify
- `src/lib/hooks/useVoice.ts` (create)

### Implementation Details
1. 録音状態管理
2. transcribe API呼び出し
3. synthesize API呼び出し
4. エラーハンドリング

### Acceptance Criteria
- [ ] 録音→変換が連携する
- [ ] 音声生成が呼び出せる

---

## Task 7: ChatInterfaceへの統合

### Description
VoiceInput/AudioPlayerをChatInterfaceに統合

### Files to Create/Modify
- `src/components/features/chat/ChatInterface.tsx` (modify)

### Implementation Details
1. VoiceInputボタン追加
2. 変換テキストを入力に反映
3. AI応答時にAudioPlayer呼び出し
4. 音声ON/OFFトグル

### Acceptance Criteria
- [ ] 音声入力が使える
- [ ] AI応答が音声で再生される

---

## Task 8: テスト実装

### Description
音声機能のテスト

### Files to Create/Modify
- `src/lib/hooks/__tests__/useVoiceRecorder.test.ts` (create)
- `app/api/voice/__tests__/transcribe.test.ts` (create)
- `app/api/voice/__tests__/synthesize.test.ts` (create)

### Acceptance Criteria
- [ ] フックのテストがパスする
- [ ] APIのテストがパスする
