# LLM Wiki — System Design v2

**Дата:** 2026-04-08  
**Основано на:** research/00-index.md + согласование с пользователем

---

## Принцип

Compile once, query many. Не RAG — wiki.

Агент платит токены один раз при ingest. Каждый query читает из compiled records (~300 токенов), не из raw (~3000 токенов). Knowledge accumulates — не пересчитывается.

```
raw/ (verbatim, immutable)
    ↓  ingest (LLM, платим один раз)
wiki/ (typed records, compressed)
    ↓  query (читаем дёшево, ~300 токенов)
answer + write-back staging
```

---

## Физическое расположение

```
/home/dev/POS/wiki/          ← корень, в POS репо
├── raw/
│   └── pos/
│       ├── issues/          ← GitHub Issues (md файлы)
│       ├── sessions/        ← session snapshots из .claude/logs/
│       ├── docs/            ← CLAUDE.md, AGENTS.md, ME.md
│       └── inbox/           ← захваченные материалы из inbox/
├── wiki/
│   └── pos/
│       ├── decisions/       ← что решили + outcome
│       ├── sessions/        ← compressed session records
│       ├── systems/         ← агенты, инструменты, скиллы, cron
│       ├── projects/        ← статус активных проектов
│       └── patterns/        ← паттерны работы с агентами
├── shared/
│   ├── indexes/
│   │   ├── topic-map.md          ← human-readable навигатор
│   │   └── record-catalog.jsonl  ← machine-readable router
│   └── logs/
│       ├── ingest-log.md         ← что и когда ingested
│       └── query-writeback.md    ← staged write-back очередь
└── schema.md                ← правила: типы, конвенции, примеры
```

Coliving добавится позже как `raw/coliving/` + `wiki/coliving/` рядом.

---

## Типы записей (5 типов, DNS-аналогия)

| Тип | Назначение | Токены |
|-----|-----------|--------|
| **SUMMARY** | Shortest useful read — ответ без дополнительных запросов | 200–400 |
| **META** | Freshness, status, entities, related topics | 100–200 |
| **SOURCE** | Pointer с контекстом → raw artifact | 50–100 |
| **ALIAS** | Redirect — синоним без дублирования контента | 20–50 |
| **COLLECTION** | Группировка: timeline, index, namespace | 100–300 |

**Agent read order:** SUMMARY → META → SOURCE → COLLECTION  
Агент останавливается как только получил достаточно. Raw читается только при явном запросе.

**Metadata format:** bold-field headers, не YAML.  
Причина: меньше prompt overhead, проще partial regeneration, надёжнее при LLM редактировании.

---

## Три операции

### Ingest
1. Обнаружить новые/изменённые raw artifacts
2. Классифицировать: vault (pos), namespace (decisions/sessions/...), topics
3. Извлечь: claims, decisions, outcomes, entities, next actions
4. Создать/обновить affected records (SUMMARY + META минимум)
5. Записать в ingest-log.md и record-catalog.jsonl

**Layer 0:** ручной запуск. Никаких daemon, queue, workers.

### Query
1. Роутинг по vault + namespace через record-catalog.jsonl
2. Загрузить SUMMARY → ответить если достаточно
3. Расширить до META/SOURCE только если нужна глубина
4. Ответить из wiki (не из raw)
5. Если ответ содержит новое знание → staged в query-writeback.md

### Lint
Отдельный pass, не часть ingest (adversarial evaluator):
- missing Sources в SUMMARY/META записях
- stale records (Updated > 30 дней, source не менялся)
- orphan pages (есть в wiki, нет в catalog)
- duplicate topics
- ALIAS без canonical target

---

## Token efficiency

| Стратегия | Экономия | Слой |
|-----------|---------|------|
| Typed records (SUMMARY first) | 5–10x меньше токенов на query | Layer 0 |
| Delta ingest (только changed) | пропорционально объёму | Layer 1 |
| Stable prompt templates | 50–90% via prompt caching | Layer 3 |
| Batch API для compilation | 50% discount | Layer 3 |

Правило: агент **никогда** не сканирует raw по умолчанию. Только wiki.

---

## MCP (Layer 0 — минимальный набор)

3 инструмента достаточно для старта:

| Tool | Что делает |
|------|-----------|
| `wiki_search(query, vault?)` | Поиск по record-catalog + SUMMARY |
| `wiki_get(topic, type?)` | Получить конкретную запись |
| `wiki_list_topics(namespace?)` | Browsing по namespace |

Позже добавятся: `wiki_ingest`, `wiki_lint`, `wiki_stage_writeback`, `wiki_review_diff`.

---

## Coliving Wiki (позже)

Та же инфраструктура, другой vault:
- `raw/coliving/` — Notion exports, meeting notes, research
- `wiki/coliving/` — namespaces: concepts, operations, initiatives, market, assets
- Cross-vault links: sparse, explicit, в `shared/crosslinks/` (добавим позже)

Не строим пока coliving-brain заморожен.
