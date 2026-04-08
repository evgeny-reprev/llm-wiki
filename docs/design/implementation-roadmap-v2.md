# Implementation Roadmap v2

**Дата:** 2026-04-08  
**Цель Layer 0:** рабочий инструмент сегодня

---

## Sequencing rule

Не строить Layer 1+ пока Layer 0 не отвечает на реальные POS вопросы из compiled wiki.

---

## Layer 0: Рабочий инструмент (цель: сегодня)

**Что получаешь:** можно спросить "что решили по X?" и получить ответ из wiki за ~300 токенов. Claude читает wiki через MCP.

### Шаги

**Шаг 1: Scaffold (5 мин)**
```bash
mkdir -p /home/dev/POS/wiki/{raw/pos/{issues,sessions,docs,inbox},wiki/pos/{decisions,sessions,systems,projects,patterns},shared/{indexes,logs}}
touch /home/dev/POS/wiki/shared/indexes/record-catalog.jsonl
touch /home/dev/POS/wiki/shared/logs/ingest-log.md
touch /home/dev/POS/wiki/shared/logs/query-writeback.md
```

**Шаг 2: Schema file (5 мин)**  
Скопировать schema.md в `/home/dev/POS/wiki/schema.md`

**Шаг 3: Захватить raw sources (15 мин)**  
Пилотный набор — достаточно 5–7 источников:
- 3 ключевых POS Issues (например #57, #65, #82)
- 2 session snapshots из `.claude/logs/sessions-*.md`
- CLAUDE.md + AGENTS.md

Формат raw файла:
```markdown
# [TASK] Сократить токены на Codex automation задачи

**Origin:** https://github.com/evgeny-reprev/POS/issues/65
**Captured:** 2026-04-08
**Source ID:** pos:github_issue:65

---
<verbatim issue body>
```

**Шаг 4: Compile (20 мин)**  
Для каждого raw source — создать SUMMARY + META record.  
Запрос к Claude: "прочитай этот raw source, создай SUMMARY и META записи по schema.md"

**Шаг 5: Заполнить catalog (5 мин)**  
Добавить строку в `record-catalog.jsonl` для каждой созданной записи.

**Шаг 6: Lint check (5 мин)**  
Проверить вручную: все SUMMARY имеют поле Sources, все topics есть в catalog.

**Шаг 7: MCP (15 мин)**  
Добавить 3 инструмента в MCP config — wiki_search, wiki_get, wiki_list_topics.

**DoD Layer 0:**
- [ ] scaffold создан
- [ ] ≥5 raw artifacts с source_id
- [ ] ≥5 compiled SUMMARY records
- [ ] record-catalog.jsonl не пустой
- [ ] lint: zero missing Sources
- [ ] реальный POS вопрос отвечается из wiki без открытия raw
- [ ] Claude/Codex видит wiki через MCP

---

## Layer 1: Повторяемый ingest (следующая неделя)

**Что добавляет:** новые Issues и сессии автоматически попадают в wiki.

- Скрипт: `wiki-ingest.py` — обнаруживает новые raw sources, компилирует только изменённые topics
- Delta ingest: rerun затрагивает только изменённые topics
- Write-back: query results стейджируются в query-writeback.md, не молча мержатся
- COLLECTION records: timeline для decisions, timeline для sessions

**DoD Layer 1:**
- [ ] rerun касается только изменённых topics
- [ ] новый Issue/сессия автоматически попадает в правильный namespace
- [ ] query loads SUMMARY first
- [ ] write-backs staged, не merged

---

## Layer 2: Coliving Vault (когда разморозим coliving-brain)

**Что добавляет:** второй vault поверх той же инфраструктуры.

- `raw/coliving/` + `wiki/coliving/` на том же контракте
- Namespaces: concepts, operations, initiatives, market, assets
- Queries могут таргетировать `pos` или `coliving` явно
- Cross-vault links: explicit, sparse, reviewable

---

## Layer 3: Оптимизация токенов (когда Layer 1 стабилен)

- Stable prompt templates → prompt caching (50–90% savings)
- Batch API для компиляции нескольких sources за раз (50% discount)
- Улучшенный ranking в catalog (title-weighted BM25)
- Evidence snippets в SOURCE records

---

## Layer 4: Reviewable automation (когда Layer 3 стабилен)

- Diff перед применением изменений wiki
- Approval gates для sensitive topics
- LLM lint (contradiction detection, stale claims)
- Staged write-back promotion pipeline
