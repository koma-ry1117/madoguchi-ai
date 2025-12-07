# Implementation Tasks

## タスクフォーマット

### 主要タスクのみ
- [ ] {{NUMBER}}. {{TASK_DESCRIPTION}}{{PARALLEL_MARK}}
  - {{詳細項目1}}
  - _Requirements: {{REQUIREMENT_IDS}}_

### 主要タスク + サブタスク構造
- [ ] {{MAJOR_NUMBER}}. {{MAJOR_TASK_SUMMARY}}
- [ ] {{MAJOR_NUMBER}}.{{SUB_NUMBER}} {{SUB_TASK_DESCRIPTION}}{{SUB_PARALLEL_MARK}}
  - {{詳細項目1}}
  - {{詳細項目2}}
  - _Requirements: {{REQUIREMENT_IDS}}_

## タスクテンプレート

### Task N: {{タスク名}}

#### Description
{{タスクの説明}}

#### Files to Create/Modify
- `path/to/file.ts` (create/modify)

#### Implementation Details
1. {{実装詳細1}}
2. {{実装詳細2}}

#### Acceptance Criteria
- [ ] {{完了条件1}}
- [ ] {{完了条件2}}

---

> **並列マーカー**: 並列実行可能なタスクには ` (P)` を付ける
> **テストカバレッジ**: テスト作業は `- [ ]*` でマーク
