# Паттерны из комментариев к Karpathy Gist

**Источник:** https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f (comments)  
**Изучено:** 2026-04-08

---

## Ключевые паттерны

### 1. Compilation > RAG
**Источник:** Karpathy + все реализации  
**Суть:** Compile once (платим LLM стоимость один раз при ingest), query many (query почти бесплатный — читаем из файла).  
**Применение:** Основа архитектуры. RAG не используем как основной путь.

---

### 2. Index файлы ломаются на масштабе
**Источник:** bitsofchris (4,000+ journal entries tested)  
**Суть:** Единый index.md перестаёт работать при большом количестве записей. Нужны дедупликация + MMR re-ranking.  
**Применение:** `record-catalog.jsonl` вместо монолитного index.md. Catalog как machine-readable router.

---

### 3. Write-back drives compounding
**Источник:** bitsofchris  
**Суть:** Когда query находит что-то ценное — записывать это обратно как новую wiki-страницу. Система учится от использования.  
**Применение:** Query операция всегда имеет write-back staging. Не merging автоматически — staged review.

---

### 4. Typed records → агент знает что читать без fetch всего
**Источник:** blink-query (83x speedup на 14K файлах)  
**Суть:** DNS-аналогия — по типу записи агент знает нужно ли читать всё или достаточно summary. Устраняет "context anxiety" от большой базы.  
**Применение:** 5 типов (SUMMARY/META/SOURCE/ALIAS/COLLECTION) + agent read order.

---

### 5. Verbatim > summary для recall задач
**Источник:** mempalace (96.6% vs ниже у summarization подходов)  
**Суть:** Summarization теряет "why" — контекст за решением, нюансы, оговорки. Verbatim raw сохраняет всё.  
**Применение:** Raw sources храним verbatim. Wiki компилирует сжатые records, но raw остаётся полным.

---

### 6. Reviewable changes — не silent rewrites
**Источник:** swarmvault  
**Суть:** Когда LLM меняет wiki-страницу — показывать diff перед применением, не писать молча.  
**Применение:** Layer 4 feature. В Layer 0 — manual review достаточно.

---

### 7. Bold-field metadata > YAML
**Источник:** asakin/llm-context-base  
**Суть:** YAML требует точного парсинга. Bold-field headers (`**Type:** SUMMARY`) проще для LLM генерировать и редактировать частично.  
**Применение:** Формат всех records в wiki.

---

### 8. Decision outcome tracking
**Источник:** emailhuynhhuy, asakin  
**Суть:** Хранить не только "что решили", но и "что получилось". Context + outcome + failure signals → operational experience.  
**Применение:** POS-специфика: каждый Decision record имеет поле Outcome.

---

### 9. Prompt caching + batch API → 50–90% экономия
**Источник:** sage-wiki  
**Суть:** Stable system prompts = cache hits. Batch API = 50% discount на async compilation.  
**Применение:** Layer 3 optimization. Промты должны быть стабильными (не включать timestamp в system prompt).

---

### 10. Adversarial evaluation ≈ lint operation
**Источник:** Anthropic Long-Running Agent Blueprint (YouTube)  
**Суть:** Lint — это не просто grammar check. Это отдельный агент-критик который проверяет качество wiki. Аналог GAN discriminator.  
**Применение:** Lint = отдельный pass с другим промтом, не часть ingest.

---

### 11. Unified vs split vault
**Источник:** sakhmedbayev (комментарий к Gist)  
**Вопрос:** Один vault для всего или раздельные по доменам?  
**Практика:** Разные домены = разные schema. Смешивать = путать агента.  
**Решение для нас:** Два vault'а (POS + Coliving), изолированные по умолчанию, sparse cross-links.

---

### 12. Harness evolves with models
**Источник:** Anthropic Long-Running Agent Blueprint (YouTube)  
**Суть:** Компоненты harness'а кодируют временные предположения. Правильная архитектура меняется с ростом модели.  
**Применение:** Не переусложнять Layer 0. Что сегодня требует сложного pipeline — завтра может делать одна модель.
