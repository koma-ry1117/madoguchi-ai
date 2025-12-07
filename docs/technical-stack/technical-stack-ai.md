# 技術スタック - AI・LLM・音声処理

> 本ドキュメントは [技術スタック詳細](../technical-stack.md) の一部です。

## AI・LLM統合（OpenAI統一）

### 会話AI

#### OpenAI GPT-4o
**用途**:
- メイン会話エンジン
- 顧客対応・会話処理
- 物件推薦・マッチング
- 感情分析（アバター表情選択用）
- HTML構造化データ抽出
- 物件説明文生成

**モデル仕様**:
- コンテキスト長: 128K トークン
- 入力価格: $5/1M トークン
- 出力価格: $15/1M トークン
- マルチモーダル対応（テキスト・画像）

**使用例**:
```typescript
import { openai } from '@ai-sdk/openai';
import { generateText, streamText } from 'ai';

// 会話生成
const { text } = await generateText({
  model: openai('gpt-4o'),
  system: 'あなたは不動産営業のAIアシスタントです...',
  prompt: userMessage,
});

// ストリーミング会話
const { textStream } = streamText({
  model: openai('gpt-4o'),
  messages: conversationHistory,
});
```

---

## AIフレームワーク

### Vercel AI SDK
**選定理由**:
- Next.jsとの完璧な統合
- OpenAI統合が簡単
- ストリーミングレスポンス対応
- Function Calling対応
- エッジランタイム対応

**使用機能**:

1. **ストリーミングレスポンス**
   ```typescript
   import { StreamingTextResponse } from 'ai';

   return new StreamingTextResponse(stream);
   ```

2. **Function Calling（物件検索実行）**
   ```typescript
   const tools = {
     searchProperties: {
       description: '物件を検索する',
       parameters: z.object({
         area: z.string(),
         rent: z.object({
           min: z.number(),
           max: z.number(),
         }),
         layout: z.string(),
       }),
       execute: async (params) => {
         return await searchProperties(params);
       },
     },
   };
   ```

3. **Tool Calling（複数ツール統合）**
   - 物件検索
   - 画像取得
   - 周辺情報取得
   - 類似物件検索

---

## ベクトル検索・RAG

### Supabase Vector (pgvector)
**用途**: セマンティック物件検索
- ベクトルデータベース
- 類似物件検索
- 顧客の希望に近い物件の発見

### OpenAI Embeddings (text-embedding-3-small)
**用途**: テキストのベクトル化
- 物件説明文のベクトル化
- 顧客の希望条件のベクトル化
- コサイン類似度による検索

**価格**: $0.02/1M トークン

**実装例**:
```typescript
import { embeddings } from '@ai-sdk/openai';

// テキストをベクトル化
const vector = await embeddings.embed({
  model: 'text-embedding-3-small',
  input: propertyDescription,
});

// Supabaseで類似検索
const { data } = await supabase.rpc('match_properties', {
  query_embedding: vector,
  match_threshold: 0.8,
  match_count: 10,
});
```

---

## 音声処理

### 音声認識 (Speech-to-Text)

#### OpenAI Whisper API
**仕様**:
- 価格: $0.006/分
- 対応形式: mp3, mp4, wav, webm等
- 高精度な日本語認識
- タイムスタンプ付き文字起こし

**使用例**:
```typescript
import { openai } from 'openai';

const transcription = await openai.audio.transcriptions.create({
  file: audioFile,
  model: 'whisper-1',
  language: 'ja',
});
```

#### Web Audio API
**用途**: ブラウザでのマイク入力処理
- マイク入力の取得
- 音声レベルメーター
- リアルタイム波形表示

#### MediaRecorder API
**用途**: 音声録音・エンコーディング
- ブラウザで音声録音
- WebM形式で保存
- Blob生成

### 音声合成 (Text-to-Speech)

#### ElevenLabs API
**選定理由**:
- 高品質な日本語音声合成
- 感情表現が豊か
- カスタム音声作成可能
- ストリーミング再生対応

**価格**:
- Starter: 30,000文字/月 - $5
- Creator: 100,000文字/月 - $11
- Pro: 500,000文字/月 - $99

**使用例**:
```typescript
import { ElevenLabsClient } from 'elevenlabs';

const audio = await elevenlabs.textToSpeech({
  voice_id: 'voice_id_here',
  text: response,
  model_id: 'eleven_multilingual_v2',
});
```

#### 音声キャッシング戦略
よくある応答を事前生成してSupabase Storageに保存

```typescript
// キャッシュ確認
const cachedAudio = await supabase.storage
  .from('audio-cache')
  .download(`${hash(text)}.mp3`);

if (cachedAudio) {
  return cachedAudio;
}

// キャッシュがなければ生成
const audio = await generateSpeech(text);
await supabase.storage
  .from('audio-cache')
  .upload(`${hash(text)}.mp3`, audio);
```

---

## Chrome拡張機能（最小構成）

### 目的
- 表示中のページのHTMLを取得→アプリに送信のみ
- 人手によるワンクリック取り込み
- 法的リスク軽減のための証跡記録

### 技術スタック

#### Manifest V3
最新のChrome拡張仕様

#### TypeScript
型安全な拡張機能開発

#### Vanilla JavaScript
フレームワーク不使用で軽量化

### 構成要素

```
chrome-extension/
├── manifest.json              # 拡張機能設定
├── src/
│   ├── background.ts          # Service Worker
│   ├── content.ts             # Content Script
│   ├── popup/
│   │   ├── popup.html         # ポップアップUI
│   │   ├── popup.ts           # ポップアップロジック
│   │   └── popup.css          # スタイル
│   └── types.ts               # 型定義
└── dist/                      # ビルド出力
```

#### manifest.json
```json
{
  "manifest_version": 3,
  "name": "madoguchi-ai Property Importer",
  "version": "1.0.0",
  "permissions": [
    "activeTab",
    "storage"
  ],
  "host_permissions": [
    "https://www.athome.co.jp/*"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://www.athome.co.jp/*"],
      "js": ["content.js"]
    }
  ],
  "action": {
    "default_popup": "popup.html"
  }
}
```

#### Content Script（HTML取得）
```typescript
// content.ts
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === 'getHTML') {
    const html = document.documentElement.outerHTML;
    const url = window.location.href;

    sendResponse({ html, url });
  }
});
```

#### Background Service Worker（API送信）
```typescript
// background.ts
chrome.action.onClicked.addListener(async (tab) => {
  const [{ result }] = await chrome.scripting.executeScript({
    target: { tabId: tab.id },
    func: () => document.documentElement.outerHTML,
  });

  // Next.js APIに送信
  const response = await fetch('https://madoguchi-ai.vercel.app/api/properties/import', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      html: result,
      url: tab.url,
    }),
  });
});
```
