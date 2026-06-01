# Open WebUI / Claude / Codex → gateway internal profile (MCP + OpenAPI)

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
| Open WebUI | internal | **никак** — он не умеет MCP, только OpenAPI |
| Obsidian, любой будущий internal клиент | internal | **никак** — нет единой точки |

### Что обнаружено в процессе планирования

`gateway/` ([D:\GitHub\AI-Assistant\gateway](gateway)) уже работает как **«single external ingress for AI-Assistant»** ([gateway/ARCHITECTURE.md](gateway/ARCHITECTURE.md)). Слушает `127.0.0.1:8780`, реализует OAuth 2.0 + PKCE, scope policy, redaction, audit hash-chain, rate-limit, response cap, маршрутизирует MCP JSON-RPC к loopback backends (ai-memory 8765, ai-memory-proposals 8770, pm-mcp-server 8766). Использован двумя ChatGPT-коннекторами через Tailscale Funnel.

Но gateway **сейчас exposes только 6 курированных tools для external** (`memory_search`/`fetch`/`propose`, `pm_list_work_items`/`get_work_item`/`propose_task`), и **internal-профиля у него вообще нет**. Поэтому ни Open WebUI, ни какой-либо другой локальный клиент не имеет «правильной» точки входа.

### Проблема исходного импульса пользователя

Изначальное предложение — «добавить OpenAPI compatibility layer в pm-mcp-server» — нарушает архитектурный инвариант `gateway = single ingress`. Это создавало бы **второй параллельный путь** к backends в обход gateway audit/scope/redaction. Технически работало бы, но идёт против уже задизайненной концепции.

### Цель этого плана

Расширить `gateway/` **internal-профилем**, который даёт всем локальным клиентам единую точку входа с обоими протоколами:

- **MCP клиенты** (Claude Desktop, Codex CLI, любой будущий MCP-aware) → `POST /internal/mcp`
- **OpenAPI клиенты** (Open WebUI, Obsidian-плагины, любой REST-клиент) → `GET /internal/openapi.json` + `POST /internal/tools/{name}`

Оба endpoint'а expose **весь tool surface обоих backends** (~85 от pm-mcp-server + ~14 от ai-memory ≈ 99 tools), без auth, привязаны строго к loopback `127.0.0.1`. External Funnel-конфиг не должен публиковать `/internal/*` наружу.

Это устраняет проблему Open WebUI **в рамках существующей архитектуры**, а не вопреки ей, и одновременно открывает дверь к будущей миграции Claude/Codex с stdio на HTTP MCP (по отдельной задаче — gateway уже будет готов их принять).

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
Internal clients (loopback only)
  ├── Open WebUI ──────HTTP+OpenAPI──────►  gateway POST /internal/tools/{name}
  ├── Obsidian (future) ─HTTP+OpenAPI────►          GET  /internal/openapi.json
  └── Claude/Codex (future migration) ───►  gateway POST /internal/mcp
                                                    │
                                                    ▼
                                          [Loopback guard middleware]
                                                    │
                                          [Unified ToolRegistry]
                                                    │
                                          (prefix-based routing)
                                                    │
                                            ┌───────┴──────────┐
                                            ▼                  ▼
                                  pm-mcp-server         ai-memory daemon
                                  127.0.0.1:8766/mcp    127.0.0.1:8765/mcp
                                  (MCP streamable-http) (MCP streamable-http)

External clients (existing, unchanged)
  ├── ChatGPT "AI-assistant (read)"  ──Tailscale Funnel──► gateway POST /mcp/read
  └── ChatGPT "AI-assistant (write)" ──Tailscale Funnel──► gateway POST /mcp/write
                                                            (OAuth+scopes+audit, curated 6 tools)
```

### Tool naming convention

Все tools, exposed через `/internal/*`, получают префикс backend'а:

- `ai-memory` tool `search_memory` → gateway exposes как `memory_search_memory`
- `ai-memory` tool `get_recent_memory` → `memory_get_recent_memory`
- `pm-mcp-server` tool `create_task` → `pm_create_task`
- `pm-mcp-server` tool `ping` → `pm_ping`

Правило простое: `<backend_prefix>_<original_name>`. Без переименований, без удаления дубликатов в имени (`memory_get_recent_memory` остаётся длинным, но предсказуемым). Это:

- избегает коллизий (`ping` в pm-mcp может однажды появиться в ai-memory)
- следует existing pattern в gateway (`memory_search`, `pm_list_work_items` уже с префиксом — единообразно)
- в Open WebUI ~99 tools сортируются alphabetically, префикс группирует визуально

Внутренние tool names в backends **не меняются** — Claude/Codex stdio продолжает видеть оригинальные `search_memory`, `create_task` и т.п.

### Discovery механизм

При старте gateway:

1. Для каждого зарегистрированного backend (ai-memory `127.0.0.1:8765/mcp`, pm-mcp-server `127.0.0.1:8766/mcp`) делает MCP JSON-RPC `initialize` → `tools/list`.
2. Кэширует результат в `ToolRegistry` с применением префиксов.
3. Failure-mode: если backend недоступен (как уже делают существующие health-checks в gateway README:91), пишет warning, registry содержит только tools от доступных backends. Health endpoint обновляется.
4. Rediscovery: на каждый рестарт gateway. Hot-reload tools — out of scope этого плана (если понадобится — отдельная задача с inotify/polling).

OpenAPI spec строится из registry динамически при первом GET `/internal/openapi.json` и кэшируется (инвалидируется при rediscovery).

### Где gateway сейчас не готов

`pm-mcp-server` сейчас **не поднимает MCP streamable-http endpoint** — только собственный custom REST (`POST /mcp/{tool_name}` в [pm-mcp-server/app/http_transport.py:130](pm-mcp-server/app/http_transport.py)). Gateway README:91 пишет «pm.* routes use `http://127.0.0.1:8766`» — фактически gateway сейчас ходит в custom REST с hardcoded списком 4 курированных tools.

Чтобы gateway мог сделать `tools/list` и discoverить все 85 tools, **`pm-mcp-server` нужно научить отдавать MCP streamable-http** (ai-memory это уже делает — [ai-memory/memory/daemon.py:376](ai-memory/memory/daemon.py)). Это маленькое изменение — одна `mount(...)` строка.

## Файлы

### Новые в `gateway/`

- [`gateway/tool_registry.py`](gateway/tool_registry.py)
  - `class ToolDef`: `prefixed_name`, `original_name`, `backend_url`, `description`, `input_schema`, `output_schema | None`
  - `class ToolRegistry`:
    - `async discover(backends: dict[prefix, url]) -> None` — `initialize` + `tools/list` JSON-RPC к каждому backend через MCP streamable-http клиент (`mcp.client.streamable_http`, уже в зависимостях ai-memory; добавить в gateway `pyproject.toml`)
    - `all_tools() -> dict[str, ToolDef]`
    - `get(prefixed_name) -> ToolDef | None`
    - `unavailable_backends() -> list[str]` — для health endpoint
  - Префиксы: `pm_` для pm-mcp-server, `memory_` для ai-memory

- [`gateway/openapi_builder.py`](gateway/openapi_builder.py)
  - `build_openapi_spec(registry: ToolRegistry, *, server_url: str) -> dict`
  - Возвращает dict OpenAPI 3.1, для каждого tool создаёт path `/internal/tools/{prefixed_name}` с `operationId=prefixed_name`, request body schema = `input_schema`, response = wrapper `{"ok": bool, "result": any, "error": object | null}`
  - Tags: `["memory"]` для memory_*, `["pm"]` для pm_*
  - `info.title = "AI-Assistant internal tools"`, `info.version = "1.0"`

- [`gateway/internal_app.py`](gateway/internal_app.py)
  - `register_internal_routes(app: Starlette, registry: ToolRegistry) -> None`
  - **`POST /internal/mcp`** — Starlette route, проксирует MCP JSON-RPC. Не хранит state, не делает auth. Принимает `initialize`/`tools/list`/`tools/call`, в `tools/list` отдаёт merged registry с префиксами, в `tools/call` находит backend по префиксу и проксирует исходный (без префикса) call.
  - **`GET /internal/openapi.json`** — отдаёт результат `openapi_builder.build_openapi_spec(...)`, кэшируется до следующей discovery.
  - **`GET /internal/docs`** — простой Swagger UI HTML, указывает на `/internal/openapi.json` (для удобства осмотра пользователем; CDN-загрузка swagger-ui-dist через `<script>`).
  - **`POST /internal/tools/{name}`** — один общий handler (не регистрируем 99 разных routes — это раздувает Starlette router без пользы):
    1. lookup `registry.get(name)` → 404 если нет
    2. читаем JSON body как dict — это arguments
    3. валидируем через `input_schema` (можно lazy, jsonschema уже в gateway зависимостях — videt в `.venv/Lib/site-packages/jsonschema/`) → 400 при invalid
    4. MCP `tools/call` к нужному backend с original name + arguments
    5. оборачиваем результат в `{"ok": True, "result": ...}` или `{"ok": False, "error": {...}}`
  - **`OPTIONS /internal/*`** — CORS preflight (`Access-Control-Allow-Origin: *` локально достаточно, потому что endpoint уже привязан к loopback).
  - **Loopback guard middleware** — мини-ASGI middleware, который для всех `/internal/*` путей проверяет `scope["client"][0]` ∈ `("127.0.0.1", "::1", "localhost")`; иначе возвращает 403. Это вторая линия защиты дополнительно к binding host'у — на случай если gateway случайно опубликуется через Tailscale Funnel целиком.

- [`gateway/tests/test_tool_registry.py`](gateway/tests/test_tool_registry.py) — mock backends (httpx MockTransport или aiohttp test server), проверка discovery с префиксами, обработка недоступного backend.

- [`gateway/tests/test_openapi_builder.py`](gateway/tests/test_openapi_builder.py) — registry с парой tools → spec валидный OpenAPI 3.1 (проверить через `jsonschema` против OpenAPI meta-schema), operationId и paths корректны.

- [`gateway/tests/test_internal_app.py`](gateway/tests/test_internal_app.py) — httpx.AsyncClient:
  - GET `/internal/openapi.json` → 200, ≥1 path
  - POST `/internal/tools/memory_search_memory` с valid body → 200 `{"ok": true, ...}`
  - POST `/internal/tools/nonexistent` → 404
  - POST `/internal/tools/pm_create_task` с invalid body → 400
  - OPTIONS preflight → 200 + CORS headers
  - Loopback guard: модифицированный client.host == "1.2.3.4" → 403
  - `POST /internal/mcp` `tools/list` → merged registry с префиксами
  - `POST /internal/mcp` `tools/call` → проксирует к backend и возвращает результат

### Изменения в `gateway/`

- [`gateway/app.py`](gateway/app.py) (или эквивалент main entry point)
  - На startup: создать `ToolRegistry`, запустить `await registry.discover({"pm": "http://127.0.0.1:8766/mcp", "memory": "http://127.0.0.1:8765/mcp"})`
  - Вызвать `register_internal_routes(app, registry)` рядом с существующими внешними routes
  - Существующие `/mcp`, `/mcp/read`, `/mcp/write`, `/token`, `/authorize`, `/register`, `/.well-known/*`, `/healthz` **не трогаем**
  - Расширить `/healthz` payload полем `internal_registry: {"pm_tools": N, "memory_tools": M, "unavailable_backends": [...]}` (без детального содержимого)

- [`gateway/pyproject.toml`](gateway/pyproject.toml) — добавить `mcp` (для `mcp.client.streamable_http`) если ещё нет. `jsonschema` уже есть.

- [`gateway/README.md`](gateway/README.md) — раздел «Internal profile»: что отдаёт, как подключить Open WebUI, что внешний Funnel не должен публиковать `/internal/*`.

- [`gateway/ARCHITECTURE.md`](gateway/ARCHITECTURE.md) — обновить диаграмму с internal/external split, явно зафиксировать loopback guard и discovery.

### Изменения в `pm-mcp-server/`

- [`pm-mcp-server/app/http_transport.py`](pm-mcp-server/app/http_transport.py) — в `create_app()` после регистрации существующих routes:
  ```python
  # Expose native MCP streamable-http для gateway internal discovery
  app.mount("/mcp-streamable", server.mcp.streamable_http_app())
  ```
  Используем path `/mcp-streamable` чтобы не конфликтовать с существующим custom REST `POST /mcp/{tool_name}`. Соответственно gateway `tool_registry.discover` использует URL `http://127.0.0.1:8766/mcp-streamable/mcp` (FastMCP внутри добавляет свой path — проверить точный).
  
  Альтернативно: мигрировать pm-mcp-server полностью с custom REST на native MCP streamable-http (удалить `/mcp/{tool_name}` и TOOL_NAMES set, поднять mount под `/mcp`). Это чище, но ломает любого client'а который сейчас использует custom REST endpoint напрямую. Решение: **оставить оба для безопасного rollout, deprecate custom REST отдельной задачей после миграции gateway**.

- [`pm-mcp-server/app/http_transport.py`](pm-mcp-server/app/http_transport.py) — auth middleware: исключить `/mcp-streamable/*` тоже из обязательного Bearer (как уже исключён `/health`), поскольку gateway работает по loopback и для FastMCP streamable-http свой механизм auth (gateway передаёт собственный internal token при необходимости — но в loopback-сценарии не нужен).
  
  Уточнение: gateway сам по себе exposed только loopback для backends. Если пользователь хочет — pm-mcp-server `/mcp-streamable` всё-таки может остаться под Bearer, gateway передаёт его при discovery (`AI_ASSISTANT_GATEWAY_PM_MCP_TOKEN` env уже есть в gateway, см. README:101).

- [`pm-mcp-server/tests/`](pm-mcp-server/tests/) — smoke test что MCP streamable-http endpoint отвечает `initialize` и `tools/list`.

### Не трогаем

- `pm-mcp-server/server.py` — все `@mcp.tool()` декораторы и логика handler'ов
- `pm-mcp-server/app/http_transport.py:130` `POST /mcp/{tool_name}` — остаётся для backwards-compat
- `ai-memory/` — целиком. Daemon на 8765 уже MCP streamable-http.
- Все external endpoints gateway (`/mcp/read`, `/mcp/write`, OAuth, `/.well-known/*`) — никаких regressions для ChatGPT
- Claude Desktop / Codex stdio configs — продолжают работать

## Verification

### 1. Backends готовы

```powershell
# ai-memory daemon (как уже работает)
cd D:\GitHub\AI-Assistant\ai-memory
uv run python -m memory.daemon                # фон

# pm-mcp-server с новым /mcp-streamable mount
cd D:\GitHub\AI-Assistant\pm-mcp-server
uv run python -m app.http_server              # фон, 8766

# Sanity:
curl http://127.0.0.1:8765/healthz            # ai-memory ok
curl http://127.0.0.1:8766/health -H "Authorization: Bearer $env:PM_MCP_HTTP_TOKEN"
```

### 2. Gateway discovery

```powershell
cd D:\GitHub\AI-Assistant\gateway
uv run ruff check .
uv run python -m unittest discover -s tests
uv run python -m gateway.app                  # 8780
curl http://127.0.0.1:8780/healthz
# В payload видим internal_registry.pm_tools ≈ 85, memory_tools ≈ 14
```

### 3. OpenAPI surface

```powershell
curl http://127.0.0.1:8780/internal/openapi.json | ConvertFrom-Json | Select-Object -ExpandProperty paths | Get-Member -MemberType NoteProperty | Measure-Object
# ≈ 99 paths под /internal/tools/*

Start-Process http://127.0.0.1:8780/internal/docs    # Swagger UI с раскрытыми tools
```

### 4. Прямые вызовы

```powershell
curl -X POST http://127.0.0.1:8780/internal/tools/pm_ping `
     -H "Content-Type: application/json" -d '{}'
# → {"ok": true, "result": "pong"}

curl -X POST http://127.0.0.1:8780/internal/tools/memory_get_recent_memory `
     -H "Content-Type: application/json" -d '{"limit": 3}'
# → {"ok": true, "result": {"status":"ok","items":[...],"count":3,...}}

curl -X POST http://127.0.0.1:8780/internal/tools/nonexistent_tool `
     -H "Content-Type: application/json" -d '{}'
# → 404
```

### 5. MCP endpoint

```powershell
# tools/list через JSON-RPC
curl -X POST http://127.0.0.1:8780/internal/mcp `
     -H "Accept: application/json, text/event-stream" `
     -H "Content-Type: application/json" `
     -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
# В ответе ~99 tools с префиксами

# tools/call
curl -X POST http://127.0.0.1:8780/internal/mcp `
     -H "Accept: application/json, text/event-stream" `
     -H "Content-Type: application/json" `
     -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"pm_ping","arguments":{}}}'
```

### 6. CORS

```powershell
curl -i -X OPTIONS http://127.0.0.1:8780/internal/tools/pm_ping `
     -H "Origin: http://localhost:3000" `
     -H "Access-Control-Request-Method: POST"
# 200 + Access-Control-Allow-Origin
```

### 7. Loopback guard

Через временную правку binding или nc/tcpdump с другого хоста в Tailnet — попытка хитнуть `/internal/*` должна получить 403. (Tailscale Funnel конфиг проверяется отдельно — `/internal/*` не должно появляться в `tailscale serve status`.)

### 8. Open WebUI end-to-end

В Open WebUI → Settings → Tools → Add tool server:
- URL: `http://127.0.0.1:8780/internal/openapi.json`
- Auth: None
- Сохранить → видим ~99 tools в каталоге

В чате с локальной Ollama: «Создай задачу 'test integration' в проекте D:/GitHub/AI-Assistant/pm-mcp-server» → Open WebUI должен вызвать `pm_create_task`. «Какие у меня недавние memory entries?» → `memory_get_recent_memory`.

### 9. External regressions (инварианты НЕ нарушены)

Прежде чем считать задачу выполненной, явно прогнать всю существующую external-схему:

- **ChatGPT (read connector)**: тестовый чат с `memory_search` через `/mcp/read` → 200, items возвращаются. Попытка вызвать write-tool (`memory_propose`) с read-токеном — отклонена с `tool_not_in_profile` (на уровне endpoint) и `invalid_scope` (на уровне scope policy).
- **ChatGPT (write connector)**: `memory_propose` через `/mcp/write` → 200, `proposal_id` возвращён. Запись **не появляется** в `search_memory` ни через ChatGPT (через read), ни локально через `memory.cli search`. Только после ручного `memory.cli proposals-approve <id>` запись становится видимой.
- **OAuth flow**: `GET /.well-known/oauth-authorization-server` и `/.well-known/oauth-protected-resource` отдают валидные metadata. DCR через `POST /register` создаёт client. PKCE `S256` обязателен.
- **Tailscale Funnel**: `tailscale serve status` не показывает `/internal/*` среди опубликованных путей (если показывает — конфиг Funnel надо явно сузить, не публиковать `/internal/*`).
- **Audit chain**: каждый external запрос пишет audit row в `data/<gateway-audit>.jsonl` с `previous_hash` и `row_hash`; цепочка верифицируется существующим тестом gateway.
- **Claude Desktop через stdio**: вызов `mcp__PM-MCP-server__ping` и `mcp__AI-memory__get_recent_memory` в Claude Code или Claude Desktop UI — продолжает работать (stdio backends не трогали).

## Risks

- **`pm-mcp-server` mount пути**: FastMCP `streamable_http_app()` может конфликтовать с настройкой `streamable_http_path` (default `/mcp`). Если `app.mount("/mcp-streamable", mcp_app)` приводит к двойному префиксу — использовать другой path или сконфигурировать FastMCP с `streamable_http_path="/"`. Тривиально проверить при имплементации.
- **Discovery failure**: если на старте gateway один из backends недоступен — gateway всё равно поднимается с warning, exposed только tools от доступных backends. При появлении backend нужен рестарт gateway (hot-reload — отдельная задача).
- **99 tools в Open WebUI UX**: возможно шумно. Пользователь может вручную выбирать subset в Open WebUI настройках, либо мы добавим query param `?profile=curated` к openapi.json в будущей итерации.
- **OpenAPI 3.1 vs 3.0**: input_schemas от FastMCP — JSON Schema 2020-12 (OpenAPI 3.1). Open WebUI принимает 3.x. Если будут проблемы — можно даунгрейдить через [openapi-spec-validator] или явный downgrade-converter.
- **MCP `streamable_http_client` в gateway**: уже используется в ai-memory (`memory/daemon.py:498` через `mcp.client.streamable_http`). В gateway возможно нужен upgrade зависимостей.

## Future work (не в этом плане)

1. **Миграция Claude Desktop / Codex CLI** со stdio на HTTP MCP через `/internal/mcp`. Технически Claude Desktop поддерживает `"transport": "http"` в `claude_desktop_config.json` ([docs](https://modelcontextprotocol.io)) — переключение конфига, никаких изменений в backends. Делаем отдельной задачей когда подтвердим стабильность internal endpoint.
2. **Объединение БД задач и AI-memory** (упомянуто пользователем). Сейчас задачи хранятся в `pm-mcp-server/.../db.sqlite3`, memory в `ai-memory/data/memory.db`. Перенос — отдельный план в духе [global-numbering-migration.md](DONE/global-numbering-migration.md).
3. **Hot-reload tool registry** в gateway — fsnotify / periodic re-discovery, если backends рестартуют и появляются новые tools.
4. **Курированный internal profile** — если 99 tools слишком много для Open WebUI UX, добавить `?profile=read|write|core` фильтрацию.
5. **Deprecation legacy `POST /mcp/{tool_name}` в pm-mcp-server** — после миграции gateway на MCP streamable-http, удалить custom REST + hardcoded `TOOL_NAMES` set. Cleanup-задача.
6. **Internal Bearer optional** — если в будущем пользователь захочет добавить минимальную защиту (например multi-user setup на одной машине), env `AI_ASSISTANT_GATEWAY_INTERNAL_TOKEN` — пустой = no auth (default), задан = Bearer required. Можно добавить без переписывания.
