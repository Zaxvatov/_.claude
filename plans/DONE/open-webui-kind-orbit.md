# AI-Assistant exchange surface: Open WebUI + native MCP backends

> Открытие: этот план не только про Open WebUI. Он закрывает тему обменов
> между internal клиентами, backends и gateway одним сшитым описанием —
> MVP про Open WebUI, плюс gated waves для остальных клиентов и форматов.
> Имя файла плана сохраняем (`open-webui-kind-orbit.md`), но содержание
> шире чем H1 предыдущей редакции.

## Context

### Видение системы

AI-Assistant — это набор из трёх ядер, который работает как самостоятельный сервис:

- **`assistant-ui/`** — морда (интерфейс)
- **`ai-memory/`** — данные (память, embeddings, history)
- **`pm-mcp-server/`** — мозг (инструментарий: задачи, цели, workflows, портфолио, бюджет, заметки, БД знаний)

К ядру подключаются **клиенты**:

| Класс | Кто | Доверие | Auth |
|---|---|---|---|
| **Internal** | Локальные на ноутбуке: Claude Desktop, Codex CLI, Open WebUI, Obsidian, … | Высокое | Не нужен (loopback) |
| **External** | В облаке: ChatGPT, Google Calendar, Todoist, … | Низкое | Максимально жёсткий (OAuth 2.0 + PKCE, scopes, Tailscale Funnel, rate-limit, audit, redaction) |

Архитектурный принцип, который пользователь хочет соблюсти: **один унифицированный путь для всех internal клиентов, один для external**, без зоопарка коннекторов к каждому backend.

### Текущее состояние

| Клиент | Класс | Как ходит сейчас |
|---|---|---|
| Claude Desktop / Codex | internal | stdio MCP напрямую к `pm-mcp-server/server.py` и `ai-memory/server.py` через `claude_desktop_config.json` |
| ChatGPT (read connector "AI-assistant (read)") | external | gateway `POST /mcp/read` через Tailscale Funnel, OAuth scopes `memory.read`/`pm.read`, 4 курированных tools |
| ChatGPT (write connector "AI-assistant (write)") | external | gateway `POST /mcp/write`, scopes `memory.propose`/`pm.propose`, 2 курированных tools (через staging-queue 8770) |
| Open WebUI | internal | **никак** — pm-mcp-server не отдаёт native MCP HTTP (только custom REST), у ai-memory есть, но Open WebUI без второго backend бесполезен; одновременно подключение Open WebUI исторически предполагало `mcpo` proxy, что добавляет процесс |
| Obsidian, любой будущий internal REST/MCP клиент | internal | **никак** — pm-mcp-server не MCP-совместим по HTTP |

### Что обнаружено в процессе планирования

**Gateway** ([D:\GitHub\AI-Assistant\gateway](gateway)) уже работает как **«single external ingress for AI-Assistant»** ([gateway/ARCHITECTURE.md](gateway/ARCHITECTURE.md)). Слушает `127.0.0.1:8780`, реализует OAuth 2.0 + PKCE, scope policy, redaction, audit hash-chain, rate-limit, response cap, маршрутизирует MCP JSON-RPC к loopback backends (ai-memory 8765, ai-memory-proposals 8770, pm-mcp-server 8766). Использован двумя ChatGPT-коннекторами через Tailscale Funnel.

**Gateway реализован на `BaseHTTPRequestHandler + ThreadingHTTPServer`** ([gateway/gateway/app.py:14](gateway/gateway/app.py)), не на ASGI (Starlette/FastAPI). Любой план который требует ASGI middleware (CORS, Pydantic body validation, async ToolRegistry) внутри текущего gateway = незаявленная миграция на Starlette с обновлением service runtime, тестов, docs.

**`gateway/scripts/expose-tailscale.ps1` публикует весь target** одной командой `tailscale serve/funnel --bg --yes --https=443 http://127.0.0.1:8780` (строки 26 и 30) — без path-filter. Любой `/internal/*` endpoint на 8780 автоматически становится internet-доступным через Funnel. Отдельный loopback-only порт или отдельный процесс — единственный безопасный способ изолировать internal от external.

**ADR-0001 D-3 + tech-stack-choices пункт 5 («Hybrid trust model»)** уже **определили целевую модель доступа**:
- External → gateway (OAuth/PKCE, scope tree, curated routes, denied destructive tools).
- Internal (Claude Desktop, Codex CLI, Open WebUI, любой будущий локальный клиент) → **прямо к loopback backends на 127.0.0.1**, trust-by-default.

Цитата из ADR-0001: «Threat model: external clients → gateway → loopback core subsystems. Loopback всегда `127.0.0.1`». Цитата из tech-stack-choices #5: «loopback (127.0.0.1) — trust-by-default, без auth; external через gateway — OAuth/PKCE + scope-enforcement + DENIED_TOOLS allowlist».

То есть «правильное место для подключения Open WebUI» **уже зафиксировано** архитектурой: backends напрямую, не через gateway. Делать `/internal/*` в gateway значило бы либо нарушить ADR, либо параллельно поддерживать два пути к одному и тому же tool set, разделённых только trust-level.

**Реальные блокеры подключения Open WebUI к backends прямо сегодня:**

1. **`pm-mcp-server` не отдаёт native MCP streamable-http**. Он поднимает только `FastAPI` с custom REST `POST /mcp/{tool_name}` (см. [pm-mcp-server/app/http_transport.py:130](pm-mcp-server/app/http_transport.py)) — это самописный JSON, не MCP протокол. `FastMCP` объект `mcp` существует в `server.py:78` со всеми ~85 tools, но он стартует только в stdio-режиме для Claude Desktop/Codex (через `server.py` как entrypoint). HTTP server этот FastMCP не выставляет. Из gateway README:91 видно, что gateway сейчас ходит к pm-mcp-server тоже через этот custom REST, а не через MCP JSON-RPC.
2. **У `pm-mcp-server` нет `runtime_contract.py`**, а это зафиксированный pattern в tech-stack-choices #5 («каждый MCP daemon экспортирует runtime_contract.py с EXPECTED_TOOLS»). Из-за этого:
   - `TOOL_NAMES` set в [http_transport.py:21-102](pm-mcp-server/app/http_transport.py) (80 имён) дрейфует относительно реальных `@mcp.tool()` декораторов в `server.py` (85 tools).
   - Нет single source of truth для тестов, gateway client fallback, smoke checks.
   - Это ровно та проблема, против которой брик в tech-stack-choices был введён.
3. **`ai-memory`** уже корректно отдаёт MCP streamable-http на `127.0.0.1:8765/mcp` ([memory/daemon.py:376](ai-memory/memory/daemon.py)) — здесь работа не требуется.
4. **Open WebUI поддержка native MCP** — согласно Open WebUI docs, **native MCP Streamable HTTP поддерживается с v0.6.31+**, тип подключения — «MCP (Streamable HTTP)». Это значит native путь — основной MVP-сценарий, OpenAPI/`mcpo` сохраняется как fallback на случай несовместимости или плохого UX в конкретной установленной версии. Источник: [docs.openwebui.com — MCP support](https://docs.openwebui.com/openapi-servers/mcp/) (URL может перенаправлять; уточнить ссылку при доступе).

### Цель этого плана

Сделать `pm-mcp-server` равноправным MCP backend (как уже сделан `ai-memory`), и подключить Open WebUI напрямую к двум backends по loopback — **в рамках уже зафиксированной архитектуры**, без расширения gateway.

Конкретно:
1. Добавить `pm-mcp-server/app/runtime_contract.py` по pattern tech-stack-choices #5; синхронизировать `TOOL_NAMES` через него.
2. Поднять native MCP streamable-http endpoint в pm-mcp-server рядом с существующим custom REST.
3. Подключить Open WebUI к `pm-mcp-server` + `ai-memory` как native MCP servers; если native MCP в данной версии Open WebUI не работает — использовать `mcpo` как тонкий прокси, без правок backends.
4. Отдельной фазой (migration discipline): мигрировать gateway с custom REST на native MCP к pm-mcp-server, после чего deprecated `POST /mcp/{tool_name}` удалить.

**Что этот план явно НЕ делает (с обоснованием):**

- **Не добавляет `/internal/*` в gateway** — нарушает ADR-0001 («loopback всегда 127.0.0.1, прямой доступ для internal»). Дополнительно: gateway сейчас на BaseHTTPRequestHandler, добавление таких endpoint'ов = неявная ASGI миграция; и `expose-tailscale.ps1` опубликовал бы их наружу через Funnel.
- **Не строит OpenAPI compatibility layer как часть MVP** — **для локального internal MVP, пока native MCP в Open WebUI работает**, OpenAPI слой не нужен. Open WebUI docs одновременно отмечают, что OpenAPI остаётся предпочтительным для многих production deployments (статичные схемы, простой auth, экосистема Swagger) — поэтому формулировка не «OpenAPI бесполезен», а «OpenAPI не входит в MVP-scope для одного локального internal клиента». При появлении второго клиента, для которого native MCP не подходит — активируется Wave 7 (см. ниже).
- **Не мигрирует Claude Desktop / Codex CLI** со stdio на HTTP MCP — они работают; миграция оправдана только когда станет очевидной потребность. Native MCP в pm-mcp-server этого пути не закрывает.

### Инварианты для external (НЕ МЕНЯТЬ)

Существующая схема для внешних клиентов — **первоклассное требование** этого плана: ни один её элемент не должен сломаться или быть обойдён.

1. **Раздельные каналы read и write.** External клиент никогда не получает write-tool через read-канал и наоборот. ChatGPT использует два разных connector'а с двумя разными OAuth scope наборами:
   - `POST /mcp/read` — scopes `memory.read` + `pm.read`, видит только read-tools (`memory_search`, `memory_fetch`, `pm_list_work_items`, `pm_get_work_item`).
   - `POST /mcp/write` — scopes `memory.propose` + `pm.propose`, видит только write-tools (`memory_propose`, `pm_propose_task`).
   - Cross-profile блокируется на двух независимых уровнях: (a) сам endpoint отвергает out-of-profile имя tool с `tool_not_in_profile`; (b) scope policy в [gateway/scope_policy.py](gateway/scope_policy.py) проверяет токен независимо.
2. **Write только через staging «отстойник» + manual approval.** Никакой external клиент не пишет в production state напрямую. Любая запись внешнего клиента — это **proposal**, который попадает в staging-таблицу `memory_proposals` (для AI-memory, daemon на 8770) или эквивалентный paradigm в PM-MCP. Promotion в production возможен **только** ручным `proposals-approve <id>` через локальный CLI ([chatgpt-write-gateway.md:13-15](DONE/chatgpt-write-gateway.md)). Auto-approve вне scope текущей реализации.
3. **OAuth 2.0 + PKCE для external.** Discovery через `/.well-known/oauth-authorization-server`, DCR через `/register`, token exchange через `/token`, refresh tokens хранятся хэшированными в `data/clients.sqlite3`. Никакого Bearer-без-OAuth для external.
4. **Tailscale Funnel** — единственный публикуемый канал наружу. `/healthz` ограничен localhost/Tailnet, CORS allowlist эксплицитный для ChatGPT-доменов.
5. **Audit hash-chain + redaction + response cap 1 MiB** работают на каждом external запросе.
6. **Курированный поверхностный набор для external**: 6 tools, перечисленных в [gateway/README.md:42-50](gateway/README.md). Расширение этого набора — отдельная задача с обязательным security review, не часть этого плана.

Все эти элементы уже реализованы (`DONE-chatgpt-read-public.md`, `DONE-chatgpt-write-gateway.md`, текущий `gateway/`). Этот план **их не трогает**, и тесты в verification явно фиксируют отсутствие регрессий.

### Paradigm для будущих external сервисов

Когда понадобится подключить Google Calendar, Todoist или другой внешний сервис — он идёт по **той же схеме что ChatGPT сегодня**, без новых архитектурных путей:

- регистрируется как новый OAuth client (через DCR `/register`)
- получает токены с подмножеством существующих scope (`memory.read`/`pm.read`/`memory.propose`/`pm.propose`/`pm.action`); новый scope добавляется только если нужна категория действий которой ещё нет
- использует `/mcp/read` или `/mcp/write` (или оба, если выполняет обе функции)
- проходит через тот же middleware stack (rate-limit, redaction, audit, scope policy)
- write попадает в тот же staging «отстойник» с тем же manual approval flow

Gateway **уже готов** принимать новых external клиентов без изменений архитектуры. Этот план это не отменяет.

## Архитектура

```
Internal clients (loopback, no auth)
  ├── Claude Desktop / Codex CLI ──stdio MCP──►   pm-mcp-server (через server.py)
  │                              ──stdio MCP──►   ai-memory     (через ai-memory/server.py)
  │   (как сейчас, не трогаем)
  │
  ├── Open WebUI (Phase 1, MVP)
  │     если native MCP работает:
  │       ──HTTP MCP──►  http://127.0.0.1:8766/mcp-streamable/mcp   (pm-mcp-server, NEW)
  │       ──HTTP MCP──►  http://127.0.0.1:8765/mcp                  (ai-memory, уже работает)
  │     если нужен mcpo fallback:
  │       ──OpenAPI──►  mcpo proxy (127.0.0.1:8788) ──MCP──► оба backends
  │
  └── Obsidian, любой будущий internal клиент → тот же путь по типу клиента

External clients (existing, без изменений)
  ├── ChatGPT "AI-assistant (read)"  ──Tailscale Funnel──► gateway POST /mcp/read
  └── ChatGPT "AI-assistant (write)" ──Tailscale Funnel──► gateway POST /mcp/write
                                                            (OAuth+PKCE, scope tree,
                                                             DENIED_TOOLS, audit-chain,
                                                             curated 6 tools, write→staging)
```

Ключевое отличие от предыдущей редакции плана: **gateway вообще не на пути internal клиентов** — это требование ADR-0001 D-3 и tech-stack-choices #5 «Hybrid trust model». Backends отдают свой native MCP интерфейс сами; Open WebUI ходит туда напрямую как любой другой internal клиент.

### Tool naming

Tools остаются с **оригинальными именами** backends: `search_memory`, `get_recent_memory`, `create_task`, `ping`. Никаких префиксов:

- Open WebUI native MCP клиент сам видит «MCP server #1: ai-memory, MCP server #2: pm-mcp-server» как два отдельных источника. Идентификация по namespace, не по prefix в имени.
- Если используется `mcpo` fallback — он либо группирует tools по серверу, либо принимает per-server конфиги; коллизии разрешаются на стороне mcpo, не на стороне backends.
- Сохраняется bit-compatibility с stdio-вызовами от Claude Desktop / Codex и с прямыми REST-вызовами через legacy `POST /mcp/{tool_name}` в pm-mcp-server.
- В gateway курированные external tools (`memory_search`, `pm_list_work_items`) **уже имеют префиксы** — это явная курация для ChatGPT, отдельный namespace. Internal не должен это копировать.

### Open WebUI: native MCP vs mcpo fallback

План имплементации проходит native MCP-путь первым. Если он не сработает (UX плохой, нестабильность, неподдержка Streamable HTTP в установленной версии Open WebUI) — добавляется `mcpo` как тонкий прокси без правок backends.

Точный синтаксис команды mcpo сверяется с [README mcpo](https://github.com/open-webui/mcpo) в момент имплементации — публичный API проекта меняется чаще, чем стабильны цитаты. Опорная форма — **config-based**: один JSON-файл со списком MCP серверов, mcpo запускается с `--config <file>`. Структура config:

```jsonc
// tools/open-webui-mcpo.config.json
{
  "mcpServers": {
    "ai-memory": {
      "type": "streamable-http",
      "url": "http://127.0.0.1:8765/mcp"
    },
    "pm-mcp-server": {
      "type": "streamable-http",
      "url": "http://127.0.0.1:8766/mcp-streamable/mcp"
    }
  }
}
```

```powershell
uvx mcpo --port 8788 --config D:\GitHub\AI-Assistant\tools\open-webui-mcpo.config.json
```

После старта mcpo **открыть в браузере `http://127.0.0.1:8788/docs`** и зафиксировать **фактические URLs**: при multi-server config mcpo обычно выставляет:
- корневой Swagger overview (`/docs`),
- per-server route и docs (например `/ai-memory/docs`, `/pm-mcp-server/docs`),
- per-server OpenAPI spec (`/ai-memory/openapi.json`, `/pm-mcp-server/openapi.json`).

**Корневой `/openapi.json` для всего merged set может не существовать** — поведение mcpo при multi-server зависит от версии. В Open WebUI добавляется столько tool server entries, сколько фактически генерируется (один на каждый mcpServer), либо один корневой если mcpo его отдаёт. Точные URLs **подтверждаются через `/docs` overview перед настройкой Open WebUI**, не вшиваются в план как догма.

Field name `type` (`streamable-http` / `sse` / `stdio`), верхний ключ `mcpServers` и флаг `--config` — это стандарт MCP server config, унаследованный mcpo от Claude Desktop / VS Code; точные опции (например, есть ли `--server-type` для inline без config) проверяются по `uvx mcpo --help` в момент имплементации, если плану нужен exact form.

Решение «native vs mcpo» принимается **в Verification §3** по реальному поведению Open WebUI, не на этапе планирования.

### Docker Open WebUI

Open WebUI часто запускают в Docker. У этого сценария есть **subtle проблема с loopback bind**:

- **Windows / Mac Docker Desktop**: контейнер доходит до хоста через `host.docker.internal`. Чтобы это **сработало**, backend на хосте должен слушать **не только `127.0.0.1`**, а также interface, видимый из Docker bridge (на Docker Desktop это специальный VM interface, не `0.0.0.0`-эквивалент по умолчанию). Если backend строго на `127.0.0.1:8765` / `127.0.0.1:8766`, Docker контейнер до него не достанет.
- **Linux Docker без `--network=host`**: то же самое — `host.docker.internal` требует bind backend шире loopback.
- **Linux Docker с `--network=host`**: контейнер делит host network namespace; `127.0.0.1` совпадает с хостовым. Этот сценарий совместим с текущим bind.

Решение в MVP:
- **Рекомендованный путь**: **запускать Open WebUI natively, не в Docker** — на Windows есть pip install / executable. Тогда loopback работает «из коробки», hybrid trust model не нарушается.
- **Если Docker неизбежен**: либо `--network=host` (Linux), либо переход на **Wave 8 Internal Bearer как prerequisite** (см. Conditional waves) — bind backends шире loopback требует auth, чтобы не нарушить trust-by-default contract.

Это документируется в `pm-mcp-server/AGENTS.md` и `ai-memory/AGENTS.md` как явное условие. В MVP принимаем «Docker Open WebUI поддерживается **только** с `--network=host` или вместе с Wave 8».

## Фазы

Реализация разбита на фазы с явной migration discipline. Каждая фаза — отдельный work item в PM-MCP (см. раздел «PM-MCP» ниже), с зависимостями через `link_task_dependency`.

### Phase 1 — pm-mcp-server: runtime_contract + native MCP HTTP

Цель: устранить TOOL_NAMES drift и сделать pm-mcp-server полноценным native MCP backend на том же port (8766), сохранив legacy custom REST для backwards-compat.

**Новый файл**: [`pm-mcp-server/app/runtime_contract.py`](pm-mcp-server/app/runtime_contract.py)

По pattern [`ai-memory/memory/runtime_contract.py`](ai-memory/memory/runtime_contract.py):
- Константы: `PM_MCP_HOST = "127.0.0.1"`, `PM_MCP_PORT = 8766`, `PM_MCP_PATH = "/mcp"`, `PM_MCP_HEALTH_PATH = "/health"`, `PM_MCP_STREAMABLE_PATH = "/mcp-streamable"` (см. ниже).
- `EXPECTED_TOOLS: tuple[str, ...]` — единый источник имён, вычисляется **из `server.mcp._tool_manager.list_tools()`** на import-time (выполняется один раз, кэшируется). Это устраняет hardcoded дублирование.
- Frozen dataclass `RuntimeContract`, helpers `get_streamable_mcp_url()`, `get_healthcheck_url()`, `get_legacy_rest_url(tool_name)`.

**Изменения**: [`pm-mcp-server/app/http_transport.py`](pm-mcp-server/app/http_transport.py)
- Удалить hardcoded `TOOL_NAMES` set (строки 21-102). Заменить на `from app.runtime_contract import EXPECTED_TOOLS as TOOL_NAMES`.
- Добавить mount native MCP streamable-http:
  ```python
  from app.runtime_contract import PM_MCP_STREAMABLE_PATH
  app.mount(PM_MCP_STREAMABLE_PATH, server.mcp.streamable_http_app())
  ```
  Path `/mcp-streamable` намеренно отличается от существующего `POST /mcp/{tool_name}` — два endpoint'а сосуществуют в одном FastAPI app без конфликтов.
- Auth middleware: освободить `/mcp-streamable/*` от обязательного Bearer (как `/health`), поскольку bind у pm-mcp-server — `127.0.0.1` и tech-stack-choices #5 явно предписывает «loopback bind, без auth для local». Существующий env `PM_MCP_HTTP_TOKEN` остаётся для legacy `POST /mcp/{tool_name}` — не ломаем поведение.
- Уточнение по FastMCP: `streamable_http_app()` возвращает Starlette ASGI app, у которого свой внутренний `/mcp` префикс через `streamable_http_path` (default `/mcp`). Финальный URL для клиентов будет `http://127.0.0.1:8766/mcp-streamable/mcp` — это документируется в README и в runtime_contract.

**Новые тесты**: [`pm-mcp-server/tests/test_streamable_mcp.py`](pm-mcp-server/tests/test_streamable_mcp.py)
- `test_initialize_responds` — JSON-RPC `initialize` возвращает корректный protocolVersion и server info.
- `test_tools_list_matches_expected` — `tools/list` возвращает ровно `EXPECTED_TOOLS` (защита от дрейфа).
- `test_tools_call_ping` — `tools/call` для `ping` возвращает `"pong"`.
- `test_legacy_rest_still_works` — `POST /mcp/ping` (legacy) тоже отвечает.
- `test_legacy_tool_names_equals_expected` — синхронизированы списки legacy `TOOL_NAMES` и `EXPECTED_TOOLS` (на случай регресса дублирования).

**Update**: [`pm-mcp-server/README.md`](pm-mcp-server/README.md), [`pm-mcp-server/ARCHITECTURE.md`](pm-mcp-server/ARCHITECTURE.md), [`pm-mcp-server/AGENTS.md`](pm-mcp-server/AGENTS.md) — добавить раздел «Two HTTP surfaces: native MCP streamable-http (canonical) and legacy custom REST (deprecated, see Phase 4)».

### Phase 2 — Open WebUI подключение (MVP — выбор native MCP vs mcpo)

Цель: получить рабочую интеграцию Open WebUI с двумя backends. Решение по способу принимается **по результату теста**, не на бумаге.

1. **Native MCP попытка**. В Open WebUI → Settings → Tools → Add tool server:
   - Server #1: type=MCP (Streamable HTTP), URL=`http://127.0.0.1:8765/mcp` (ai-memory), label=`AI-memory`.
   - Server #2: type=MCP (Streamable HTTP), URL=`http://127.0.0.1:8766/mcp-streamable/mcp` (pm-mcp-server), label=`PM-MCP-server`.
   - Auth: None.
   - Сохранить, в каталоге tools видим merged список 85 + 16 = 101 tools (без префиксов, разделены по серверу в UI).
2. **Acceptance criterion — Function Name Filter List (tool allowlist)**. После того как tools видны, **обязательно** настроить Open WebUI Function Name Filter для каждого сервера. Это не security boundary (это `DENIED_TOOLS` в gateway для external), а **safety gate**: предотвращает случайный вызов LLM destructive / admin tools. Default allowlist для MVP:
   - **AI-memory** (server): `search_memory`, `get_recent_memory`, `search_by_workitem`, `search_by_metadata`, `preview_summary`, `propose_memory`. **Скрыть из default**: `store_memory`, `store_memory_batch`, `store_summary` (это **direct production writes** — для LLM-интерфейса безопаснее путь через `propose_memory` со staging+review; разрешать direct write только отдельным explicit решением пользователя с фиксацией `kind=decision`); все `get_*_workflow_*`, `get_batch_run_history`, `get_portfolio_run_history` (операционные данные, для них есть Assistant-UI); `approve_memory_proposal`, `reject_memory_proposal`, `list_memory_proposals` (proposals админ — через CLI/Assistant-UI).
   - **PM-MCP-server** (server): `list_projects`, `read_tasks`, `read_tasks_range`, `create_task`, `update_task`, `get_next_task`, `pick_next_task`, `get_project_summary`, `project_brief`, `get_project_context`, `recommend_next_project_task`, `recommend_next_global`, `list_work_items`, `get_work_item`, `morning_brief`, `weekly_review_report`, `ping`. **Скрыть**: `set_process_state`, `register_process`, `rename_process_key`, `unregister_process`, `get_process_state_audit`, `bulk_update_tasks`, `move_task`, `complete_task`, `close_task`, `audit_query`, `audit_export`, `run_workflow`, `resume_workflow`, `replan_workflow`, `recover_workflow`, `task_queue`, `batch_run`, все `*_batch_workflow_*`, все `*_portfolio_workflow_*`, `portfolio_run`, `approve_portfolio_workflow_transition`, calendar/todoist mutating tools, `archive_goal`, `create_goal`, `update_goal`, `set_user_constraints`.
   - Эти списки фиксируются в `pm-mcp-server/AGENTS.md` / `ai-memory/AGENTS.md` как «recommended Open WebUI allowlist». Открытый список может расти — но осознанно, не по умолчанию.
   - **Альтернатива**: явно принять «full surface tools доступна модели» с фиксацией в AI-memory `kind=decision` и обоснованием. Default — закрытый список выше.
3. **Smoke chat**. В чате с локальной Ollama: «Создай задачу 'test integration' в проекте `D:/GitHub/AI-Assistant/pm-mcp-server`» → Open WebUI вызывает `create_task`. «Покажи недавнюю memory» → `get_recent_memory`. «Удали процесс ai-memory-daemon» (попытка вызвать скрытый `set_process_state`) — Open WebUI **не должен** дать LLM это сделать, потому что tool вне allowlist.
4. **Если native MCP работает** → Phase 2 завершена, mcpo не нужен. Документируем версию Open WebUI и подключение в AI-memory `kind=fact, project=AI-Assistant`.
5. **Если native MCP не работает** (UX плохой, нет tool discovery, неполный output schema rendering, ошибки JSON-RPC, нестабильно) — **fallback smoke через mcpo**:
   - Создать `tools/open-webui-mcpo.config.json` по шаблону из раздела «Open WebUI: native MCP vs mcpo fallback» (config-based подход, корректная для mcpo схема `{"mcpServers": {...}}` с `"type": "streamable-http"`).
   - Запустить foreground: `uvx mcpo --port 8788 --config D:\GitHub\AI-Assistant\tools\open-webui-mcpo.config.json`. Если синтаксис флага отличается в установленной версии — сверить по `uvx mcpo --help`.
   - Открыть `http://127.0.0.1:8788/docs`, посмотреть фактические URLs (см. примечание про multi-server routes в разделе «Open WebUI: native MCP vs mcpo fallback»). В Open WebUI: удалить MCP-сервера, добавить **каждый** обнаруженный per-server OpenAPI endpoint (например `http://127.0.0.1:8788/ai-memory/openapi.json` и `http://127.0.0.1:8788/pm-mcp-server/openapi.json`), либо один корневой `/openapi.json` если он есть. Tool allowlist (шаг 2) переносится либо в Open WebUI Function Name Filter, либо в mcpo конфиг.
   - Повторить smoke chat (шаг 3).
   - Зафиксировать в AI-memory `kind=fact`: «native MCP в Open WebUI vN.N.N не подошёл по причине X, MVP использует mcpo proxy на 8788». **Productionization mcpo через NSSM** — Wave 6 этого плана, не MVP.

Решение «native vs mcpo» = факт-запись в AI-memory. Дальнейшие планы могут на неё опираться.

### Phase 3 — Verification + ChatGPT regression

Цель: убедиться что external контур (ChatGPT через gateway) **не сломался** при добавлении новых HTTP endpoint'ов в pm-mcp-server. См. раздел `Verification` ниже.

### Phase 4 — Migration discipline: gateway → native MCP к pm-mcp-server

Цель (отдельная фаза): мигрировать gateway с custom REST к pm-mcp-server на native MCP, после чего удалить legacy `POST /mcp/{tool_name}` целиком. Это **обязательная** часть, не «future work» — иначе остаётся неподдерживаемый дублирующий путь.

Шаги:
1. В `gateway/gateway/backends.py` (на основании AI-memory id 1371 — там уже есть `MCPStreamableHttpClient` через httpx.AsyncClient): добавить вариант с URL `http://127.0.0.1:8766/mcp-streamable/mcp` для PM-MCP вместо текущего custom REST. Возможно env-flag `AI_ASSISTANT_GATEWAY_PM_MCP_NATIVE=true` для постепенного перехода.
2. Регрессионные тесты gateway: `pm_list_work_items`, `pm_get_work_item`, `pm_propose_task` идут через native MCP с тем же результатом что и раньше.
3. После стабилизации (минимум 1 неделя в проде):
   - **Удалить** `POST /mcp/{tool_name}` route в `pm-mcp-server/app/http_transport.py` и весь WRITE_TOOL_NAMES audit-wrapper в этой ветке (audit перенести в FastMCP middleware если нужен).
   - **Удалить** `from app.runtime_contract import EXPECTED_TOOLS as TOOL_NAMES` alias — TOOL_NAMES больше не нужно.
   - **Удалить** env `PM_MCP_HTTP_TOKEN` если он был только для custom REST.
   - Documentation sweep: `grep -r "TOOL_NAMES\|/mcp/{tool_name}" pm-mcp-server/` пустым.
   - AI-memory `kind=change` запись «pm-mcp-server: legacy custom REST endpoint удалён, остался только native MCP streamable-http».
4. По migration-discipline skill: end state выглядит так, как будто custom REST никогда не существовал.

### Не трогаем

- **`gateway/`** — никаких изменений в этом плане кроме Phase 4 backend rewiring к native MCP. Никакого `/internal/*`, никакой ASGI миграции, никакой публикации tools каталога наружу.
- **`ai-memory/`** — целиком. Daemon на 8765 уже MCP streamable-http, runtime_contract есть, EXPECTED_TOOLS работает.
- **External клиенты** — ChatGPT через gateway `/mcp/read` и `/mcp/write`, OAuth, scope policy, audit-chain, write→staging+manual approval — **ни один элемент не меняется** (см. «Инварианты для external» выше).
- **Claude Desktop / Codex stdio** — продолжают подключаться через `python server.py` к каждому backend. Никаких config-правок.

## PM-MCP

После утверждения этого плана и до начала кода — завести work items в Assistant-UI/PM-MCP по AGENTS.md K.1. Имя файла плана **не переименовывается** (соответствует текущему central-plan-workflow); все id ниже фиксируются в этом файле строкой `PM-MCP: #<id>` после `create_task`.

### MVP work items (создать сразу)

- **PM-MCP: #752** Phase 1 — pm-mcp-server `runtime_contract.py` + native MCP HTTP mount + tests, проект `pm-mcp-server`.
- **PM-MCP: #753** Phase 2 — Open WebUI native MCP smoke + tool allowlist + (если нужно) mcpo fallback, проект `AI-Assistant` (cross-cutting). Depends on #752.
- **PM-MCP: #754** Phase 3 — Verification + ChatGPT regression + black-box external probe, проект `gateway`. Depends on #753.
- **PM-MCP: #755** Phase 4a — rewire PM-MCP backend в gateway с custom REST на native MCP (env-flag, тесты), проект `gateway`. Depends on #754.
- **PM-MCP: #756** Phase 4b — удалить legacy `POST /mcp/{tool_name}` в pm-mcp-server + documentation sweep, проект `pm-mcp-server`. Depends on #755.

Цепочка `link_task_dependency`: 752 → 753 → 754 → 755 → 756 заведена через cross-project зависимости. Phase 4a (#755) явно гейтится «Phase 2 (#753) завершена и стабильна минимум неделю» — это правило соблюдается на уровне approval статуса, не автоматически.

### Conditional waves (создать только при срабатывании триггера)

Эти задачи **не создаются заранее**, чтобы не засорять backlog. Каждая создаётся только когда соответствующий триггер сработал; на момент создания добавляется зависимость от MVP-задач.

- `PM-MCP: #<id-5>` Wave 5 (Claude/Codex HTTP MCP migration) — триггер: пользователь решил мигрировать клиентов. Cross-subsystem: одна задача в `AI-Assistant` (root) + по одной в `claude-desktop-config` (если есть отдельный проект) или просто заметка в AGENTS.md.
- `PM-MCP: #<id-6>` Wave 6 (mcpo NSSM) — триггер: Phase 2 завершилась mcpo путём. Проект `AI-Assistant` (cross-cutting: `tools/`).
- `PM-MCP: #<id-7>` Wave 7 (OpenAPI first-class) — триггер: mcpo недостаточен И ≥2 разных клиентов подтверждены. Cross-subsystem: `pm-mcp-server` + `ai-memory` (две связанные задачи).
- `PM-MCP: #<id-8>` Wave 8 (Internal Bearer) — триггер: Docker сценарий, multi-user или non-loopback bind. Cross-subsystem: `pm-mcp-server` + `ai-memory` + клиентские конфиги.

Принцип: **cross-subsystem работа = отдельные задачи по `project_path`**, не одна общая. Связь через `link_task_dependency` сохраняет видимость в обоих проектах и не размывает audit одной подсистемы.

## Verification

### 1. Phase 1 — pm-mcp-server: runtime_contract + native MCP

**Основной smoke — через MCP `ClientSession`** (учитывает полный lifecycle: `initialize` → `notifications/initialized` → `tools/list`/`tools/call`, корректный `Mcp-Session-Id` header). Curl ниже — только грубый transport probe: он **не** воспроизводит полный MCP lifecycle и может давать false-positive на отдельных вызовах.

```powershell
cd D:\GitHub\AI-Assistant\pm-mcp-server
uv sync --dev
uv run ruff check .
uv run pytest tests/test_streamable_mcp.py -v
# 5 тестов проходят (initialize, tools/list, tools/call ping, legacy still works, TOOL_NAMES==EXPECTED_TOOLS)
```

Тесты используют `mcp.client.streamable_http` + `ClientSession` (образец — [ai-memory/memory/daemon.py:498](ai-memory/memory/daemon.py)).

**Грубый transport probe** (опционально, для быстрой sanity-check, не замена тестов):

```powershell
uv run python -m app.http_server                        # background, 8766

# Health (legacy, не MCP):
curl http://127.0.0.1:8766/health                       # → {"ok": true}

# Legacy custom REST (всё ещё работает):
curl -X POST http://127.0.0.1:8766/mcp/ping `
     -H "Authorization: Bearer $env:PM_MCP_HTTP_TOKEN" `
     -H "Content-Type: application/json" -d '{"arguments": {}}'
# → "pong"

# Native MCP — только проверка, что endpoint отвечает 200 на initialize.
# Полный lifecycle (notifications/initialized после, sticky session id) — в pytest выше.
curl -i -X POST http://127.0.0.1:8766/mcp-streamable/mcp `
     -H "Accept: application/json, text/event-stream" `
     -H "Content-Type: application/json" `
     -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"smoke","version":"1.0"}}}'
# → 200 + JSON-RPC ответ + header Mcp-Session-Id
```

### 2. ai-memory не сломан

**Основной smoke — через `ClientSession`** (полный MCP lifecycle с initialize → notifications/initialized → tools/list, sticky `Mcp-Session-Id`):

```powershell
cd D:\GitHub\AI-Assistant\ai-memory
uv run pytest tests/test_daemon.py::test_ping_daemon -v
# или эквивалентный smoke который вызывает ping_daemon_sync() / streamable_http_client
```

Существующий `ping_daemon_sync` ([memory/daemon.py:522](ai-memory/memory/daemon.py)) уже делает корректный full-lifecycle smoke — переиспользовать его.

**Грубый transport probe** (опционально, без MCP lifecycle — false-positives возможны):

```powershell
curl http://127.0.0.1:8765/healthz                          # liveness
curl -i -X POST http://127.0.0.1:8765/mcp `
     -H "Accept: application/json, text/event-stream" `
     -H "Content-Type: application/json" `
     -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"smoke","version":"1.0"}}}'
# → 200 + JSON-RPC ответ + Mcp-Session-Id. Полную проверку tools/list даёт pytest выше.
```

### 3. Phase 2 — Open WebUI native MCP попытка

В Open WebUI:
- Settings → Tools → Add tool server → MCP → `http://127.0.0.1:8765/mcp`, label `AI-memory`, auth None.
- Settings → Tools → Add tool server → MCP → `http://127.0.0.1:8766/mcp-streamable/mcp`, label `PM-MCP-server`, auth None.
- Save.

В каталоге tools должны появиться оба сервера с раздельными namespace. Открыть один из tools (например `create_task`) — должна быть видна полная input schema из FastMCP.

Smoke в чате с локальной Ollama (например `qwen2.5:7b-instruct`):
- «Создай задачу 'test integration' в проекте `D:/GitHub/AI-Assistant/pm-mcp-server`, статус Бэклог» — модель должна выбрать `create_task` от `PM-MCP-server`.
- Проверить в Assistant-UI: задача появилась.
- «Покажи мои недавние memory entries» — модель выбирает `get_recent_memory` от `AI-memory`, возвращает реальные items.

**Если native MCP работает** — фиксируем в AI-memory `kind=fact, project=AI-Assistant`: «Open WebUI vN.N.N успешно подключается к ai-memory (port 8765) и pm-mcp-server (port 8766/mcp-streamable/mcp) как native MCP streamable-http servers, mcpo не требуется». Phase 2 завершена.

**Если native MCP не работает / UX неприемлемый** — переход на mcpo:

```powershell
# uvx запустит mcpo во временной venv:
uvx mcpo --port 8788 --config tools/open-webui-mcpo.config.json
```

Открыть `http://127.0.0.1:8788/docs`, посмотреть фактические URLs (см. раздел «Open WebUI: native MCP vs mcpo fallback» — multi-server mcpo обычно даёт per-server routes `/ai-memory/openapi.json`, `/pm-mcp-server/openapi.json` с собственными `/docs`; корневой `/openapi.json` может отсутствовать в зависимости от версии). В Open WebUI удалить MCP-сервера, добавить **каждый** обнаруженный per-server OpenAPI endpoint как отдельный tool server, либо один корневой если он реально есть. Повторить smoke chat. Зафиксировать в AI-memory: «native MCP в Open WebUI vN.N.N не подошёл по причине X, используем mcpo proxy на 8788, фактические endpoint'ы такие-то».

### 4. Phase 3 — External regressions (ChatGPT через gateway НЕ сломался)

Phase 1 добавляет mount в pm-mcp-server, Phase 2 ничего не трогает в backends. Но регрессию проверяем отдельно:

- **ChatGPT (read connector)**: тестовый чат «найди в памяти упоминания gateway» → `memory_search` через `/mcp/read` → 200, items возвращаются. Попытка вызвать write-tool (`memory_propose`) с read-токеном — отклонена с `tool_not_in_profile` (на уровне endpoint) и `invalid_scope` (на уровне scope policy).
- **ChatGPT (write connector)**: «запомни что мне нравятся MM роли» → `memory_propose` через `/mcp/write` → 200, `proposal_id` возвращён. Запись **не появляется** в `search_memory` ни через ChatGPT (через read), ни локально через `memory.cli search`. Только после ручного `memory.cli proposals-approve <id>` запись становится видимой.
- **OAuth flow**: `GET https://<funnel-host>/.well-known/oauth-authorization-server` и `/.well-known/oauth-protected-resource` отдают валидные metadata. DCR через `POST /register` создаёт client. PKCE `S256` обязателен.
- **Tailscale Funnel и pm-mcp-server**: `tailscale serve status` и `tailscale funnel status` показывают только gateway (8780). pm-mcp-server (8766) **не должен появляться** в публикуемых targets. Это инвариант — pm-mcp-server **остаётся loopback-only daemon**, даже с новым `/mcp-streamable` mount.
- **Black-box external probe**: с другого хоста в tailnet (или эмулируя извне через `curl --resolve` к публичному Funnel-домену) сделать `GET https://<funnel-host>/mcp-streamable/mcp` → должна быть 404 от gateway (этого пути в gateway нет; gateway не проксирует наружу новые pm-mcp-server endpoint'ы). Это ключевое подтверждение, что добавление native MCP в pm-mcp-server **не утекает** через Funnel.
- **Audit chain**: каждый external запрос пишет audit row в `data/<gateway-audit>.jsonl` с `previous_hash` и `row_hash`; цепочка верифицируется существующим тестом gateway.
- **Claude Desktop / Codex через stdio**: вызов `mcp__PM-MCP-server__ping` и `mcp__AI-memory__get_recent_memory` в Claude Code или Claude Desktop — работает (stdio backends не трогали).

### 5. Phase 4 — Migration verification (после rewiring gateway)

После того как gateway переключен на native MCP к pm-mcp-server и legacy custom REST удалён:

- Gateway тесты `pm_list_work_items`, `pm_get_work_item`, `pm_propose_task` — зелёные.
- `grep -r "TOOL_NAMES" pm-mcp-server/` — пусто (или только в архивных docs).
- `curl http://127.0.0.1:8766/mcp/ping` (старый legacy path) — 404, потому что route удалён. Это **подтверждение завершённой миграции**, не регресс.
- ChatGPT smoke chat — `pm_list_work_items` через `/mcp/read` всё ещё работает, теперь через native MCP под капотом.

## Risks

- **FastMCP `streamable_http_app()` mount path** — внутри FastMCP добавляется свой `streamable_http_path` (default `/mcp`). Финальный URL для клиентов = `http://127.0.0.1:8766/mcp-streamable/mcp` (двойной `/mcp`). Это неудобно, но не блокер. Проверить при имплементации; если двойной префикс ломает Open WebUI клиента — переименовать mount в `/native` или сконфигурировать FastMCP с `streamable_http_path="/"`. Поправка чисто в runtime_contract.
- **Open WebUI session handling в MCP клиенте**. MCP Streamable HTTP использует `Mcp-Session-Id` header (см. `gateway/gateway/app.py:40`). Если Open WebUI не управляет сессией корректно (например, не передаёт session id после initialize) — будут странные ошибки. mcpo fallback это нивелирует. Документируется в результате Phase 2.
- **`tools/list` от pm-mcp-server большой** (~85 tools × средний input_schema). Это нагрузка на cold-start initialize для Open WebUI. Если ответ медленный — добавить кэширование на стороне FastMCP (есть из коробки).
- **Docker Open WebUI и loopback bind** — backends в MVP биндятся строго на `127.0.0.1`. Из Docker-контейнера это **не доступно через `host.docker.internal`** на Windows/Mac (трафик приходит с IP контейнера/Docker-bridge, который не loopback для backend); на Linux работает только с `--network=host`. Поддерживаемые сценарии: (a) **Open WebUI native, не в Docker** — рекомендованный путь; (b) **Linux Docker с `--network=host`** — работает с текущим bind; (c) **Docker с `host.docker.internal`** — **только вместе с Wave 8 Internal Bearer + non-loopback bind**, иначе нарушает hybrid trust model. См. раздел «Docker Open WebUI» в Архитектуре для деталей.
- **MCP `streamable_http_client` в gateway** — уже используется в gateway (AI-memory id 1371: `MCPStreamableHttpClient updated for current MCP SDK by passing timeout through httpx.AsyncClient`). Phase 4 переиспользует существующий клиент, не вводит новых зависимостей.
- **Drift между TOOL_NAMES и реальными tools** — основная защита это `test_legacy_tool_names_equals_expected` в Phase 1 tests + EXPECTED_TOOLS из `mcp._tool_manager.list_tools()` как single source. Если кто-то регрессирует — тест ломается.

## Conditional waves in this exchange plan

Этот план называется «exchange between clients ↔ backends ↔ gateway», а не «Open WebUI integration» — потому что все условные продолжения ниже относятся к той же поверхности обменов. Они **остаются в этом плане** как gated waves: каждая активируется по конкретному триггеру, не блокируют MVP, но фиксируются здесь, чтобы тема обменов не разваливалась на хвосты.

### Wave 5 — Claude Desktop / Codex CLI HTTP MCP migration

**Триггер**: либо очевидный выигрыш в latency / debugging UX (по факту наблюдения), либо желание унифицировать конфиги между Claude и Codex. Не блокирует MVP — stdio работает.

Содержание:
- **Заранее проверить поддержку каждого клиента отдельно**: Claude Desktop поддерживает MCP Streamable HTTP через `"type": "http"` / URL в `claude_desktop_config.json` (заявлено в `docs.anthropic.com`); Codex CLI — TBD по установленной версии, проверить через `codex config --show` или эквивалент.
- Переключение клиентских конфигов на `http://127.0.0.1:8765/mcp` (ai-memory) и `http://127.0.0.1:8766/mcp-streamable/mcp` (pm-mcp-server).
- Никаких изменений в backends — pure client config-change.
- Smoke: вызов одного tool из Claude Desktop и Codex после переключения, сравнение latency со stdio.
- Фиксация в AI-memory `kind=change`.

### Wave 6 — mcpo productionization через NSSM

**Триггер**: Phase 2 завершилась mcpo путём (зафиксировано в AI-memory `kind=fact`). Если native MCP сработал — wave неактивна.

Содержание:
- `tools/install_mcpo_service.ps1` — NSSM регистрация mcpo как Windows service по pattern tech-stack-choices #1. Port 8788, working dir, log path, env vars.
- mcpo config — JSON файл со списком backends, см. Phase 2 пример.
- Single-instance guard (tech-stack-choices #8).
- Документация в корневом `tools/README.md`.

### Wave 7 — OpenAPI first-class artifact в backends

**Триггер**: оба условия должны быть выполнены:
- mcpo недостаточен по конкретной причине (нужны custom auth, route shaping, per-client allowlists, или иной control which mcpo не даёт).
- Подтверждено что REST/OpenAPI нужен ≥2 разным клиентам (Obsidian REST plugin, custom скрипты, Make/Zapier-style автоматизации, ChatGPT Custom GPT Actions, и т.п.) — не просто гипотетически.

Содержание:
- В pm-mcp-server (FastAPI уже есть): динамическая генерация OpenAPI 3.1 из `EXPECTED_TOOLS` + input/output schemas, routes `POST /tools/{name}`.
- В ai-memory: через FastMCP `custom_route`, ручная сборка OpenAPI спека (по pattern из исходного плана редакции 1).
- CORS allowlist (не `*` — конкретные origins для Open WebUI и других клиентов).
- Tool allowlist через `?profile=read|core-write|admin` query param или header.

### Wave 8 — Internal Bearer (security wave)

**Триггер**: любое из трёх:
- **Docker Open WebUI требует bind backends шире чем `127.0.0.1`** (например, на `0.0.0.0` или на специфичный interface, чтобы контейнер достал хост). В этом случае Internal Bearer **перестаёт быть future-work и становится prerequisite Docker-сценария** — без него bind за пределы loopback нарушает hybrid trust model. Если пользователь принимает решение «Open WebUI запускаем natively на Windows, не в Docker» — wave остаётся опциональной.
- Многопользовательский runtime на одной машине.
- Bind на Tailscale interface для cross-device доступа в tailnet.

Содержание:
- Env `PM_MCP_INTERNAL_BEARER`, `AI_MEMORY_INTERNAL_BEARER` (пусто = no auth, default; задан = Bearer required).
- Auth middleware для `/mcp-streamable/*` в pm-mcp-server и для FastMCP custom_route в ai-memory.
- Keyring-storage для token (по pattern ai-memory `memory/secrets.py`).
- CLI `<service>-internal-token-set/rotate/show-fingerprint`.
- Обновление документации и client setup (Open WebUI Bearer header, Claude/Codex config).

## Strategic out of scope (не в этом плане, требуют отдельных планов)

Тематически за пределами exchange surface — требуют отдельного планирования.

### Объединение БД задач и AI-memory — **решено НЕ делать**

Пользователь упоминал идею перенести задачи из `pm-mcp-server` в `ai-memory` («задачи тоже память о работе»). После обсуждения **решение зафиксировано**: **не объединять физически**. Причины:

- `ai-memory` — **долговременная смысловая память**: решения, факты, выводы, контекст задач, важные изменения. Отвечает на вопрос «что нужно помнить дальше?».
- `pm-mcp-server` — **control plane / operational source of truth**: задачи, статусы, зависимости, active task guard, workflow runs, audit, process state. Отвечает на «что делать сейчас, в каком состоянии работа, что заблокировано?».
- Перенос задач в ai-memory сделает её ответственной за транзакционные статусы, зависимости, workflow audit, блокировки, очереди и правила выполнения — это размоет назначение.
- Цена объединения: миграция schemas, смена source-of-truth, риски для task guard и workflow, новые retention/privacy вопросы.
- Выигрыш сейчас в основном эстетический, не функциональный.

Объединение оправдано, **если когда-нибудь** одновременно: дублирование данных стало источником багов; нужны атомарные транзакции «обновить задачу + записать память»; почти все запросы требуют сложных join; появляется единая доменная модель «work graph». Сейчас этого нет.

### Семантическая связка PM-MCP tasks ↔ AI-memory entries — **draft для отдельного плана**

Вместо физического объединения — усилить семантические связи между двумя источниками, оставив их физически разделёнными. Это **более полезная доработка**, но **тематически отдельная** от exchange surface — заслуживает собственного плана и отдельной сессии планирования.

Outline (для будущего плана):

- **AI-memory**: в metadata всех `task_context`/`decision`/`change` записях, связанных с задачей, обязательно хранить `task: "#<id>"` или `tasks: ["#<id1>", ...]`. Валидация на write.
- **PM-MCP-server**: при отображении задачи (`get_work_item`, `project_brief`) показывать связанные memory entries из AI-memory по metadata.task — отдельный composite tool `get_task_memory_context`.
- **AI-memory tools**: `search_related_memory_for_task(task_ref)` — обёртка над `search_by_metadata(field="task", value=task_ref)`.
- **Assistant-UI**: на странице задачи — sidebar с её memory entries (decisions, notes, recent changes); на странице memory entry — ссылка обратно к связанной задаче.
- **Чёткие конвенции** в AGENTS.md обоих subsystems: что писать в metadata, когда создавать memory entry от задачи (decision при architectural choice; change после landed PR; note при insight).
- Никакой миграции схем, никаких физических join'ов — только конвенции + read-side tools.

Этот outline сохраняется здесь только для трассировки. Создание полноценного плана — отдельная сессия с central-plan-workflow.

## Tech-stack-choices: возможные новые кирпичики (предложить после approval)

После утверждения этого плана и Phase 1 — следующие кирпичики достойны фиксации в `D:\GitHub\_engineering_rules\tech-stack-choices.md` (только с подтверждением пользователя):

- **«FastMCP daemon без auth для loopback + opt-in Bearer для external через gateway»** — расширение пункта 5: явно зафиксировать паттерн «двойной HTTP surface» (native MCP + legacy custom REST с deprecation path) для daemons, которые исторически имели свой REST.
- **«Tailscale `serve/funnel` публикует target целиком — изолировать internal endpoint'ы отдельным портом или отдельным процессом»** — отдельный пункт безопасности. Сейчас это знание неявное.
- **«MCP discovery через `mcp._tool_manager.list_tools()` как single source for runtime_contract.EXPECTED_TOOLS»** — формализация компиляции EXPECTED_TOOLS, чтобы не было нового drift.

Каждый предлагается отдельным сообщением пользователю с обоснованием — не правится в `tech-stack-choices.md` инициативно.
