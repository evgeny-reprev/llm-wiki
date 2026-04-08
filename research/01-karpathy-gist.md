# Karpathy LLM Wiki — исходная идея

**Source:** https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f  
**Изучено:** 2026-04-08

## Тезис

Вместо RAG (переизвлечения при каждом запросе) — LLM компилирует **persistent wiki** из raw sources один раз. Knowledge compounds over time, а не пересчитывается заново.

> "The wiki is a persistent, compounding artifact"

## Три-слойная архитектура

```
Raw sources (immutable)
        ↓  [LLM ingest]
Wiki layer (LLM-generated markdown, обновляется)
        ↓  [query]
Answer + write-back → Raw sources
```

**Raw sources** — единственный source of truth. Никогда не меняются, только добавляются.  
**Wiki** — derived artifact. Противоречия резолвятся здесь.  
**Schema document** — конфигурация структуры и конвенций.

## Три операции

| Операция | Что делает |
|----------|-----------|
| **Ingest** | Обработать новые raw sources → обновить wiki pages → обновить index/log |
| **Query** | Поиск по wiki → синтез ответа → write-back ценных результатов назад |
| **Lint** | Health-check: противоречия, stale claims, orphan pages |

## Поддерживающие файлы

- `index.md` — content-oriented catalog
- `log.md` — chronological append-only record

## Почему это лучше RAG

| RAG | LLM Wiki |
|-----|----------|
| Re-synthesize при каждом запросе | Compile once, answer many |
| Токены = каждый запрос | Токены = только при ingest |
| Знания не накапливаются | Knowledge compounds |
| Контекст теряется | Persistent artifact |
