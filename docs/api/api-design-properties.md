# 物件API

> 本ドキュメントは [API設計](../api-design.md) の一部です。

## GET /api/properties

**説明**: 物件一覧取得（オペレーター管理画面用）

**認証**: 必須（operator または admin）

**用途**: オペレーターが自社物件を管理するためのAPI。キオスク利用者は直接このAPIにアクセスしない（AI会話内で内部的に検索される）。

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| area | string | エリア（文京区等） |
| rent_min | number | 最低賃料 |
| rent_max | number | 最高賃料 |
| layout | string | 間取り（1K, 2LDK等） |
| building_type | string | 建物種別 |
| is_public | boolean | 公開ステータス |
| page | number | ページ番号（デフォルト: 1） |
| limit | number | 取得件数（デフォルト: 20） |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "properties": [
      {
        "id": "uuid",
        "title": "文京区本郷 2LDK マンション",
        "address": "東京都文京区本郷1-2-3",
        "area": "文京区",
        "rent": 150000,
        "layout": "2LDK",
        "building_type": "マンション",
        "floor_area": 55.5,
        "station": "本郷三丁目駅",
        "distance_from_station": 5,
        "images": [
          {
            "url": "https://..."
          }
        ]
      }
    ],
    "pagination": {
      "total": 100,
      "page": 1,
      "limit": 20,
      "total_pages": 5
    }
  }
}
```

**実装例**:
```typescript
// app/api/properties/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);

  const area = searchParams.get('area');
  const rent_min = Number(searchParams.get('rent_min')) || 0;
  const rent_max = Number(searchParams.get('rent_max')) || 999999999;
  const layout = searchParams.get('layout');
  const page = Number(searchParams.get('page')) || 1;
  const limit = Number(searchParams.get('limit')) || 20;

  const supabase = createRouteHandlerClient({ cookies });

  let query = supabase
    .from('properties')
    .select('*, property_images(*)', { count: 'exact' })
    .eq('is_public', true)
    .gte('rent', rent_min)
    .lte('rent', rent_max)
    .range((page - 1) * limit, page * limit - 1);

  if (area) query = query.eq('area', area);
  if (layout) query = query.eq('layout', layout);

  const { data, error, count } = await query;

  if (error) {
    return NextResponse.json(
      { success: false, error: { code: 'DB_ERROR', message: error.message } },
      { status: 500 }
    );
  }

  return NextResponse.json({
    success: true,
    data: {
      properties: data,
      pagination: {
        total: count,
        page,
        limit,
        total_pages: Math.ceil(count / limit),
      },
    },
  });
}
```

---

## GET /api/properties/[id]

**説明**: 物件詳細取得

**認証**: 必須

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "title": "...",
    "address": "...",
    "rent": 150000,
    "layout": "2LDK",
    "description": "...",
    "images": [...]
  }
}
```

---

## POST /api/properties/import

**説明**: Chrome拡張からの物件取り込み

**認証**: 必須（operator or admin）

**リクエスト**:
```json
{
  "html": "<html>...</html>",
  "url": "https://www.athome.co.jp/..."
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "property_id": "uuid",
    "import_log_id": "uuid",
    "message": "物件を取り込みました"
  }
}
```

**実装例**:
```typescript
// app/api/properties/import/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { openai } from '@ai-sdk/openai';
import { generateObject } from 'ai';
import { z } from 'zod';

const PropertySchema = z.object({
  title: z.string(),
  address: z.string(),
  area: z.string(),
  rent: z.number(),
  layout: z.string(),
  building_type: z.string(),
  floor_area: z.number().optional(),
  station: z.string().optional(),
  distance_from_station: z.number().optional(),
  description: z.string().optional(),
});

export async function POST(request: Request) {
  const { html, url } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // 認証チェック
  const { data: { user }, error: authError } = await supabase.auth.getUser();
  if (authError || !user) {
    return NextResponse.json(
      { success: false, error: { code: 'UNAUTHORIZED', message: '認証が必要です' } },
      { status: 401 }
    );
  }

  // 権限チェック
  const { data: userData } = await supabase
    .from('auth.users')
    .select('role')
    .eq('id', user.id)
    .single();

  if (!['operator', 'admin'].includes(userData.role)) {
    return NextResponse.json(
      { success: false, error: { code: 'FORBIDDEN', message: '権限がありません' } },
      { status: 403 }
    );
  }

  // GPT-4oで構造化データ抽出
  const { object } = await generateObject({
    model: openai('gpt-4o'),
    schema: PropertySchema,
    prompt: `以下のHTMLから物件情報を抽出してください：\n\n${html}`,
  });

  // Embedding生成
  const embeddingResponse = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: `${object.title} ${object.description}`,
  });

  // Supabaseに保存
  const { data: property, error: insertError } = await supabase
    .from('properties')
    .insert({
      ...object,
      embedding: embeddingResponse.data[0].embedding,
      created_by: user.id,
    })
    .select()
    .single();

  if (insertError) {
    return NextResponse.json(
      { success: false, error: { code: 'DB_ERROR', message: insertError.message } },
      { status: 500 }
    );
  }

  // HTMLスナップショット保存
  const htmlPath = `html-snapshots/${property.id}.html`;
  await supabase.storage.from('snapshots').upload(htmlPath, html, {
    contentType: 'text/html',
  });

  // 取り込みログ保存
  const { data: importLog } = await supabase
    .from('property_import_logs')
    .insert({
      property_id: property.id,
      source_url: url,
      html_snapshot_path: htmlPath,
      imported_by: user.id,
    })
    .select()
    .single();

  return NextResponse.json({
    success: true,
    data: {
      property_id: property.id,
      import_log_id: importLog.id,
      message: '物件を取り込みました',
    },
  });
}
```

---

## POST /api/properties/search

**説明**: セマンティック検索（pgvector）

**認証**: 必須

**リクエスト**:
```json
{
  "query": "ペット可で広めの2LDK、駅近希望"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "properties": [
      {
        "id": "uuid",
        "title": "...",
        "similarity": 0.92
      }
    ]
  }
}
```

**実装例**:
```typescript
// app/api/properties/search/route.ts
export async function POST(request: Request) {
  const { query } = await request.json();

  // クエリのベクトル化
  const embeddingResponse = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: query,
  });

  // pgvectorで類似検索
  const { data } = await supabase.rpc('match_properties', {
    query_embedding: embeddingResponse.data[0].embedding,
    match_threshold: 0.8,
    match_count: 10,
  });

  return NextResponse.json({
    success: true,
    data: {
      properties: data,
    },
  });
}
```
