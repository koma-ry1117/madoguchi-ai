# Development Environment

macOS（Apple Silicon）環境での開発セットアップと最適化ガイドライン。

## Required Tools

| ツール | バージョン | 用途 |
|--------|-----------|------|
| Node.js | 22.x | ランタイム |
| pnpm | 9.x | パッケージマネージャー |
| Docker Desktop | 最新 | コンテナ（ローカルSupabase） |
| fnm | 最新 | Node.jsバージョン管理 |

## Node.js Version Management

### fnm（Fast Node Manager）の使用

nvmではなく**fnm**を使用。Rust製で起動オーバーヘッドがほぼゼロ。

```bash
# インストール
brew install fnm

# シェル設定（~/.zshrc）
eval "$(fnm env --use-on-cd)"

# プロジェクトのNode.jsを使用
fnm use
```

### バージョン固定

プロジェクトルートの `.nvmrc` でバージョンを固定：

```
v22.11.0
```

### Corepack有効化

パッケージマネージャーのバージョン差異を防止：

```bash
corepack enable
```

`package.json` で固定：

```json
{
  "packageManager": "pnpm@9.15.0"
}
```

## Docker Configuration

### VirtioFS の強制適用

Apple Silicon環境でのI/O遅延を解消するため、VirtioFSを使用：

1. Docker Desktop > Settings > General
2. "Use Virtualization framework" を有効化
3. "File sharing implementation" で **VirtioFS** を選択

**効果**: ファイルシステム操作の時間を最大98%短縮、Next.jsのホットリロードが高速化。

### Colima（代替）

Docker Desktopの代替として、より軽量なColimaも使用可能：

```bash
# インストール
brew install colima

# 起動（推奨設定）
colima start --cpu 4 --memory 8 --mount-type virtiofs --vm-type vz --vz-rosetta
```

**利点**: Rosetta 2によるx86_64バイナリの高速変換、ライセンス不要。

## VS Code Settings

プロジェクトレベルで設定を統一。`.vscode/settings.json`:

```json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "explicit"
  },
  "files.exclude": {
    "**/.next": true,
    "**/node_modules": true
  }
}
```

**重要**: プロジェクトの `node_modules` 内のTypeScriptを使用することで、CI環境と型推論の挙動が一致。

## Recommended Extensions

`.vscode/extensions.json`:

```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "bradlc.vscode-tailwindcss",
    "prisma.prisma"
  ]
}
```

## Environment Variables

### ローカル開発

`.env.local` に環境変数を設定（gitignore対象）：

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# AI
GOOGLE_GENERATIVE_AI_API_KEY=...
OPENAI_API_KEY=...

# Email
RESEND_API_KEY=re_...
```

### 環境変数バリデーション

`env.mjs` でZodによる起動時検証を実装し、デプロイ前に不足を検知。

## Common Commands

```bash
# 開発サーバー起動
pnpm dev

# 本番ビルド
pnpm build

# 型チェック
pnpm type-check

# Lint
pnpm lint

# テスト
pnpm test

# Supabase ローカル起動
pnpm supabase:start

# Supabase 型生成
pnpm supabase:gen-types
```

## Troubleshooting

### ポート競合

```bash
# 使用中のポートを確認
lsof -i :3000
lsof -i :54321

# プロセス終了
kill -9 <PID>
```

### Docker関連

```bash
# Docker状態確認
docker ps

# Supabaseコンテナ再起動
pnpm supabase:stop && pnpm supabase:start

# キャッシュクリア
docker system prune -a
```

### Node.js関連

```bash
# node_modules再インストール
rm -rf node_modules pnpm-lock.yaml
pnpm install

# fnmキャッシュクリア
fnm uninstall <version> && fnm install <version>
```

---
_Focus on setup reproducibility and developer onboarding efficiency_
