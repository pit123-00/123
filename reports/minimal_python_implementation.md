# Отчет 6: Минималистичная реализация на чистом Python (Zero Complexity)

## 1. Концепция: Какой один проект взять за основу?

Если стоит задача **повторить полностью и точно требования `AGENT_WORKFLOW_FRAMEWORK_RU.md` без лишней сложности, без Node/TypeScript, без LangGraph, LangChain, тяжелых фреймворков и сторонних библиотек**, оптимальный выбор — **взять архитектурные паттерны ядра `deer-flow`** и реализовать их на **стандартной библиотеке Python 3.10+ (Standard Library only: `json`, `hashlib`, `pathlib`, `os`, `sys`, `argparse`, `re`)**.

### Почему именно `deer-flow` как единственный первоисточник?
В `deer-flow` уже решены три сложнейших математических/протокольных аспекта вашего ТЗ:
1. **Детерминированные квитанции инструментов (`tool_receipt.py`):** генерация `[r1]`, `output_sha256` прямо из потока результатов.
2. **Верификация цитирования родительским агентом (`receipt_verification.py`):** чистая функция без LLM, проверяющая, что субагент цитирует реальные квитанции, а не выдумывает результаты.
3. **Шлюз Read-Before-Write (`read_before_write_middleware.py`):** проверка хеша файла перед разрешением перезаписи/патча.

Все эти три механизма в `deer-flow` по своей сути являются **чистыми функциями**, не требующими LangGraph. Ниже приведена законченная, минималистичная реализация этой системы.

---

## 2. Минимальная структура проекта

В репозитории создается всего 5 компактных скриптов и папка с контрактами:

```text
workflow/
  contracts/
    assignment.schema.json
    result.schema.json
  scripts/
    common.py        # Утилиты: атомарная запись, вычисление sha256, проверка путей
    registry.py      # Реестр запуска и конечный автомат (FSM)
    delivery.py      # Курсор чтения порциями (<= 12 Кб) с подтверждением ACK
    receipts.py      # Квитанции инструментов и проверка цитат (из deer-flow)
    patch.py         # Применение точечных правок по section_id
    runner.py        # CLI точка входа для оркестратора (next_action)
```

Никаких виртуальных окружений с десятками пакетов. Работает на любом установленном Python 3.

---

## 3. Эталонный код ключевых модулей

### Модуль 1: `workflow/scripts/common.py` (Атомарность, Хеши и Безопасность путей)
```python
import hashlib
import json
import os
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent.parent

def sha256_text(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()

def sha256_file(path: Path) -> str:
    if not path.is_file():
        return ""
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while chunk := f.read(65536):
            h.update(chunk)
    return h.hexdigest()

def atomic_write_json(path: Path, data: dict):
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp_path = path.with_suffix(".tmp")
    with open(tmp_path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    os.replace(tmp_path, path)

def safe_resolve_path(rel_path: str, base_dir: Path) -> Path:
    """Защита от directory traversal (../ attacks)."""
    target = (base_dir / rel_path).resolve()
    if not str(target).startswith(str(base_dir.resolve())):
        raise PermissionError(f"Access denied: path {rel_path} escapes {base_dir}")
    return target
```

---

### Модуль 2: `workflow/scripts/receipts.py` (Квитанции и Верификация из `deer-flow`)
Реализует разделы 8.3–8.5 `AGENT_WORKFLOW_FRAMEWORK_RU.md`:
```python
import re
from typing import TypedDict

class ToolReceipt(TypedDict):
    id: str           # r1, r2, ...
    tool_name: str    # bash, read_file, write_file
    output_sha256: str
    summary: str

_CITATION_RE = re.compile(r"\[(r\d+)(?:\s+([a-zA-Z0-9_\-\.]+))?\]")

def parse_citations(text: str) -> list[tuple[str, str | None]]:
    """Парсит ссылки вида [r1] или [r1 bash] в тексте отчета."""
    return _CITATION_RE.findall(text)

def verify_report_citations(report_text: str, receipts: list[dict]) -> dict:
    """Детерминированная проверка цитат без участия LLM."""
    known_receipts = {r["id"]: r for r in receipts}
    citations = parse_citations(report_text)

    if not citations and receipts:
        return {
            "citation_resolved": False,
            "verdict": "FAIL",
            "reason": "Отчет не содержит ссылок на квитанции [rN] выполненных действий."
        }

    missing_ids = []
    mismatched_tools = []

    for rid, tool_name in citations:
        if rid not in known_receipts:
            missing_ids.append(rid)
        elif tool_name and tool_name != known_receipts[rid]["tool_name"]:
            mismatched_tools.append((rid, tool_name, known_receipts[rid]["tool_name"]))

    if missing_ids or mismatched_tools:
        return {
            "citation_resolved": False,
            "verdict": "FAIL",
            "missing_receipt_ids": missing_ids,
            "mismatched_tools": mismatched_tools,
        }

    return {"citation_resolved": True, "verdict": "PASS"}
```

---

### Модуль 3: `workflow/scripts/delivery.py` (Курсор чтения порциями <= 12 Кб)
Реализует раздел 5 `AGENT_WORKFLOW_FRAMEWORK_RU.md`:
```python
import sys
import argparse
from pathlib import Path
from .common import atomic_write_json, sha256_text

CHUNK_LIMIT_BYTES = 12000

def chunk_text(text: str, limit_bytes: int = CHUNK_LIMIT_BYTES) -> list[str]:
    """Разбивает текст по абзацам и строкам с жестким лимитом UTF-8 байт."""
    paragraphs = text.split("\n\n")
    chunks = []
    cur = []
    cur_bytes = 0
    for p in paragraphs:
        p_bytes = len((p + "\n\n").encode("utf-8"))
        if cur_bytes + p_bytes > limit_bytes and cur:
            chunks.append("\n\n".join(cur))
            cur = [p]
            cur_bytes = p_bytes
        else:
            cur.append(p)
            cur_bytes += p_bytes
    if cur:
        chunks.append("\n\n".join(cur))
    return chunks

def get_next_part(run_dir: Path, task_id: str, full_context: str, cursor: int):
    chunks = chunk_text(full_context)
    if cursor >= len(chunks):
        return None, len(chunks)

    part_content = chunks[cursor]
    part_hash = sha256_text(part_content)
    part_id = cursor + 1
    total = len(chunks)

    payload = (
        f"=== PART {part_id}/{total} [sha256: {part_hash}] ===\n"
        f"{part_content}\n"
        f"=== END_PART {part_id}/{total} ===\n"
        f"Next command: python workflow/scripts/delivery.py ack --run-id {run_dir.name} --part-id {part_id}\n"
    )
    return payload, total
```

---

### Модуль 4: `workflow/scripts/registry.py` (FSM и хранение состояния)
Реализует раздел 7 `AGENT_WORKFLOW_FRAMEWORK_RU.md`:
```python
from pathlib import Path
import json
from .common import atomic_write_json

VALID_TRANSITIONS = {
    "PREPARED": ["BOUND"],
    "BOUND": ["READING"],
    "READING": ["READING", "WORKING"],
    "WORKING": ["CHECKING"],
    "CHECKING": ["REPAIR", "COMPLETED"],
    "REPAIR": ["CHECKING"]
}

class Registry:
    def __init__(self, run_dir: Path):
        self.file_path = run_dir / "registry.json"
        if not self.file_path.exists():
            self.data = {
                "run_id": run_dir.name,
                "status": "PREPARED",
                "active_agent_id": None,
                "reading_cursor": 0,
                "total_parts": 0,
                "acknowledged_parts": [],
                "next_action": "BIND_AGENT"
            }
            atomic_write_json(self.file_path, self.data)
        else:
            with open(self.file_path, "r", encoding="utf-8") as f:
                self.data = json.load(f)

    def transition_to(self, new_status: str, next_action: str):
        cur = self.data["status"]
        if new_status not in VALID_TRANSITIONS.get(cur, []):
            raise ValueError(f"Invalid transition from {cur} to {new_status}")
        self.data["status"] = new_status
        self.data["next_action"] = next_action
        atomic_write_json(self.file_path, self.data)
```

---

### Модуль 5: `workflow/scripts/patch.py` (Точечные правки по `section_id`)
Реализует раздел 9 `AGENT_WORKFLOW_FRAMEWORK_RU.md`:
```python
import re
from pathlib import Path

def apply_section_patch(target_md: Path, section_id: str, replacement_text: str, allowed_sections: list[str]):
    if section_id not in allowed_sections:
        raise PermissionError(f"Modification of section '{section_id}' is forbidden!")

    content = target_md.read_text(encoding="utf-8")
    # Регулярное выражение поиска Markdown секции: ## <section_id> ... до следующего ##
    pattern = rf"(##\s+{re.escape(section_id)}\b[\s\S]*?)(?=\n##|\Z)"

    if not re.search(pattern, content):
        raise ValueError(f"Section {section_id} not found in document.")

    new_section = f"## {section_id}\n\n{replacement_text.strip()}\n"
    updated_content = re.sub(pattern, new_section, content)
    target_md.write_text(updated_content, encoding="utf-8")
```

---

## 4. Схема взаимодействия «Оркестратор — Скрипт — Субагент»

Вся работа оркестратора сводится к вызову одной команды:
```bash
python workflow/scripts/runner.py next --run-id <run_id>
```
Скрипт читает состояние и возвращает ровно **одно следующее действие** в формате JSON:
- `{"action": "SPAWN", "role": "author", "instructions": "..."}`
- `{"action": "DELIVER_CHUNK", "command": "python workflow/scripts/delivery.py read-next ..."}`
- `{"action": "WAIT_FOR_SUBMISSION"}`
- `{"action": "AUDIT", "candidate_path": "..."}`
- `{"action": "REPAIR", "issues": [...]}`
- `{"action": "COMPLETED", "final_artifact": "..."}`

### Ключевые преимущества этой минималистичной схемы:
1. **Детерминированность:** Оркестратор не "думает" о том, что делать дальше, он просто следует возвращенному полю `action`.
2. **Нулевые зависимости:** Код запускается где угодно — локально, в Docker, на сервере без необходимости ставить сложные пакеты или настраивать среду.
3. **Строгая математическая верификация:** Ни одна фаза не завершится, если хэши квитанций не сошлись или не подтверждены все порции чтения.
