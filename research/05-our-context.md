# Наш контекст — что уже существует в POS

**Изучено:** 2026-04-08

## Репозитории

- **POS:** https://github.com/evgeny-reprev/POS — личная OS фаундера
- **coliving-brain:** https://github.com/evgeny-reprev/coliving-brain — business domain knowledge
- **llm-wiki:** https://github.com/evgeny-reprev/llm-wiki — этот репо, движок wiki

## Что уже есть как raw sources

### POS
| Источник | Путь | Объём |
|----------|------|-------|
| GitHub Issues | `gh issue list --repo evgeny-reprev/POS` | 20+ open issues |
| Session logs | `.claude/logs/sessions-*.md` | ежедневно |
| Daily logs | `.claude/logs/daily/` | ежедневно |
| Inbox | `inbox/` | временный сброс |
| Agent memory | `.claude/projects/.../memory/` | feedback, project, reference |
| YouTube transcripts | `algorithms/product_div/Multi_agent_framework/` | 2+ видео |
| Skills docs | `resources/{inventory}*.md` | каталоги |

### Coliving (уже частично в coliving-brain)
| Источник | Статус |
|----------|--------|
| Notion extraction v4 | ✅ 4944 страниц, 225 БД, completed 2026-04-02 |
| Use layer (Phase 4) | ⚠️ нужно перестроить из блоков (issue coliving-brain#22) |
| Company strategy experiment | ✅ Phases A-H в experiments/2026-04-03-company-strategy-restart-from-zero/ |

## Два vault'а — обоснование

**POS Wiki** — мета-уровень:
- Личные решения и паттерны работы
- Архитектура POS (как работает система)
- Lessons learned по агентам и инструментам
- Hypothesis tracking
- Schema: decisions, sessions, systems, projects, patterns

**Coliving Wiki** — бизнес-домен:
- CRM, маркетинг, конкуренты, операции, жильцы
- По сути = то что строим в coliving-brain
- Schema: concepts, operations, initiatives, market, assets

**Почему раздельные:**
- Разные schema → разные namespace → разные агент-промты
- Смешивать = путать агента (он не знает в каком контексте отвечать)
- Но инфраструктура (tooling, pipeline) — **общая**

## Текущие pain points которые wiki должна решить

1. **Context anxiety** — при большой истории Issues агент перегружается читая всё подряд
2. **Нет compiled knowledge** — каждый раз заново синтезируем из raw (Issues, logs)
3. **Session snapshots теряются** — есть в Issues, но не структурированы как база знаний
4. **Coliving use layer сломан** — читался по заголовкам, не по блокам (issue #22)
5. **Нет query layer** — нельзя спросить "что мы решили про X" и получить быстрый ответ

## Что хотим от системы

- Задать вопрос "что мы решили по авторизации в POS?" → получить ответ из compiled wiki за ~300 токенов
- Ingested session → автоматически обновить relevant pages
- Lint раз в неделю → найти stale claims, orphan pages
- MCP tools → Claude/Codex могут читать и писать в wiki напрямую
