# データベース設計

## データベース構成

Supabase PostgreSQL + pgvector を使用

---

## ER図

```
┌─────────────────┐
│   auth.users    │ (Supabase管理)
│─────────────────│
│ id (uuid) PK    │
│ email           │
│ role            │ ← 'admin', 'operator'
│ created_at      │
└────────┬────────┘
         │
         │ 1:1 (role='operator'の場合)
         │
┌────────▼──────────────────┐
│      operators            │ ← オペレーター企業テーブル
│───────────────────────────│
│ id (uuid) PK              │
│ user_id (uuid) FK         │ ← auth.users.id (role='operator')
│ company_name              │
│ company_address           │
│ phone                     │
│ email                     │
│ subscription_status       │ ← 'active', 'suspended', 'cancelled'
│ subscription_plan         │ ← 'basic', 'standard', 'premium'
│ contract_start_date       │
│ contract_end_date         │
│ is_active                 │
│ created_at                │
│ updated_at                │
└────────┬──────────────────┘
         │
         │ 1:N
         │
┌────────▼────────────────────────┐
│      properties                 │
│─────────────────────────────────│
│ id (uuid) PK                    │
│ operator_id (uuid) FK           │ ← どのオペレーターの物件か
│ title                           │
│ address                         │
│ area                            │
│ rent                            │
│ layout                          │
│ building_type                   │
│ floor_area                      │
│ station                         │
│ distance_from_station           │
│ description                     │
│ embedding (vector)              │ ← pgvector
│ is_public                       │
│ created_by (uuid) FK            │ ← auth.users.id
│ created_at                      │
│ updated_at                      │
└────────┬────────────────────────┘
         │
         │ 1:N
         │
┌────────▼──────────────┐
│  property_images      │
│───────────────────────│
│ id (uuid) PK          │
│ property_id (uuid) FK │
│ storage_path          │
│ display_order         │
│ created_at            │
└───────────────────────┘

┌──────────────────────────┐
│ property_import_logs     │
│──────────────────────────│
│ id (uuid) PK             │
│ property_id (uuid) FK    │
│ operator_id (uuid) FK    │
│ source_url               │
│ html_snapshot_path       │
│ imported_by (uuid) FK    │
│ imported_at              │
└──────────────────────────┘

┌──────────────────┐
│   customers      │
│──────────────────│
│ id (uuid) PK     │
│ operator_id (FK) │ ← どのオペレーターの顧客か
│ name             │ ← 名前
│ phone            │ ← 電話番号
│ email            │ ← メールアドレス
│ created_at       │
└────────┬─────────┘
※ user_idは不要（キオスク利用者は認証アカウントを持たない）
         │
         │ 1:N
         │
┌────────▼──────────────┐
│ customer_preferences  │
│───────────────────────│
│ id (uuid) PK          │
│ customer_id (uuid) FK │
│ area                  │
│ rent_min              │
│ rent_max              │
│ layout                │
│ building_type         │
│ updated_at            │
└───────────────────────┘

┌──────────────────────┐
│   conversations      │
│──────────────────────│
│ id (uuid) PK         │
│ customer_id (uuid) FK│
│ operator_id (uuid) FK│
│ started_at           │
│ ended_at             │
└────────┬─────────────┘
         │
         │ 1:N
         │
┌────────▼─────────┐
│    messages      │
│──────────────────│
│ id (uuid) PK     │
│ conversation_id  │
│ role             │
│ content          │
│ emotion          │
│ timestamp        │
└──────────────────┘

┌──────────────────┐
│   inquiries      │
│──────────────────│
│ id (uuid) PK     │
│ customer_id (FK) │
│ property_id (FK) │
│ operator_id (FK) │
│ status           │
│ message          │
│ created_at       │
└──────────────────┘

┌──────────────────┐
│   viewings       │
│──────────────────│
│ id (uuid) PK     │
│ customer_id (FK) │
│ property_id (FK) │
│ operator_id (FK) │
│ viewing_date     │
│ status           │
│ created_at       │
└──────────────────┘
```

---

## ドキュメント一覧

本ドキュメントは以下のサブドキュメントに分割されています：

| ドキュメント | 説明 |
|-------------|------|
| [テーブル定義](./database/database-design-tables.md) | operators, auth.users, properties, customers, conversations, viewings等 |
| [メール関連テーブル](./database/database-design-email.md) | email_logs, email_suppressions, marketing_campaigns等 |
| [セキュリティ・検索・運用](./database/database-design-security.md) | RLS設定、pgvector検索、全文検索、トリガー、バックアップ |

---

次のドキュメント: [API設計](./api-design.md)
