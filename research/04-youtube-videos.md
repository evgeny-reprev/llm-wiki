# YouTube видео — обработанные транскрипты

**Изучено:** 2026-04-08  
**Полные транскрипты:** `/home/dev/POS/algorithms/product_div/Multi_agent_framework/`

---

## Miras Agentic Framework

**Источник:** Improvado, "Introducing Miras, a Holistic Agentic Framework for Enhanced Collaboration"  
**YouTube:** https://www.youtube.com/watch?v=2T9yZKzJg3A  
**Дата:** 2026-04-03

### Тезис
Miras = ответ на "что если Obsidian проектировали сегодня для human + agent collaboration". Agent sessions как first-class objects.

### Три слоя

```
Data Sources → Agent Ingestion → Model + Governance → Knowledge Graph
                                                              ↓
                                                    Session Context → Decision Chain → Execution → Review + Diffs
```

### Session-first подход
- Каждая сессия = concise decision chain + diagrams + artifacts + next actions
- Bidirectional link session ↔ terminal (iTerm)
- "What did I do in this session?" без ручного восстановления

### "Token metabolism" model
Жизнь = последовательность токенов через множество систем. Агент ingest'ит raw data (code sessions, calls, CRM, Jira, GitHub, ads, docs) → моделирует → governance layer → knowledge graph.

### Partial graph sharing
Подграфы для разных контекстов. POS wiki ≠ Coliving wiki, но edges между ними возможны.

### Применение для нас
| Концепция Miras | Как применяем |
|-----------------|---------------|
| Session как first-class | POS Wiki: каждая сессия = raw source |
| Knowledge graph = backbone | Wiki объясняет откуда взялся факт |
| Partial graph | POS vault ≠ Coliving vault, sparse crosslinks |
| Plan → approve → execute | Reviewable changes (Layer 4) |

---

## Anthropic Long-Running Agent Blueprint

**Источник:** The AI Automators, "Anthropic Just Dropped the New Blueprint for Long-Running AI Agents"  
**YouTube:** https://www.youtube.com/watch?v=9d5bzxVsocw  
**Дата:** 2026-04-04

### Тезис
Harness design — главный control layer для long-running агентов, не мощность модели.

### Три failure mode

**1. Context anxiety**
По мере заполнения контекста агент начинает торопиться — пропускать шаги, заканчивать задачу раньше времени. Не просто "память кончается", а поведенческий сбой.  
→ Для wiki: typed records решают (агент читает SUMMARY 300 токенов, не весь файл)

**2. Слабая self-evaluation**
Агент оценивает своё же качество и хвалит посредственный результат.  
→ Для wiki: lint = отдельный evaluator pass, не часть ingest

**3. Reset vs compaction**
Anthropic пробовали reset (чистый контекст + артефакт прогресса) vs compaction (сжатие треда). С новыми моделями compaction достаточен.  
→ Для wiki: delta ingest (только новые/изменённые sources) = аналог compaction

### Harness evolves with models
Компоненты harness'а = временные предположения о модели. Simplify as models improve.  
→ Для нас: Layer 0 максимально простой. Не строить сложный pipeline который модель скоро сделает сама.

### Generator + Evaluator loop
Adversarial evaluation: generator agent создаёт артефакт, evaluator agent критикует внутри delivery цикла.  
→ Для wiki: ingest (generator) + lint (evaluator) — разные промты, разные агенты.
