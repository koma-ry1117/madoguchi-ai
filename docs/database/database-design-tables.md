# データベース設計 - テーブル定義

> 本ドキュメントは [データベース設計](../database-design.md) の一部です。

## 1. operators（オペレーター企業マスタ）

```sql
CREATE TABLE operators (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- ユーザーアカウント紐付け
  user_id UUID UNIQUE NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  -- 企業情報
  company_name VARCHAR(255) NOT NULL,
  company_address TEXT,
  phone VARCHAR(20),
  email VARCHAR(255) NOT NULL,

  -- サブスクリプション情報
  subscription_status VARCHAR(20) DEFAULT 'active' CHECK (
    subscription_status IN ('active', 'suspended', 'cancelled', 'trial')
  ),
  subscription_plan VARCHAR(20) DEFAULT 'basic' CHECK (
    subscription_plan IN ('basic', 'standard', 'premium')
  ),

  -- 契約情報
  contract_start_date DATE NOT NULL,
  contract_end_date DATE,

  -- ステータス
  is_active BOOLEAN DEFAULT true,

  -- メタ情報
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- インデックス作成
CREATE INDEX idx_operators_user_id ON operators(user_id);
CREATE INDEX idx_operators_subscription_status ON operators(subscription_status);
CREATE INDEX idx_operators_is_active ON operators(is_active);

-- 更新日時の自動更新
CREATE TRIGGER update_operators_updated_at
BEFORE UPDATE ON operators
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

**カラム説明**:
| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PK | オペレーターID |
| user_id | uuid | FK, UNIQUE | auth.usersへの参照 |
| company_name | varchar(255) | NOT NULL | 企業名 |
| company_address | text | | 企業住所 |
| phone | varchar(20) | | 電話番号 |
| email | varchar(255) | NOT NULL | 企業メールアドレス |
| subscription_status | varchar(20) | NOT NULL | サブスク状態 |
| subscription_plan | varchar(20) | NOT NULL | プラン種別 |
| contract_start_date | date | NOT NULL | 契約開始日 |
| contract_end_date | date | | 契約終了日 |
| is_active | boolean | DEFAULT true | 有効フラグ |
| created_at | timestamptz | DEFAULT NOW() | 作成日時 |
| updated_at | timestamptz | DEFAULT NOW() | 更新日時 |

---

## 2. auth.users（Supabase管理テーブル）

```sql
-- Supabaseが自動生成・管理
-- カスタムフィールドを追加

ALTER TABLE auth.users
ADD COLUMN role VARCHAR(20) DEFAULT 'operator';

-- roleの制約（キオスク利用者は認証しないためcustomerは不要）
ALTER TABLE auth.users
ADD CONSTRAINT check_role
CHECK (role IN ('operator', 'admin'));
```

**カラム説明**:
| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PK | ユーザーID（Supabase自動生成） |
| email | varchar | UNIQUE | メールアドレス |
| role | varchar(20) | NOT NULL | ロール（operator/admin） |
| created_at | timestamptz | NOT NULL | 作成日時 |

**注意**: `customer`ロールは存在しない。キオスク利用者は認証アカウントを持たない。

---

## 3. properties（物件マスタ）

```sql
CREATE TABLE properties (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- オペレーター紐付け
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,

  -- 基本情報
  title VARCHAR(255) NOT NULL,
  address TEXT NOT NULL,
  area VARCHAR(100) NOT NULL, -- 文京区、千代田区など

  -- 賃料・売買
  transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('rent', 'sale')),
  rent INTEGER, -- 賃料（円/月） ※賃貸の場合
  sale_price BIGINT, -- 売買価格（円） ※売買の場合
  management_fee INTEGER, -- 管理費（円/月）
  key_money INTEGER, -- 礼金（ヶ月）
  deposit INTEGER, -- 敷金（ヶ月）

  -- 物件詳細
  layout VARCHAR(50) NOT NULL, -- 1K、1LDK、2LDKなど
  building_type VARCHAR(50) NOT NULL, -- マンション、一戸建て、アパート
  floor_area DECIMAL(6,2), -- 専有面積（㎡）
  floor_number INTEGER, -- 階数
  total_floors INTEGER, -- 総階数
  year_built INTEGER, -- 築年
  structure VARCHAR(50), -- 構造（RC造、木造など）

  -- 設備
  amenities JSONB, -- {balcony: true, parking: true, pet: false, ...}

  -- アクセス
  station VARCHAR(100), -- 最寄り駅
  distance_from_station INTEGER, -- 駅徒歩（分）

  -- 説明文
  description TEXT,

  -- ベクトル検索用（pgvector）
  embedding vector(1536), -- OpenAI Embeddings

  -- 公開設定
  is_public BOOLEAN DEFAULT true,

  -- メタ情報
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- インデックス作成
CREATE INDEX idx_properties_operator_id ON properties(operator_id);
CREATE INDEX idx_properties_area ON properties(area);
CREATE INDEX idx_properties_rent ON properties(rent);
CREATE INDEX idx_properties_layout ON properties(layout);
CREATE INDEX idx_properties_transaction_type ON properties(transaction_type);

-- 複合インデックス
CREATE INDEX idx_properties_search
ON properties(area, rent, layout, operator_id)
WHERE is_public = true;

-- ベクトル検索用インデックス（pgvector）
CREATE INDEX idx_properties_embedding
ON properties
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 全文検索用インデックス
CREATE INDEX idx_properties_fulltext
ON properties
USING gin(to_tsvector('japanese', title || ' ' || description));

-- 更新日時の自動更新
CREATE TRIGGER update_properties_updated_at
BEFORE UPDATE ON properties
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

**カラム説明**:
| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PK | 物件ID |
| operator_id | uuid | FK, NOT NULL | オペレーターID（どの不動産会社の物件か） |
| title | varchar(255) | NOT NULL | 物件タイトル |
| address | text | NOT NULL | 住所 |
| area | varchar(100) | NOT NULL | エリア（文京区等） |
| transaction_type | varchar(20) | NOT NULL | 取引種別（rent/sale） |
| rent | integer | | 賃料（円/月） |
| sale_price | bigint | | 売買価格（円） |
| layout | varchar(50) | NOT NULL | 間取り |
| building_type | varchar(50) | NOT NULL | 建物種別 |
| floor_area | decimal(6,2) | | 専有面積（㎡） |
| station | varchar(100) | | 最寄り駅 |
| distance_from_station | integer | | 駅徒歩（分） |
| description | text | | 物件説明 |
| embedding | vector(1536) | | ベクトル（セマンティック検索用） |
| is_public | boolean | DEFAULT true | 公開フラグ |
| created_by | uuid | FK | 登録者（auth.users.id） |
| created_at | timestamptz | DEFAULT NOW() | 作成日時 |
| updated_at | timestamptz | DEFAULT NOW() | 更新日時 |

---

## 4. property_images（物件画像）

```sql
CREATE TABLE property_images (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  storage_path TEXT NOT NULL, -- Supabase Storageのパス
  display_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_property_images_property_id ON property_images(property_id);
CREATE INDEX idx_property_images_display_order ON property_images(property_id, display_order);
```

---

## 5. property_import_logs（取り込みログ）

```sql
CREATE TABLE property_import_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,
  source_url TEXT NOT NULL, -- 元のURL（アットホーム等）
  html_snapshot_path TEXT, -- Supabase Storageに保存したHTMLパス
  imported_by UUID NOT NULL REFERENCES auth.users(id),
  imported_at TIMESTAMPTZ DEFAULT NOW(),

  -- メタ情報
  import_method VARCHAR(50) DEFAULT 'chrome_extension', -- インポート方法
  status VARCHAR(20) DEFAULT 'success' CHECK (status IN ('success', 'failed', 'pending')),
  error_message TEXT
);

CREATE INDEX idx_import_logs_operator_id ON property_import_logs(operator_id);
CREATE INDEX idx_import_logs_imported_by ON property_import_logs(imported_by);
CREATE INDEX idx_import_logs_imported_at ON property_import_logs(imported_at DESC);
```

**用途**: 法的証跡・監査ログ

---

## 6. customers（顧客情報）

キオスク利用者の情報を保存。認証アカウントは作成しないため、user_idは不要。

```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE, -- どのオペレーターの顧客か
  name VARCHAR(100) NOT NULL,      -- 名前
  phone VARCHAR(20) NOT NULL,      -- 電話番号
  email VARCHAR(255) NOT NULL,     -- メールアドレス
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_customers_operator_id ON customers(operator_id);
CREATE INDEX idx_customers_email ON customers(email);
```

**注意**: `user_id`は不要。キオスク利用者は認証アカウントを持たない。

---

## 7. customer_preferences（顧客の希望条件）

```sql
CREATE TABLE customer_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,

  -- 希望条件
  area VARCHAR(100),
  rent_min INTEGER,
  rent_max INTEGER,
  layout VARCHAR(50),
  building_type VARCHAR(50),
  station VARCHAR(100),
  max_distance_from_station INTEGER, -- 駅徒歩上限（分）

  -- JSONB形式での詳細条件
  conditions JSONB, -- {balcony: true, pet_allowed: true, ...}

  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_customer_preferences_customer_id ON customer_preferences(customer_id);
```

---

## 8. conversations（会話セッション）

```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,
  started_at TIMESTAMPTZ DEFAULT NOW(),
  ended_at TIMESTAMPTZ,

  -- 会話のメタ情報
  total_messages INTEGER DEFAULT 0,
  satisfaction_score INTEGER CHECK (satisfaction_score BETWEEN 1 AND 5)
);

CREATE INDEX idx_conversations_customer_id ON conversations(customer_id);
CREATE INDEX idx_conversations_operator_id ON conversations(operator_id);
CREATE INDEX idx_conversations_started_at ON conversations(started_at DESC);
```

---

## 9. messages（会話履歴）

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,

  role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
  content TEXT NOT NULL,

  -- アバター表情（assistantの場合のみ）
  emotion VARCHAR(20) CHECK (emotion IN ('neutral', 'happy', 'thinking', 'apologetic', 'excited')),

  timestamp TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_messages_timestamp ON messages(conversation_id, timestamp);
```

---

## 10. inquiries（問い合わせ）

```sql
CREATE TABLE inquiries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,

  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'completed', 'cancelled')),

  message TEXT NOT NULL,
  response TEXT,

  created_at TIMESTAMPTZ DEFAULT NOW(),
  responded_at TIMESTAMPTZ
);

CREATE INDEX idx_inquiries_customer_id ON inquiries(customer_id);
CREATE INDEX idx_inquiries_operator_id ON inquiries(operator_id);
CREATE INDEX idx_inquiries_status ON inquiries(status);
CREATE INDEX idx_inquiries_created_at ON inquiries(created_at DESC);
```

---

## 11. viewings（内見予約）

```sql
CREATE TABLE viewings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,

  -- 予約日時
  viewing_date TIMESTAMPTZ NOT NULL,
  duration_minutes INTEGER DEFAULT 30, -- 所要時間（分）

  -- ステータス
  status VARCHAR(20) DEFAULT 'confirmed' CHECK (status IN ('confirmed', 'completed', 'cancelled', 'no_show')),

  -- 担当者
  assigned_to UUID REFERENCES auth.users(id), -- オペレーター or 管理者

  -- メモ
  notes TEXT,
  customer_notes TEXT, -- 顧客からの特記事項

  -- メタ情報
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  cancelled_at TIMESTAMPTZ,
  cancellation_reason TEXT
);

CREATE INDEX idx_viewings_customer_id ON viewings(customer_id);
CREATE INDEX idx_viewings_property_id ON viewings(property_id);
CREATE INDEX idx_viewings_operator_id ON viewings(operator_id);
CREATE INDEX idx_viewings_viewing_date ON viewings(viewing_date);
CREATE INDEX idx_viewings_status ON viewings(status);
CREATE INDEX idx_viewings_assigned_to ON viewings(assigned_to);

-- 更新日時の自動更新
CREATE TRIGGER update_viewings_updated_at
BEFORE UPDATE ON viewings
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```
