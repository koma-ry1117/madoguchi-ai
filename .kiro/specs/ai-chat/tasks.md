# Implementation Tasks

## Task 1: キオスクセッション管理

### Description
キオスクセッションの状態管理（Zustand）

### Files to Create/Modify
- `src/lib/stores/kiosk-store.ts` (create)

### Implementation Details
1. セッション状態（welcome, conversation, contact, complete, goodbye）
2. メッセージ履歴
3. 選択物件・予約日時
4. リセット機能
5. タイムアウト管理

### Acceptance Criteria
- [ ] 状態遷移が正しく動作する
- [ ] リセットで全状態がクリアされる

---

## Task 2: POST /api/kiosk/chat API

### Description
キオスク専用の会話API（認証なし）

### Files to Create/Modify
- `app/api/kiosk/chat/route.ts` (create)
- `src/lib/ai/kiosk-system-prompt.ts` (create)

### Implementation Details
1. operator_idによるデータ分離
2. GPT-4o + Function Calling
3. 物件検索ツール
4. 内見予約ツール
5. 感情分析
6. SSEストリーミング

### Acceptance Criteria
- [ ] 認証なしでアクセス可能
- [ ] 物件検索が機能する
- [ ] 内見予約提案が機能する

---

## Task 3: キオスク用AI System Prompt

### Description
不動産接客に特化したシステムプロンプト

### Files to Create/Modify
- `src/lib/ai/kiosk-system-prompt.ts` (create)

### Implementation Details
1. 丁寧な接客口調
2. 高齢者向けの分かりやすい説明
3. 条件ヒアリングの流れ
4. 物件提案の仕方
5. 内見予約への誘導

### Acceptance Criteria
- [ ] 自然な会話が生成される
- [ ] 適切なタイミングで物件検索する

---

## Task 4: AI Function Calling Tools

### Description
キオスク用のFunction Callingツール群

### Files to Create/Modify
- `src/lib/ai/tools/kiosk-tools.ts` (create)

### Implementation Details
1. searchProperties: 物件検索
2. getPropertyDetail: 物件詳細
3. proposeViewing: 内見提案
4. confirmViewing: 内見確定

### Acceptance Criteria
- [ ] 各ツールが正しく動作する
- [ ] 検索結果が適切に返る

---

## Task 5: POST /api/kiosk/complete API

### Description
予約完了と顧客情報登録API

### Files to Create/Modify
- `app/api/kiosk/complete/route.ts` (create)

### Implementation Details
1. 顧客情報バリデーション
2. customers INSERT
3. viewings INSERT
4. conversations UPDATE
5. 顧客確認メール送信
6. オペレーター通知メール送信

### Acceptance Criteria
- [ ] 顧客情報が保存される
- [ ] 予約が保存される
- [ ] メールが送信される

---

## Task 6: KioskInterfaceコンポーネント

### Description
キオスク全体のコンテナコンポーネント

### Files to Create/Modify
- `src/components/features/kiosk/KioskInterface.tsx` (create)

### Implementation Details
1. フェーズに応じた表示切り替え
2. Welcome → Conversation → Contact → Complete
3. タイムアウト検知
4. 自動リセット

### Acceptance Criteria
- [ ] フェーズ遷移が正しく動作する
- [ ] タイムアウトでリセットする

---

## Task 7: AvatarDisplayコンポーネント

### Description
感情表現付きアバター表示

### Files to Create/Modify
- `src/components/features/kiosk/AvatarDisplay.tsx` (create)
- `public/avatars/` (5種類の表情画像)

### Implementation Details
1. 5種類の表情画像
2. Framer Motionアニメーション
3. 感情状態の購読

### Acceptance Criteria
- [ ] 表情が切り替わる
- [ ] アニメーションが滑らか

---

## Task 8: ConversationPanelコンポーネント

### Description
会話履歴と物件カード表示

### Files to Create/Modify
- `src/components/features/kiosk/ConversationPanel.tsx` (create)
- `src/components/features/kiosk/MessageBubble.tsx` (create)
- `src/components/features/kiosk/PropertyCard.tsx` (create)

### Implementation Details
1. メッセージバブル（user/assistant）
2. 物件カードインライン表示
3. 自動スクロール
4. ローディング表示

### Acceptance Criteria
- [ ] メッセージが表示される
- [ ] 物件カードが表示される

---

## Task 9: VoiceInputコンポーネント

### Description
音声入力機能

### Files to Create/Modify
- `src/components/features/kiosk/VoiceInput.tsx` (create)
- `src/components/features/kiosk/TextInput.tsx` (create)

### Implementation Details
1. マイクボタン
2. 録音中の波形表示
3. Whisper API呼び出し
4. テキスト入力フォールバック

### Acceptance Criteria
- [ ] 音声録音ができる
- [ ] テキスト変換される

---

## Task 10: AudioPlayerコンポーネント

### Description
AI応答の音声読み上げ

### Files to Create/Modify
- `src/components/features/kiosk/AudioPlayer.tsx` (create)

### Implementation Details
1. ElevenLabs TTS呼び出し
2. 自動再生
3. キャッシュ利用

### Acceptance Criteria
- [ ] 応答が読み上げられる

---

## Task 11: ContactFormコンポーネント

### Description
連絡先入力フォーム

### Files to Create/Modify
- `src/components/features/kiosk/ContactForm.tsx` (create)

### Implementation Details
1. 名前入力
2. 電話番号入力（バリデーション）
3. メールアドレス入力（バリデーション）
4. 送信ボタン
5. ローディング状態

### Acceptance Criteria
- [ ] 入力バリデーションが機能する
- [ ] 送信でAPI呼び出し

---

## Task 12: CompletionScreenコンポーネント

### Description
予約完了画面

### Files to Create/Modify
- `src/components/features/kiosk/CompletionScreen.tsx` (create)

### Implementation Details
1. 「ありがとうございました」メッセージ
2. 予約内容サマリー
3. 30秒後自動リセット
4. 終了ボタン

### Acceptance Criteria
- [ ] 完了画面が表示される
- [ ] 自動リセットする

---

## Task 13: GoodbyeScreenコンポーネント

### Description
予約なし終了画面

### Files to Create/Modify
- `src/components/features/kiosk/GoodbyeScreen.tsx` (create)

### Implementation Details
1. 「またのご来店をお待ちしています」メッセージ
2. 自動リセット

### Acceptance Criteria
- [ ] 終了画面が表示される
- [ ] 自動リセットする

---

## Task 14: キオスクページ

### Description
キオスク用のメインページ

### Files to Create/Modify
- `app/kiosk/[operatorId]/page.tsx` (create)
- `app/kiosk/[operatorId]/layout.tsx` (create)

### Implementation Details
1. operatorIdをURLから取得
2. KioskInterface配置
3. フルスクリーンレイアウト
4. キオスクモード用CSS

### Acceptance Criteria
- [ ] ページが表示される
- [ ] operator_idが正しく取得される

---

## Task 15: メールテンプレート

### Description
キオスク予約用のメールテンプレート

### Files to Create/Modify
- `src/lib/email/templates/KioskBookingConfirmation.tsx` (create)
- `src/lib/email/templates/KioskBookingNotification.tsx` (create)

### Implementation Details
1. 顧客向け確認メール
2. オペレーター向け通知メール
3. 予約詳細表示

### Acceptance Criteria
- [ ] メールが正しくレンダリングされる

---

## Task 16: テスト実装

### Description
キオスク機能のテスト

### Files to Create/Modify
- `src/lib/stores/__tests__/kiosk-store.test.ts` (create)
- `app/api/kiosk/__tests__/chat.test.ts` (create)
- `app/api/kiosk/__tests__/complete.test.ts` (create)

### Acceptance Criteria
- [ ] ストアのテストがパスする
- [ ] APIのテストがパスする
