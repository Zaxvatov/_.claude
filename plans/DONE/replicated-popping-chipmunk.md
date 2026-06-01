# План: вынести `budget` в отдельный портфельный модуль

## Контекст

Сейчас функционал бюджета живёт внутри `assistant-ui/app/budget/` (2 640 LOC, 16 файлов) и БД в `assistant-ui/data/budget.db`. По концепции `assistant-ui` — UI-слой, а домены (PM, память, бюджет) живут как самостоятельные модули портфеля наравне с `ai-memory/` и `pm-mcp-server/`.

Триггер выноса: появятся **внешние потребители-агенты** (Codex, Claude Code, в перспективе ChatGPT через Gateway — отдельной фазой) — как минимум для аналитики, в перспективе для добавления записей. Без MCP-границы агенты не могут обращаться к бюджету напрямую.

**Ревизия после Codex-review**: первая редакция плана недооценивала coupling. Реальный объём изменений в Phase 1 больше — 15+ внутренних `from app.budget…` импортов нужно переписать на `from budget…`, а не один.

**Используемые кирпичики `_engineering_rules/tech-stack-choices.md`** (явно, без отклонений):

| # | Кирпичик | Где применяется |
|---|---|---|
| 1 | NSSM-обёртка для daemon | `budget/scripts/run_server.ps1` под будущий NSSM-сервис |
| 3 | SQLite + WAL | Уже используется в `budget.db`; добавляем WAL-checkpoint в migration discipline |
| 4 | Namespaced env vars | `BUDGET_DB_PATH`, `BUDGET_MCP_PORT` (host **не** конфигурируется — жёсткий loopback, см. ниже) |
| 5 | FastMCP + loopback (без auth для local) | `budget/server.py` через `mcp.server.fastmcp.FastMCP`, bind 127.0.0.1:8767 |
| 7 | uv + per-subsystem `.venv` | Новый `budget/.venv`, в assistant-ui `uv add --editable ../budget` |
| 8 | Singleton-guard через named mutex / lock-file | Для нового daemon на порту 8767 |

## Рекомендуемый подход

**Гибридная архитектура потребления**:

| Потребитель | Способ доступа | Почему |
|-------------|----------------|--------|
| `assistant-ui` route handlers (~26 endpoints в `app/main.py:897-1258`) | Direct Python import (`from budget.services import transactions`) через editable path dependency | Страница `/budget` бьёт ~9 сервисных вызовов на один рендер. MCP roundtrip умножит latency на 9 без выигрыша — UI и budget живут в одном процессе. |
| Codex, Claude Code, внутренний агентский чат assistant-ui | MCP через streamable-http на `127.0.0.1:8767` | Стандартный паттерн портфеля: PM-MCP — 8766, AI-memory — 8765, gateway — 8780. |
| ChatGPT через Gateway | **Вне v1** — отдельная фаза с ADR | Финансовые данные чувствительные. Внешний доступ требует scope policy / DENIED_TOOLS / audit / redaction. |

`budget` физически становится sibling-пакетом рядом с `ai-memory/`, со своим `pyproject.toml`, `.venv`, `data/budget.db`, `server.py`, тестами, документацией.

**MCP-транспорт v1**: только loopback streamable-http (без stdio). Один режим, один порт, NSSM-совместимый запуск. Subprocess-spawn как у ai-memory stdio не нужен — все клиенты могут пойти по HTTP.

**Tools v1**: read + analytics. Write-tools (создание транзакций агентом) — отдельная итерация после валидации.

## Целевая структура `D:/GitHub/AI-Assistant/budget/`

```
budget/
├── pyproject.toml          # name: budget; deps: pydantic, openpyxl, mcp
├── uv.lock
├── .python-version
├── .venv/                  # per-subsystem (#7)
├── .gitignore              # игнорит data/, *.sqlite-wal, *.bak, logs/
├── README.md
├── ARCHITECTURE.md
├── AGENTS.md
├── CLAUDE.md
├── server.py               # entrypoint: streamable-http daemon
├── budget/                 # core subpackage (то, что было app/budget/)
│   ├── __init__.py
│   ├── config.py           # NEW: db_path() читает BUDGET_DB_PATH
│   ├── runtime_contract.py # NEW: BUDGET_HOST/PORT/PATH/HEALTH_PATH, EXPECTED_TOOLS
│   ├── singleton.py        # NEW: named mutex / lock-file для daemon (#8)
│   ├── mcp_app.py          # NEW: build_mcp_server() по образцу ai-memory
│   ├── db.py
│   ├── schemas.py
│   ├── repositories.py
│   ├── importer.py
│   └── services/
│       ├── __init__.py
│       ├── store.py        # импорт из ../config, не из app.config
│       ├── accounts.py
│       ├── analytics.py
│       ├── balances.py
│       ├── categories.py
│       ├── fx_chain.py
│       ├── lookups.py
│       ├── products.py
│       ├── settings.py
│       └── transactions.py
├── data/                   # gitignored (`data/` в корневом .gitignore уже игнорит)
│   └── budget.db
├── scripts/
│   ├── import_budget_xlsx.py
│   ├── backup_budget_db.ps1   # NEW: daily WAL checkpoint + copy
│   └── run_server.ps1         # NSSM-compatible (#1)
├── docs/
│   ├── MCP_TOOLS.md           # контракт каждого tool
│   └── RUNTIME.md             # port, healthcheck, singleton-guard, NSSM
└── tests/
    ├── conftest.py
    ├── test_balances.py       # переехал из assistant-ui/tests/
    ├── test_internal_imports.py  # NEW: assert no `from app.budget` остатков
    └── test_server_smoke.py      # NEW: live FastMCP fixture
```

## Dependency model (явно)

В этом монорепо **нет** root `uv workspace` (проверено: только `pm-mcp-server/`, `gateway/`, `ai-memory/`, `assistant-ui/` имеют свои `pyproject.toml`; root `pyproject.toml` отсутствует). Кирпичик #7 — per-subsystem `.venv`.

Решение:
- `budget/` получает свой `.venv` через `uv venv` внутри `budget/`.
- `budget/pyproject.toml`: `name = "budget"`, deps `pydantic>=2`, `mcp>=1.27`, `openpyxl>=3.1.5` (для importer); dev `pytest`, `ruff`.
- В `assistant-ui/pyproject.toml` добавить `budget` как editable path: команда `uv add --editable ../budget` из `assistant-ui/`.
- Все runtime/test/lint команды в плане — `uv run ...` внутри соответствующего subsystem (`uv run pytest`, `uv run ruff check .`, `uv run python server.py`).

## SQLite migration discipline

`assistant-ui/data/budget.db` — **runtime data**, уже игнорится корневым `.gitignore` (строка 51 `data/`).

Команды NSSM stop/start требуют **elevated PowerShell**. Агент **не выполняет** их сам — выдаёт пользователю готовый snippet ниже, пользователь запускает с правами администратора.

```powershell
# Run in elevated PowerShell.
# 1. Stop UI (правильное имя сервиса — AI-Assistant-Assistant-UI, не assistant-ui).
nssm stop AI-Assistant-Assistant-UI

# 2. WAL checkpoint — сбросить WAL в основной файл.
sqlite3 "D:\GitHub\AI-Assistant\assistant-ui\data\budget.db" "PRAGMA wal_checkpoint(TRUNCATE);"

# 3. Integrity check — должно вернуть `ok`.
sqlite3 "D:\GitHub\AI-Assistant\assistant-ui\data\budget.db" "PRAGMA integrity_check;"

# 4. Backup — `*.bak` уже в корневом .gitignore.
Copy-Item "D:\GitHub\AI-Assistant\assistant-ui\data\budget.db" `
          "D:\GitHub\AI-Assistant\assistant-ui\data\budget.db.pre-extract.bak"

# 5. Подготовить целевой каталог и перенести exact paths (без wildcard, чтобы не захватить .bak).
New-Item -ItemType Directory -Force -Path "D:\GitHub\AI-Assistant\budget\data" | Out-Null
foreach ($name in @("budget.db", "budget.db-wal", "budget.db-shm")) {
    $src = "D:\GitHub\AI-Assistant\assistant-ui\data\$name"
    if (Test-Path $src) {
        Move-Item $src "D:\GitHub\AI-Assistant\budget\data\$name"
    }
}

# 6. Sanity-check счёта строк.
sqlite3 "D:\GitHub\AI-Assistant\budget\data\budget.db" "SELECT COUNT(*) FROM transactions;"

# 7. Запустить UI заново.
nssm start AI-Assistant-Assistant-UI
```

После рестарта открыть `/budget` — журнал и баланс должны идентично сматчиться с пред-миграционным состоянием. Дополнительная гарантия — `test_balances.py` проверяет invariant `account_balance_after` (window function).

**Backup-конфиг**: внешний MCP-Daily-Backup существует в `G:\Мой диск\Бэкапы, инструкции, настройки, синхронизации\AI-assistent\backup-config.json` (за пределами репозитория) и сейчас **не включает** `budget.db` — он бэкапит conversation store и другие assistant-ui артефакты. После Phase 2 нужно:

1. Обновить внешний `backup-config.json`: добавить новый source-path `D:\GitHub\AI-Assistant\budget\data\budget.db` рядом с существующими entries. Это **операционная задача** — отдельный PM-MCP work item для `[ops]` или соответствующего subsystem'а. Файл доступен на чтение в текущей среде, но **изменять только по явному подтверждению пользователя** (он управляет MCP-Daily-Backup pipeline и должен видеть diff перед применением).
2. Локально в репозитории создать `budget/scripts/backup_budget_db.ps1` как fallback / ручной backup: `PRAGMA wal_checkpoint(TRUNCATE)` → `Copy-Item budget/data/budget.db budget/data/backups/budget-YYYYMMDD.db`. Это **не заменяет** внешний daily backup, а дополняет на случай его отказа.
3. Зафиксировать оба пункта в `budget/docs/RUNTIME.md` как operational checklist с явным указанием, что внешний backup-config — единственный автоматизированный путь.

## Untangling импортов (Phase 1, исправление к редакции 1)

Полный список внутренних `from app.budget…` импортов (`grep -n "^from app\." assistant-ui/app/budget/`):

- `__init__.py:7` — `from app.budget.db import SCHEMA_VERSION, init_db`
- `repositories.py:11` — `from app.budget.schemas import …`
- `importer.py:14-20` — 7 импортов: `repositories`, `schemas`, `services.balances`, `services.lookups`, `services.categories.base_defaults_for_item`, `services.fx_chain.{FxInput, compute_chain}`, `services.store.open_connection`
- `services/store.py:9` — `from app.budget import db as budget_db`
- `services/store.py:10` — `from app.config import budget_db_path` (единственный assistant-ui-coupling)
- `services/accounts.py`, `categories.py`, `lookups.py`, `settings.py`, `analytics.py`, `balances.py`, `products.py`, `transactions.py` — суммарно ~15 импортов вида `from app.budget…` (см. результат grep)

Замена при копировании в `budget/budget/`: `from app.budget` → `from budget`. Для `services/store.py` дополнительно `from app.config import budget_db_path` → `from budget.config import db_path`.

Чтобы поймать пропуски — добавить `tests/test_internal_imports.py`, который проверяет `ast.parse` всех `.py` в `budget/budget/` и assert'ит, что ни один импорт не содержит модуля `app.*`.

## MCP tool surface v1 (read + analytics)

«MCP tool» — функция, регистрируемая через `@mcp.tool()`-декоратор; агент видит её имя + JSON-схему. Например, Codex в чате может попросить «покажи баланс по счетам на конец прошлого месяца» — внутри вызовет `get_balances` с параметром `as_of`.

| Группа | Tool | Назначение |
|--------|------|------------|
| Analytics | `get_analytics_snapshot(period)` | Снимок аналитики по периоду |
| Analytics | `get_monthly_product(month)` | Месячный «продукт»: budget left, daily budget, плановое/факт |
| Analytics | `list_available_months()` | Список месяцев, для которых есть данные |
| Read | `list_transactions(limit, offset, filters)` | Журнал, с фильтрами по дате/счёту/категории |
| Read | `get_balances(as_of?)` | Баланс по счетам и валютам |
| Read | `list_categories()` / `list_accounts()` / `list_settings()` | Справочники |
| Read | `list_operation_types()` / `list_need_kinds()` | Лукапы |

Не входит в v1 (явно): write-CRUD `create_transaction`, `update_transaction`, `delete_transaction` и аналоги для категорий/счетов/настроек — отдельная итерация после read-only валидации и решения по guardrails (idempotency, дневной лимит, диалог подтверждения).

## MCP daemon hardening checklist (Phase 3)

**Reuse mandate**: брать паттерны напрямую из `ai-memory/`, не писать параллельный стиль. Конкретные образцы:

- `ai-memory/memory/runtime_contract.py` — структура констант + dataclass + helper-функции (`get_background_mcp_url()`, `get_healthcheck_url()`).
- `ai-memory/memory/mcp_app.py::build_mcp_server()` — FastMCP wiring, tool registration через `_register_tool()`, healthcheck через `@mcp.custom_route(…)`.
- `ai-memory/memory/daemon.py` — singleton-guard (`get_daemon_lock_path()`), startup orchestration, `register_process_state_with_retry()`, идемпотентность при уже-запущенном instance (`status: "already_running"` + healthcheck-сверка).

Чеклист:

- [ ] `budget/budget/runtime_contract.py` с константами `BUDGET_HOST = "127.0.0.1"`, `BUDGET_PORT = 8767`, `BUDGET_PATH = "/mcp"`, `BUDGET_HEALTH_PATH = "/healthz"`, `EXPECTED_TOOLS = (…)`, dataclass `BudgetRuntimeContract`, helper-функции `get_background_mcp_url()`, `get_healthcheck_url()`. **Структуру копируем из `ai-memory/memory/runtime_contract.py`.**
- [ ] `budget/budget/mcp_app.py::build_mcp_server()` — создаёт `FastMCP("budget", host, port, log_level, streamable_http_path)`, регистрирует tools через приватные функции `register_read_tools()`, добавляет `@mcp.custom_route(BUDGET_HEALTH_PATH, methods=["GET"])` healthcheck, возвращающий `{"status": "ok", "schema_version": …, "db_size_bytes": …, "pid": …}`. **Шаблон — `ai-memory/memory/mcp_app.py::build_mcp_server()`.**
- [ ] `budget/server.py` — тонкая точка входа: `if __name__ == "__main__"`: singleton-guard → healthcheck-сверка → `init_db()` → `register_process_state_with_retry()` → `build_mcp_server().run(transport="streamable-http")`. **Шаблон — `ai-memory/memory/daemon.py::run_daemon_service()` (строка 322) + `register_process_state_with_retry()` (строка 395).**
- [ ] `budget/budget/singleton.py` — lock-file `data/budget.pid` + advisory lock (`msvcrt.locking` на Windows / `fcntl.flock` на Unix) (#8). При уже занятом lock — healthcheck живого daemon; если живой → graceful `sys.exit(0)` с лог-сообщением «Another instance running (PID X)»; если мёртвый → переtake lock. **Шаблон — функции `get_daemon_lock_path()` и lock-логика в `ai-memory/memory/daemon.py`.**
- [ ] Log path: stdout + `budget/logs/budget-daemon.log` (rotating handler). Каталог в `.gitignore`.
- [ ] Preflight: `Test-NetConnection 127.0.0.1 -Port 8767` перед NSSM-регистрацией. На момент проверки порт свободен.
- [ ] `budget/scripts/run_server.ps1` — NSSM-совместимый shim: `cd budget && .venv/Scripts/python.exe server.py`. NSSM-конфиг: `AppExit 0 = Exit`, `AppThrottle 60000`, `AppRestartDelay 30000` (#8).
- [ ] Регистрация в PM-MCP Process State Manager при startup. Реальная сигнатура `pm-mcp register_process` (`pm-mcp-server/server.py:3575`):

  ```python
  arguments = {
      "project_path": str(Path(__file__).resolve().parents[1]),
      "process_key": "budget-daemon",
      "pid": os.getpid(),
      "command": f"{sys.executable} server.py",
      "executable": sys.executable,
      "supports_idle": True,
      "allow_stop": False,
      "state": "Active",
  }
  call_pm_mcp_tool("register_process", arguments)  # с retry, см. ai-memory/memory/daemon.py:395
  ```

- [ ] `budget/tests/test_server_smoke.py` — фикстура с поднятым `FastMCP` в отдельном процессе, MCP клиент через `mcp.client.streamable_http.streamable_http_client` (импорт точно такой же, как в `ai-memory/memory/daemon.py:19`) + `mcp.ClientSession`, вызовы `list_transactions(limit=1)` и `get_analytics_snapshot`, проверка JSON-схемы. Дополнительно тест «занятый порт»: при попытке второго старта daemon должен log + exit 0.

## MCP-клиент в assistant-ui (Phase 4)

Текущий `assistant-ui/app/mcp_client.py` имеет два клиента:
- `PMMCPHttpClient` — **custom HTTP** к `/mcp/{tool_name}` (не streamable-http MCP протокол). Не подходит для нашего FastMCP-сервера.
- `AIMemoryStdioClient` — MCP `ClientSession` через **stdio**. Не подходит для нашего HTTP-сервера.

Решение — добавить **третий** клиент `BudgetStreamableHttpClient`, использующий `mcp.ClientSession` через `mcp.client.streamable_http.streamable_http_client` (точное имя helper'а — см. `ai-memory/memory/daemon.py:19`). Это generic-клиент, который потом можно переиспользовать, если ai-memory daemon станет первичным транспортом.

Скелет (новый файл `assistant-ui/app/mcp_client.py` дополняется). Имя helper'а — `streamable_http_client` (а не `streamablehttp_client`), точно как в `ai-memory/memory/daemon.py:19`:

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client
# ...
class BudgetStreamableHttpClient:
    def __init__(self, url: str | None = None, timeout_seconds: float | None = None) -> None:
        self.url = url or os.environ.get("ASSISTANT_BUDGET_MCP_URL") or "http://127.0.0.1:8767/mcp"
        self.timeout_seconds = timeout_seconds or _timeout_seconds("ASSISTANT_BUDGET_TIMEOUT_SECONDS")
    # list_tools / call_tool: открыть streamable_http_client(url) → ClientSession → session.list_tools() / call_tool()
```

Регистрация в `MCPClients`-aggregate: добавить `budget` поле, default `BudgetStreamableHttpClient()`.

В `app/tool_router.py` зарегистрировать budget-tools в **отдельный** toolset `"budget"` (не сливать с общим `"mixed"`), чтобы маленькие чаты не тащили его в контекст без причины. Routing keywords: «бюджет», «трат», «баланс», «budget», «expense», «category» — точная подборка в коде роутера.

**Fail-open requirement** (v1 — обязательно): если budget daemon не запущен/недоступен, это **не должно ломать небюджетные чаты**. Решение v1 — **static EXPECTED_TOOLS fallback**, по образцу `PM_MCP_TOOL_NAMES` fallback в `mcp_client.py:177`:

- `BudgetStreamableHttpClient.list_tools()` при healthcheck-fail возвращает **static** список из `EXPECTED_TOOLS` (импортируется из `budget.runtime_contract`). Это позволяет агентскому tool-loop'у иметь схему даже без живого daemon — `MCPClientManager.list_tools()` собирает карту по всем namespace без падения.
- `BudgetStreamableHttpClient.call_tool()` при connection error возвращает structured error `{"ok": False, "error": {"code": "budget_unavailable", "message": "..."}}`, не raise. Агент видит ошибку и может ответить пользователю «бюджет временно недоступен», не падая.

Lazy registration (подключать toolset только при budget-keywords в промте) — **не в v1**: требует менять текущую модель `MCPClientManager`, которая строит карту через `clients.list_tools()` по всем namespace сразу. Зафиксировать как возможную оптимизацию для будущей итерации.

## PM-MCP `app/adapters/budget.py` (резолюция коллизии)

В `pm-mcp-server/app/adapters/budget.py` уже есть модуль с тем же именем, но **другой семантикой**: он принимает `raw_items: list[dict]` и превращает их в `WorkItem(domain="finance", type="budget_item", source="budget_stub")`. Это work-item stub для finance-домена PM-системы, **не** связан с реальным `budget.db` и журналом транзакций.

Чтобы избежать семантической путаницы для агентов и читателей:
- Переименовать `pm-mcp-server/app/adapters/budget.py` → `pm-mcp-server/app/adapters/finance_stub.py`.
- Соответствующий import в pm-mcp-server обновить.
- В `pm-mcp-server/AGENTS.md` или `ARCHITECTURE.md` добавить заметку: «finance/budget_item — work-item stub, не финансовый домен; реальный бюджет см. `budget/` модуль».
- В `budget/AGENTS.md` обратная ссылка: «не путать с `pm-mcp-server/app/adapters/finance_stub.py`».

Это **минимальное** касание PM-MCP — переименование + ссылки, без изменения логики.

## Trust model в v1

- Local агенты (Claude Code, Codex CLI, agent_loop внутри assistant-ui) — **трастятся**, ходят напрямую через loopback streamable-http без auth (#5, hybrid trust model).
- Host hardcoded `127.0.0.1` в `runtime_contract.py::BUDGET_HOST`. Env var для host **не предусматриваем** — это сохраняет инвариант FastMCP loopback trust (#5). Если по какой-то операционной причине host всё-таки надо сделать override-абельным (например, IPv6 dual-stack), валидация принимает **только** `127.0.0.1`, `localhost`, `::1`; любой другой хост → fail-fast на startup с лог-сообщением «non-loopback host rejected». `0.0.0.0` запрещён архитектурно.
- Gateway (ChatGPT через интернет) — **запрещён** в v1. Никакого route в `gateway/scope_policy.py` для budget-tools пока не добавляем.
- Расширение для external клиента — отдельная фаза с собственным ADR (по аналогии с ADR-0001 D-3, `gateway/scope_policy.py` `DENIED_TOOLS`). Минимум: scope tree `budget.read`, audit-логирование, redaction чувствительных полей (account number, balances), explicit allowlist tools.

## Фазы миграции

Каждая фаза оставляет систему рабочей.

### Phase 1. Создать пакет `budget/`

- Создать `D:/GitHub/AI-Assistant/budget/` со скелетом структуры выше, но **без** `server.py` (Phase 3), **без** переноса БД (Phase 2), **без** singleton/runtime_contract (Phase 3).
- `uv venv` внутри `budget/`. `uv add pydantic openpyxl`. `uv add --dev pytest ruff`.
- Скопировать содержимое `assistant-ui/app/budget/` → `budget/budget/`.
- **Переписать 15+ внутренних импортов** `from app.budget…` → `from budget…` (см. секцию «Untangling импортов»).
- Создать `budget/budget/config.py::db_path()`, читающий `BUDGET_DB_PATH` (без fallback). На время Phase 1 переменная указывает на старый путь `assistant-ui/data/budget.db`.
- В `budget/budget/services/store.py` заменить:
  - `from app.budget import db as budget_db` → `from budget import db as budget_db`
  - `from app.config import budget_db_path` → `from budget.config import db_path`
- Перенести `assistant-ui/tests/test_budget_balances.py` → `budget/tests/test_balances.py`, обновить импорты.
- Добавить `budget/tests/test_internal_imports.py` (assert no `app.*` imports).
- Из `budget/`: `uv run pytest tests/` зелёный, `uv run ruff check .` чистый.

В assistant-ui ничего не меняется — старый код продолжает работать. Это безопасный first commit.

### Phase 2. Переключить assistant-ui на новый пакет + физический перенос БД

- В `assistant-ui/`: `uv add --editable ../budget` (добавится в `pyproject.toml` секцию `dependencies` или отдельную path-секцию).
- В `assistant-ui/app/main.py:23-30` обновить **только импорты** (~8 строк). Aliases (`budget_schemas`, `budget_accounts`, `budget_analytics`, `budget_categories`, `budget_lookups`, `budget_products`, `budget_settings`, `budget_transactions`) уже есть в импортах и используются на callsite'ах — callsites в `app/main.py:897-1258` менять **не нужно**:
  - `from app.budget import schemas as budget_schemas` → `from budget import schemas as budget_schemas`
  - `from app.budget.services import accounts as budget_accounts` → `from budget.services import accounts as budget_accounts`
  - и аналогично для остальных модулей с aliases.
- В `assistant-ui/app/config.py:19-24` удалить `budget_db_path()` (функция уехала).
- **Перенести БД** по протоколу SQLite migration discipline (см. выше) — snippet пользователь запускает сам в elevated shell.
- В `budget/budget/config.py` дефолт `db_path()` теперь — `<package_root>/data/budget.db`, env-override `BUDGET_DB_PATH` остаётся.
- Удалить `assistant-ui/app/budget/` (старый каталог).
- Удалить `assistant-ui/tests/test_budget_balances.py` (переехал).
- Перенести `assistant-ui/scripts/import_budget_xlsx.py` → `budget/scripts/import_budget_xlsx.py`, обновить shebang/импорты, проверить что путь к исходному xlsx передаётся параметром (а не зашит относительно `assistant-ui/`).
- В обоих модулях: `uv run pytest`, `uv run ruff check .`. Зелёное.
- Запустить UI: `uv run assistant-ui serve`. Открыть `/budget`, `/budget/settings`, `/api/settings/budget` — сверить с пред-миграционным состоянием.
- Обновить `assistant-ui/ARCHITECTURE.md`, `AGENTS.md`, `CLAUDE.md`: убрать `app.budget` из таблицы Key modules, упомянуть `budget` как editable path dependency. Добавить ссылку на `budget/ARCHITECTURE.md`.

Jinja-фильтры `num_or_blank`, `num_or_dash` в `app/main.py:113-137` **остаются** в assistant-ui — это презентация.

### Phase 3. Добавить MCP-сервер (daemon)

- Создать `budget/budget/runtime_contract.py` со всеми константами и dataclass (см. checklist выше).
- Создать `budget/budget/singleton.py` — Windows named mutex / lock-file (#8).
- Создать `budget/budget/mcp_app.py::build_mcp_server()` с регистрацией tools v1 (см. таблицу). Healthcheck route.
- Создать `budget/server.py` — entrypoint.
- Создать `budget/tests/test_server_smoke.py` — поднимает live server, mcp.ClientSession через `streamable_http_client`, exercises каждый tool, asserts JSON schema. Тест на «занятый порт» — graceful exit.
- `budget/scripts/run_server.ps1` — NSSM-compatible shim.
- `budget/docs/MCP_TOOLS.md` — контракт каждого tool (input schema, output schema, пример).
- `budget/docs/RUNTIME.md` — port, healthcheck URL, singleton-guard, NSSM-конфиг, backup pending item.
- Smoke: из `budget/` в одном терминале `uv run python server.py`, в другом — `curl http://127.0.0.1:8767/healthz` → 200; из `budget/` `uv run pytest tests/test_server_smoke.py` зелёный.

### Phase 4. Подключить клиентов

- Добавить `BudgetStreamableHttpClient` в `assistant-ui/app/mcp_client.py` (см. секцию «MCP-клиент»). Регистрация в `MCPClients`.
- В `assistant-ui/app/tool_router.py` зарегистрировать budget-tools в отдельный toolset `"budget"`.
- В Claude Code MCP config добавить `budget` сервер: URL `http://127.0.0.1:8767/mcp`.
- В Codex MCP config — то же.
- В PM-MCP вызвать `register_process` для `budget-daemon` (либо одноразовым скриптом, либо вручную).
- Smoke: в Claude Code задать «покажи мой баланс» — проверить что вызов прошёл через `get_balances`, ответ корректный. В Codex — то же.

### Phase 5 (отдельная итерация, не в этом плане)

Write-tools: `create_transaction`, `update_transaction`, `delete_transaction`. Требует:
- Решения по guardrails (dry-run mode, дневной лимит на агентские write'ы, UI-подтверждение).
- Audit-логирование каждой агентской write-операции в conversation store или ai-memory.
- Возможно отдельного scope `budget.write` если когда-нибудь экспонируем через Gateway.

### Phase 6 (опционально, отдельная итерация)

Gateway-экспозиция для ChatGPT. Требует:
- Новый ADR по аналогии с ADR-0001 D-3.
- `gateway/scope_policy.py` `BUDGET_DENIED_TOOLS` + `BUDGET_ALLOWED_TOOLS`.
- Redaction чувствительных полей.
- OAuth scope `budget.read`.

## PM-MCP коллизия (резолюция)

Параллельно с Phase 1-2: переименовать `pm-mcp-server/app/adapters/budget.py` → `finance_stub.py`, обновить импорт + добавить cross-ref в обоих AGENTS.md. Это **отдельный мелкий атомарный commit** в pm-mcp-server, не блокирующий budget extraction.

Верификация переименования:

```powershell
cd pm-mcp-server
# 1. Убедиться что не осталось ссылок на старое имя.
rg "adapters\.budget|adapters/budget\.py|from app\.adapters\.budget" -n
# 2. Lint + tests adapters-уровня.
uv run ruff check .
uv run pytest tests/  # минимум — adapters/registry-related тесты
```

## Критические файлы

В `assistant-ui/`:

- `app/main.py:23-30` — **только 8 импортов в шапке**. Callsites в `app/main.py:897-1258` остаются как есть (используют aliases `budget_*`).
- `app/config.py:19-24` — удалить `budget_db_path()`.
- `app/mcp_client.py` — добавить `BudgetStreamableHttpClient` + регистрация в `MCPClients`.
- `app/tool_router.py` — отдельный toolset `"budget"`.
- `pyproject.toml` — `uv add --editable ../budget`.
- `scripts/import_budget_xlsx.py` — удалить (переехал).
- `tests/test_budget_balances.py` — удалить (переехал).
- `ARCHITECTURE.md`, `AGENTS.md`, `CLAUDE.md` — убрать `app.budget`, упомянуть external dep.

В `pm-mcp-server/`:

- `app/adapters/budget.py` → переименовать в `finance_stub.py`.
- `app/adapters/registry.py` — обновить import + регистрацию адаптера на новое имя.
- `tests/test_obsidian_budget_gmail_adapters.py` — обновить импорты (и при желании переименовать в `test_obsidian_finance_stub_gmail_adapters.py`, но это опционально — внешнее имя файла не критично).
- `AGENTS.md` или `ARCHITECTURE.md` — заметка о distinction между `finance_stub` work-item adapter и новым `budget/` модулем.

В новом `budget/`: создаётся целиком, см. файловое дерево.

## Риски и open questions

1. **Backup-расписание budget.db** — Windows Scheduled Task для `backup_budget_db.ps1` нужно поставить отдельной задачей после Phase 2. В план не включаем как блокер, фиксируем в `RUNTIME.md`.
2. **Port 8767** — на момент проверки свободен; preflight `Test-NetConnection` перед NSSM-регистрацией обязателен.
3. **Importer xlsx-путь** — нужно убедиться, что `import_budget_xlsx.py` принимает путь к xlsx параметром, а не хардкодит относительно `assistant-ui/`. Если хардкодит — параметризовать.
4. **ToolRouter routing keywords** — точная подборка ключевых слов для budget toolset (рус/eng) — решить при имплементации, обкатать на тестовых промтах.
5. **NSSM-регистрация** — Phase 4 опционально может включать `tools/register_services.ps1` обновление под новый daemon, либо это отдельная operational задача.
6. **PM-MCP переименование** — нужно подтвердить с владельцем PM-MCP что переименование `budget.py → finance_stub.py` не сломает внешних callers (probably none — это adapter, вызывается только внутри pm-mcp).

## Верификация

После **Phase 1**:

```powershell
cd budget
uv run ruff check .
uv run pytest tests/  # test_balances + test_internal_imports
```

После **Phase 2** (UI работает на новом пакете):

```powershell
cd budget && uv run pytest tests/ && uv run ruff check .
cd ../assistant-ui && uv run pytest tests/ && uv run ruff check .
cd .. && uv run assistant-ui serve
# Browser smoke:
# - http://127.0.0.1:8000/budget — журнал, аналитика, балансы рендерятся идентично
# - http://127.0.0.1:8000/budget/settings — настройки читаются/пишутся
# - http://127.0.0.1:8000/api/settings/budget — JSON ответ
# - POST /api/budget/transactions через UI — создаётся транзакция, видна в журнале
cd budget && uv run python scripts/import_budget_xlsx.py --reference-only --dry-run
```

После **Phase 3** (MCP-сервер):

```powershell
cd budget && uv run python server.py  # в одном терминале
# в другом:
curl http://127.0.0.1:8767/healthz  # → {"status":"ok",...}
uv run pytest budget/tests/test_server_smoke.py  # MCP клиент проверяет все tools v1
```

После **Phase 4** (агенты):

- Claude Code: «покажи мой баланс на сегодня» — должен вызвать `get_balances`, ответ корректный.
- Codex: «сколько потратил в апреле» — должен вызвать `get_analytics_snapshot`.
- `tool_router` отбирает budget toolset только для бюджетных/финансовых запросов, не подмешивает в `mixed`.

**Полный equivalence check**: журнал `/budget` страницы до и после Phase 2 должен показывать идентичные строки и суммы (включая `account_balance_after`, который материализуется window-функцией — это покрыто `test_balances.py`).

## Process после approval

1. Создать PM-MCP work items по затронутым subsystem paths (K.2 в `AGENTS.md`):
   - `[budget]` — Phase 1, 2 (создание пакета + extraction), Phase 3 (MCP daemon).
   - `[assistant-ui]` — Phase 2 (switch на новый пакет + удаление старого), Phase 4 (client + tool_router).
   - `[pm-mcp-server]` — переименование `budget.py → finance_stub.py`.
   - `[ops]` или отдельный subsystem — обновление внешнего `backup-config.json` для включения `budget.db`.
2. После создания корневой PM-MCP задачи (для `[budget]`) — переименовать план: `replicated-popping-chipmunk.md` → `<id>-replicated-popping-chipmunk.md`, используя её ID.
3. Обновить локальный pointer `C:\Users\Zaxva\.claude\plans\replicated-popping-chipmunk.md` на новый путь.
4. **Git policy** (J.1, J.4 в `AGENTS.md`): код коммитится **напрямую в `main`**, без branches/worktrees/PR. Каждая фаза — атомарный commit, привязанный к соответствующей PM-MCP задаче. Cross-subsystem изменения (Phase 2 — `assistant-ui` + `budget` одновременно) идут одним атомарным коммитом (J.4). Push в remote и создание PR — **только по явному запросу пользователя** (J.2).
