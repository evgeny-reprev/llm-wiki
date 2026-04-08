# Raw Layer Spec

## Objective

Store immutable evidence cheaply and regenerate both vaults from it.

## Invariants

- raw is source of truth
- raw is append-only
- derived wiki is regenerable
- every artifact has global `source_id`
- every record cites one or more `source_id`s

## Source Classes

POS:
- `github_issue`
- `github_comment`
- `session_snapshot`
- `agent_log`
- `project_doc`
- `inbox_item`
- `external_capture`

Coliving:
- `notion_page`
- `notion_database_row`
- `meeting_note`
- `business_doc`
- `research_capture`
- `external_capture`

## Folder Contract

```
raw/
  pos/github/issues/YYYY/ISSUE_ID.md
  pos/github/comments/YYYY/ISSUE_ID/COMMENT_ID.md
  pos/sessions/YYYY/MM/DD/SESSION_ID.md
  pos/logs/YYYY/MM/DD/LOG_ID.md
  pos/docs/PATH_SLUG.md
  pos/inbox/YYYY/MM/ITEM_ID.md
  pos/external/YYYY/SOURCE_ID.md
  coliving/notion/pages/SPACE/PAGE_ID.md
  coliving/notion/databases/DB_ID/ROW_ID.md
  coliving/meetings/YYYY/MM/MEETING_ID.md
  coliving/docs/PATH_SLUG.md
  coliving/research/YYYY/SOURCE_ID.md
  coliving/external/YYYY/SOURCE_ID.md
  manifests/sources.jsonl
  manifests/checkpoints.jsonl
```

## source_id Format

`<vault>:<class>:<native_id>`

Examples:
- `pos:github_issue:57`
- `pos:session_snapshot:2026-04-08-startup`
- `coliving:notion_page:8f3d2a`

## Manifest Schema

Each line in `sources.jsonl`:

```json
{
  "source_id": "pos:github_issue:57",
  "vault": "pos",
  "class": "github_issue",
  "path": "raw/pos/github/issues/2026/57.md",
  "title": "[TASK] Codex startup parity",
  "created_at": "2026-03-31T11:30:58Z",
  "captured_at": "2026-04-08T07:00:00Z",
  "checksum": "sha256:...",
  "status": "active",
  "tags": ["agent-task", "agent/codex"]
}
```

## Immutability Policy

- never edit stored raw body
- upstream change creates a new capture/checkpoint
- manifests may enrich metadata, but old entries remain valid

## Minimal Raw Markdown Format

```
# <title>

**Origin:** <url or local path>
**Captured:** <ISO timestamp>
**Source ID:** <source_id>

---

<verbatim body>
```

## Layer 0 Capture Scope

Only:
- selected POS GitHub issues (2–3)
- selected POS session snapshots (2–3)
- a small fixed set of core docs (1–2)

Reason: keep setup under one hour. Coliving capture begins at Layer 2 on the same contract.
