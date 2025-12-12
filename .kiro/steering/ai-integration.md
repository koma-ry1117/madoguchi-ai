# AI Integration

Gemini 2.5とVercel AI SDKを使用したAI機能の実装ガイドライン。

## Architecture Overview

```
┌─────────────────────────────────────────┐
│           Client Component              │
│  useChat() / useCompletion()           │
└─────────────────┬───────────────────────┘
                  │ Streaming
                  ▼
┌─────────────────────────────────────────┐
│      API Route (Node.js Runtime)        │
│  /api/chat/route.ts                     │
│  - Vercel AI SDK                        │
│  - Rate Limiting                        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Google Gemini API               │
│  - Gemini 2.5 Flash (メイン)            │
│  - Gemini 2.5 Pro (高精度タスク)        │
└─────────────────────────────────────────┘
```

## Runtime Selection

### Node.js Runtime を使用

GoogleのAIライブラリはNode.js固有のAPIに依存するため、**Edge Runtime ではなく Node.js Runtime** を使用：

```typescript
// app/api/chat/route.ts
export const runtime = 'nodejs'; // 明示的に指定

export const maxDuration = 60; // タイムアウト延長（秒）
```

**理由**:
- gRPC依存関係の互換性
- HTTP/2実装の安定性
- ストリーミングの信頼性

### タイムアウト設定

Gemini 2.5 Proは応答に時間がかかる場合があるため、`maxDuration` を設定：

| プラン | maxDuration 上限 |
|--------|-----------------|
| Hobby | 10秒 |
| Pro | 60秒 |
| Enterprise | 900秒 |

## Vercel AI SDK Integration

### 基本セットアップ

```typescript
// app/api/chat/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: google('gemini-2.5-flash-preview-05-20'),
    messages,
    system: `あなたは不動産会社の窓口AIアシスタントです。
丁寧な敬語で対応し、物件の提案や内見予約のサポートを行います。`,
  });

  return result.toDataStreamResponse();
}
```

### クライアント側の使用

```typescript
'use client';

import { useChat } from 'ai/react';

export function ChatInterface() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat({
    api: '/api/chat',
  });

  return (
    <form onSubmit={handleSubmit}>
      {messages.map((m) => (
        <div key={m.id}>
          <strong>{m.role}:</strong> {m.content}
        </div>
      ))}
      <input
        value={input}
        onChange={handleInputChange}
        disabled={isLoading}
        placeholder="メッセージを入力..."
      />
    </form>
  );
}
```

## Function Calling (Tool Use)

### Zodスキーマで型安全に定義

```typescript
import { streamText, tool } from 'ai';
import { z } from 'zod';

const result = streamText({
  model: google('gemini-2.5-flash-preview-05-20'),
  messages,
  tools: {
    searchProperties: tool({
      description: '条件に合う物件を検索します',
      parameters: z.object({
        area: z.string().describe('検索エリア（例：文京区）'),
        rentMin: z.number().optional().describe('最低家賃（万円）'),
        rentMax: z.number().optional().describe('最高家賃（万円）'),
        layout: z.string().optional().describe('間取り（例：2LDK）'),
      }),
      execute: async ({ area, rentMin, rentMax, layout }) => {
        // Supabaseから検索（RLSは適用されないので注意）
        const supabase = await createClient();
        const query = supabase.from('properties').select('*');

        if (area) query.ilike('address', `%${area}%`);
        if (rentMin) query.gte('rent', rentMin * 10000);
        if (rentMax) query.lte('rent', rentMax * 10000);
        if (layout) query.eq('layout', layout);

        const { data } = await query.limit(5);
        return data;
      },
    }),

    createViewingReservation: tool({
      description: '内見予約を作成します',
      parameters: z.object({
        propertyId: z.string().uuid(),
        preferredDate: z.string().describe('希望日時'),
        customerName: z.string(),
        customerPhone: z.string(),
      }),
      execute: async (params) => {
        // 予約処理
        return { success: true, reservationId: '...' };
      },
    }),
  },
});
```

### セキュリティ上の注意

Function Calling内でのDB操作は**RLSが適用されない場合がある**ため：

1. 必要に応じてService Roleではなく認証済みクライアントを使用
2. ユーザー入力のバリデーションを徹底
3. 機密操作には追加の認可チェックを実装

## Rate Limiting & Error Handling

### レート制限対策

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'), // 1分10リクエスト
});

export async function POST(req: Request) {
  // IPベースのレート制限
  const ip = req.headers.get('x-forwarded-for') ?? '127.0.0.1';
  const { success, remaining } = await ratelimit.limit(ip);

  if (!success) {
    return new Response('アクセスが集中しています。しばらく待ってから再試行してください。', {
      status: 429,
      headers: { 'X-RateLimit-Remaining': remaining.toString() },
    });
  }

  // AI処理続行
}
```

### 429エラーのハンドリング

Gemini APIのレート制限エラーを適切に処理：

```typescript
try {
  const result = await streamText({ /* ... */ });
  return result.toDataStreamResponse();
} catch (error) {
  if (error instanceof Error && error.message.includes('429')) {
    return new Response('APIが混み合っています。しばらくお待ちください。', {
      status: 429,
    });
  }
  throw error;
}
```

### フォールバック戦略

Gemini 2.5 Proが制限された場合、Flashにフォールバック：

```typescript
const models = [
  google('gemini-2.5-pro-preview-05-06'),
  google('gemini-2.5-flash-preview-05-20'),
];

async function generateWithFallback(messages: Message[]) {
  for (const model of models) {
    try {
      return await streamText({ model, messages });
    } catch (error) {
      if (isRateLimitError(error)) continue;
      throw error;
    }
  }
  throw new Error('All models are rate limited');
}
```

## Structured Output

### Zodスキーマで構造化出力

```typescript
import { generateObject } from 'ai';
import { z } from 'zod';

const propertyExtractionSchema = z.object({
  name: z.string().describe('物件名'),
  address: z.string().describe('住所'),
  rent: z.number().describe('家賃（円）'),
  layout: z.string().describe('間取り'),
  features: z.array(z.string()).describe('特徴・設備'),
});

export async function extractPropertyInfo(rawText: string) {
  const { object } = await generateObject({
    model: google('gemini-2.5-flash-preview-05-20'),
    schema: propertyExtractionSchema,
    prompt: `以下のテキストから物件情報を抽出してください:\n\n${rawText}`,
  });

  return object; // 型安全な構造化データ
}
```

## Voice Integration

音声入出力との統合については `.kiro/specs/voice-io/design.md` を参照。

### STT → AI → TTS フロー

```typescript
// 1. 音声入力をテキストに変換（Whisper）
const transcription = await transcribe(audioBlob);

// 2. AIで応答生成（Gemini）
const response = await generateResponse(transcription);

// 3. 応答を音声に変換（Google Cloud TTS）
const audioBuffer = await synthesize(response);
```

## System Prompt Best Practices

```typescript
const systemPrompt = `あなたは「まどぐちAI」という不動産会社の窓口AIアシスタントです。

## 役割
- 顧客の物件探しをサポート
- 条件のヒアリングと物件提案
- 内見予約の受付

## コミュニケーションスタイル
- 丁寧な敬語を使用
- 簡潔で分かりやすい説明
- 高齢者にも配慮した表現

## 制約
- 契約や法的アドバイスは行わない
- 個人情報は最小限のみ収集
- 不明点は正直に「担当者に確認します」と伝える

## 出力形式
- 長文は避け、要点を箇条書きで提示
- 物件提案時は主要な特徴を3つまで
`;
```

## Cost Optimization

| モデル | 用途 | コスト |
|--------|------|--------|
| Gemini 2.5 Flash | 一般的な会話、物件検索 | 低 |
| Gemini 2.5 Pro | 複雑な条件の解析、構造化抽出 | 高 |

**戦略**:
- 通常の会話はFlashを使用
- 物件情報の構造化抽出など精度が必要な場合のみProを使用
- キャッシュ可能なレスポンス（定型文等）はキャッシュ

---
_AI is a tool, not magic. Design for failure and fallback._
