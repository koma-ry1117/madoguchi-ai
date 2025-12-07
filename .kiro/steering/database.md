# Database Standards

Supabase（PostgreSQL）を使用。Row Level Security（RLS）によるマルチテナント分離。

## Philosophy
- Model the domain first; optimize after correctness
- RLSでセキュリティを担保、アプリケーション層でバイパス不可
- Prefer explicit constraints; let database enforce invariants

## Naming & Types
- Tables: `snake_case`, plural (`properties`, `customers`, `email_logs`)
- Columns: `snake_case` (`created_at`, `operator_id`)
- FKs: `{table}_id` referencing `{table}.id`
- ID: UUID（`gen_random_uuid()`）
- Timestamps: `TIMESTAMPTZ`（タイムゾーン付き）
- Money: `INTEGER`（円単位）

## Core Tables

### マルチテナント構造
```
operators（不動産会社）
  ├── properties（物件）
  ├── customers（顧客）
  ├── conversations（会話）
  ├── viewings（内見予約）
  ├── inquiries（問い合わせ）
  └── email_logs（メール送信ログ）
```

### 営業メール関連
- `email_suppressions` - 配信停止リスト（バウンス・苦情）
- `marketing_campaigns` - キャンペーン管理
- `marketing_campaign_recipients` - 配信対象者
- `marketing_email_queue` - メールキュー・リトライ

## Row Level Security (RLS)

### パターン
```sql
-- オペレーター：自社データのみ
CREATE POLICY "operator_own_data"
ON {table} FOR ALL
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
);

-- 顧客：自分が所属するオペレーターのデータのみ
CREATE POLICY "customer_own_operator_data"
ON {table} FOR SELECT
TO authenticated
USING (
  operator_id IN (
    SELECT operator_id FROM customers WHERE user_id = auth.uid()
  )
);

-- 管理者：全データ
CREATE POLICY "admin_all"
ON {table} FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

## Migrations
- Supabase CLI: `supabase migration new {name}`
- 命名: `{timestamp}_{action}_{object}`（例: `20250115_add_email_suppressions`）
- 本番適用前にローカルでテスト

## Query Patterns

### Supabaseクライアント使用
```typescript
// 自動的にRLSが適用される
const { data } = await supabase
  .from('properties')
  .select('*')
  .eq('is_public', true);
```

### 複雑なクエリはRPC
```sql
-- 営業推奨ユーザー検索
CREATE FUNCTION get_marketing_recipients(...)
RETURNS TABLE(...) AS $$
BEGIN
  -- 配信停止リスト除外
  -- 直近送信済み除外
  -- 条件マッチング
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Indexes
- FK列には必ずインデックス
- `WHERE`で頻繁に使用する列
- `ORDER BY`で使用する列
- 複合インデックスは選択性の高い列を先頭に

```sql
CREATE INDEX idx_properties_operator_id ON properties(operator_id);
CREATE INDEX idx_email_logs_status_sent_at ON email_logs(status, sent_at DESC);
```

## Data Integrity
- `NOT NULL`を積極的に使用
- `CHECK`制約でenum値を制限
- `UNIQUE`制約で重複防止
- カスケード削除は慎重に（`ON DELETE CASCADE` vs `SET NULL`）

---
_Focus on patterns and decisions. No environment-specific settings._
