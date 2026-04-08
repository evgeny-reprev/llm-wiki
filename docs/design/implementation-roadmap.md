# Implementation Roadmap

## Goal

Prove a tiny compiled wiki reduces token burn for POS before building broader infrastructure.

**Sequencing rule:** Do not build Layers 2–4 before Layer 0 proves useful on real POS questions. The first bet is not breadth. The first bet is lower token cost and lower context anxiety with a tiny compiled wiki.

---

## Layer 0: One-Hour Working Core

**Objective:** A local file-based wiki that ingests a tiny POS subset and answers from compiled records immediately.

**Scope:**
- `raw/` + `wiki/` + `shared/` scaffold
- `sources.jsonl` manifest
- Five record type templates (SUMMARY, META, SOURCE, ALIAS, COLLECTION)
- Manually compile 5–10 pilot POS sources
- `record-catalog.jsonl`
- Deterministic lint (missing Sources, orphan pages)

**Suggested pilot sources:**
- 2–3 POS GitHub issues (e.g. #57, #65, #82)
- 2–3 session snapshots from `.claude/logs/`
- 1–2 core docs (CLAUDE.md, AGENTS.md)

**Minimal commands:**
- `ingest-pilot` — compile selected raw sources into wiki records
- `query-pilot` — answer a real POS question from wiki only
- `lint-pilot` — check for missing Sources and orphan pages

**Definition of Done:**
- [ ] scaffold exists: `raw/`, `wiki/pos/`, `wiki/shared/indexes/`, `wiki/shared/logs/`
- [ ] `sources.jsonl` has ≥5 entries with valid `source_id`
- [ ] ≥5 compiled records across SUMMARY and META types
- [ ] `record-catalog.jsonl` routes each topic to a file
- [ ] lint finds zero missing `Sources` fields
- [ ] one real POS question answered from compiled records without opening raw

**Review gate:** Confirm non-empty `Sources`, catalog findability, and <1 hour setup from empty folder to first query.

---

## Layer 1: Repeatable POS Ingest

**Scope:**
- Deterministic source discovery for selected POS classes (github_issue, session_snapshot, agent_log)
- Update rules by topic — rerun touches only changed topics
- Query-time write-back staging
- COLLECTION records for decision timelines and session timelines

**Definition of Done:**
- [ ] rerun only touches topics with changed sources
- [ ] new POS issue/session automatically lands in correct topic pages
- [ ] query loads SUMMARY first, expands only if needed
- [ ] write-backs are staged, not silently merged into wiki

---

## Layer 2: Coliving Vault

**Scope:**
- Same raw manifest contract applied to Coliving sources
- Coliving namespaces and record templates
- Explicit cross-vault links in `shared/crosslinks/`
- Default vault isolation (queries scoped to `pos` or `coliving`)

**Definition of Done:**
- [ ] Coliving raw stored under same contract as POS raw
- [ ] ≥5 Coliving compiled records (SUMMARY + META)
- [ ] queries can target `pos` or `coliving` explicitly
- [ ] cross-vault links are explicit, sparse, and reviewable

---

## Layer 3: Efficiency and Ranking

**Scope:**
- Batched compilation (multiple sources per LLM call)
- Stable prompt templates for cache reuse (prompt caching 50–90% savings)
- Better catalog ranking — SUMMARY records promoted over raw
- Evidence snippets in SOURCE records
- Stronger deterministic lint (duplicate topics, stale records)

**Definition of Done:**
- [ ] ingest supports batch mode (≥3 sources per call)
- [ ] prompt templates are stable and cache-friendly
- [ ] query opens fewer files than Layer 1 on equivalent questions
- [ ] duplicate-topic and stale-record lint checks are active

---

## Layer 4: Reviewable Automation

**Scope:**
- Diff generation before applying wiki changes
- Approval gates for sensitive topic pages
- LLM-assisted contradiction lint (flags likely unsupported claims)
- Staged write-back promotion (query-derived artifacts move through review)

**Definition of Done:**
- [ ] wiki changes are reviewable as diffs before apply
- [ ] sensitive topics require explicit approval
- [ ] LLM lint flags likely unsupported claims
- [ ] query-derived write-backs move through staged promotion path

---

## Reference: Architecture Stance

Start here, not at vectors/graph/orchestration:

```
raw/ (immutable, verbatim)
  └─► ingest (LLM, once per source)
        └─► wiki/ (typed markdown records)
              └─► query (SUMMARY-first, low token)
                    └─► write-back (staged, reviewable)
```

Each layer adds capability without replacing the layer below it.
