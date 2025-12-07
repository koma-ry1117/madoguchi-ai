# キオスク・AI会話API

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## キオスクAPI（オペレーター認証必須）

キオスクAPIはオペレーターがログインした状態で呼び出す。セッションからoperator_idを自動取得するため、リクエストにoperator_idを含める必要はない。

### POST /api/kiosk/chat

**説明**: キオスク専用AI会話エンドポイント（SSEストリーミング対応）

**認証**: 必須（operator）- セッションからoperator_idを取得

**リクエスト**:
```json
{
  "session_id": "uuid",
  "message": "文京区で2LDKの物件を探しています",
  "context": {
    "selected_property_id": "uuid",
    "viewing_date": "2025-01-20"
  }
}
```

**レスポンス**: Server-Sent Events (SSE)
```
data: {"type":"text","content":"かしこまりました。"}
data: {"type":"emotion","emotion":"thinking"}
data: {"type":"property","property":{...}}
data: {"type":"booking_confirm","property_id":"...","date":"..."}
data: {"type":"request_contact"}
data: {"type":"done"}
```

---

### POST /api/kiosk/complete

**説明**: キオスク予約完了・顧客情報登録

**認証**: 必須（operator）- セッションからoperator_idを取得

**リクエスト**:
```json
{
  "session_id": "uuid",
  "customer": {
    "name": "山田太郎",
    "phone": "090-1234-5678",
    "email": "yamada@example.com"
  },
  "viewing": {
    "property_id": "uuid",
    "viewing_date": "2025-01-20T14:00:00+09:00"
  }
}
```

**注意**: 顧客情報は名前・電話番号・メールアドレスのみ。認証アカウントは作成しない。

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "customer_id": "uuid",
    "viewing_id": "uuid",
    "message": "ご予約ありがとうございます"
  }
}
```

---

## AI会話API（オペレーター管理画面用）

### POST /api/chat

**説明**: AI会話エンドポイント（ストリーミング対応）- オペレーター管理画面でのテスト用

**認証**: 必須（operator または admin）

**リクエスト**:
```json
{
  "messages": [
    {
      "role": "user",
      "content": "文京区で2LDKの物件を探しています"
    }
  ],
  "conversation_id": "uuid" // 既存の会話の場合
}
```

**レスポンス**: Server-Sent Events (SSE)

```
data: {"type":"text","content":"かしこまりました。"}
data: {"type":"text","content":"文京区で2LDK"}
data: {"type":"text","content":"の物件ですね。"}
data: {"type":"emotion","emotion":"thinking"}
data: {"type":"function_call","name":"searchProperties","args":{"area":"文京区","layout":"2LDK"}}
data: {"type":"function_result","result":[...]}
data: {"type":"text","content":"以下の物件がございます..."}
data: {"type":"emotion","emotion":"happy"}
data: {"type":"done"}
```

**実装例**:
```typescript
// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function POST(request: Request) {
  const { messages, conversation_id } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // 認証チェック
  const { data: { user }, error } = await supabase.auth.getUser();
  if (error || !user) {
    return new Response('Unauthorized', { status: 401 });
  }

  // 会話履歴の保存・取得
  let conversationId = conversation_id;
  if (!conversationId) {
    const { data } = await supabase
      .from('conversations')
      .insert({
        customer_id: user.id,
      })
      .select()
      .single();
    conversationId = data.id;
  }

  // AI生成
  const result = await streamText({
    model: openai('gpt-4o'),
    system: `あなたは不動産営業のAIアシスタントです。
丁寧で親しみやすく、高齢者にも分かりやすい説明を心がけてください。`,
    messages,
    tools: {
      searchProperties: {
        description: '物件を検索する',
        parameters: z.object({
          area: z.string().optional(),
          rent_min: z.number().optional(),
          rent_max: z.number().optional(),
          layout: z.string().optional(),
        }),
        execute: async (params) => {
          const { data } = await supabase
            .from('properties')
            .select('*')
            .eq('area', params.area)
            .gte('rent', params.rent_min || 0)
            .lte('rent', params.rent_max || 999999999)
            .eq('layout', params.layout)
            .eq('is_public', true)
            .limit(10);

          return data;
        },
      },
    },
  });

  // メッセージをDBに保存（非同期）
  result.onFinish(async ({ text }) => {
    await supabase.from('messages').insert([
      {
        conversation_id: conversationId,
        role: 'user',
        content: messages[messages.length - 1].content,
      },
      {
        conversation_id: conversationId,
        role: 'assistant',
        content: text,
      },
    ]);
  });

  return result.toAIStreamResponse();
}
```
