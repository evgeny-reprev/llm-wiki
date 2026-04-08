# LLM Wiki Schema

## Shared Record Types

### SUMMARY
```
**Type:** SUMMARY
**Vault:** <pos|coliving>
**Topic:** <namespace/slug>
**Updated:** <YYYY-MM-DD>
**Sources:** <source_id, source_id, ...>
**Confidence:** <high|medium|low>

<200–400 token summary — shortest useful read>
```

### META
```
**Type:** META
**Vault:** <pos|coliving>
**Topic:** <namespace/slug>
**Updated:** <YYYY-MM-DD>
**Source Count:** <N>
**Primary Entities:** <entity, entity, ...>
**Status:** <active|stale|archived>
**Freshness:** <YYYY-MM-DD — date of most recent raw source>
**Related:** <topic, topic, ...>
```

### SOURCE
```
**Type:** SOURCE
**Vault:** <pos|coliving>
**Topic:** <namespace/slug>
**Source ID:** <source_id>
**Origin:** <url or path>
**Evidence:** <1–2 sentence summary of what this source proves>
```

### ALIAS
```
**Type:** ALIAS
**Vault:** <pos|coliving>
**Alias:** <alternate name>
**Canonical Topic:** <namespace/slug>
**Reason:** <why this alias exists>
```

### COLLECTION
```
**Type:** COLLECTION
**Vault:** <pos|coliving>
**Topic:** <namespace/slug>
**Collection Kind:** <timeline|group|index>
**Members:** <topic, topic, ...>
**Rule:** <how members are selected>
```

## Metadata Format

Bold-field headers only — no YAML frontmatter.

Why:
- lower prompt overhead than YAML parsing
- easier partial regeneration (update one field without touching whole block)
- less fragile for LLM edits

## POS Vault

Namespaces and topic conventions:
- `decisions/<slug>` — durable outcomes, not chat history
- `sessions/<yyyy-mm-dd>-<slug>` — compressed session snapshots
- `systems/<slug>` — tools, automations, integrations
- `projects/<slug>` — active project state
- `patterns/<slug>` — reusable agent/workflow patterns

POS-specific notes:
- Decision SUMMARY: captures what was decided and why, not the deliberation
- Session SUMMARY: decision chain + touched files + next actions
- Pattern SUMMARY: when to use, what it solves, example invocation

## Coliving Vault

Namespaces and topic conventions:
- `concepts/<slug>` — domain vocabulary, product definitions
- `operations/<slug>` — SOPs, recurring processes
- `initiatives/<slug>` — active marketing/product/ops initiatives
- `market/<slug>` — competitor intel, positioning, pricing
- `assets/<slug>` — templates, creatives, landing pages

Coliving-specific notes:
- Operation pages compress repeated business knowledge from raw Notion exports
- Market SUMMARY captures current state, not historical analysis
- Initiative COLLECTION groups related operations and assets

## Cross-Vault Schema

Sparse cross-vault links stored in `shared/crosslinks/pos-coliving-links.md`:

```
**Type:** CROSSLINK
**POS Topic:** <pos namespace/slug>
**Coliving Topic:** <coliving namespace/slug>
**Rationale:** <why these connect>
**Status:** <active|under-review>
**Updated:** <YYYY-MM-DD>
```

Default: vault isolation. Cross-vault links are explicit, sparse, reviewable.

## Agent Read Order

1. **SUMMARY** — always first (200–400 tokens)
2. **META** — if freshness/status needed (100–200 tokens)
3. **SOURCE** — if evidence depth needed
4. **COLLECTION** — if topic grouping needed
5. Cross-vault links — only when explicitly following

This read order is the main token-saving behavior in the schema. Agents must not scan raw by default.
