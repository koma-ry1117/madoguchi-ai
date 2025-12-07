# 音声処理API

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## POST /api/voice/transcribe

**説明**: 音声→テキスト変換（OpenAI Whisper）

**認証**: 必須

**リクエスト**: multipart/form-data
```
audio: <audio file (mp3, wav, webm)>
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "text": "文京区で2LDKの物件を探しています",
    "duration": 3.5
  }
}
```

**実装例**:
```typescript
// app/api/voice/transcribe/route.ts
import { NextResponse } from 'next/server';
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export async function POST(request: Request) {
  const formData = await request.formData();
  const audio = formData.get('audio') as File;

  if (!audio) {
    return NextResponse.json(
      { success: false, error: { code: 'MISSING_AUDIO', message: '音声ファイルが必要です' } },
      { status: 400 }
    );
  }

  const transcription = await openai.audio.transcriptions.create({
    file: audio,
    model: 'whisper-1',
    language: 'ja',
  });

  return NextResponse.json({
    success: true,
    data: {
      text: transcription.text,
    },
  });
}
```

---

## POST /api/voice/synthesize

**説明**: テキスト→音声変換（ElevenLabs）+ キャッシング

**認証**: 必須

**リクエスト**:
```json
{
  "text": "かしこまりました。文京区で2LDKの物件ですね。"
}
```

**レスポンス**: audio/mpeg (音声バイナリ)

**実装例**:
```typescript
// app/api/voice/synthesize/route.ts
import { NextResponse } from 'next/server';
import { ElevenLabsClient } from 'elevenlabs';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import crypto from 'crypto';

const elevenlabs = new ElevenLabsClient({
  apiKey: process.env.ELEVENLABS_API_KEY,
});

export async function POST(request: Request) {
  const { text } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // テキストのハッシュ値を生成
  const hash = crypto.createHash('sha256').update(text).digest('hex');
  const cacheKey = `audio-cache/${hash}.mp3`;

  // キャッシュ確認
  const { data: cachedAudio } = await supabase.storage
    .from('audio-cache')
    .download(cacheKey);

  if (cachedAudio) {
    return new Response(cachedAudio, {
      headers: {
        'Content-Type': 'audio/mpeg',
      },
    });
  }

  // 音声生成
  const audio = await elevenlabs.textToSpeech.convert('voice_id_here', {
    text,
    model_id: 'eleven_multilingual_v2',
  });

  const audioBuffer = Buffer.from(await audio.arrayBuffer());

  // キャッシュ保存
  await supabase.storage.from('audio-cache').upload(cacheKey, audioBuffer, {
    contentType: 'audio/mpeg',
  });

  return new Response(audioBuffer, {
    headers: {
      'Content-Type': 'audio/mpeg',
    },
  });
}
```
