# 技術スタック - 開発ツール・テスト・セキュリティ

> 本ドキュメントは [技術スタック詳細](../technical-stack.md) の一部です。

## テスト・品質保証

### ユニットテスト

#### Vitest
**選定理由**:
- Jest互換API
- Viteベースで高速
- ESM完全対応
- TypeScript完全サポート

#### Testing Library (React Testing Library)
**用途**: Reactコンポーネントテスト

#### MSW (Mock Service Worker)
**用途**: API モック

**テスト例**:
```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';

describe('PropertyCard', () => {
  it('物件情報が正しく表示される', () => {
    render(<PropertyCard property={mockProperty} />);
    expect(screen.getByText('2LDK')).toBeInTheDocument();
  });
});
```

### E2Eテスト

#### Playwright
**選定理由**:
- クロスブラウザテスト
- 高速・安定
- 音声入力シミュレーション可能
- Chrome拡張機能テスト対応

**テスト例**:
```typescript
import { test, expect } from '@playwright/test';

test('物件検索フロー', async ({ page }) => {
  await page.goto('http://localhost:3000');
  await page.click('[aria-label="音声入力開始"]');
  await page.fill('input[name="条件"]', '文京区 2LDK');
  await page.click('button:has-text("検索")');

  await expect(page.locator('.property-card')).toHaveCount(10);
});
```

### 型チェック・リンター

- **TypeScript**: 静的型チェック
- **ESLint**: コード品質チェック
- **Prettier**: コードフォーマット
- **Husky**: Git pre-commit hooks

---

## モニタリング・分析

### エラートラッキング

#### Sentry
**機能**:
- フロントエンドエラー監視
- APIエラー監視
- パフォーマンスモニタリング
- ユーザーセッション再生
- ソースマップ対応

**価格**:
- Developer: 無料（5,000イベント/月）
- Team: $26/月（50,000イベント/月）

### アナリティクス

#### Vercel Analytics
- Core Web Vitals測定
- リアルタイムアクセス解析
- デバイス・ブラウザ分析

#### PostHog（オプション）
- イベントトラッキング
- ファネル分析
- セッションリプレイ
- A/Bテスト

### ログ管理

#### Axiom
- リアルタイムログ分析
- Vercel統合
- ダッシュボード作成

#### Supabase Logs
- データベースクエリログ
- 遅いクエリの検出

---

## 開発ツール・ワークフロー

### AI駆動開発

#### Cursor
- AIペアプログラミングIDE
- コード補完・生成
- リファクタリング提案
- 価格: $20/月

#### Claude Pro (Max Plan)
- 技術相談・アーキテクチャ設計
- コードレビュー
- ドキュメント作成
- 複雑な問題解決
- 価格: $100/月（¥15,000）

**Max Planの価値**:
- 通常のClaude Proより5倍多いメッセージ送信
- より長い会話コンテキスト
- 複雑な技術問題の解決に必須

### バージョン管理・CI/CD

#### GitHub Actions
```yaml
name: CI/CD

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - run: pnpm install
      - run: pnpm type-check
      - run: pnpm lint
      - run: pnpm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
```

### 開発環境

#### pnpm
**選定理由**:
- npm/yarnより高速
- ディスク容量節約
- モノレポ対応

---

## セキュリティ

### 認証・認可
- Supabase Auth（JWT）
- Row Level Security (RLS)
- Refresh Token

### API保護
- CORS設定
- Rate Limiting
- API Key管理
- Request Validation (Zod)

### セキュリティヘッダー
```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline';"
  );

  return response;
}
```

### データ保護
- 暗号化（機密データ）
- SQLインジェクション対策
- XSS対策
- CSRF対策

---

## 技術スタック選定理由まとめ

### なぜこのスタックなのか

1. **Next.js + Vercel**
   - サーバーレスアーキテクチャで運用コスト最小化
   - 自動スケーリング対応
   - エッジコンピューティングで低レイテンシ

2. **Supabase**
   - PostgreSQL + Auth + Storage + Realtime が統合
   - オープンソースで将来的な移行可能性
   - RLSによるセキュアなデータアクセス

3. **OpenAI統一**
   - GPT-4o単体で会話・構造化・分析すべてカバー
   - Whisper APIで高精度な日本語音声認識
   - コスト管理が容易（単一プロバイダー）

4. **Vercel AI SDK**
   - Next.jsとの完璧な統合
   - ストリーミング対応
   - Function Calling対応

5. **TypeScript**
   - 型安全性による開発効率向上
   - AIコード生成との相性が良い
   - 大規模化時のメンテナンス性

6. **Claude Pro (Max Plan)**
   - 学生開発者の技術メンター
   - 複雑な設計判断のサポート
   - コードレビュー・最適化提案
