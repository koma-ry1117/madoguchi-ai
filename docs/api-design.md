# API設計

## API構成

Next.js App Router の **Route Handlers** を使用

```
app/api/
├── auth/
│   ├── login/route.ts
│   ├── logout/route.ts
│   └── refresh/route.ts
├── chat/
│   └── route.ts
├── voice/
│   ├── transcribe/route.ts
│   └── synthesize/route.ts
├── properties/
│   ├── route.ts
│   ├── [id]/route.ts
│   ├── import/route.ts
│   └── search/route.ts
├── customers/
│   └── preferences/route.ts
└── admin/
    ├── users/route.ts
    └── logs/route.ts
```

---

## 共通仕様

### ベースURL
```
開発環境: http://localhost:3000/api
本番環境: https://madoguchi-ai.vercel.app/api
```

### 認証方式
- **Bearer Token** (JWT)
- Supabase Authで発行

```http
Authorization: Bearer <access_token>
```

### レスポンス形式

#### 成功レスポンス
```json
{
  "success": true,
  "data": { ... }
}
```

#### エラーレスポンス
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": { ... }
  }
}
```

### HTTPステータスコード
| コード | 説明 |
|--------|------|
| 200 | 成功 |
| 201 | 作成成功 |
| 400 | リクエストエラー |
| 401 | 認証エラー |
| 403 | 権限エラー |
| 404 | リソースが見つからない |
| 429 | レート制限超過 |
| 500 | サーバーエラー |

---

## APIドキュメント一覧

本ドキュメントは以下のサブドキュメントに分割されています：

| ドキュメント | 説明 |
|-------------|------|
| [認証API](./api/api-design-auth.md) | ログイン・ログアウト・トークンリフレッシュ |
| [キオスク・AI会話API](./api/api-design-kiosk.md) | キオスクAPI、AI会話エンドポイント |
| [音声処理API](./api/api-design-voice.md) | 音声→テキスト変換、テキスト→音声変換 |
| [物件API](./api/api-design-properties.md) | 物件一覧・詳細・取り込み・検索 |
| [内見予約API](./api/api-design-viewings.md) | 内見予約の作成・取得・更新 |
| [メール送信・営業メールAPI](./api/api-design-emails.md) | メール送信、キャンペーン管理、配信停止リスト |
| [管理者API](./api/api-design-admin.md) | オペレーター管理、システム統計 |
| [レート制限・エラーコード](./api/api-design-errors.md) | レート制限実装、エラーコード一覧 |

---

次のドキュメント: [コスト見積もり](./cost-estimation.md)
