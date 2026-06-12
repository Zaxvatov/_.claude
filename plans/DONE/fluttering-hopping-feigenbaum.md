# Open WebUI dual-protocol connectors (in-process MCP→OpenAPI)

PM-MCP: #812 (T1, root) · #813 (T2, pm-mcp-server) · #814 (T3, ai-memory) · #815 (T4, root) · interim #788

## Context

Цель пользователя: подключить `ai-memory` и `pm-mcp-server` к локальному Open WebUI
(`http://127.0.0.1:8080`, версия **0.9.6**), чтобы модели OWUI пользовались ими так
же, как другие агенты (Claude, Codex, ChatGPT).

Что выяснено в живой системе 2026-06-03:
- OWUI 0.9.6 **не потребляет native MCP** в чате. Регистрация `type:"mcp"` проходит
  server-side verify (tools/list виден), но модель в чате не получает инструменты
  (qwen прямо ответил «это не мой функционал»), а ai-memory вообще не появлялся в
  пикере. Браузер не делает client-side запрос к `/mcp`.
- OWUI 0.9.6 потребляет **только OpenAPI tool servers** (`/api/v1/configs/tool_servers`,
  `type:"openapi"`): фронт client-side тянет `<url><path>` (`/openapi.json`) — нужен
  **CORS** на origin `http://127.0.0.1:8080`.
- Временный мост `uvx mcpo` (порт 8788, OpenAPI + CORS) доказал путь: deepseek-v4-pro
  в OWUI реально вызвал `pm-mcp-server→list_projects` (16 проектов) и
  `ai-memory→search_memory` (10 записей) с tool-call цитатами. Сейчас admin-конфиг
  OWUI указывает на 8788. mcpo запущен как **временный процесс сессии** — умрёт по
  её завершении, и оба инструмента отвалятся.

Решение пользователя: **не плодить новый сервис** (6 NSSM-служб уже есть). Допустимо
**расширить существующую службу**. Значит мост MCP→OpenAPI должен жить **внутри уже
работающих процессов** `ai-memory` и `pm-mcp-server`, а не отдельным mcpo-демоном.

Итог: коннекторы становятся **двупротокольными** — MCP для Claude/Codex/ChatGPT,
OpenAPI для Open WebUI и любого будущего OpenAPI-агента, **без новой инфраструктуры**.

Соответствует tech-stack-choices: brick **#12** (фоновые задачи как mount внутри
существующего FastAPI-процесса, ADR-0001 D-8 «минимизация daemon'ов») и brick **#14**
(один сервис-слой — несколько протоколов потребления). Loopback-доступ без auth —
brick **#5** hybrid trust (локальные агенты = trust-by-default).

## Decision (рекомендуемый подход)

**Собственный тонкий in-process mount** (НЕ embed mcpo): в каждом сервисе добавить
небольшой модуль `openapi_bridge.py`, который из `await mcp.list_tools()` строит
OpenAPI-3.1 документ и по одному POST-роуту `/<tool_name>` на инструмент, вызывая
`await mcp.call_tool(name, arguments)` **в том же процессе** (без self-HTTP). Монтируется
на существующем порту по префиксу `/openapi`, с `CORSMiddleware` для точного OWUI-origin
(`http://127.0.0.1:8080`; детали allowlist/CORS/normalization — в Bridge module ниже).

Почему не embed mcpo: mcpo тянет свои зависимости и поднимает MCP-client-сессию к
loopback-URL самого себя (self-HTTP, циклично). Тонкий mount — без новой зависимости,
прямые in-process вызовы (brick #14), полный контроль CORS и формы схемы. Форму
OpenAPI копируем с проверенного вывода mcpo (`operationId: tool_<name>_post`, POST,
`requestBody` из `inputSchema`).

OWUI после готовности указывает на:
- ai-memory: url `http://127.0.0.1:8765/openapi`, path `openapi.json`
- pm-mcp:    url `http://127.0.0.1:8766/openapi`, path `openapi.json`

После переключения временный mcpo (8788) останавливается и удаляется из документации.

## Design

### Bridge module `openapi_bridge.py` (копия в каждом сервисе — изоляция как `runtime_contract.py`)
`build_openapi_tools_app(mcp, *, title, allowed_origins, allowlist) -> Starlette`:
- `GET /openapi.json` — OpenAPI 3.1 из `mcp.list_tools()` (name/description/inputSchema),
  **только** инструменты из `allowlist` (operationId `tool_<name>_post`, как у mcpo).
- `POST /<tool>` — только из `allowlist`; body JSON → `await mcp.call_tool(tool, args)`.
- **Result/error normalization contract**: `call_tool()` отдаёт structured dict ИЛИ
  список content-block'ов — bridge нормализует в `{"ok":true,"result":<json>}`; ошибка
  инструмента → `{"ok":false,"error":{"code","message"}}` (HTTP 4xx/5xx);
  неизвестный/не-в-allowlist tool → 404; **response cap** ~256 KiB (как read-daemon
  ai-memory) с флагом усечения. Иначе OWUI может получить невалидный tool-response.
- **Allowlist обязателен (closed allow-list, не block-list, не optional)** — ровно
  перечисленные read-only имена, **без wildcard/`…`**. Всё, чего нет в списке (в т.ч. ВСЕ
  write-tools), не попадает ни в `openapi.json`, ни в POST-роуты. Каждое имя в T1 сверяется
  с live-registry как read-only; при сомнении — исключать.
  - **ai-memory (read-only, 11)**: `search_memory`, `get_recent_memory`,
    `search_by_workitem`, `search_by_metadata`, `preview_summary`,
    `get_latest_workflow_memory`, `get_workflow_run_history`, `get_batch_run_history`,
    `get_portfolio_run_history`, `traverse_memory`, `get_tag_graph` (последние два —
    read-only граф-чтения по ADR-0010; **включаем явно**).
  - **pm-mcp (read-only)**: `ping`, `list_projects`, `get_project_context`,
    `get_project_summary`, `get_projects_overview`, `list_work_items`, `get_work_item`,
    `get_work_item_history`, `read_tasks`, `read_tasks_range`, `get_blockers`,
    `get_next_task`, `get_dashboard_snapshot`, `get_drift_report`, `morning_brief`,
    `project_brief`, `proactive_suggestions`, `recommend_next_global`,
    `recommend_next_project_task`, `task_queue`, `list_goals`, `get_goal_tree`,
    `get_goal_path`, `get_goal_decomposition`, `get_user_constraints`,
    `get_portfolio_overview`, `get_portfolio_initiatives`, `weekly_review_report`,
    `export_tasks_md`, `audit_query`, `list_registered_processes`,
    `get_process_state_audit`.
  - **Writes**: НЕ открываются никаким «enable-all» флагом. Только точечно — env
    `<SERVICE>_OPENAPI_TOOLS=<csv точных имён>` добавляет именно перечисленные tools к
    read-дефолту (отдельно для ai-memory и pm-mcp). Нет csv → нет writes.
- **CORS ≠ auth**: `allow_origins=["http://127.0.0.1:8080"]` (exact), методы
  `GET,POST,OPTIONS`, headers `content-type`. Доп. origin — только через env
  `<SERVICE>_OPENAPI_ALLOWED_ORIGINS` (exact list, без широкого `localhost:*` regex).
- **Local-only surface**: bind только loopback (порт сервиса на 127.0.0.1); это НЕ новый
  внешний ingress — внешний доступ остаётся только через Gateway (ADR-0001 D-3).

### pm-mcp-server (меньшее изменение — outer FastAPI уже есть)
- `app/http_transport.py:create_app()` — добавить `app.mount("/openapi",
  build_openapi_tools_app(server.mcp, ...))`.
- В `auth_middleware` добавить `/openapi` в bypass (loopback, без Bearer) рядом с
  `/health` и `PM_MCP_STREAMABLE_MOUNT_PATH`.
- Новый файл `app/openapi_bridge.py`.

### ai-memory (большее изменение — сейчас `mcp.run()` напрямую)
- **Preflight spike (до правки service runtime)**: в изолированном тесте собрать ASGI из
  `mcp.streamable_http_app()` + outer lifespan `session_manager.run()` + `/mcp` +
  `/healthz` + `/openapi/openapi.json` + один POST read-tool, прогнать через
  `httpx.ASGITransport` на ephemeral-порту с временной БД. Менять `daemon.py` только
  после зелёного спайка — это снижает риск самого опасного пункта плана.
- `memory/daemon.py:run_daemon_service()` — вместо `mcp.run(transport="streamable-http")`
  собрать ASGI-приложение из `mcp.streamable_http_app()` (его lifespan поднимает
  `session_manager`), навесить `CORSMiddleware`, примонтировать `/openapi`, и запустить
  через `uvicorn.run(app, host, port)`. **Сохранить**: singleton-lock, `/healthz`
  (custom_route уже в `build_mcp_server(include_healthcheck=True)`), регистрацию в
  PM-MCP Process State Manager, log/lock контракт. Шаблон композиции — pm-mcp
  `app/http_transport.py` (lifespan `async with server.mcp.session_manager.run()`).
- Новый файл `memory/openapi_bridge.py` (копия общего моста).

### OWUI repoint + retirement (root AI-Assistant) — финальный rollout-gate
- **Сначала выкатить и проверить сервисы** (admin step пользователя: рестарт NSSM
  `AI-Assistant-AI-memory` и `AI-Assistant-PM-MCP-server` + проверка свежего PID/health;
  агент НЕ рестартит службы сам — nssm требует admin).
- Через admin-API OWUI (браузер, JS как в этой сессии) заменить два mcpo-эндпоинта на
  in-process: `8765/openapi` + `8766/openapi`.
- Только после зелёной проверки — **остановить временный mcpo (8788)** и повторить smoke.
- `tools/README.md` + `tools/open-webui-mcpo.config.json`: in-process OpenAPI —
  основной путь; mcpo — пометить как emergency-only либо удалить (migration discipline
  Q.2: документация описывает только текущий путь).
- Новый **ADR `docs/adrs/0014-openwebui-dual-protocol.md`** (0009–0013 заняты; см.
  `docs/adrs/README.md`) + обновить индекс в README: ограничение native-MCP в OWUI 0.9.6,
  решение о двупротокольных коннекторах, in-process OpenAPI mount, **local-only/loopback
  boundary со ссылкой на ADR-0001** (внешний доступ только через Gateway), allowlist
  read-only-by-default, ретайр mcpo.

### Docs
pm-mcp + ai-memory `AGENTS.md`/`ARCHITECTURE.md`/`README.md` (новый OpenAPI surface,
порт/путь, CORS, loopback-trust); root `README.md`. После аппрува предложить правки
tech-stack-choices brick #5 (OpenAPI surface для OpenAPI-only агентов) и #14 (третий
путь потребления) — по правилу skill только после подтверждения пользователя.

## Critical files
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\http_transport.py` (mount + auth bypass)
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\openapi_bridge.py` (новый)
- `D:\GitHub\AI-Assistant\ai-memory\memory\daemon.py` (`run_daemon_service`: uvicorn + mount)
- `D:\GitHub\AI-Assistant\ai-memory\memory\openapi_bridge.py` (новый)
- `D:\GitHub\AI-Assistant\docs\adrs\0014-openwebui-dual-protocol.md` (новый) + `docs\adrs\README.md` (индекс)
- `tools/README.md`, `tools/open-webui-mcpo.config.json`, профильные `AGENTS/ARCHITECTURE/README`

## Reuse
- `await mcp.list_tools()` / `await mcp.call_tool()` (FastMCP, пакет `mcp`).
- `runtime_contract.EXPECTED_TOOLS` в обоих сервисах — sanity-чек набора инструментов.
- pm-mcp `app/http_transport.py` lifespan-паттерн — шаблон для ai-memory.
- Форма OpenAPI — с проверенного вывода mcpo (verify в этой сессии вернул именно её).

## Verification

### Offline-first (без живых данных, в pytest — основной gate)
1. In-process через `httpx.ASGITransport` (порт не bind-ится) ИЛИ ephemeral-порт — но НЕ
   canonical 8765/8766; временная БД:
   - `GET /openapi/openapi.json` → 200, operationId `tool_<name>_post`, в схеме **только**
     allowlist (write-tools отсутствуют при дефолте);
   - `POST /openapi/<read_tool>` → нормализованный `{"ok":true,...}`; ошибка → `{"ok":false,...}`;
     response cap соблюдён;
   - CORS-заголовок только для `http://127.0.0.1:8080`, не для произвольного origin;
   - pm-mcp `/openapi` доступен без Bearer (auth bypass), прочие пути — 401;
   - write-tool недоступен без явного `*_OPENAPI_TOOLS=<csv>`; write-smoke **запрещает**
     canonical-порты (только in-process ASGITransport / ephemeral + temp БД).
2. `uv run ruff check .` + `uv run pytest` в обеих подсистемах. ai-memory preflight-спайк
   зелёный до правки `daemon.py`.

### Live rollout-gate (финальный, ручной)
3. Admin step пользователя: рестарт NSSM `AI-Assistant-AI-memory` (cold-start ~45s,
   sentence-transformers) и `AI-Assistant-PM-MCP-server`, проверка свежего PID/health.
4. `curl` 8765/8766 `/openapi/openapi.json` → 200 + CORS; `/healthz`/`/mcp` живы.
5. OWUI: repoint → reload → пикер показывает оба сервера без broken-иконок →
   deepseek-v4-pro вызывает по одному read-инструменту каждого.
6. **Остановить временный mcpo (8788)** и повторить (5) — инструменты обязаны работать
   через in-process эндпоинты.
7. Перезагрузка Windows → службы поднимаются (NSSM AUTO_START), OWUI-инструменты живы.

## Risks
- **ai-memory daemon refactor** (`mcp.run()` → uvicorn+mount) — самый рискованный пункт:
  не сломать singleton-lock, `/healthz`, PM-MCP register, streamable `/mcp`. Митигировать
  копией pm-mcp-паттерна и прогоном `ping_daemon`/health после рестарта.
- **Browser-facing CORS surface ≠ private MCP trust** (ключевой риск по Codex): любой
  локальный origin под CORS мог бы дёргать tools. Митигировано: allow-list read-only
  by default, writes только через env opt-in, exact CORS origin, local-only loopback bind,
  result/error+cap нормализация. Это НЕ паритет с private MCP Codex/Claude.
- **Дублирование** ~120 строк моста в двух сервисах — осознанно (изоляция подсистем,
  как `runtime_contract.py`); вынести в общий модуль только если появится 3-й потребитель.
- **Окно простоя**: между остановкой mcpo и готовностью in-process пути OWUI-инструменты
  недоступны — порядок: сначала выкатить сервисы, проверить, потом repoint+ретайр mcpo.

## Acceptance
- Offline pytest (ASGITransport/temp БД) зелёный: openapi.json содержит только allowlist,
  write-tools скрыты при дефолте, CORS только для `:8080`, нормализация result/error+cap,
  pm-mcp `/openapi` без Bearer. Write-smoke не трогает canonical-порты.
- Оба сервера видны в OWUI-пикере без broken-иконок; deepseek вызывает по одному
  read-инструменту каждого через `127.0.0.1:8765/openapi` и `:8766/openapi`.
- Временный mcpo (8788) остановлен/удалён; инструменты работают без него.
- Переживает перезагрузку (NSSM), без нового сервиса. Рестарт служб — admin step
  пользователя, не автодействие агента.
- Тесты обеих подсистем зелёные; ADR-0014 + индекс + docs обновлены; правки
  tech-stack-choices (brick #5/#14) предложены пользователю (не применены без аппрува).

## Phased order (меньший blast radius, по Codex)
Порядок: контракт → меньший сервис → рискованный сервис → rollout → docs sweep.

## Tasks (PM-MCP — создать после аппрува через pm-mcp-task-flow; явная цепочка)
- **T1 (#812) — root AI-Assistant: ADR-0014 + bridge contract** (closed read-only allowlists с
  точными именами, exact CORS origins, result/error+response-cap schema,
  local-only/loopback boundary, индекс ADR). **Старт цепочки, без зависимостей.**
- **T2 (#813) — pm-mcp-server** (первым: outer FastAPI уже есть): `app/openapi_bridge.py` + mount
  `/openapi` + auth bypass + read-only allowlist + offline tests + профильные docs.
  Зависит от T1 (`dependency_project_path` → AI-Assistant root для ADR).
- **T3 (#814) — ai-memory** (вторым): preflight-спайк → ASGI/uvicorn composition в `daemon.py`
  (сохранить lock/`/healthz`/PM-MCP register/`/mcp`) + `memory/openapi_bridge.py` +
  read-only allowlist + offline tests + профильные docs. Зависит от T1.
- **T4 (#815) — root AI-Assistant: rollout + docs sweep**: repoint OWUI на in-process → smoke →
  остановить mcpo → `tools/README.md`/`open-webui-mcpo.config.json` к одному текущему пути.
  Зависит от T2 и T3.
- **`#788` вне dependency-графа T1–T4** (иначе цикл `#788`→T4→T1→`#788`): interim mcpo
  live-rollout функционально выполнен и проверен в этой сессии; закрывается как Готово
  **отдельно** (сейчас или при ретайре mcpo на T4); T1–T4 ссылаются на него как контекст,
  не как блокер.
- Кросс-подсистемные связи — через `link_task_dependency` с `dependency_project_path`
  (K.7). Правки tech-stack-choices brick #5/#14 — предложить после аппрува (skill-правило).
