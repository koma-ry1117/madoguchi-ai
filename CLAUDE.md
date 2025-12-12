# AI-DLC and Spec-Driven Development

Kiro-style Spec Driven Development implementation on AI-DLC (AI Development Life Cycle)

## Project Context

### Paths
- Steering: `.kiro/steering/`
- Specs: `.kiro/specs/`

### Steering vs Specification

**Steering** (`.kiro/steering/`) - Guide AI with project-wide rules and context
**Specs** (`.kiro/specs/`) - Formalize development process for individual features

### Active Specifications (9 specs)
| # | Spec | 概要 | 依存関係 |
|---|------|------|----------|
| 1 | `foundation` | プロジェクト基盤（Next.js, Supabase, UI） | なし |
| 2 | `auth` | 認証システム（ログイン, ロール管理） | foundation |
| 3 | `property-management` | 物件管理（CRUD, 検索） | auth |
| 4 | `operator-management` | オペレーター管理（Admin向け） | auth |
| 5 | `ai-chat` | AIキオスク会話 | foundation, property |
| 6 | `voice-io` | 音声入出力 | ai-chat |
| 7 | `viewing-reservation` | 内見予約管理 | ai-chat, property |
| 8 | `chrome-extension` | 物件取り込み拡張機能 | auth, property |
| 9 | `marketing-email` | 営業メール配信 | auth, property |

**推奨実装順序**: foundation → auth → property-management → operator-management → ai-chat → voice-io → viewing-reservation → chrome-extension → marketing-email

- Use `/kiro:spec-status [feature-name]` to check progress

## Development Guidelines
- Think in English, generate responses in Japanese. All Markdown content written to project files (e.g., requirements.md, design.md, tasks.md, research.md, validation reports) MUST be written in the target language configured for this specification (see spec.json.language).

## Minimal Workflow
- Phase 0 (optional): `/kiro:steering`, `/kiro:steering-custom`
- Phase 1 (Specification):
  - `/kiro:spec-init "description"`
  - `/kiro:spec-requirements {feature}`
  - `/kiro:validate-gap {feature}` (optional: for existing codebase)
  - `/kiro:spec-design {feature} [-y]`
  - `/kiro:validate-design {feature}` (optional: design review)
  - `/kiro:spec-tasks {feature} [-y]`
- Phase 2 (Implementation): `/kiro:spec-impl {feature} [tasks]`
  - `/kiro:validate-impl {feature}` (optional: after implementation)
- Progress check: `/kiro:spec-status {feature}` (use anytime)

## Development Rules
- 3-phase approval workflow: Requirements → Design → Tasks → Implementation
- Human review required each phase; use `-y` only for intentional fast-track
- Keep steering current and verify alignment with `/kiro:spec-status`
- Follow the user's instructions precisely, and within that scope act autonomously: gather the necessary context and complete the requested work end-to-end in this run, asking questions only when essential information is missing or the instructions are critically ambiguous.

## Steering Configuration
- Load entire `.kiro/steering/` as project memory
- Default files: `product.yaml`, `tech.yaml`, `structure.yaml`
- Custom files are supported (managed via `/kiro:steering-custom`)
- Current custom files: `security.yaml`, `ai-integration.yaml`, `architecture.yaml`, `development.yaml`, `supabase.yaml`, `database.yaml`, `api-standards.yaml`
