# Wiki Schema v2

**Дата:** 2026-04-08

---

## Форматы записей с примерами

### SUMMARY

```markdown
**Type:** SUMMARY
**Vault:** pos
**Topic:** decisions/codex-startup-parity
**Updated:** 2026-04-08
**Sources:** pos:github_issue:57, pos:session_snapshot:2026-03-31
**Confidence:** high
**Outcome:** implemented — startup now takes <30s, token cost reduced 40%

Codex запускается через общий AGENTS.md + WORKFLOW.md контракт.
При старте читает только: ME.md, AGENTS.md, CODEX.md, активный issue.
Не читает всю историю issues подряд.
Решение принято 2026-03-31 после диагностики issue #57.
```

### META

```markdown
**Type:** META
**Vault:** pos
**Topic:** decisions/codex-startup-parity
**Updated:** 2026-04-08
**Source Count:** 3
**Primary Entities:** Codex, AGENTS.md, startup protocol
**Status:** active
**Freshness:** 2026-04-04
**Related:** systems/codex-cli, patterns/agent-handoff
```

### SOURCE

```markdown
**Type:** SOURCE
**Vault:** pos
**Topic:** decisions/codex-startup-parity
**Source ID:** pos:github_issue:57
**Origin:** https://github.com/evgeny-reprev/POS/issues/57
**Evidence:** Issue описывает проблему долгого старта Codex и root cause — чтение лишних файлов при инициализации.
```

### ALIAS

```markdown
**Type:** ALIAS
**Vault:** pos
**Alias:** codex cold start
**Canonical Topic:** decisions/codex-startup-parity
**Reason:** часто используется как "холодный старт кодекса" в сессиях
```

### COLLECTION

```markdown
**Type:** COLLECTION
**Vault:** pos
**Topic:** collections/codex-decisions-timeline
**Collection Kind:** timeline
**Members:** decisions/codex-startup-parity, decisions/codex-token-reduction, decisions/codex-model-routing
**Rule:** все decisions связанные с Codex, сортировка по Updated desc
```

---

## POS Vault — namespaces и конвенции

| Namespace | Что хранит | Topic slug |
|-----------|-----------|-----------|
| `decisions/` | Что решили + outcome + why | `decisions/<action-object>` |
| `sessions/` | Compressed session: decisions + files + next | `sessions/<yyyy-mm-dd>-<slug>` |
| `systems/` | Инструменты, скиллы, агенты, cron | `systems/<tool-name>` |
| `projects/` | Текущий статус проекта | `projects/<name>` |
| `patterns/` | Паттерны работы с агентами | `patterns/<pattern-name>` |

**Naming rules:**
- Только lowercase, дефис как разделитель
- Без дат в slug (дата — в поле Updated)
- Максимум 5 слов

---

## Coliving Vault — namespaces (позже)

| Namespace | Что хранит |
|-----------|-----------|
| `concepts/` | Домен: термины, определения, продукт |
| `operations/` | SOP, повторяющиеся процессы |
| `initiatives/` | Активные маркетинговые/продуктовые инициативы |
| `market/` | Конкуренты, позиционирование, цены |
| `assets/` | Шаблоны, тексты, лендинги |

---

## record-catalog.jsonl формат

Одна строка на record:

```json
{"topic": "decisions/codex-startup-parity", "vault": "pos", "type": "SUMMARY", "path": "wiki/pos/decisions/codex-startup-parity.md", "updated": "2026-04-08", "confidence": "high", "tags": ["codex", "startup", "performance"]}
```

Catalog используется для быстрого routing без открытия файлов.

---

## schema.md (корень wiki/)

Копируется в `/home/dev/POS/wiki/schema.md` при init.  
Содержит: типы записей, naming rules, agent read order, примеры.  
User-editable — агенты читают при каждом ingest.
