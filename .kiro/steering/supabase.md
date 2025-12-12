# Supabase Patterns

Next.js 15 App Router環境でのSupabase統合パターン。

## Package Migration

`@supabase/auth-helpers-nextjs` は**非推奨**。`@supabase/ssr` を使用：

```bash
pnpm add @supabase/supabase-js @supabase/ssr
```

## Client Creation

### Server Component用クライアント

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies(); // Next.js 15では await が必須

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Server Componentからの呼び出し時は無視
          }
        },
      },
    }
  );
}
```

### Client Component用クライアント

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

## Middleware Session Management

**重要**: Server Componentsはクッキーを読み取れるが、書き込みはできない。トークンリフレッシュはMiddlewareで行う。

```typescript
// middleware.ts
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({
    request,
  });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          // 1. リクエストのクッキーを更新
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          // 2. レスポンスを再作成
          supabaseResponse = NextResponse.next({
            request,
          });
          // 3. レスポンスのクッキーを更新
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // getUser() でセッション検証（getSession()は使用禁止）
  const {
    data: { user },
  } = await supabase.auth.getUser();

  // 認証が必要なパスの保護
  if (
    !user &&
    !request.nextUrl.pathname.startsWith('/login') &&
    !request.nextUrl.pathname.startsWith('/auth') &&
    !request.nextUrl.pathname.startsWith('/chat') // キオスクは認証不要
  ) {
    const url = request.nextUrl.clone();
    url.pathname = '/login';
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

### 重要ポイント

1. **`getUser()` を使用**: `getSession()` はクライアントから改ざん可能なため、サーバーサイドでは必ず `getUser()` を使用
2. **レスポンスを返す**: クッキーがセットされた `supabaseResponse` を必ず return
3. **両方のクッキーを更新**: `request.cookies` と `response.cookies` の両方を更新

## Type Generation

### 自動生成コマンド

```bash
# ローカルDBから生成
supabase gen types typescript --local > src/types/database.types.ts

# リモートDBから生成
supabase gen types typescript --project-id <project-id> > src/types/database.types.ts
```

### 型の使用

```typescript
// types/database.types.ts は自動生成（手動編集禁止）
import { Database } from '@/types/database.types';

// テーブル行の型
type Property = Database['public']['Tables']['properties']['Row'];

// Insert用の型
type PropertyInsert = Database['public']['Tables']['properties']['Insert'];

// Update用の型
type PropertyUpdate = Database['public']['Tables']['properties']['Update'];

// クライアントに型を適用
const supabase = createClient<Database>();
```

### Zodスキーマとの同期

`supazod` などのツールでDBスキーマからZodスキーマを自動生成：

```typescript
// schemas/property.ts（自動生成）
import { z } from 'zod';

export const propertySchema = z.object({
  id: z.string().uuid(),
  name: z.string().max(100),
  price: z.number().positive(),
  // ...
});
```

## Data Fetching Patterns

### Server Componentでの取得

```typescript
// app/properties/page.tsx
export default async function PropertiesPage() {
  const supabase = await createClient();

  const { data: properties, error } = await supabase
    .from('properties')
    .select(`
      *,
      operator:operators(name)
    `)
    .eq('is_active', true)
    .order('created_at', { ascending: false });

  if (error) {
    throw error; // error.tsx でキャッチ
  }

  return <PropertyList properties={properties} />;
}
```

### Server Actionでの変更

```typescript
// actions/properties.ts
'use server';

import { revalidatePath } from 'next/cache';
import { createClient } from '@/lib/supabase/server';
import { propertySchema } from '@/schemas/property';

export async function createProperty(formData: FormData) {
  const supabase = await createClient();

  // 認証チェック
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return { error: 'Unauthorized' };
  }

  // バリデーション
  const parsed = propertySchema.omit({ id: true }).safeParse(
    Object.fromEntries(formData)
  );
  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }

  // 挿入（RLSも適用される）
  const { error } = await supabase
    .from('properties')
    .insert({
      ...parsed.data,
      operator_id: user.id,
    });

  if (error) {
    console.error('Insert error:', error);
    return { error: '保存に失敗しました' };
  }

  revalidatePath('/properties');
  return { success: true };
}
```

## Realtime Subscriptions

Client Componentで使用：

```typescript
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';

export function RealtimeMessages({ chatId }: { chatId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const supabase = createClient();

  useEffect(() => {
    // 初期データ取得
    const fetchMessages = async () => {
      const { data } = await supabase
        .from('messages')
        .select('*')
        .eq('chat_id', chatId)
        .order('created_at');
      if (data) setMessages(data);
    };
    fetchMessages();

    // リアルタイム購読
    const channel = supabase
      .channel(`chat:${chatId}`)
      .on(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'messages',
          filter: `chat_id=eq.${chatId}`,
        },
        (payload) => {
          setMessages((prev) => [...prev, payload.new as Message]);
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [chatId, supabase]);

  return (/* render messages */);
}
```

## Storage

### ファイルアップロード

```typescript
const { data, error } = await supabase.storage
  .from('property-images')
  .upload(`${propertyId}/${file.name}`, file, {
    cacheControl: '3600',
    upsert: false,
  });
```

### 署名付きURL取得

```typescript
const { data } = await supabase.storage
  .from('property-images')
  .createSignedUrl(path, 3600); // 1時間有効
```

## Common Pitfalls

### 1. クッキーの非同期アクセス

```typescript
// ❌ Next.js 15でエラー
const cookieStore = cookies();

// ✅ await が必要
const cookieStore = await cookies();
```

### 2. getSession vs getUser

```typescript
// ❌ サーバーサイドで使用禁止（改ざん可能）
const { data: { session } } = await supabase.auth.getSession();

// ✅ サーバーサイドでは必ず getUser
const { data: { user } } = await supabase.auth.getUser();
```

### 3. Middlewareからレスポンスを返さない

```typescript
// ❌ クッキーが更新されない
await supabase.auth.getUser();
return NextResponse.next();

// ✅ 更新されたレスポンスを返す
await supabase.auth.getUser();
return supabaseResponse;
```

---
_Supabase SSR is the foundation - get it right from the start_
