# Design Document

## Overview

**Purpose**: システム管理者がオペレーター（不動産会社）を管理し、システム全体を監視する機能。

**Users**: システム管理者（gyam様）のみ

### Goals
- オペレーターのライフサイクル管理
- サブスクリプション管理
- システム全体の可視化
- 監査ログ

### Non-Goals
- 課金・決済処理（外部システム）
- 自動サブスクリプション更新

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    Admin Dashboard                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │ OperatorList    │  │ OperatorDetail  │  │ SystemStats    │  │
│  │ (Table)         │  │ (Edit Form)     │  │ (Charts)       │  │
│  └────────┬────────┘  └────────┬────────┘  └───────┬────────┘  │
└───────────┼──────────────────────┼─────────────────┼────────────┘
            │                      │                 │
            ▼                      ▼                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Next.js API Routes                            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │ /api/admin/      │  │ /api/admin/      │  │ /api/admin/    │  │
│  │ operators        │  │ operators/[id]   │  │ stats          │  │
│  └──────────────────┘  └──────────────────┘  └────────────────┘  │
└───────────────────────────────┬──────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Supabase                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ operators    │  │ auth.users   │  │ properties, customers  │  │
│  │              │  │ (role=operator)│ │ (for stats)           │  │
│  └──────────────┘  └──────────────┘  └────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Component | Choice | Role |
|-----------|--------|------|
| Frontend | React 19 + Next.js 15 | 管理画面 |
| UI | shadcn/ui + TanStack Table | テーブル、フォーム |
| Charts | Recharts | 統計グラフ |
| Auth | Supabase Auth | 管理者認証 |
| Database | Supabase PostgreSQL | データ |

## System Flows

### オペレーター登録フロー

```mermaid
sequenceDiagram
    participant A as Admin
    participant UI as OperatorForm
    participant API as /api/admin/operators
    participant Auth as Supabase Auth
    participant DB as Supabase DB

    A->>UI: フォーム入力
    UI->>API: POST /api/admin/operators

    API->>API: admin権限チェック
    API->>Auth: admin.createUser()
    Auth-->>API: user (role=operator)

    API->>DB: auth.users UPDATE (role)
    API->>DB: operators INSERT
    DB-->>API: operator

    API-->>UI: 成功
    UI-->>A: 一覧に戻る
```

## Components and Interfaces

### Frontend Components

#### OperatorList
| Field | Detail |
|-------|--------|
| Intent | オペレーター一覧テーブル |
| Requirements | 2 |

**Features**
- ソート（会社名、作成日、ステータス）
- フィルター（ステータス）
- 検索（会社名）
- ページネーション

#### OperatorForm
| Field | Detail |
|-------|--------|
| Intent | オペレーター登録・編集フォーム |
| Requirements | 1, 3 |

**Fields**
```typescript
interface OperatorFormData {
  email: string;
  password?: string; // 新規のみ
  company_name: string;
  company_address?: string;
  phone?: string;
  subscription_plan: 'basic' | 'standard' | 'premium';
  subscription_status: 'active' | 'suspended' | 'cancelled' | 'trial';
  contract_start_date: Date;
  contract_end_date?: Date;
  is_active: boolean;
}
```

#### SystemStats
| Field | Detail |
|-------|--------|
| Intent | システム統計ダッシュボード |
| Requirements | 5 |

**Stats**
- オペレーター数（状態別）
- 物件数
- 顧客数
- 会話数（直近30日）

### API Routes

#### GET /api/admin/operators

**Query Parameters**
| Param | Type | Description |
|-------|------|-------------|
| subscription_status | string | ステータスフィルター |
| is_active | boolean | 有効状態フィルター |
| search | string | 会社名検索 |
| page | number | ページ番号 |
| limit | number | 取得件数 |

**Response**
```json
{
  "success": true,
  "data": {
    "operators": [
      {
        "id": "uuid",
        "company_name": "〇〇不動産",
        "subscription_status": "active",
        "stats": {
          "property_count": 150,
          "customer_count": 45
        }
      }
    ],
    "pagination": { "total": 50, "page": 1 }
  }
}
```

#### POST /api/admin/operators

**Request**
```json
{
  "email": "operator@example.com",
  "password": "securepassword",
  "company_name": "〇〇不動産株式会社",
  "company_address": "東京都文京区...",
  "phone": "03-1234-5678",
  "subscription_plan": "standard",
  "contract_start_date": "2025-01-15"
}
```

#### PATCH /api/admin/operators/[id]

**Request**
```json
{
  "company_name": "更新後の会社名",
  "subscription_status": "suspended",
  "is_active": false
}
```

#### DELETE /api/admin/operators/[id]

**Response**
```json
{
  "success": true,
  "data": {
    "message": "オペレーターを削除しました"
  }
}
```

#### GET /api/admin/stats

**Response**
```json
{
  "success": true,
  "data": {
    "operators": {
      "total": 50,
      "active": 45,
      "suspended": 3,
      "cancelled": 2
    },
    "properties": {
      "total": 5000,
      "public": 4500
    },
    "customers": {
      "total": 1200
    },
    "conversations": {
      "total": 3500,
      "last_30_days": 450
    }
  }
}
```

## Data Models

### Operator
```typescript
interface Operator {
  id: string;
  user_id: string;
  company_name: string;
  company_address?: string;
  phone?: string;
  email: string;
  subscription_status: 'active' | 'suspended' | 'cancelled' | 'trial';
  subscription_plan: 'basic' | 'standard' | 'premium';
  contract_start_date: Date;
  contract_end_date?: Date;
  is_active: boolean;
  created_at: Date;
  updated_at: Date;
}
```

## Security Considerations

- **認証**: admin roleのみアクセス可能
- **権限チェック**: 全APIでadmin権限を検証
- **監査ログ**: 重要な操作（作成、削除、ステータス変更）をログ

## Error Handling

### API Errors
- 403 Forbidden: admin以外のアクセス
- 404 Not Found: オペレーターが存在しない
- 409 Conflict: メールアドレス重複

## Testing Strategy

### Unit Tests
- OperatorService: CRUD操作
- 権限チェック

### Integration Tests
- オペレーター登録→ユーザー作成
- オペレーター削除→CASCADE確認

### E2E Tests
- 登録→一覧表示→編集→削除
- 統計表示
