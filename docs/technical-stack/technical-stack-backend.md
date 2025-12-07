# 技術スタック - バックエンド・インフラ

> 本ドキュメントは [技術スタック詳細](../technical-stack.md) の一部です。

## データベース・ストレージ

### Supabase
**選定理由**:
- PostgreSQL + Auth + Storage + Realtime が統合
- オープンソースで将来的な移行可能性
- RLS（Row Level Security）による強力なセキュリティ
- REST API + Realtime subscriptions

**使用機能**:

1. **PostgreSQL データベース**
   - リレーショナルデータベース
   - JSON型サポート
   - 全文検索（FTS）
   - トリガー・関数

2. **Row Level Security (RLS)**
   - データベースレベルでのアクセス制御
   - ユーザーロール別の権限管理
   - SQLポリシーによる柔軟な設定

3. **Realtime subscriptions**
   - データベース変更のリアルタイム通知
   - WebSocketベースの双方向通信
   - 複数クライアント間の同期

4. **Storage**
   - 物件画像ストレージ
   - 音声キャッシュストレージ
   - アバター画像ホスティング
   - HTMLスナップショット保存

---

## ホスティング・デプロイ

### Vercel
**選定理由**:
- Next.jsとの完璧な統合
- 自動スケーリング
- グローバルCDN
- プレビューデプロイ（PRごと）
- ゼロコンフィグデプロイ

**使用機能**:

1. **Vercel Edge Functions**
   - エッジコンピューティング
   - 低レイテンシ
   - グローバル展開

2. **Vercel Edge Middleware**
   - レート制限
   - リダイレクト・リライト
   - 認証チェック
   - セキュリティヘッダー

3. **Vercel Analytics**
   - Webパフォーマンス分析
   - Core Web Vitals測定
   - リアルタイムアクセス解析

---

## 認証・認可システム

### Supabase Auth

**認証方式**:
- メール/パスワード認証
- OAuth認証（Google, GitHub等）サポート
- マジックリンク認証（パスワードレス）

**セッション管理**:
- JWT (JSON Web Token) ベースのステートレス認証
- Refresh Token による自動セッション更新
- マルチセッション管理

**セキュリティ機能**:
- パスワードハッシュ化（bcrypt）
- パスワードリセット機能
- メール確認機能
- セッション有効期限管理

### 権限管理（Row Level Security）

```sql
-- オペレーター：自分が取り込んだ物件の編集権限
CREATE POLICY "operator_own_properties"
ON properties FOR UPDATE
USING (auth.uid() = created_by);

-- 管理者：全物件の編集・削除権限
CREATE POLICY "admin_all_properties"
ON properties FOR ALL
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);

-- 顧客：公開物件の閲覧のみ
CREATE POLICY "customer_view_public"
ON properties FOR SELECT
USING (is_public = true);
```

### ユーザーロール

| ロール | 権限 |
|--------|------|
| customer | 物件閲覧、会話履歴閲覧（自分のみ） |
| operator | 物件取り込み、自分の物件編集、顧客対応履歴確認 |
| admin | 全物件編集・削除、ユーザー管理、システム設定 |

---

## メール送信機能

### Resend

**選定理由**:
- Next.jsとの完璧な統合
- シンプルなAPI
- 無料枠が充実（月100通まで無料）
- 配信率が高い
- Vercelとの相性が良い

**価格**:
- Free: 100通/月 - 無料
- Pro: 50,000通/月 - $20
- Business: カスタム

**使用例**:
```typescript
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);

await resend.emails.send({
  from: 'madoguchi-ai <noreply@madoguchi-ai.jp>',
  to: customer.email,
  subject: '内見予約が確定しました',
  react: ViewingConfirmationEmail({
    customerName: customer.name,
    propertyTitle: property.title,
    viewingDate: '2025/11/20 14:00',
  }),
});
```

### React Email

**用途**: 美しいHTMLメールテンプレート作成

**選定理由**:
- Reactコンポーネントでメール作成
- レスポンシブデザイン対応
- プレビュー機能
- TypeScript完全サポート

**テンプレート例**:
```typescript
// emails/viewing-confirmation.tsx
import {
  Body, Container, Head, Heading, Html,
  Preview, Section, Text, Button
} from '@react-email/components';

export default function ViewingConfirmationEmail({
  customerName,
  propertyTitle,
  viewingDate
}) {
  return (
    <Html>
      <Head />
      <Preview>内見予約が確定しました</Preview>
      <Body style={main}>
        <Container style={container}>
          <Heading style={h1}>
            内見予約確定のお知らせ
          </Heading>
          <Text style={text}>
            {customerName} 様
          </Text>
          <Text style={text}>
            以下の物件の内見予約が確定しました。
          </Text>
          <Section style={propertyInfo}>
            <Text><strong>物件名:</strong> {propertyTitle}</Text>
            <Text><strong>内見日時:</strong> {viewingDate}</Text>
          </Section>
          <Button
            href="https://madoguchi-ai.jp/viewings"
            style={button}
          >
            予約詳細を確認
          </Button>
        </Container>
      </Body>
    </Html>
  );
}
```

### メール種別

#### 1. 内見予約確定メール
- 顧客への確認メール
- オペレーターへの通知
- カレンダー招待（.ics）添付

#### 2. システムお知らせメール
- 新着物件のお知らせ
- メンテナンス通知
- 重要な変更の連絡

#### 3. 問い合わせ確認メール
- 問い合わせ受付の自動返信
- 担当者アサイン通知

#### 4. 認証関連メール
- パスワードリセット（Supabase Auth）
- メール確認
- アカウント登録完了

### カレンダー招待（.ics）生成

```typescript
import ical from 'ical-generator';

const calendar = ical({
  name: '内見予約',
  timezone: 'Asia/Tokyo'
});

calendar.createEvent({
  start: new Date('2025-11-20T14:00:00+09:00'),
  end: new Date('2025-11-20T15:00:00+09:00'),
  summary: `物件内見: ${property.title}`,
  description: `住所: ${property.address}`,
  location: property.address,
  url: `https://madoguchi-ai.jp/properties/${property.id}`,
  organizer: {
    name: 'madoguchi-ai',
    email: 'noreply@madoguchi-ai.jp',
  },
  attendees: [
    {
      name: customer.name,
      email: customer.email,
    },
  ],
});

// .icsファイルをメールに添付
await resend.emails.send({
  from: 'madoguchi-ai <noreply@madoguchi-ai.jp>',
  to: customer.email,
  subject: '内見予約が確定しました',
  react: ViewingConfirmationEmail({ ... }),
  attachments: [
    {
      filename: 'viewing.ics',
      content: calendar.toString(),
    },
  ],
});
```
