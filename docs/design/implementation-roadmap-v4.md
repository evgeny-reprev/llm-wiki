# Implementation Roadmap v4 — от и до

**Дата:** 2026-04-08  
**Изменения vs v3:**
- 0.4 Search: per-type rank boost (blink-query pattern)
- 0.6 MCP: `wiki_brief()` как 4й инструмент (mempalace progressive loading)
- Layer 1: 2-pass ingest, drift detection `--status`, ALIAS auto-creation
- Layer 2: coliving vault → отдельный инстанс в `coliving-brain`, не в POS. Отложен на после POS wiki.

---

## Стек

| Компонент | Откуда | Токены |
|-----------|--------|--------|
| LLM для ingest | OpenRouter Gemma 3-12b (free) | ~$0 |
| LLM для сложного query | Claude Sonnet (только если нужно) | минимум |
| Search | `search_content.py` из coliving — адаптируем | без LLM |
| Catalog | `phase2_common.py` паттерны — адаптируем | без LLM |
| Delta tracking | `processing_state.json` паттерн | без LLM |
| MCP server | FastMCP Python (~60 строк) поверх search | без LLM |

---

## Layer 0 — Рабочий инструмент

**Цель:** спросить "что решили по X?" → ответ из wiki. Claude читает через MCP.  
**Объём:** ~200 строк нового кода + конфиг файлы

---

### Шаг 0.1 — Scaffold и AGENTS.md

**Что делаем:** создаём структуру папок + главный конфиг файл который агент читает при каждом ingest.

```bash
mkdir -p /home/dev/POS/wiki/{raw/pos/{issues,sessions,docs,inbox},wiki/pos/{decisions,sessions,systems,projects,patterns},shared/{indexes,logs},scripts}
touch /home/dev/POS/wiki/shared/indexes/record-catalog.jsonl
echo '[]' > /home/dev/POS/wiki/shared/indexes/ingest-state.json
touch /home/dev/POS/wiki/shared/logs/ingest-log.md
touch /home/dev/POS/wiki/shared/logs/query-writeback.md
```

**Создать `/home/dev/POS/wiki/AGENTS.md`** — domain schema, агент читает при каждом ingest:

```markdown
# POS Wiki — Domain Schema

## Vault: pos

### Namespaces
- decisions/  — что решили, почему, и что получилось (Outcome обязателен)
- sessions/   — compressed session: decisions + touched files + next actions
- systems/    — инструменты, скиллы, агенты, cron, MCP
- projects/   — текущий статус активных проектов
- patterns/   — паттерны работы с агентами и системой

### Record types (agent read order: SUMMARY → META → SOURCE)
- SUMMARY: 200-400 токенов, должен отвечать на вопрос без дополнительных запросов
- META: 100-200 токенов, freshness + entities + related
- SOURCE: pointer назад к raw artifact
- ALIAS: redirect без дублирования
- COLLECTION: timeline или группировка топиков

### Metadata format — bold-field, не YAML
**Type:** SUMMARY  
**Vault:** pos  
**Topic:** decisions/slug  
**Updated:** YYYY-MM-DD  
**Sources:** pos:class:id, ...  
**Confidence:** high|medium|low  
**Outcome:** что получилось в итоге (обязательно для decisions/)

### Source ID format
pos:github_issue:57  
pos:session_snapshot:2026-03-31  
pos:doc:claude-md
```

**DoD 0.1:**
- [ ] все папки созданы
- [ ] AGENTS.md существует и содержит все 5 namespaces
- [ ] `ingest-state.json` содержит `[]`

---

### Шаг 0.2 — Захват raw sources (пилот)

**Что делаем:** скачиваем 7 реальных POS источников в `raw/pos/`.

**Скрипт `/home/dev/POS/wiki/scripts/capture.sh`:**
```bash
#!/bin/bash
# Захватить GitHub Issue в raw/
# Usage: ./capture.sh github_issue 65
WIKI=/home/dev/POS/wiki
VAULT=pos
CLASS=$1
ID=$2

case $CLASS in
  github_issue)
    BODY=$(gh issue view $ID --repo evgeny-reprev/POS --json title,body,number,labels,createdAt -q '.')
    TITLE=$(echo $BODY | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['title'])")
    OUT="$WIKI/raw/$VAULT/issues/$ID.md"
    echo "# $TITLE" > $OUT
    echo "" >> $OUT
    echo "**Origin:** https://github.com/evgeny-reprev/POS/issues/$ID" >> $OUT
    echo "**Captured:** $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $OUT
    echo "**Source ID:** $VAULT:$CLASS:$ID" >> $OUT
    echo "" >> $OUT
    echo "---" >> $OUT
    echo "" >> $OUT
    gh issue view $ID --repo evgeny-reprev/POS >> $OUT
    echo "Captured: $OUT"
    ;;
  doc)
    SRC=$3
    SLUG=$(basename $SRC .md)
    OUT="$WIKI/raw/$VAULT/docs/$SLUG.md"
    echo "**Origin:** $SRC" > $OUT
    echo "**Captured:** $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $OUT
    echo "**Source ID:** $VAULT:doc:$SLUG" >> $OUT
    echo "" >> $OUT
    echo "---" >> $OUT
    echo "" >> $OUT
    cat $SRC >> $OUT
    echo "Captured: $OUT"
    ;;
esac
```

**Пилотный набор (запустить вручную):**
```bash
cd /home/dev/POS/wiki/scripts
chmod +x capture.sh
./capture.sh github_issue 65   # Codex token reduction
./capture.sh github_issue 57   # Codex startup parity
./capture.sh github_issue 82   # Shared skills layer
./capture.sh github_issue 84   # Gmail MCP repair
./capture.sh doc /home/dev/POS/CLAUDE.md
./capture.sh doc /home/dev/POS/AGENTS.md
./capture.sh doc /home/dev/POS/ME.md
```

**DoD 0.2:**
- [ ] ≥7 файлов в `raw/pos/`
- [ ] каждый файл содержит `**Source ID:**`
- [ ] `grep -r "Source ID" /home/dev/POS/wiki/raw/pos/ | wc -l` → ≥7

---

### Шаг 0.3 — Ingest script (compile raw → wiki records)

**Что делаем:** скрипт читает raw/ → вызывает Gemma через OpenRouter → пишет SUMMARY + META.

> **Layer 0:** один вызов LLM → оба record. В Layer 1 переходим на 2-pass (сначала SUMMARY, потом META с SUMMARY как контекстом — лучше качество).

**Файл `/home/dev/POS/wiki/scripts/ingest.py`:**

```python
#!/usr/bin/env python3
"""Wiki ingest: raw/ → wiki/ typed records via free OpenRouter models."""
from __future__ import annotations
import json, re, sys
from pathlib import Path
from urllib import request, error

WIKI = Path("/home/dev/POS/wiki")
ENV_PATH = Path("/home/dev/POS/.env")
OPENROUTER_URL = "https://openrouter.ai/api/v1/chat/completions"
FREE_MODELS = ["google/gemma-3-12b-it:free", "google/gemma-3-4b-it:free"]

INGEST_PROMPT = """You are a wiki compiler. Read the raw source below and create two markdown records.

AGENTS.md schema is:
{agents_md}

Create exactly two records separated by ---RECORD---:
1. SUMMARY record (200-400 tokens) — direct answer to "what is this about?"
2. META record (100-200 tokens) — freshness, entities, related topics

Use bold-field format. Detect the best namespace from: decisions, sessions, systems, projects, patterns.
Infer a slug from the content. For decisions, Outcome field is mandatory.

RAW SOURCE:
{raw_content}

Output only the two records, no explanation."""

def load_api_key() -> str:
    for line in ENV_PATH.read_text().splitlines():
        if line.startswith("OPENROUTER_API_KEY="):
            return line.split("=", 1)[1].strip()
    raise RuntimeError("OPENROUTER_API_KEY not found")

def call_llm(prompt: str) -> str:
    key = load_api_key()
    for model in FREE_MODELS:
        body = json.dumps({
            "model": model,
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 800,
            "temperature": 0.2,
        }).encode()
        req = request.Request(
            OPENROUTER_URL,
            data=body,
            headers={"Authorization": f"Bearer {key}", "Content-Type": "application/json"}
        )
        try:
            with request.urlopen(req, timeout=30) as r:
                data = json.load(r)
                return data["choices"][0]["message"]["content"]
        except Exception as e:
            print(f"  [{model}] failed: {e}", file=sys.stderr)
    raise RuntimeError("All models failed")

def extract_topic(record: str) -> str | None:
    m = re.search(r"\*\*Topic:\*\*\s*(.+)", record)
    return m.group(1).strip() if m else None

def update_catalog(topic: str, vault: str, record_type: str, path: str):
    catalog = WIKI / "shared/indexes/record-catalog.jsonl"
    entry = json.dumps({
        "topic": topic, "vault": vault, "type": record_type,
        "path": str(path), "updated": __import__("datetime").date.today().isoformat()
    })
    lines = [l for l in catalog.read_text().splitlines()
             if not (f'"topic": "{topic}"' in l and f'"type": "{record_type}"' in l)] if catalog.stat().st_size > 0 else []
    lines.append(entry)
    catalog.write_text("\n".join(lines) + "\n")

def ingest_file(raw_path: Path):
    agents_md = (WIKI / "AGENTS.md").read_text()
    raw_content = raw_path.read_text()
    source_id_match = re.search(r"\*\*Source ID:\*\*\s*(.+)", raw_content)
    source_id = source_id_match.group(1).strip() if source_id_match else raw_path.stem

    print(f"Ingesting {raw_path.name}...")
    prompt = INGEST_PROMPT.format(agents_md=agents_md[:1000], raw_content=raw_content[:3000])
    result = call_llm(prompt)

    parts = result.split("---RECORD---")
    records = [p.strip() for p in parts if p.strip()]

    for record_text in records:
        type_match = re.search(r"\*\*Type:\*\*\s*(\w+)", record_text)
        topic = extract_topic(record_text)
        if not type_match or not topic:
            continue
        record_type = type_match.group(1).upper()
        namespace = topic.split("/")[0]
        slug = topic.split("/")[-1]
        out_dir = WIKI / "wiki/pos" / namespace
        out_dir.mkdir(parents=True, exist_ok=True)
        out_path = out_dir / f"{slug}.md"
        out_path.write_text(record_text)
        update_catalog(topic, "pos", record_type, out_path)
        print(f"  → {record_type}: wiki/pos/{namespace}/{slug}.md")

    log = WIKI / "shared/logs/ingest-log.md"
    with log.open("a") as f:
        f.write(f"- {__import__('datetime').datetime.utcnow().isoformat()[:16]} | {source_id} | {raw_path.name}\n")

if __name__ == "__main__":
    raw_dir = WIKI / "raw/pos"
    files = list(raw_dir.rglob("*.md"))
    print(f"Found {len(files)} raw files")
    for f in files:
        try:
            ingest_file(f)
        except Exception as e:
            print(f"  ERROR {f.name}: {e}", file=sys.stderr)
    print("Done.")
```

**DoD 0.3:**
- [ ] `python3 scripts/ingest.py` отрабатывает без ошибок
- [ ] `ls wiki/pos/decisions/` → ≥1 файл
- [ ] `wc -l shared/indexes/record-catalog.jsonl` → ≥5 строк
- [ ] каждый wiki файл содержит `**Sources:**`

---

### Шаг 0.4 — Search (адаптация search_content.py)

**Что делаем:** BM25-like scorer с title-weighting и per-type rank boost (паттерн blink-query — 25–91x быстрее grep на 14K файлах).

**Файл `/home/dev/POS/wiki/scripts/search.py`:**

```python
#!/usr/bin/env python3
"""Wiki search: title-weighted BM25 + per-type boost over record-catalog."""
from __future__ import annotations
import json, re, sys
from pathlib import Path

WIKI = Path("/home/dev/POS/wiki")
WORD_RE = re.compile(r"\w+", re.UNICODE)

# Per-type rank boost (blink-query pattern): агент получает SUMMARY первым
TYPE_BOOST = {"SUMMARY": 10, "ALIAS": 5, "META": 3, "SOURCE": 0, "COLLECTION": 1}

def load_catalog() -> list[dict]:
    catalog = WIKI / "shared/indexes/record-catalog.jsonl"
    return [json.loads(l) for l in catalog.read_text().splitlines() if l.strip()]

def score(entry: dict, terms: list[str]) -> int:
    topic = entry.get("topic", "").lower()
    content = ""
    p = Path(entry.get("path", ""))
    if p.exists():
        content = p.read_text().lower()
    s = TYPE_BOOST.get(entry.get("type", ""), 0)  # per-type boost
    for term in terms:
        s += topic.count(term) * 8   # title-weighted
        s += content.count(term)
    return s

def search(query: str, vault: str = "pos", top_k: int = 5) -> list[dict]:
    terms = [t.lower() for t in WORD_RE.findall(query)]
    catalog = [e for e in load_catalog() if e.get("vault") == vault]
    scored = [(score(e, terms), e) for e in catalog]
    scored.sort(key=lambda x: -x[0])
    results = []
    for s, entry in scored[:top_k]:
        if s == 0:
            continue
        result = dict(entry)
        p = Path(entry.get("path", ""))
        result["snippet"] = p.read_text()[:400] if p.exists() else ""
        results.append(result)
    return results

if __name__ == "__main__":
    query = " ".join(sys.argv[1:])
    results = search(query)
    for r in results:
        print(f"\n## {r['topic']} ({r['type']})")
        print(r["snippet"][:300])
```

**DoD 0.4:**
- [ ] `python3 scripts/search.py "codex токены"` → возвращает ≥1 результат
- [ ] первый результат — SUMMARY запись, не SOURCE или META

---

### Шаг 0.5 — Lint (проверка качества)

**Файл `/home/dev/POS/wiki/scripts/lint.py`:**

```python
#!/usr/bin/env python3
"""Wiki lint: deterministic quality checks (adversarial evaluator pass)."""
from __future__ import annotations
import json, re
from pathlib import Path

WIKI = Path("/home/dev/POS/wiki")
errors = []

catalog = [json.loads(l) for l in (WIKI / "shared/indexes/record-catalog.jsonl").read_text().splitlines() if l.strip()]
catalog_topics = {e["topic"] for e in catalog}

for entry in catalog:
    p = Path(entry["path"])
    if not p.exists():
        errors.append(f"ORPHAN: {entry['topic']} → path not found: {p}")
        continue
    content = p.read_text()
    if "**Sources:**" not in content:
        errors.append(f"MISSING_SOURCES: {entry['topic']}")
    if entry["type"] == "SUMMARY" and entry.get("vault") == "pos":
        ns = entry["topic"].split("/")[0]
        if ns == "decisions" and "**Outcome:**" not in content:
            errors.append(f"MISSING_OUTCOME: {entry['topic']}")

# Wiki файлы не в каталоге
for f in (WIKI / "wiki/pos").rglob("*.md"):
    topic_guess = f.parent.name + "/" + f.stem
    if topic_guess not in catalog_topics:
        errors.append(f"NOT_IN_CATALOG: {topic_guess}")

if errors:
    print(f"LINT FAILED — {len(errors)} issues:")
    for e in errors:
        print(f"  ✗ {e}")
    raise SystemExit(1)
else:
    print(f"LINT OK — {len(catalog)} records checked")
```

**DoD 0.5:**
- [ ] `python3 scripts/lint.py` → `LINT OK`

---

### Шаг 0.6 — MCP server

**Файл `/home/dev/POS/wiki/scripts/wiki_mcp.py`** — 4 инструмента:

```python
#!/usr/bin/env python3
"""Wiki MCP server — 4 tools for Layer 0."""
import json
from pathlib import Path
from mcp.server.fastmcp import FastMCP
from search import search, load_catalog, WIKI

mcp = FastMCP("wiki")

@mcp.tool()
def wiki_brief() -> str:
    """Returns a ~150 token overview: namespaces + available topics.
    Call FIRST before any search to understand what's in the wiki."""
    catalog = load_catalog()
    by_ns: dict[str, list[str]] = {}
    for e in catalog:
        if e.get("type") == "SUMMARY":
            ns = e["topic"].split("/")[0]
            by_ns.setdefault(ns, []).append(e["topic"].split("/")[-1])
    lines = ["## POS Wiki — available knowledge"]
    for ns, topics in sorted(by_ns.items()):
        lines.append(f"- {ns}/: {', '.join(topics[:6])}")
    return "\n".join(lines)

@mcp.tool()
def wiki_search(query: str, vault: str = "pos") -> str:
    """Search wiki by query. Returns top-5 records, SUMMARY first."""
    results = search(query, vault=vault)
    if not results:
        return "No results found."
    out = []
    for r in results:
        out.append(f"## {r['topic']} ({r['type']})\n{r['snippet']}")
    return "\n\n---\n\n".join(out)

@mcp.tool()
def wiki_get(topic: str, record_type: str = "SUMMARY") -> str:
    """Get a specific wiki record by topic and type."""
    catalog = load_catalog()
    for entry in catalog:
        if entry["topic"] == topic and entry.get("type", "SUMMARY") == record_type:
            p = Path(entry["path"])
            return p.read_text() if p.exists() else f"File not found: {p}"
    return f"Topic not found: {topic}"

@mcp.tool()
def wiki_list_topics(namespace: str = "", vault: str = "pos") -> str:
    """List available topics, optionally filtered by namespace."""
    catalog = load_catalog()
    topics = [e["topic"] for e in catalog
              if e.get("vault") == vault and e.get("type") == "SUMMARY"
              and (not namespace or e["topic"].startswith(namespace))]
    return "\n".join(sorted(topics)) if topics else "No topics found."

if __name__ == "__main__":
    mcp.run()
```

**Добавить в `~/.claude.json` MCP config:**
```json
{
  "mcpServers": {
    "wiki": {
      "command": "python3",
      "args": ["/home/dev/POS/wiki/scripts/wiki_mcp.py"],
      "cwd": "/home/dev/POS/wiki/scripts"
    }
  }
}
```

**DoD 0.6:**
- [ ] `python3 scripts/wiki_mcp.py` запускается без ошибок
- [ ] Claude видит 4 инструмента: `wiki_brief`, `wiki_search`, `wiki_get`, `wiki_list_topics`
- [ ] `wiki_brief()` возвращает список namespaces без дополнительных запросов
- [ ] `wiki_search("codex токены")` возвращает содержимое SUMMARY записи

---

### Layer 0 — итоговый DoD

- [ ] scaffold создан
- [ ] AGENTS.md написан
- [ ] ≥7 raw artifacts с `**Source ID:**`
- [ ] ≥7 compiled SUMMARY+META records
- [ ] `record-catalog.jsonl` ≥7 строк
- [ ] `python3 scripts/lint.py` → LINT OK
- [ ] `python3 scripts/search.py "codex"` → SUMMARY запись первой
- [ ] MCP 4 tools видны в Claude
- [ ] реальный вопрос отвечается из wiki

---

## Layer 1 — Автоматический ingest (следующая неделя)

**Ключевые улучшения:**

**1. Delta mode + drift detection**
```bash
python3 scripts/ingest.py --status   # показать: N stale, M new, K up-to-date
python3 scripts/ingest.py --delta    # обработать только новые/изменённые
```
`ingest-state.json` хранит checksums. При `--status` сравнивает raw файлы с последним ingest — находит устаревшие.

**2. 2-pass ingest (лучше качество)**
- Pass 1: SUMMARY из raw
- Pass 2: META с уже готовым SUMMARY как контекстом

Результат: META точнее отражает freshness и entities, потому что видит скомпилированный summary, а не только raw.

**3. ALIAS auto-creation**
При появлении `[[другая-тема]]` в SUMMARY — автоматически создаётся ALIAS запись. Агент находит запись по синониму без дублирования контента.

**4. Auto-capture новых Issues**
`capture.sh` расширяется: `--since` flag через `gh issue list --since`.

**5. Write-back staging**
После query Claude пишет ценные синтезы в `query-writeback.md`. Раз в неделю: review → promote → ingest как новый SOURCE.

**6. COLLECTION records**
Timeline для decisions: что решали и в каком порядке.

---

## Layer 2 — Coliving vault (после завершения POS wiki)

> **Архитектурное решение (2026-04-08):** coliving vault живёт в `evgeny-reprev/coliving-brain`, не в POS.
>
> **Почему:** coliving данные уже там (4944 страниц Notion extraction). Phase 4 (use layer) в coliving-brain = это и есть coliving wiki. Строим один слой, не два.
>
> **Как:** движок (ingest.py, search.py, wiki_mcp.py, lint.py) копируется из POS/wiki/scripts/ в coliving-brain/wiki/scripts/. Данные переезжают внутри coliving-brain: `data/notion/` → `wiki/raw/coliving/` (mv, не cp). Отдельный MCP server с vault="coliving".

**Шаги (когда придёт время):**
- Скопировать scripts/ из POS wiki
- `mv data/notion/ wiki/raw/coliving/` внутри coliving-brain
- Создать `wiki/AGENTS.md` с coliving namespaces: concepts, operations, initiatives, market, assets
- Запустить ingest на пилотном наборе (20-30 страниц, не все 4944 сразу)
- Добавить MCP config для coliving wiki в `~/.claude.json`

---

## Layer 3 — Оптимизация (когда Layer 1 стабилен)

- Stable ingest prompts → prompt caching 50-90% (sage-wiki паттерн)
- Batch compilation: ≥3 sources за вызов
- **SQLite FTS5** вместо jsonl scan — подтверждено blink-query (25-91x speedup) и sage-wiki. Адаптация из `search_content.py` coliving-brain.
- Evidence snippets в SOURCE records

---

## Layer 4 — Reviewable automation

- Diff перед записью wiki изменений (swarmvault паттерн)
- LLM lint (contradiction detection) через Gemma free
- Staged write-back promotion
- God-node detection: записи-мосты между namespaces (swarmvault community detection)

---

## Порядок выполнения сегодня

```
0.1 Scaffold + AGENTS.md     (~10 мин)
0.2 Capture 7 raw sources    (~15 мин, bash)
0.3 Ingest script + запуск   (~20 мин)
0.4 Search script            (~10 мин)
0.5 Lint                     (~10 мин)
0.6 MCP server (4 tools)     (~15 мин)
────────────────────────────
Итого:                       ~80 мин
```

---

## Источники паттернов

| Паттерн | Источник |
|---------|---------|
| Compile once, query many | Karpathy Gist |
| 5 типов записей (SUMMARY/META/SOURCE/ALIAS/COLLECTION) | blink-query |
| Per-type rank boost в search | blink-query |
| Bold-field metadata | asakin/llm-context-base |
| Decision Outcome tracking | asakin/llm-context-base |
| Write-back drives compounding | bitsofchris/openaugi |
| Reviewable changes | swarmvault |
| Progressive loading (wiki_brief) | mempalace |
| Verbatim raw sources | mempalace |
| Drift detection (--status) | hsuanguo/llm-wiki |
| ALIAS auto-creation | blink-query |
| SQLite FTS5 (Layer 3) | blink-query + sage-wiki |
| Prompt caching + batch API (Layer 3) | sage-wiki |
| Lint = отдельный evaluator pass | Anthropic Long-Running Agent Blueprint |
| 2-pass ingest (Layer 1) | sage-wiki 5-pass pipeline |
| Vault isolation + sparse crosslinks | sakhmedbayev + Miras Framework |
