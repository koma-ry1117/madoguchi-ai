# データベース設計 - セキュリティ・検索・運用

> 本ドキュメントは [データベース設計](../database-design.md) の一部です。

## Row Level Security（RLS）設定

### operators テーブル

```sql
-- RLS有効化
ALTER TABLE operators ENABLE ROW LEVEL SECURITY;

-- オペレーター：自分の企業情報のみ閲覧・更新
CREATE POLICY "operator_own_data"
ON operators FOR ALL
TO authenticated
USING (
  user_id = auth.uid()
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべてのオペレーター情報を閲覧・管理
CREATE POLICY "admin_all_operators"
ON operators FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

---

### properties テーブル

キオスク利用者は認証しないため、物件検索はオペレーターのセッションで行われる。

```sql
-- RLS有効化
ALTER TABLE properties ENABLE ROW LEVEL SECURITY;

-- オペレーター：自社の物件を閲覧・更新
CREATE POLICY "operator_manage_own_properties"
ON properties FOR ALL
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
)
WITH CHECK (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべての物件を閲覧・管理
CREATE POLICY "admin_all_properties"
ON properties FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

**注意**: キオスク利用者は認証しない。物件検索はオペレーターのセッションで実行されるため、顧客向けのRLSポリシーは存在しない。

---

### property_import_logs テーブル

```sql
ALTER TABLE property_import_logs ENABLE ROW LEVEL SECURITY;

-- オペレーター：自社の取り込みログのみ閲覧
CREATE POLICY "operator_view_own_logs"
ON property_import_logs FOR SELECT
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべてのログを閲覧
CREATE POLICY "admin_view_all_logs"
ON property_import_logs FOR SELECT
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

---

### customers テーブル

キオスク利用者は認証アカウントを持たないため、顧客向けのRLSポリシーは不要。

```sql
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

-- オペレーター：自社の顧客情報のみ閲覧・管理
CREATE POLICY "operator_manage_own_customers"
ON customers FOR ALL
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべて閲覧・管理可能
CREATE POLICY "admin_all_customers"
ON customers FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

**注意**: 顧客（kiosk_user）は認証しないため、顧客向けのRLSポリシーは存在しない。

---

### conversations & messages

キオスク利用者は認証しないため、顧客向けのRLSポリシーは不要。

```sql
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- オペレーター：自社の会話を閲覧・作成
CREATE POLICY "operator_manage_own_conversations"
ON conversations FOR ALL
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- オペレーター：自社の会話メッセージを閲覧・作成
CREATE POLICY "operator_manage_own_messages"
ON messages FOR ALL
TO authenticated
USING (
  conversation_id IN (
    SELECT c.id FROM conversations c
    WHERE c.operator_id IN (
      SELECT id FROM operators WHERE user_id = auth.uid()
    )
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべて閲覧可能
CREATE POLICY "admin_view_all_conversations"
ON conversations FOR SELECT
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);

CREATE POLICY "admin_view_all_messages"
ON messages FOR SELECT
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

**注意**: 会話データはオペレーターのセッションで作成・管理される。顧客向けのRLSポリシーは存在しない。

---

## pgvector による類似物件検索

### ベクトル検索関数

```sql
CREATE OR REPLACE FUNCTION match_properties(
  query_embedding vector(1536),
  match_threshold float DEFAULT 0.8,
  match_count int DEFAULT 10
)
RETURNS TABLE (
  id uuid,
  title varchar,
  address text,
  rent integer,
  layout varchar,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    p.id,
    p.title,
    p.address,
    p.rent,
    p.layout,
    1 - (p.embedding <=> query_embedding) AS similarity
  FROM properties p
  WHERE 1 - (p.embedding <=> query_embedding) > match_threshold
    AND p.is_public = true
  ORDER BY p.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

### 使用例

```typescript
// 顧客の希望条件をベクトル化
const queryText = "文京区で2LDK、家賃15万円以内のペット可物件";
const embedding = await openai.embeddings.create({
  model: 'text-embedding-3-small',
  input: queryText,
});

// Supabaseで類似検索
const { data } = await supabase.rpc('match_properties', {
  query_embedding: embedding.data[0].embedding,
  match_threshold: 0.8,
  match_count: 10,
});
```

---

## 全文検索

### 全文検索関数

```sql
CREATE OR REPLACE FUNCTION search_properties_fulltext(
  search_query text
)
RETURNS TABLE (
  id uuid,
  title varchar,
  address text,
  rent integer,
  layout varchar,
  rank float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    p.id,
    p.title,
    p.address,
    p.rent,
    p.layout,
    ts_rank(
      to_tsvector('japanese', p.title || ' ' || p.description),
      plainto_tsquery('japanese', search_query)
    ) AS rank
  FROM properties p
  WHERE to_tsvector('japanese', p.title || ' ' || p.description) @@
        plainto_tsquery('japanese', search_query)
    AND p.is_public = true
  ORDER BY rank DESC;
END;
$$;
```

---

## トリガー・関数

### 更新日時の自動更新

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- properties テーブルに適用
CREATE TRIGGER update_properties_updated_at
BEFORE UPDATE ON properties
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

### 会話終了時のメッセージ数カウント

```sql
CREATE OR REPLACE FUNCTION update_conversation_message_count()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE conversations
  SET total_messages = (
    SELECT COUNT(*) FROM messages WHERE conversation_id = NEW.conversation_id
  )
  WHERE id = NEW.conversation_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER count_messages_on_insert
AFTER INSERT ON messages
FOR EACH ROW
EXECUTE FUNCTION update_conversation_message_count();
```

---

## データ保持ポリシー

### 古いデータの削除

```sql
-- 1年以上前の会話ログを削除
CREATE OR REPLACE FUNCTION cleanup_old_conversations()
RETURNS void AS $$
BEGIN
  DELETE FROM conversations
  WHERE started_at < NOW() - INTERVAL '1 year';
END;
$$ LANGUAGE plpgsql;

-- 定期実行（Supabase Cron Jobs）
SELECT cron.schedule(
  'cleanup-old-conversations',
  '0 3 * * 0', -- 毎週日曜 3:00AM
  'SELECT cleanup_old_conversations();'
);
```

---

## バックアップ戦略

### Supabase自動バックアップ
- 毎日自動バックアップ（過去7日分保持）
- Point-in-Time Recovery (PITR) 対応

### 手動バックアップ

```bash
# pg_dumpでエクスポート
pg_dump -h <supabase-host> -U postgres -d postgres > backup.sql

# 復元
psql -h <supabase-host> -U postgres -d postgres < backup.sql
```
