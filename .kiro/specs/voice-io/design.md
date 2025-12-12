# Design Document

## Overview

**Purpose**: キオスク端末での音声会話インターフェースを提供し、タイピングが苦手な顧客（特に高齢者）でも快適に物件を探せるようにする。

**Users**: キオスク端末を利用する店舗来店客（認証不要）

### Goals
- 自然な日本語音声認識
- 高品質な音声合成
- 低レイテンシーの応答
- コスト効率の良いキャッシング
- **高齢者にも使いやすいUI**: 音声入力ボタンを画面中央に大きく配置

### Non-Goals
- 多言語対応（日本語のみ）
- リアルタイム字幕生成
- 音声ファイルのアップロード
- オペレーター管理画面での音声機能（テキストベースのみ）

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    キオスク端末（店舗設置）                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    VoiceInterface                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │  │
│  │  │ VoiceInput  │  │ AudioPlayer │  │ VoiceWaveform   │   │  │
│  │  │ (Record)    │  │ (Playback)  │  │ (Visualization) │   │  │
│  │  └──────┬──────┘  └──────┬──────┘  └─────────────────┘   │  │
│  └─────────┼────────────────┼───────────────────────────────┘  │
└────────────┼────────────────┼──────────────────────────────────┘
             │                │
             ▼                ▼
┌────────────────────┐  ┌────────────────────┐
│ POST /api/voice/   │  │ POST /api/voice/   │
│ transcribe         │  │ synthesize         │
│                    │  │                    │
│ OpenAI Whisper API │  │ Google Cloud TTS   │
└────────────────────┘  │ + Supabase Cache   │
                        └────────────────────┘
```

### Technology Stack

| Layer | Choice / Version | Role |
|-------|------------------|------|
| Audio Recording | Web Audio API, MediaRecorder | ブラウザ音声録音 |
| STT | OpenAI Whisper API | 音声→テキスト |
| TTS | Google Cloud TTS（WaveNet） | テキスト→音声 |
| Cache | Supabase Storage | 音声キャッシュ |
| Audio Playback | HTML5 Audio API | 音声再生 |

## System Flows

### 音声入力フロー（キオスクUI統合）

```mermaid
sequenceDiagram
    participant U as キオスク利用者
    participant UI as VoiceInput
    participant Kiosk as KioskInterface
    participant API as /api/voice/transcribe
    participant W as OpenAI Whisper

    U->>UI: マイクボタンクリック
    UI->>UI: MediaRecorder開始
    UI-->>U: 波形表示（録音中）

    U->>UI: 録音停止（または自動停止）
    UI->>UI: Blob生成
    UI->>API: POST (audio file)
    API->>W: transcriptions.create()
    W-->>API: text
    API-->>UI: { text }
    UI->>Kiosk: onTranscribe(text)
    Kiosk->>Kiosk: AI会話APIに送信
```

### 音声出力フロー（ストリーミングTTS）

**設計方針**: テキストを先行表示し、音声はバックグラウンドで準備。ユーザーは待ち時間中にテキストを読める。

```mermaid
sequenceDiagram
    participant AI as Gemini (Stream)
    participant Kiosk as KioskInterface
    participant AP as AudioPlayer
    participant API as /api/voice/synthesize
    participant Cache as Supabase Storage
    participant GC as Google Cloud TTS

    AI-->>Kiosk: テキストストリーム開始
    Kiosk->>Kiosk: テキスト即座に表示開始

    Note over Kiosk: 文単位でチャンク分割

    loop 各チャンク（文単位）
        Kiosk->>AP: チャンク送信
        AP->>API: POST { text: chunk }
        API->>API: SHA256(chunk) → hash
        API->>Cache: download(hash.mp3)

        alt Cache Hit
            Cache-->>API: audio data
        else Cache Miss
            API->>GC: synthesizeSpeech()
            GC-->>API: audio stream
            API->>Cache: upload(hash.mp3)
        end

        API-->>AP: audio/mpeg
        AP->>AP: AudioQueueに追加
    end

    Note over AP: 最初のチャンク準備完了で再生開始
    AP->>AP: AudioQueue順次再生
    AP-->>Kiosk: onPlayStart()
    Note over Kiosk: アバター感情切り替え
    AP-->>Kiosk: onPlayEnd()
```

### チャンク分割ロジック

```typescript
// 日本語の文末で分割（。！？で区切る）
const splitIntoChunks = (text: string): string[] => {
  const chunks = text.split(/(?<=[。！？])/);
  return chunks.filter(chunk => chunk.trim().length > 0);
};

// 例: "こちらの物件は駅から徒歩5分です。おすすめですよ。"
// → ["こちらの物件は駅から徒歩5分です。", "おすすめですよ。"]
```

### 音声再生キュー

```typescript
interface AudioQueue {
  chunks: AudioBuffer[];
  currentIndex: number;
  isPlaying: boolean;

  enqueue(audio: AudioBuffer): void;
  play(): void;
  skip(): void;  // 読み上げスキップ
}
```

## Components and Interfaces

### Frontend Components

#### VoiceInput（キオスク専用）
| Field | Detail |
|-------|--------|
| Intent | 音声録音とテキスト変換 |
| Requirements | 1 |

**UI設計（高齢者対応）**
```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                                                         │
│              ┌─────────────────────┐                   │
│              │                     │                   │
│              │    🎤               │                   │
│              │                     │                   │
│              │   タップして        │                   │
│              │   お話しください     │                   │
│              │                     │                   │
│              └─────────────────────┘                   │
│                  (120px × 120px)                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ [___テキスト入力___]              [送信]        │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**設計ポイント**
- 音声ボタンは画面中央に大きく配置（120px × 120px以上）
- ボタンには「タップしてお話しください」のラベルを表示
- 録音中は波形アニメーションとカウントダウン表示
- テキスト入力は補助的な位置（下部に小さく配置）

**State**
```typescript
interface VoiceInputState {
  isRecording: boolean;
  audioBlob: Blob | null;
  isProcessing: boolean;
  recordingDuration: number;
}
```

**Props**
```typescript
interface VoiceInputProps {
  onTranscribe: (text: string) => void;
  disabled?: boolean;
  maxDuration?: number; // デフォルト60秒
  size?: 'large' | 'medium'; // デフォルト'large'（キオスク用）
}
```

#### AudioPlayer（キオスク専用）
| Field | Detail |
|-------|--------|
| Intent | ストリーミング音声再生とアバター連携 |
| Requirements | 2, 4 |

**Props**
```typescript
interface AudioPlayerProps {
  chunks: string[];           // 文単位のチャンク配列
  autoPlay: boolean;          // キオスクでは常にtrue
  onPlayStart?: () => void;
  onPlayEnd?: () => void;
  onInterrupt?: () => void;   // ユーザーが話し始めた時
  onSkip?: () => void;        // スキップボタン押下時
}
```

**State**
```typescript
interface AudioPlayerState {
  queue: AudioBuffer[];
  currentIndex: number;
  isPlaying: boolean;
  isSkipped: boolean;
}
```

**スキップ機能**
- ユーザーが読み上げ中に「スキップ」ボタンを押すと即座に停止
- 残りのチャンクのTTSリクエストもキャンセル
- テキストは表示済みなので情報は失われない

#### VoiceWaveform
| Field | Detail |
|-------|--------|
| Intent | 録音中の波形表示 |
| Requirements | 1 |

**Props**
```typescript
interface VoiceWaveformProps {
  analyser: AnalyserNode | null;
  isRecording: boolean;
}
```

### API Routes

#### POST /api/voice/transcribe

認証なしでアクセス可能（キオスク用）

**Request**: multipart/form-data
```
audio: File (webm/mp3)
operator_id: string (レート制限用)
```

**Response**
```json
{
  "success": true,
  "data": {
    "text": "文京区で2LDKの物件を探しています",
    "duration": 3.5,
    "language": "ja"
  }
}
```

#### POST /api/voice/synthesize

認証なしでアクセス可能（キオスク用）

**Request**
```json
{
  "text": "かしこまりました。文京区で2LDKの物件ですね。",
  "operator_id": "uuid"
}
```

**Response**: audio/mpeg (binary)

**Headers**
```
X-Cache-Status: HIT | MISS
Content-Type: audio/mpeg
```

## Data Models

### Audio Cache Key
```
audio-cache/{operator_id}/{sha256_hash}.mp3
```

### Supabase Storage Bucket
- Bucket: `audio-cache`
- Public: No
- Max file size: 5MB
- Allowed types: audio/mpeg
- Lifecycle: 30日後自動削除

### Pre-cached Phrases（事前キャッシュ）

**目的**: キャッシュヒット率を高め、TTS待ち時間を削減

```typescript
const PRECACHED_PHRASES = [
  // 挨拶・基本応答
  "いらっしゃいませ。",
  "何かお探しでしょうか？",
  "かしこまりました。",
  "ありがとうございます。",
  "またのご来店をお待ちしております。",

  // 確認パターン
  "ご予算はいくらくらいをお考えですか？",
  "間取りのご希望はありますか？",
  "エリアのご希望はありますか？",
  "駅からの距離はどのくらいがよろしいですか？",
  "その他にご希望の条件はありますか？",

  // 提案パターン
  "こちらの物件はいかがでしょうか。",
  "おすすめの物件がございます。",
  "条件に合う物件が見つかりました。",

  // 内見予約
  "内見のご予約を承ります。",
  "ご都合の良い日時をお聞かせください。",
  "ご連絡先をお教えください。",
  "ご予約ありがとうございます。",
  "担当者よりご連絡いたします。",
];
```

**キャッシュ戦略**
- 起動時に事前キャッシュフレーズを非同期でTTS生成・保存
- 文単位のチャンクがキャッシュフレーズと完全一致すればキャッシュヒット
- キャッシュヒット率をモニタリングし、頻出フレーズを随時追加

## Error Handling

### Transcription Errors
- マイク許可拒否 → テキスト入力にフォールバック、許可を促すメッセージ
- 録音失敗 → 再試行ボタン表示
- Whisper APIエラー → テキスト入力を促す
- 無音検出 → 「お話が聞き取れませんでした」メッセージ

### Synthesis Errors
- Google Cloud TTS APIエラー → テキストのみ表示（音声なし）
- キャッシュ保存失敗 → ログ出力（サイレント失敗）
- 音声再生失敗 → 再生ボタン表示

## Testing Strategy

### Unit Tests
- VoiceInput: 録音状態管理
- AudioPlayer: 再生コントロール
- キャッシュキー生成

### Integration Tests
- /api/voice/transcribe: Whisper API呼び出し
- /api/voice/synthesize: Google Cloud TTS + キャッシュ

### E2E Tests
- 録音→変換→AI送信
- AI応答→音声生成→自動再生

## Performance Considerations

### 基本設定
- 音声ファイルサイズ: 最大5MB
- 録音最大時間: 60秒
- Google Cloud TTS レート制限: operator_idごとにレート制限

### ストリーミングTTS最適化
- **テキスト先行表示**: 音声準備完了を待たずにテキストを即座に表示
- **チャンク分割**: 文単位で分割し、最初のチャンク準備完了で再生開始
- **並列リクエスト**: 複数チャンクのTTSを並列で生成
- **体感待ち時間**: 全文一括より50%以上削減を目標

### キャッシュ最適化
- キャッシュヒット率の監視（目標: 40%以上）
- 事前キャッシュ: よく使うフレーズを起動時にキャッシュ
- 文単位キャッシュ: 同じ文は再利用される確率が高い

### UX設計
- スキップボタン: ユーザーが読み上げをスキップ可能
- 音声なしフォールバック: TTS失敗時はテキストのみ表示（情報は失われない）
