# Research Index — LLM Wiki

**Дата исследования:** 2026-04-08  
**Статус:** Завершено, готово к проектированию

## Файлы

| Файл | Содержание |
|------|-----------|
| [01-karpathy-gist.md](./01-karpathy-gist.md) | Исходная идея: 3-слойная архитектура, 3 операции, почему лучше RAG |
| [02-ecosystem-repos.md](./02-ecosystem-repos.md) | 7 репозиториев: mempalace, qmd, sage-wiki, openaugi, swarmvault, blink-query, llm-wiki, llm-context-base |
| [03-community-patterns.md](./03-community-patterns.md) | 12 паттернов из комментариев к Gist + YouTube |
| [04-youtube-videos.md](./04-youtube-videos.md) | Miras Framework + Anthropic Long-Running Agent Blueprint |
| [05-our-context.md](./05-our-context.md) | Что уже есть в POS, pain points, два vault'а |

## Ключевые решения из исследования

| Вопрос | Решение | Источник |
|--------|---------|----------|
| RAG vs Wiki | Wiki (compile once) | Karpathy |
| Формат metadata | Bold-field headers | asakin/llm-context-base |
| Типы записей | SUMMARY/META/SOURCE/ALIAS/COLLECTION | blink-query |
| Один vault или два | Два (POS + Coliving), изолированные | sakhmedbayev discussion |
| Index файл или catalog | `record-catalog.jsonl` (machine-readable) | bitsofchris (scale failure) |
| Verbatim или summary | Raw = verbatim, Wiki = compressed | mempalace (96.6% recall) |
| Write-back | Staged, reviewable | swarmvault |
| Cost optimization | Prompt caching + batch API | sage-wiki |
| Lint как операция | Отдельный evaluator pass | Anthropic blueprint |

## Что берём из каждого репо

| Репо | Что берём |
|------|-----------|
| mempalace | Verbatim raw, progressive loading, namespace isolation |
| qmd | Hybrid search (BM25 + semantic + LLM rerank) |
| sage-wiki | Prompt caching, batch API, 5-pass pipeline |
| openaugi | Write-back drives compounding, context blocks = META |
| swarmvault | Reviewable changes, schema.md как user config, multi-agent rules |
| blink-query | **5 типов записей** (основа schema), title-weighted BM25 |
| llm-wiki | AGENTS.md domain schema, drift detection, cascade updates |
| llm-context-base | Bold-field metadata, decision outcome tracking |
