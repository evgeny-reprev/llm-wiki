# Экосистема реализаций — изученные репозитории

**Изучено:** 2026-04-08  
**Источник:** комментарии к Karpathy Gist + самостоятельный поиск

---

## milla-jovovich/mempalace

**Stars:** 19,759 | **Lang:** Python | **Updated:** 2026-04-08  
**Repo:** https://github.com/milla-jovovich/mempalace  
**Claim:** "The highest-scoring AI memory system ever benchmarked. And it's free."

### Что делает
Хранит **всё verbatim** (без summarization) + semantic search через ChromaDB. Recall 96.6% на LongMemEval R@5.

### Архитектура
- **Wings** → Projects/People
- **Rooms** → Topic categories (auth, billing, deploy)
- **Halls** → Memory types (facts, events, preferences)
- **Closets** → Indexed summaries pointing to originals
- **Drawers** → Verbatim source files

### Ключевая идея
Verbatim > summary для recall задач. Summarization теряет "why" — контекст за решением.

### MCP
19 tools, Python API для local models. ~170 токенов critical facts при старте → search on-demand.

### Техстек
Python, ChromaDB, MCP

### Что взять
- Принцип verbatim хранения raw sources
- Progressive loading (170 токенов старт → on-demand search)
- Palace structure = namespace isolation

---

## tobi/qmd

**Stars:** 19,726 | **Lang:** TypeScript | **Updated:** 2026-04-08  
**Repo:** https://github.com/tobi/qmd  
**Desc:** "mini cli search engine for your docs, knowledge bases, meeting notes"

### Что делает
On-device search engine с hybrid pipeline: BM25 + vector + LLM rerank. Всё локально через node-llama-cpp.

### Pipeline
1. BM25 indexing (fast lexical matching)
2. Vector embeddings (semantic meaning)
3. LLM reranking (surface most relevant)

Combines via reciprocal rank fusion.

### Техстек
TypeScript/Node.js, SQLite, GGUF models via node-llama-cpp, MCP server, npm package `@tobilu/qmd`

### Что взять
- Hybrid search как стандарт (BM25 + semantic)
- MCP server подход
- Typed sub-queries (lexical, vector, HyDE expansion)

---

## xoai/sage-wiki

**Stars:** 197 | **Lang:** Go | **Updated:** 2026-04-08  
**Repo:** https://github.com/xoai/sage-wiki  
**Desc:** "An LLM-compiled personal knowledge base"

### Что делает
5-pass compilation: summarization → concept extraction → article generation → image captioning → cross-reference discovery.

### Интерфейсы
- **TUI** (bubbletea): article browsing, fuzzy search, Q&A, live compilation monitoring
- **Web UI** (Preact + Tailwind): knowledge graph visualization, streaming Q&A
- **MCP**: 14 tools, stdio и SSE transports

### Cost optimization
- Prompt caching: 50–90% savings on input tokens
- Batch API: 50% cost reduction (async submission)
- Cost estimation preview до компиляции

### Техстек
Pure Go, SQLite + FTS5, single binary (no CGO), cross-platform

### Что взять
- Cost optimization: prompt caching + batch API → 50–90% экономия
- 5-pass pipeline как образец для ingest
- TUI как Layer 3 опция
- Единый бинарь без зависимостей

---

## bitsofchris/openaugi

**Stars:** 83 | **Lang:** Python | **Updated:** 2026-04-08  
**Repo:** https://github.com/bitsofchris/openaugi  
**Desc:** "The LLM wiki and personal knowledge base that scales beyond 100 articles"

### Что делает
Knowledge graph из Obsidian vault → queryable SQLite → MCP server → Claude integration.

### Processing pipeline
Vault ingestion → content splitting → metadata extraction → semantic embedding → SQLite → MCP server

### "Navigational metadata layer"
- **Data blocks**: raw content с deterministic identity
- **Context blocks**: compiled summaries для relevance assessment
- **Typed links**: graph edges для traversal

### 5 Query Modes
Semantic search, keyword matching, graph traversal, time-based queries, direct lookups.

### Field findings (4,000+ journal entries tested)
- Index файлы ломаются на масштабе → нужна дедупликация + MMR re-ranking
- Links работают как first-class graph nodes
- **Write-back drives compounding** — ключевой инсайт

### Техстек
Python, SQLite, MCP, Cloudflare Tunnel для remote access

### Что взять
- Write-back drives compounding — встроить в query операцию
- Graph traversal как query mode
- Context blocks = наш META record type

---

## swarmclawai/swarmvault

**Stars:** 8 | **Lang:** TypeScript | **Updated:** 2026-04-08  
**Repo:** https://github.com/swarmclawai/swarmvault  
**Desc:** "Local-first LLM knowledge base compiler"

### Что делает
Full pipeline: ingest → compile → query → save → review. Vault = продукт, не чат.

### Storage model
```
raw/         — immutable source files
wiki/        — compiled pages (sources, concepts, entities, code, outputs)
state/       — graph.json, SQLite, sessions, approvals
swarmvault.schema.md — user-editable instructions
```

### Graph system
Deterministic knowledge graph: querying, pathfinding, community detection, "god-node" identification (узлы-мосты между community).

### Code awareness
Parser-backed analysis: JS, TS, Python, Go, Rust, Java, C#, C++, PHP, Ruby, PowerShell.

### Providers
OpenAI, Anthropic, Gemini, Ollama, OpenRouter, Groq, Together, xAI, Cerebras, custom OpenAI-compatible.

### Ключевая философия
**Reviewable workflows** — staged approval queues, не silent mutations.

### Техстек
TypeScript, 3 packages (@swarmvaultai/cli, engine, viewer), pnpm workspaces, Node.js ≥24

### Что взять
- Reviewable changes = staged approval (Layer 4)
- Schema.md как user-editable конфиг
- God-node detection для cross-vault links
- Multi-agent rules (Claude Code, Codex, Cursor)

---

## arpitnath/blink-query

**Stars:** 2 | **Lang:** TypeScript | **Updated:** 2026-04-08  
**Repo:** https://github.com/arpitnath/blink-query  
**Desc:** "A typed wiki for LLMs — markdown on disk, resolution in the library"

### Что делает
DNS-аналогия для знаний: typed records + SQLite index → 25–91x быстрее grep на 14K файлах.

### 5 типов записей (DNS-аналогия)
| Тип | Назначение |
|-----|-----------|
| **SUMMARY** | Прямой ответ — "агент не читает ничего больше" |
| **META** | Structured JSON: config, logs, entity attributes |
| **SOURCE** | Pointer с контекстом — "summary here, fetch source if needed" |
| **ALIAS** | Auto-create из wikilinks, redirect без дублирования |
| **COLLECTION** | Auto-generated namespace indexes для browsing |

### Performance
- 25x–91x быстрее grep на 14,251-файловом corpus
- grep: ~1,212 нерейтингованных файлов → blink: top-5 ranked
- title-weighted BM25 over FTS5 + per-type rank offsets → 242x меньше файлов для чтения

### MCP
11 tools: read (resolve, search, query), write (save, move, delete), zone management.

### Что взять
- **5 типов записей** — это наша основа schema
- Title-weighted BM25 + per-type rank offsets
- Runtime knowledge authoring (агент пишет назад)
- ALIAS auto-creation из wikilinks

---

## hsuanguo/llm-wiki

**Stars:** 3 | **Lang:** Python | **Updated:** 2026-04-08  
**Repo:** https://github.com/hsuanguo/llm-wiki  
**Desc:** "LLM wiki that evolves with you"

### Что делает
Skills-based подход: agent skill (`skills/llm-wiki/`) + Python CLI (`lwiki/`).

### Структура
```
wiki-name/
├── AGENTS.md        (domain schema)
├── CLAUDE.md        (auto-imports AGENTS.md)
├── raw/
└── wiki/
    ├── summaries/
    ├── concepts/
    ├── entities/
    ├── insights/
    └── index.md, log.md, overview.md
```

### 5 операций
INIT, INGEST, QUERY, UPDATE, LINT

### CLI
`lwiki init`, `lwiki raw status`, `lwiki raw sync`

### Что взять
- AGENTS.md как domain schema document
- Drift detection (raw status command)
- Cascade updates — новая инфо автоматом обновляет связанные страницы

---

## asakin/llm-context-base

**Stars:** 1 | **Lang:** None | **Updated:** 2026-04-08  
**Repo:** https://github.com/asakin/llm-context-base

### Что делает
Starter framework без index файлов — scan at query time. Bold-field metadata вместо YAML.

### Ключевые решения
- **Нет index файлов** — scan at query time (anti-pattern для больших баз, но проще на старте)
- **Bold-field metadata > YAML** — проще для LLM
- **30-day training period** — фаза обучения перед production
- **Decision outcome tracking** — не только факты, но результаты решений

### Что взять
- Bold-field metadata как формат
- Decision outcome tracking
