# LLM Wiki System Design v1

## 1. Raw Layer

Raw is the only source of truth. Wiki pages are derived artifacts.

POS raw sources: GitHub issues/comments, session snapshots, agent logs, core docs, inbox captures, locally stored external captures.
Coliving raw sources: Notion exports, extracted business docs, meeting notes, research captures, locally stored external captures.

Rules:
- append-only raw
- stable `source_id` per artifact
- every wiki claim cites one or more `source_id`s
- no in-place edits of raw content

Folder shape:
```
raw/
  pos/
  coliving/
  manifests/sources.jsonl
  manifests/ingest-log.jsonl
```

Why: compile once, query many; keep verbatim recall in raw; pay LLM cost on ingest, not on every query.

## 2. Schema Design

Use typed markdown records, not free-form pages.

Record families:
- **SUMMARY**: shortest useful read
- **META**: freshness, status, entities, related topics
- **SOURCE**: evidence routing back to raw
- **ALIAS**: redirects/synonyms
- **COLLECTION**: grouped topics/timelines

POS namespaces:
- decisions
- sessions
- systems
- projects
- patterns

Coliving namespaces:
- concepts
- operations
- initiatives
- market
- assets

Metadata format: bold-field headers, not YAML.

Reason:
- less prompt overhead
- easier partial regeneration
- less fragile for LLM edits

## 3. Wiki Layer Structure

```
wiki/
  pos/
  coliving/
  shared/indexes/
  shared/logs/
  shared/crosslinks/
```

Support files:
- `shared/indexes/topic-map.md`
- `shared/indexes/record-catalog.jsonl`
- `shared/logs/ingest-log.md`
- `shared/logs/query-writeback.md`
- `shared/crosslinks/pos-coliving-links.md`

Cross-vault links are explicit and sparse. Default is vault isolation.

Why: predictable namespaces reduce search width; catalog avoids opening many files; sparse crosslinks prevent token sprawl.

## 4. Pipeline Design

**Ingest:**
1. Detect new raw artifacts
2. Classify source and vault
3. Extract durable claims/entities/decisions/next actions
4. Update or create affected records
5. Append to ingest log and catalog

**Query:**
1. Route to vault and namespace
2. Load catalog and top SUMMARY records
3. Expand to META/COLLECTION only if needed
4. Answer from wiki first
5. Stage durable new synthesis as write-back candidate

**Lint:**
- missing source links
- stale records
- duplicate topics
- orphan pages
- confidence/conflict mismatch

Layer 0 is manual CLI batch only. No daemon, queue, or workers.

## 5. Token Efficiency

Primary strategies:
- compile once, answer many
- typed records first
- raw stays verbatim, wiki stays compressed
- delta ingest only
- progressive disclosure: SUMMARY → META → SOURCE → COLLECTION

Concrete budgets:
- SUMMARY: 200–400 tokens
- META: 100–200 tokens

Avoid:
- default raw retrieval at query time
- giant index pages
- YAML-heavy headers
- full-page regeneration for tiny edits
- dense auto-linking across both vaults

## 6. MCP Integration

Layer 0 tools:
- `wiki_search_records(query, vault, type)`
- `wiki_get_record(topic, vault, type)`
- `wiki_list_sources(topic)`
- `wiki_run_ingest(source_ids?)`
- `wiki_run_lint(scope)`

Later:
- `wiki_stage_writeback(answer_artifact)`
- `wiki_review_diff(topic)`
- `wiki_follow_crosslinks(topic)`

Rule: expose wiki primitives, not raw connector complexity.

## 7. Implementation Roadmap

- **Layer 0**: local scaffold, raw manifest, typed templates, pilot ingest, catalog, deterministic lint
- **Layer 1**: repeatable POS ingest + write-back staging
- **Layer 2**: Coliving vault on same infrastructure
- **Layer 3**: ranking, batching, cache gains, evaluator-lite lint
- **Layer 4**: reviewable diffs, approvals, stronger contradiction lint

Architecture stance: do not start with vectors, knowledge graph, or multi-agent orchestration. The first proof should be a file-based compiler plus typed markdown records.
