# Подключение ChatGPT Plus к AI-memory через публичный read-only MCP endpoint

## Context

Пользователь хочет, чтобы его ChatGPT Plus читал контекст из локального
AI-memory и опирался на ту же долговременную память, в которую уже пишут
Claude Code, Codex CLI и Assistant-UI+Ollama. На тарифе **Plus** OpenAI
блокирует write-tools в Custom MCP Connectors — доступны только
read/fetch tools (полноценная запись через MCP требует
Business/Enterprise/Edu).

С учётом этого:
- запись остаётся за локальными агентами (они уже пишут);
- ChatGPT получает только **read-доступ** к памяти, и этого достаточно;
- по нашим источникам, Developer Mode может отключать встроенную ChatGPT
  Memory; это поведение **проверяется во время pre-flight** и
  фиксируется как `kind=fact` в AI-memory вместо того, чтобы
  закладываться на него заранее.

Дополнительное требование пользователя: **максимальная защита от утечки
данных и взломозащищённость**. Поэтому архитектура построена по принципу
defense-in-depth — каждый слой независимо ограничивает поверхность атаки.

Источники по ограничениям ChatGPT (май 2026):
- [Developer mode and MCP apps in ChatGPT — OpenAI Help](https://help.openai.com/en/articles/12584461-developer-mode-apps-and-full-mcp-connectors-in-chatgpt-beta)
- [Building MCP servers for ChatGPT Apps — OpenAI Developers](https://developers.openai.com/api/docs/mcp)
- [Connectors and MCP servers — OpenAI Platform](https://platform.openai.com/docs/guides/tools-connectors-mcp)
- [Connectors in ChatGPT — OpenAI Help](https://help.openai.com/en/articles/11487775-connectors-in-chatgpt)
- [Configuring actions in GPTs — OpenAI Help](https://help.openai.com/en/articles/9442513-configuring-actions-in-gpts)

## Pre-flight: сверить, что в ChatGPT Plus реально есть Developer Mode

Документация OpenAI по Plus/Pro неоднозначна: на одних страницах Custom
MCP Connectors в Developer Mode заявлены доступными для Plus, на других
— только для Business/Enterprise/Edu. **Перед началом любого кода
пользователь открывает ChatGPT в браузере и проверяет:**

1. ChatGPT → Settings → Connectors (или Apps) → Advanced → видит ли
   тумблер «Developer mode».
2. После включения — есть ли «Add custom connector» с полем URL и
   поддержкой Bearer auth.

Если в UI Plus развилка отсутствует, переходим в **Fallback B**
(Assistant-UI как chat-клиент через OpenAI API) — он даёт полноценный
read+write контур и не зависит от capabilities ChatGPT-сайта. Если
доступен Developer Mode + Custom MCP — выполняем основной план.

Запасной путь **Fallback A** через Custom GPT Actions с OpenAPI
работоспособен. По состоянию на май 2026 OpenAI продолжает поддерживать
GPT Actions (API key/OAuth, создание actions описано в [Configuring
actions in GPTs](https://help.openai.com/en/articles/9442513-gpt-actions-domain-settings-chatgpt-enterprise)
и [GPTs in ChatGPT](https://help.openai.com/en/articles/8554407)),
однако Apps/MCP — современный путь, и ставка на Actions считается
legacy. Включаем Fallback A как мост, если Plus имеет Actions, но не
Custom MCP.

## Целевая архитектура

```text
ChatGPT Plus
    │ HTTPS, header: Authorization: Bearer <secret-from-Windows-Credential-Manager>
    ▼
Tailscale Funnel  https://<machine>.<tailnet>.ts.net/{mcp,healthz}
    │ TLS terminated by Tailscale → HTTP loopback
    ▼
AI-memory public daemon  127.0.0.1:8767  (read-only)
    │ ┌─────────────────────────────────────────────────────────┐
    │ │ ASGI middleware stack (внешнее → внутреннее):           │
    │ │  1. SecurityHeadersMiddleware  (HSTS, nosniff, no-store)│
    │ │  2. BodySizeMiddleware         (≤ 1 MiB)                │
    │ │  3. RateLimitMiddleware        (token-bucket per-IP)    │
    │ │  4. BearerAuthMiddleware       (chained tokens, keyring)│
    │ │  5. AuditMiddleware            (JSONL, без секретов)    │
    │ │  6. FastMCP streamable-http (12 read-tools)             │
    │ │     └─ tools обёрнуты ProjectAllowlist + LimitCap       │
    │ │     └─ MetadataRedaction post-processor                 │
    │ └─────────────────────────────────────────────────────────┘
    │
    ▼  read-only handle: sqlite3 connect(?mode=ro) + PRAGMA query_only=ON
    ▼
data/memory.db  ◀──  AI-memory private daemon 127.0.0.1:8765 (read+write, без auth)
                       (Claude/Codex/Assistant-UI как сейчас)
```

Приватный 8765 endpoint **не трогаем** — байт-стабильно для Claude/Codex.
Публичный 8767 — отдельный процесс, отдельный lock, отдельный лог,
отдельный SQLite-handle (read-only). Tailscale Funnel — бесплатный
HTTPS-туннель со стабильным URL, без своего домена.

## Threat model

| Угроза | Слой защиты |
|---|---|
| Перехват трафика | TLS терминация на Tailscale (Let's Encrypt) |
| Подмена клиента | Bearer ≥ 32 байта, `hmac.compare_digest`, ротация (active+previous) |
| Утечка токена через env / child process | Хранение в Windows Credential Manager (keyring), env только как fallback |
| Утечка токена в логи | `access_log=False`, middleware никогда не логирует header `Authorization`, audit пишет только tool name + status |
| Brute-force токена | Rate limit 60 req/min per-IP по умолчанию + 5-секундный backoff после 5 подряд 401 |
| Запись через MCP | Public FastMCP физически регистрирует только 12 read-tools |
| Запись через SQL injection / эксплойт в FastMCP | SQLite connection открыт в `?mode=ro` с `PRAGMA query_only=ON` — на уровне ОС файл всё равно писать нельзя |
| Утечка чувствительных проектов | **Default-deny** allow-list `AI_MEMORY_PUBLIC_ALLOWED_PROJECTS`; пустое или незаданное = доступа нет ни к одному проекту; tools-обёртки фильтруют ответы |
| Утечка секретов в `metadata` | Post-processor `redact_metadata` маскирует значения по ключам `*token*`, `*secret*`, `*password*`, `*api_key*`, `*private_key*`, `*credential*` |
| Утечка секретов в `text` | Post-processor `redact_text` применяет regex-паттерны для известных форматов (OpenAI `sk-...`, GitHub `ghp_...`, Slack `xox[bp]-...`, AWS `AKIA...`, JWT `eyJ...`, generic high-entropy строки длиной ≥40 после `password=`/`token=`/`secret=`) |
| Объёмная экфильтрация | Hard cap на `limit ≤ 50` (вне зависимости от запрошенного), `BodySizeMiddleware` (1 MiB), response cap (общий размер ответа ≤ 256 KiB после редактирования) |
| Эскалация прав через файловую систему | Daemon запускается **под отдельным low-privilege Windows-пользователем** (`ai-memory-public`), у которого ACL даёт только `Read` на `data/memory.db` и `Write` только в свой лог-каталог |
| Утечка stack-trace | Глобальный exception handler возвращает `{"error":"internal"}` 500 без деталей |
| DoS | Rate limit per-IP + body size limit + uvicorn workers=1 (намеренно: меньше поверхность) |
| Перехват через MITM в Tailscale | Tailscale аккаунт защищён 2FA; Funnel ACL ограничивает publish-права на конкретную машину |
| Утечка через ChatGPT Conversations | OpenAI хранит запросы и ответы; ChatGPT-сторона неуправляема — поэтому метаданные редактируются ДО возврата клиенту |
| Replay-атака | Read-only — replay безвреден; для defense-in-depth добавляем `Cache-Control: no-store` |
| Подмена Tailscale-узла | Tailscale node identity (WireGuard public key) аутентифицирует машину; контроль через https://login.tailscale.com/admin/machines |

## Политика секретов в AI-memory

`redact_text` и `redact_metadata` — **последняя линия обороны**, не
основной механизм защиты. Основной механизм — **секреты в принципе не
должны попадать в AI-memory**. Это фиксируется в [AGENTS.md](D:/GitHub/AI-memory/AGENTS.md)
(уже есть пункт 4 в разделе AI-memory: «Do not store secrets, tokens,
passwords, personal data, `.env` contents, large logs, transient errors,
or raw file dumps») и подкрепляется в [docs/CHATGPT.md](D:/GitHub/AI-memory/docs/CHATGPT.md):

> Любые секреты (API ключи, токены, пароли, .env содержимое,
> credential JSON) **не должны** записываться в AI-memory ни одним
> агентом. Если secret всё-таки попал — `redact_text/redact_metadata`
> попытаются его замаскировать на выдаче ChatGPT, но эта защита
> рассчитана на типовые форматы и может пропустить нестандартные. Не
> полагайтесь на неё; чистите память через `archive` при подозрении
> на утечку.

В [tests/test_public_filters.py](D:/GitHub/AI-memory/tests/test_public_filters.py)
тесты redaction явно помечают, что закрывают только known-format
patterns, и не дают гарантии для произвольных строк.

## Тулзы публичного endpoint (до 12 read, по умолчанию 2 — Wave 0)

`search_memory`, `get_recent_memory`, `search_by_workitem`,
`get_chat_history`, `get_token_usage_history`, `analyze_session_outcomes`,
`summarize_history`, `get_process_resource_state`,
`get_latest_workflow_memory`, `get_workflow_run_history`,
`get_batch_run_history`, `get_portfolio_run_history`.

Исключены: `store_memory`, `store_memory_batch`,
`set_process_resource_state`.

Каждый из 12 tools оборачивается обёртками на регистрации (в порядке
применения, снаружи внутрь):
1. `with_project_allowlist(allow=AI_MEMORY_PUBLIC_ALLOWED_PROJECTS)` —
   **default-deny**: пустой/незаданный allow-list → ничего не выдаём.
   Фильтрует возвращаемые items по разрешённым `project=`.
2. `with_limit_cap(max_limit=50)` — клампит `limit`.
3. `with_response_cap(max_bytes=262_144)` — серверный лимит на общий
   размер ответа после JSON-сериализации; при превышении урезаем до
   меньшего числа items и выставляем `truncated=True` в payload.
4. После выполнения — `redact_metadata(items)` (ключи) и
   `redact_text(items)` (regex по содержимому `text`) стирают
   потенциальные секреты.

### Поэтапное включение tools (staged enablement)

Управление — env-флагом `AI_MEMORY_PUBLIC_ENABLED_TOOLS` (CSV).
Default-значение в [memory/runtime_contract.py](D:/GitHub/AI-memory/memory/runtime_contract.py):
`PUBLIC_DEFAULT_ENABLED_TOOLS = PUBLIC_COMPAT_TOOLS = ("search","fetch")`.

Константы:
- `PUBLIC_COMPAT_TOOLS = ("search", "fetch")` — Wave 0.
- `PUBLIC_NATIVE_READ_TOOLS = ("search_memory", "get_recent_memory",
  "summarize_history", "search_by_workitem", "get_chat_history",
  "analyze_session_outcomes", "get_token_usage_history",
  "get_latest_workflow_memory", "get_workflow_run_history",
  "get_batch_run_history", "get_portfolio_run_history",
  "get_process_resource_state")` — 12 родных read-tools.
- `PUBLIC_PUBLIC_REGISTRABLE = PUBLIC_COMPAT_TOOLS + PUBLIC_NATIVE_READ_TOOLS`
  — все 14, что вообще можно включить.
- Любое имя из `EXPECTED_TOOLS \ PUBLIC_PUBLIC_REGISTRABLE`
  (т.е. write-tools и `store_memory_batch`) **никогда** не
  регистрируется в публичном app, даже если явно перечислено в env.

Волны:

- **Wave 0 (по умолчанию)**: `search` и `fetch` — тонкие адаптеры с
  **минимальной схемой, ожидаемой ChatGPT/deep research**. `search`
  обязан вернуть как минимум `id` и `title`; `url` стандартом
  ожидается, добавляем; `snippet` — опциональное поле:
  - `search(query: str) -> {"results": [{"id","title","url",
    "snippet"?}, ...]}`. Под капотом — `search_memory(query)`. `id` —
    строковое представление `memory.id`; `title` — первые ≤80
    символов `text` (после `redact_text`); `url` — `mcp://ai-memory/<id>`
    (синтетический resolvable identifier для последующего `fetch`,
    не реальный HTTP URL).
  - `fetch(id: str) -> {"id","title","text","metadata"}` — достаёт
    конкретную запись по `id` через **новый read-only helper
    `get_memory_by_id(id)`** в [memory/search.py](D:/GitHub/AI-memory/memory/search.py).
    Helper делает SELECT по primary key с применением project
    allow-list (если запись не в allowed projects — возвращает
    `{"error":"not_found"}`, без раскрытия факта существования
    записи), redaction, `with_response_cap`. **Не использует**
    `get_recent_memory` — это была бы некорректная семантика.
  - Источник схемы: [Connectors and MCP servers — OpenAI](https://platform.openai.com/docs/guides/tools-connectors-mcp),
    [MCP servers for ChatGPT — OpenAI Developers](https://developers.openai.com/api/docs/mcp).
- **Wave 1**: + `search_memory`, `get_recent_memory`,
  `summarize_history` (родные read-tools AI-memory с богаче ответом).
- **Wave 2**: + `search_by_workitem`, `get_chat_history`,
  `analyze_session_outcomes`, `get_token_usage_history`.
- **Wave 3**: + `get_latest_workflow_memory`, `get_workflow_run_history`,
  `get_batch_run_history`, `get_portfolio_run_history`,
  `get_process_resource_state`. Полный read-набор.

Tools, не указанные в `AI_MEMORY_PUBLIC_ENABLED_TOOLS`, физически не
регистрируются — `tools/list` их не покажет. Write-tools и
`store_memory_batch` отфильтровываются из списка регистрации
white-list’ом, не black-list’ом — даже опечатка в env не откроет write.

## Файлы

### Создать
- [memory/public_app.py](D:/GitHub/AI-memory/memory/public_app.py) —
  `build_public_mcp_server()` (12 read-tools с обёртками),
  `build_public_starlette_app()`. Получает FastMCP, берёт его
  `streamable_http_app()`, надевает middleware-стек.
- [memory/public_security.py](D:/GitHub/AI-memory/memory/public_security.py) —
  `BearerAuthMiddleware` (chained tokens, OPTIONS/healthz exempt),
  `RateLimitMiddleware` (in-memory token-bucket per-IP),
  `BodySizeMiddleware` (1 MiB), `SecurityHeadersMiddleware`,
  `AuditMiddleware` (запись JSONL append-only с ротацией по размеру).
- [memory/public_filters.py](D:/GitHub/AI-memory/memory/public_filters.py) —
  `with_project_allowlist` (default-deny: `[]`/`None` = no projects),
  `with_limit_cap`, `with_response_cap` декораторы;
  `redact_metadata(items)` (ключи) и `redact_text(items)` (regex по
  содержимому `text`) пост-процессоры. Регистры паттернов: OpenAI keys
  (`sk-[A-Za-z0-9]{32,}`, `sk-proj-...`), GitHub (`ghp_[A-Za-z0-9]{36}`,
  `gho_/ghu_/ghs_/ghr_`), Slack (`xox[bp]-[A-Za-z0-9-]+`),
  AWS access key (`AKIA[A-Z0-9]{16}`), Google API
  (`AIza[A-Za-z0-9_-]{35}`), JWT (`eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+`),
  generic high-entropy строки длиной ≥40 после `password=`/`token=`/
  `api_key=`/`secret=`. Список расширяемый через env
  `AI_MEMORY_PUBLIC_REDACTION_EXTRA` (regex CSV).
  **`with_response_cap`** применяется на уровне **tool-wrapper**, не
  ASGI: оборачивает результат tool до возврата FastMCP. Считает
  байты JSON-сериализации `result` (включая поля, которые FastMCP
  затем размножит в `structuredContent` и `content[].text`); если
  превышает 256 KiB — урезает массив `items` (или эквивалент) до тех
  пор, пока размер не уложится, и добавляет `truncated=true`,
  `truncated_reason="response_cap"` в payload. Применять до FastMCP
  сериализации проще и безопаснее, чем переписывать ASGI-stream
  (streamable-http может идти как SSE — ASGI-wrapper там сложно).
  Тест-инвариант (см. `test_public_filters.py`) проверяет, что
  итоговый MCP JSON/SSE payload (включая `structuredContent` и
  `content[].text`) не превышает порог даже при искусственно
  раздутом источнике.
- [memory/public_daemon.py](D:/GitHub/AI-memory/memory/public_daemon.py) —
  single-instance daemon (свой lock, свой лог), запускает uvicorn с
  `access_log=False`, `workers=1`. Загружает токен через
  `memory.secrets`, открывает read-only SQLite handle. Содержит две
  функции payload’а:
  - `build_public_health_payload_minimal()` — `{"status":"ok",
    "role":"public-readonly"}`. Используется публичным `/healthz`,
    без auth. Не раскрывает enabled tools, проекты, пути, версии,
    hostname, pid.
  - `build_public_status_payload()` — расширенная диагностика
    (uptime, endpoint, enabled_tools, allowed_projects_count,
    lock/log paths, hostname). Доступна только через локальный CLI
    `public-status` (читает напрямую без HTTP) — полный payload не
    выдаётся наружу никогда.
- [memory/secrets.py](D:/GitHub/AI-memory/memory/secrets.py) —
  тонкий слой над `keyring` (Windows Credential Manager). Функции:
  `read_public_token()`, `read_public_token_previous()`, `write_public_token(value)`,
  `clear_public_token()`. Env-fallback `AI_MEMORY_PUBLIC_TOKEN[_PREVIOUS]`
  — только если в keyring пусто и явно разрешено флагом
  `AI_MEMORY_ALLOW_ENV_SECRETS=1` (для CI).
- [memory/public_db.py](D:/GitHub/AI-memory/memory/public_db.py) —
  `open_readonly_connection()` возвращает sqlite3.Connection,
  открытое как `file:...?mode=ro` с `PRAGMA query_only=ON`. Public
  search/get_* функции из `memory/search.py` оборачиваются так, чтобы
  использовать этот connection (либо patch через context, либо отдельный
  set of read-only функций — выбираем context-var, чтобы не дублировать
  код search.py).
- [scripts/generate_chatgpt_token.py](D:/GitHub/AI-memory/scripts/generate_chatgpt_token.py) —
  `secrets.token_urlsafe(32)`, по умолчанию пишет в keyring через
  `memory.secrets.write_public_token`. Печатает токен в stdout только
  при флаге `--print`. Никогда не пишет в env.
- [scripts/rotate_chatgpt_token.py](D:/GitHub/AI-memory/scripts/rotate_chatgpt_token.py) —
  двигает текущий токен в `_PREVIOUS`, генерирует новый, кладёт в
  keyring. Обоснование: ChatGPT-сторона при ротации некоторое время
  использует старый — chained validation позволяет zero-downtime.
- [scripts/configure_tailscale_funnel.ps1](D:/GitHub/AI-memory/scripts/configure_tailscale_funnel.ps1) —
  идемпотентная настройка `tailscale funnel` для `/mcp` и `/healthz`,
  плюс проверка наличия 2FA-флага через `tailscale status --json` и
  предупреждение, если 2FA не активна. Поддерживает `-Remove`.
- [docs/CHATGPT.md](D:/GitHub/AI-memory/docs/CHATGPT.md) — пошаговое
  руководство, threat model, security checklist, troubleshooting.
- [tests/test_public_app.py](D:/GitHub/AI-memory/tests/test_public_app.py) —
  по умолчанию (без env) ожидаем **только Wave 0** tools (`search`,
  `fetch`); при `AI_MEMORY_PUBLIC_ENABLED_TOOLS="search,fetch,
  search_memory,get_recent_memory,summarize_history,…"` — ровно тот
  набор, что в env; **write-tools отсутствуют ВСЕГДА** при любом
  значении env, даже если их явно перечислить в env. Отдельный
  тест-инвариант проверяет, что `register_write_tools` не вызывается
  из public-кода.
- [tests/test_public_security.py](D:/GitHub/AI-memory/tests/test_public_security.py) —
  Bearer 401/200, OPTIONS exempt, /healthz exempt, body-size 413,
  rate-limit 429, security headers присутствуют.
- [tests/test_public_filters.py](D:/GitHub/AI-memory/tests/test_public_filters.py) —
  project allow-list режет items, limit cap clamp'ит,
  `redact_metadata` маскирует подозрительные ключи и значения.
- [tests/test_public_db.py](D:/GitHub/AI-memory/tests/test_public_db.py) —
  попытка SQL `INSERT/UPDATE/DELETE` через read-only connection
  падает с `sqlite3.OperationalError`.
- [tests/test_public_audit.py](D:/GitHub/AI-memory/tests/test_public_audit.py) —
  AuditMiddleware пишет JSONL, не пишет токен/Authorization,
  ротация по размеру срабатывает.

### Изменить
- [memory/mcp_app.py](D:/GitHub/AI-memory/memory/mcp_app.py) — вынести
  тела `@mcp.tool()` в `register_read_tools(mcp, *, wrappers=None)` и
  `register_write_tools(mcp)`. `wrappers` — опциональный список
  декораторов-обёрток для read-tools (используется только публичным
  app'ом). `build_mcp_server()` зовёт оба регистратора без обёрток —
  поведение приватного 8765 не меняется.
- [memory/runtime_contract.py](D:/GitHub/AI-memory/memory/runtime_contract.py) —
  `PUBLIC_HOST="127.0.0.1"`, `PUBLIC_PORT=8767`, `PUBLIC_PATH="/mcp"`,
  `PUBLIC_HEALTH_PATH="/healthz"`,
  `PUBLIC_TOKEN_KEYRING_SERVICE="ai-memory-public"`,
  `PUBLIC_TOKEN_KEYRING_USER="active"`,
  `PUBLIC_TOKEN_PREVIOUS_KEYRING_USER="previous"`,
  `PUBLIC_TOOLS` (12-кортеж), `PUBLIC_LIMIT_CAP=50`,
  `PUBLIC_BODY_SIZE_BYTES=1_048_576`,
  `PUBLIC_RATE_LIMIT_PER_MINUTE=60`,
  `PUBLIC_FAILED_AUTH_BACKOFF_SECONDS=5`,
  `PUBLIC_FAILED_AUTH_THRESHOLD=5`. Хелперы
  `get_public_base_url/mcp_url/healthcheck_url`. Расширить
  `RuntimeContract`.
- [memory/config.py](D:/GitHub/AI-memory/memory/config.py) —
  `DEFAULT_PUBLIC_DAEMON_LOCK_PATH`, `DEFAULT_PUBLIC_DAEMON_LOG_PATH`,
  `DEFAULT_PUBLIC_AUDIT_LOG_PATH`,
  `DEFAULT_PUBLIC_AUDIT_MAX_BYTES = 10 * 1024 * 1024`,
  хелперы для них. Чтение allow-list проектов **default-deny**:
  `get_public_allowed_projects()` — читает env
  `AI_MEMORY_PUBLIC_ALLOWED_PROJECTS` (CSV) → list[str]; если переменная
  не задана или пуста — возвращает `[]` (никакой проект не разрешён).
  Чтение списка включённых tools: `get_public_enabled_tools()` — env
  `AI_MEMORY_PUBLIC_ENABLED_TOOLS` (CSV); default —
  `PUBLIC_COMPAT_TOOLS` (`search`, `fetch`). Финальный список =
  пересечение env-набора и `PUBLIC_PUBLIC_REGISTRABLE` (white-list).
- [memory/cli.py](D:/GitHub/AI-memory/memory/cli.py) — субкоманды
  `public-daemon`, `public-healthcheck`, `public-token-set`,
  `public-token-rotate`, `public-token-show-fingerprint`
  (последняя печатает SHA-256[:8] токена для проверки, что в keyring
  лежит ожидаемое значение, без раскрытия). Lazy import.
- [pyproject.toml](D:/GitHub/AI-memory/pyproject.toml) — runtime deps:
  `uvicorn>=0.30,<1.0`, `keyring>=25,<26`. Dev deps: `bandit>=1.7,<2.0`.
  Закоммитить `uv.lock`.
- [README.md](D:/GitHub/AI-memory/README.md) — раздел «Подключение из
  ChatGPT Plus» с happy-path и ссылкой на `docs/CHATGPT.md` и
  threat model.
- [AGENTS.md](D:/GitHub/AI-memory/AGENTS.md) — раздел «Public read-only
  endpoint» (English по L.3): порт 8767, токен в Windows Credential
  Manager (`ai-memory-public/active`), 12 read-tools,
  read-only SQLite, audit log в `data/logs/ai-memory-public-audit.jsonl`,
  ротация через `cli public-token-rotate`. Явно: токен **никогда** не
  попадает в `AI-memory`-записи и в код.
- [ARCHITECTURE.md](D:/GitHub/AI-memory/ARCHITECTURE.md) — диаграмма
  двух endpoints, threat model краткой формой, схема middleware-стека.
- [tests/test_runtime_contract.py](D:/GitHub/AI-memory/tests/test_runtime_contract.py) —
  ассерты на `PUBLIC_*` константы и `PUBLIC_TOOLS ⊂ EXPECTED_TOOLS`.
- [.gitignore](D:/GitHub/AI-memory/.gitignore) — убедиться, что
  `data/logs/ai-memory-public-audit.jsonl*` и
  `data/ai-memory-public-daemon.lock` исключены (вероятно покрыты
  существующим `data/`-glob, проверить).

### Не трогать
- [memory/daemon.py](D:/GitHub/AI-memory/memory/daemon.py),
  `server.py`, `memory/stdio_bridge.py` — приватный endpoint и stdio
  bridge остаются без изменений.

## Ключевой псевдокод

### `BearerAuthMiddleware` с chained tokens и backoff

```python
class BearerAuthMiddleware(BaseHTTPMiddleware):
    """Static Bearer с поддержкой текущего и предыдущего токенов
    (zero-downtime ротация). Backoff для адреса после серии 401."""

    def __init__(self, app, *, get_active_token, get_previous_token,
                 healthcheck_path, fail_threshold, fail_backoff_seconds):
        super().__init__(app)
        self._get_active = get_active_token
        self._get_previous = get_previous_token
        self._healthcheck_path = healthcheck_path
        self._fail_threshold = fail_threshold
        self._fail_backoff = fail_backoff_seconds
        self._fail_counters: dict[str, tuple[int, float]] = {}  # ip -> (cnt, until)

    async def dispatch(self, request, call_next):
        if request.method == "OPTIONS" or request.url.path == self._healthcheck_path:
            return await call_next(request)

        ip = (request.client.host if request.client else "unknown")
        now = time.monotonic()
        cnt, until = self._fail_counters.get(ip, (0, 0.0))
        if now < until:
            return JSONResponse({"error": "throttled"}, status_code=429,
                                headers={"Retry-After": str(int(until - now) + 1)})

        scheme, _, token = request.headers.get("authorization", "").partition(" ")
        active, previous = self._get_active(), self._get_previous()
        ok = (scheme.lower() == "bearer" and token and (
            (active and hmac.compare_digest(token, active)) or
            (previous and hmac.compare_digest(token, previous))
        ))
        if not ok:
            cnt += 1
            until = now + (self._fail_backoff if cnt >= self._fail_threshold else 0.0)
            self._fail_counters[ip] = (cnt, until)
            LOGGER.warning("public auth: rejected ip=%s path=%s cnt=%d",
                           ip, request.url.path, cnt)
            return JSONResponse({"error": "unauthorized"}, status_code=401,
                                headers={"WWW-Authenticate": "Bearer"})

        # Успех: сбрасываем счётчик
        self._fail_counters.pop(ip, None)
        return await call_next(request)
```

### `RateLimitMiddleware` token-bucket

```python
class RateLimitMiddleware(BaseHTTPMiddleware):
    """Простой in-memory token-bucket per-IP. Достаточно для одиночного
    daemon на 8767 — мы намеренно держим workers=1."""

    def __init__(self, app, *, capacity, refill_per_minute, healthcheck_path):
        super().__init__(app)
        self._capacity = capacity
        self._refill_rate = refill_per_minute / 60.0  # tokens/sec
        self._healthcheck_path = healthcheck_path
        self._buckets: dict[str, tuple[float, float]] = {}  # ip -> (tokens, last_ts)

    async def dispatch(self, request, call_next):
        if request.url.path == self._healthcheck_path:
            return await call_next(request)
        ip = request.client.host if request.client else "unknown"
        now = time.monotonic()
        tokens, last = self._buckets.get(ip, (float(self._capacity), now))
        tokens = min(self._capacity, tokens + (now - last) * self._refill_rate)
        if tokens < 1.0:
            self._buckets[ip] = (tokens, now)
            return JSONResponse({"error": "rate_limited"}, status_code=429,
                                headers={"Retry-After": "60"})
        self._buckets[ip] = (tokens - 1.0, now)
        return await call_next(request)
```

### `SecurityHeadersMiddleware`

```python
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["Strict-Transport-Security"] = "max-age=31536000"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["Referrer-Policy"] = "no-referrer"
        response.headers["Cache-Control"] = "no-store"
        response.headers["X-Frame-Options"] = "DENY"
        return response
```

### `BodySizeMiddleware`

Реализуется как ASGI middleware (не BaseHTTPMiddleware), чтобы успеть
прервать тело до `await receive()` буферизации. Стандартный паттерн
Starlette: считаем накопленный размер чанков, при превышении возвращаем
413.

### Аудит: HTTP-уровень + tool-уровень

Аудит разделён на две независимые точки записи, чтобы не трогать body
запроса в Starlette middleware (это конфликтует с FastMCP streamable-http
/ SSE).

**A. `AuditHttpMiddleware`** — пишет одну строку JSON на каждый запрос,
только метаданные транспорта: `ip`, `method`, `path`, `status`, `ms`.
Никогда не читает body, никогда не логирует заголовки. Этого достаточно
чтобы видеть auth-failures (401/429), ошибки клиента (4xx) и общую
нагрузку.

```python
class AuditHttpMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, *, log_path, max_bytes):
        super().__init__(app)
        self._writer = AuditWriter(log_path, max_bytes)

    async def dispatch(self, request, call_next):
        started = time.monotonic()
        ip = request.client.host if request.client else "unknown"
        method, path = request.method, request.url.path
        try:
            response = await call_next(request)
            status = response.status_code
        except Exception:
            self._writer.write({"ts": _utc_iso(), "kind": "http",
                                "ip": ip, "method": method, "path": path,
                                "status": 500, "ms": _ms(started),
                                "error": "exception"})
            raise
        self._writer.write({"ts": _utc_iso(), "kind": "http",
                            "ip": ip, "method": method, "path": path,
                            "status": status, "ms": _ms(started)})
        return response
```

**B. `with_audit` tool-wrapper** — пишет вторую строку JSON ТОЛЬКО с
именем tool, временем выполнения, статусом (`ok`/`error`) и числом
items в ответе. Без аргументов, без содержимого. Это место — внутри
FastMCP, после получения распарсенных параметров, поэтому ни Starlette
body, ни SSE не задеваются:

```python
def with_audit(tool_name: str, audit_writer: AuditWriter):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapped(*args, **kwargs):
            started = time.monotonic()
            try:
                result = fn(*args, **kwargs)
                items = (result or {}).get("items") if isinstance(result, dict) else None
                audit_writer.write({"ts": _utc_iso(), "kind": "tool",
                                    "tool": tool_name, "status": "ok",
                                    "ms": _ms(started),
                                    "item_count": len(items) if items else None})
                return result
            except Exception:
                audit_writer.write({"ts": _utc_iso(), "kind": "tool",
                                    "tool": tool_name, "status": "error",
                                    "ms": _ms(started)})
                raise
        return wrapped
    return decorator
```

`AuditWriter` (общий для обеих точек) инкапсулирует append-only запись
в JSONL и ротацию по `max_bytes`, под `threading.Lock`. Корреляция между
http- и tool-записями — по близкому `ts` и одинаковому окну запроса
(полный correlation-id вводить не будем — это требовало бы передавать
context через FastMCP, что усложняет).

    def _write(self, record):
        with self._lock:
            self._rotate_if_needed()
            with self._log_path.open("a", encoding="utf-8") as fh:
                fh.write(json.dumps(record, ensure_ascii=False) + "\n")

    def _rotate_if_needed(self):
        try:
            if self._log_path.stat().st_size > self._max_bytes:
                rotated = self._log_path.with_suffix(self._log_path.suffix + ".1")
                rotated.unlink(missing_ok=True)
                self._log_path.replace(rotated)
        except FileNotFoundError:
            pass
```

### Сборка ASGI-приложения

```python
def build_public_starlette_app(*, log_level="INFO",
                               health_payload_factory=None) -> Starlette:
    mcp = build_public_mcp_server(log_level=log_level)
    @mcp.custom_route(PUBLIC_HEALTH_PATH, methods=["GET"], include_in_schema=False)
    async def healthcheck(_):
        return JSONResponse((health_payload_factory or (lambda: {"status": "ok"}))())
    inner = mcp.streamable_http_app()
    inner.user_middleware.extend([
        Middleware(SecurityHeadersMiddleware),
        Middleware(BodySizeMiddleware,
                   max_bytes=PUBLIC_BODY_SIZE_BYTES,
                   healthcheck_path=PUBLIC_HEALTH_PATH),
        Middleware(RateLimitMiddleware,
                   capacity=PUBLIC_RATE_LIMIT_PER_MINUTE,
                   refill_per_minute=PUBLIC_RATE_LIMIT_PER_MINUTE,
                   healthcheck_path=PUBLIC_HEALTH_PATH),
        Middleware(BearerAuthMiddleware,
                   get_active_token=secrets_mod.read_public_token,
                   get_previous_token=secrets_mod.read_public_token_previous,
                   healthcheck_path=PUBLIC_HEALTH_PATH,
                   fail_threshold=PUBLIC_FAILED_AUTH_THRESHOLD,
                   fail_backoff_seconds=PUBLIC_FAILED_AUTH_BACKOFF_SECONDS),
        Middleware(AuditHttpMiddleware,                # только метаданные
                   log_path=get_public_audit_log_path(),
                   max_bytes=DEFAULT_PUBLIC_AUDIT_MAX_BYTES),
        # Имя tool пишется отдельной строкой через with_audit на уровне
        # tool-обёртки в register_read_tools(...wrappers=[with_audit, ...]).
    ])
    inner.middleware_stack = inner.build_middleware_stack()
    return inner
```

Порядок (внешний → внутренний): SecurityHeaders → BodySize → RateLimit
→ BearerAuth → Audit → FastMCP. RateLimit стоит до BearerAuth намеренно
— чтобы brute-force tools тоже отжимался по rate limit'у.

### Read-only SQLite handle

```python
def open_readonly_connection(db_path: Path) -> sqlite3.Connection:
    """Открывает SQLite в read-only режиме. Двойная защита: URI mode=ro
    + PRAGMA query_only=ON."""
    uri = f"file:{db_path.as_posix()}?mode=ro&immutable=0"
    conn = sqlite3.connect(uri, uri=True, check_same_thread=False)
    conn.execute("PRAGMA query_only = ON")
    conn.execute("PRAGMA temp_store = MEMORY")
    return conn
```

Public daemon переиспользует функции `memory.search.*` через
context-var: на старте — `set_db_connection_provider(open_readonly_connection)`,
существующий код, который сейчас открывает RW-соединение, через
context-var получает RO-handle. Это локальное изменение в
[memory/db.py](D:/GitHub/AI-memory/memory/db.py): обёрнутый в context-var
factory, default — current behavior (RW).

### Pre-call wrapping read-tools для allow-list и limit cap

```python
def with_project_allowlist(allow: list[str]):
    """Default-deny: allow == [] или None означает «никакой проект не разрешён».
    allow-all не существует как режим — это всегда явный список."""
    def decorator(fn):
        @functools.wraps(fn)
        def wrapped(*args, **kwargs):
            if not allow:
                return _empty_collection_response(reason="no_projects_allowed")
            requested = kwargs.get("project")
            if requested is not None and requested not in allow:
                return _empty_collection_response(reason="project_not_allowed")
            if requested is None:
                # tool без явного project — принудительно сужаем до allow-list
                kwargs["project"] = None  # search.py поймёт фильтр через allow
            result = fn(*args, **kwargs)
            return _filter_items_by_project(result, allow)
        return wrapped
    return decorator

def with_limit_cap(max_limit: int):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapped(*args, **kwargs):
            if "limit" in kwargs and isinstance(kwargs["limit"], int):
                kwargs["limit"] = min(kwargs["limit"], max_limit)
            return fn(*args, **kwargs)
        return wrapped
    return decorator
```

Эти декораторы передаются в `register_read_tools(mcp, wrappers=[...])`.

### `redact_metadata` post-processor

```python
SENSITIVE_KEY_RE = re.compile(
    r"(token|secret|password|api[_-]?key|private[_-]?key|credential)",
    re.IGNORECASE,
)

def redact_metadata(items: list[dict]) -> list[dict]:
    """Возвращает новый список items с замаскированными чувствительными
    ключами в metadata. Не модифицирует исходные объекты."""
    redacted = []
    for item in items:
        meta = item.get("metadata") or {}
        if isinstance(meta, dict) and any(SENSITIVE_KEY_RE.search(k) for k in meta):
            meta = {k: ("***redacted***" if SENSITIVE_KEY_RE.search(k) else v)
                    for k, v in meta.items()}
            item = {**item, "metadata": meta}
        redacted.append(item)
    return redacted
```

Применяется внутри обёрток read-tools перед возвратом.

## Шаги реализации

Каждый пункт ниже — отдельный work item в Assistant-UI согласно AGENTS K.2,
с зависимостями через `link_task_dependency`.

0. **Pre-flight (без кода)** — пользователь открывает ChatGPT в
   браузере и подтверждает наличие Developer Mode + Custom MCP
   Connectors на тарифе Plus. Если их нет — переключаемся в Fallback B
   (Assistant-UI как chat-клиент с OpenAI API). Этот шаг
   фиксируется заметкой в AI-memory (`kind=fact`,
   `project=AI-memory`), чтобы не возвращаться к проверке.
1. **Создать дерево задач** в Assistant-UI: родительский work item
   «ChatGPT Plus integration via public read-only AI-memory endpoint
   (defense-in-depth)», `project="AI-memory"`. 12 подзадач (по одной на
   шаги 2–13 + опциональный hardening 14), статус «Бэклог». Запросить
   у пользователя подтверждение → «К выполнению» (K.2). Без этого код
   не трогаем.
2. **Refactor [memory/mcp_app.py](D:/GitHub/AI-memory/memory/mcp_app.py)** —
   `register_read_tools(mcp, *, wrappers=None)`, `register_write_tools(mcp)`.
   Поведение приватного 8765 не меняется. Коммит:
   «refactor(mcp_app): вынес регистрацию read/write tools, поддержка wrappers».
3. **Расширить
   [memory/runtime_contract.py](D:/GitHub/AI-memory/memory/runtime_contract.py)** —
   константы и хелперы; обновить
   [tests/test_runtime_contract.py](D:/GitHub/AI-memory/tests/test_runtime_contract.py).
   Коммит: «feat(runtime_contract): зафиксированы публичный порт 8767 и security-параметры».
4. **Создать [memory/secrets.py](D:/GitHub/AI-memory/memory/secrets.py)**
   с keyring-обёрткой + env fallback. Зависимость `keyring` в
   `pyproject.toml`. Коммит: «feat(secrets): keyring-хранилище публичного Bearer-токена».
5. **Создать [memory/public_db.py](D:/GitHub/AI-memory/memory/public_db.py)**
   + context-var в [memory/db.py](D:/GitHub/AI-memory/memory/db.py).
   Тесты read-only поведения. Коммит: «feat(public_db): read-only SQLite handle через context-var».
6. **Создать
   [memory/public_filters.py](D:/GitHub/AI-memory/memory/public_filters.py)**
   + тесты `tests/test_public_filters.py`. Коммит:
   «feat(public_filters): project allow-list, limit cap, metadata redaction».
7. **Создать
   [memory/public_security.py](D:/GitHub/AI-memory/memory/public_security.py)**
   + тесты `tests/test_public_security.py`. Коммит:
   «feat(public_security): Bearer/RateLimit/BodySize/SecurityHeaders/Audit middleware».
8. **Создать
   [memory/public_app.py](D:/GitHub/AI-memory/memory/public_app.py)**
   + тесты `tests/test_public_app.py` и `tests/test_public_audit.py`.
   Коммит: «feat(public_app): сборка read-only ASGI приложения с middleware-стеком».
9. **Создать
   [memory/public_daemon.py](D:/GitHub/AI-memory/memory/public_daemon.py)**
   + расширить [memory/config.py](D:/GitHub/AI-memory/memory/config.py).
   Коммит: «feat(public_daemon): single-instance публичный daemon на 8767».
10. **CLI** — субкоманды в
    [memory/cli.py](D:/GitHub/AI-memory/memory/cli.py):
    `public-daemon`, `public-healthcheck`, `public-token-set`,
    `public-token-rotate`, `public-token-show-fingerprint`. Коммит:
    «feat(cli): команды управления публичным daemon и токеном».
11. **Скрипты** —
    `scripts/generate_chatgpt_token.py`,
    `scripts/rotate_chatgpt_token.py`,
    `scripts/configure_tailscale_funnel.ps1`. Коммит:
    «feat(scripts): генерация/ротация Bearer и настройка Tailscale Funnel».
12. **Документация** —
    [docs/CHATGPT.md](D:/GitHub/AI-memory/docs/CHATGPT.md) (включая
    threat model и security checklist), разделы в
    [README.md](D:/GitHub/AI-memory/README.md),
    [AGENTS.md](D:/GitHub/AI-memory/AGENTS.md),
    [ARCHITECTURE.md](D:/GitHub/AI-memory/ARCHITECTURE.md). Коммит:
    «docs: подключение ChatGPT Plus, threat model и security checklist».
13. **`pyproject.toml` + `uv.lock` + bandit** — `uvicorn`, `keyring` в
    runtime; `bandit` в dev. Запустить `uv run bandit -r memory/`.
    Коммит: «chore(deps): uvicorn, keyring и bandit; security-линт чистый».
14. **Hardening — отдельный low-privilege Windows user. Это
    обязательный security gate перед включением Tailscale Funnel
    наружу**. Без него RCE в public daemon (FastMCP / uvicorn /
    middleware / зависимостях) даёт права основного пользователя — а
    через Funnel этот daemon виден всему интернету. Подзадачи:
    - Создать локальную учётку `ai-memory-public` без членства в
      Administrators (`net user ai-memory-public <random> /add`).
    - ACL на исходники, venv и кэши uv:
      - `D:\GitHub\AI-memory\memory\` — Read+Execute для
        `ai-memory-public`.
      - `D:\GitHub\AI-memory\.venv\` — Read+Execute (нужен для
        запуска интерпретатора).
      - `D:\GitHub\AI-memory\pyproject.toml`, `uv.lock` — Read.
      - Отдельный кэш uv для этой учётки:
        `C:\Users\ai-memory-public\.cache\uv\` (env
        `UV_CACHE_DIR=...`); основной пользователь не делит кэш с
        public daemon’ом.
    - ACL на данные SQLite — обязательно покрыть **все три файла**
      WAL-режима:
      - `data\memory.db` — Read для `ai-memory-public`; Full для
        основного пользователя.
      - `data\memory.db-wal`, `data\memory.db-shm` — Read для
        `ai-memory-public`; Full для основного пользователя. Без
        этого SQLite в WAL-режиме упадёт при чтении даже с
        `mode=ro`, потому что ему нужен read-доступ к WAL-файлу.
    - ACL на каталог логов и lock’а:
      - `data\logs\public\` — Write для `ai-memory-public`. Все
        public-логи и audit-JSONL пишутся сюда; основной daemon в
        этот каталог не пишет.
      - `data\ai-memory-public-daemon.lock` — Write для
        `ai-memory-public`.
    - Запуск public daemon через Task Scheduler (`schtasks /create
      /RU ai-memory-public /RP ...`) или Windows Service с
      `RunAsUser=ai-memory-public`. Скрипт-обёртка
      `scripts/install_public_daemon_service.ps1`.
    - Keyring: токен сохраняется именно в Credential Manager этой
      учётки. Установка токена выполняется один раз через
      `runas /user:ai-memory-public "uv run python -m memory.cli
      public-token-set"`.
    - Documentation update в `docs/CHATGPT.md` (раздел «Hardening»).
    Коммит: «feat(hardening): отдельный low-priv пользователь
    ai-memory-public для public daemon».

После каждого шага: `uv run ruff check .` (M.2). После шагов 5–10 ещё
`uv run python -m unittest discover -s tests` (M.3). После шага 13 —
`uv run bandit -r memory/`. Push в remote — только по явному запросу
пользователя (N.1).

## Verification end-to-end

### 1. Линтер, тесты, security-static-analysis
```powershell
uv run ruff check .
uv run python -m unittest discover -s tests
uv run bandit -r memory/
```

### 2. Установка токена
```powershell
uv run python -m memory.cli public-token-set    # генерит и кладёт в keyring
uv run python -m memory.cli public-token-show-fingerprint
# выводит SHA-256[:8] хранимого токена; значение нигде не печатается
```

### 3. Локальный smoke
```powershell
Start-Process -NoNewWindow uv -ArgumentList @('run','python','-m','memory.cli','public-daemon')
uv run python -m memory.cli public-healthcheck
```
Ожидание payload’а с `"role":"public-readonly"`, `"port":8767`,
`"tool_names":[12]`, без значения токена.

### 4. Negative-матрица через curl (локально)
```powershell
$tok = (uv run python -c "import memory.secrets as s; print(s.read_public_token())")
# 401 без токена
curl.exe -i -H "Accept: application/json, text/event-stream" -X POST http://127.0.0.1:8767/mcp `
  -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
# 401 с неправильным
curl.exe -i -H "Authorization: Bearer wrong" -X POST http://127.0.0.1:8767/mcp `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
# повторить 5 раз → 6-й должен вернуть 429 (backoff)
# 200 с правильным
curl.exe -i -H "Authorization: Bearer $tok" -X POST http://127.0.0.1:8767/mcp `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
# 413 на большом теле
[byte[]]$buf = New-Object byte[] (2*1024*1024)
[IO.File]::WriteAllBytes("$env:TEMP\big.bin", $buf)
curl.exe -i -H "Authorization: Bearer $tok" -X POST http://127.0.0.1:8767/mcp `
  -H "Content-Type: application/octet-stream" --data-binary "@$env:TEMP/big.bin"
# /healthz без токена — 200 (но без чувствительных данных)
curl.exe -i http://127.0.0.1:8767/healthz
```

### 5. Read-only гарантия
```powershell
# Прямая попытка записи через read-only handle (должно упасть)
uv run python -c "from pathlib import Path; from memory.public_db import open_readonly_connection; \
  c = open_readonly_connection(Path('data/memory.db')); \
  c.execute('INSERT INTO memory(text) VALUES (\\'x\\')')"
# ожидание: sqlite3.OperationalError: attempt to write a readonly database
```

### 5a. Default-deny allow-list
```powershell
# Без AI_MEMORY_PUBLIC_ALLOWED_PROJECTS список пуст → ничего не возвращается
Remove-Item Env:\AI_MEMORY_PUBLIC_ALLOWED_PROJECTS -ErrorAction SilentlyContinue
curl.exe -i -H "Authorization: Bearer $tok" -X POST http://127.0.0.1:8767/mcp `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" `
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_recent_memory","arguments":{"limit":5}}}'
# ожидание: пустой items[] и filters.reason="project_not_allowed" в payload
```

### 5b. Text/metadata redaction
```powershell
# Положить в private daemon тестовую запись с секретом
uv run python -m memory.cli store --text "OPENAI_API_KEY=sk-proj-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA test" `
  --project portfolio --agent test --kind note `
  --metadata '{"token":"ghp_1234567890123456789012345678901234567890","note":"hi"}'
# Запросить через public endpoint
$env:AI_MEMORY_PUBLIC_ALLOWED_PROJECTS = "portfolio"
curl.exe -H "Authorization: Bearer $tok" -X POST http://127.0.0.1:8767/mcp `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" `
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_memory","arguments":{"query":"OPENAI_API_KEY","limit":5}}}'
# ожидание в ответе: text заменён на "***redacted***" в местах sk-proj-... и
# metadata.token == "***redacted***"
```

### 5c. Staged enablement
```powershell
# По умолчанию должны быть только Wave 0 (compat) tools — search и fetch
Remove-Item Env:\AI_MEMORY_PUBLIC_ENABLED_TOOLS -ErrorAction SilentlyContinue
# Перезапустить daemon, затем:
curl.exe -H "Authorization: Bearer $tok" -X POST http://127.0.0.1:8767/mcp `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" `
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
# ожидание: ровно 2 tool — search и fetch
```

### 5d. Write-tools отсутствуют ВСЕГДА
```powershell
# Попытка явно включить write-tool через env — должно игнорироваться
$env:AI_MEMORY_PUBLIC_ENABLED_TOOLS = "search,fetch,store_memory,store_memory_batch,set_process_resource_state"
# Перезапустить daemon, затем:
curl.exe -H "Authorization: Bearer $tok" -X POST http://127.0.0.1:8767/mcp `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" `
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
# ожидание: write-tools (store_memory, store_memory_batch, set_process_resource_state)
# отсутствуют в ответе. Тест-инвариант test_public_app.py проверяет это.
Remove-Item Env:\AI_MEMORY_PUBLIC_ENABLED_TOOLS
```

### 5e. Audit log содержит tool name
```powershell
# Сделать вызов tool и проверить, что в audit log есть строка с tool=search
curl.exe -H "Authorization: Bearer $tok" -X POST http://127.0.0.1:8767/mcp `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" `
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search","arguments":{"query":"test"}}}'
Get-Content data\logs\ai-memory-public-audit.jsonl -Tail 1 | ConvertFrom-Json | Select-Object tool, status
# ожидание: tool="search", status=200; arguments в логе НЕТ
```

### 6. Audit log
```powershell
Get-Content data\logs\ai-memory-public-audit.jsonl -Tail 20
# Каждая строка — JSON с ts/ip/method/path/status/ms; никакого Authorization, никакого тела.
```

### 7. Tailscale Funnel
```powershell
pwsh -File scripts\configure_tailscale_funnel.ps1
# Скрипт: проверяет наличие 2FA в tailscale status, идемпотентно настраивает /mcp и /healthz,
# распечатывает https://<machine>.<tailnet>.ts.net/mcp.
curl.exe -i "https://<machine>.<tailnet>.ts.net/healthz"
curl.exe -i -H "Authorization: Bearer $tok" -X POST "https://<machine>.<tailnet>.ts.net/mcp" `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### 8. ChatGPT
1. ChatGPT → Settings → Connectors → Advanced → Developer mode ON.
   **Зафиксировать наблюдаемое поведение** относительно встроенной
   ChatGPT Memory (отключилась / не отключилась / частично) — записать
   как `kind=fact, project="AI-memory"` в приватный AI-memory.
2. Add custom MCP connector: name `AI-memory`, URL
   `https://<machine>.<tailnet>.ts.net/mcp`, auth — Bearer + токен из
   keyring (вытащить через `public-token-show-fingerprint` нельзя —
   значение раскрывает только `--print` ключ скрипта, выполнить вручную).
3. В чате: «Use AI-memory: get_recent_memory project=portfolio limit=5».
4. Negative: убедиться, что `store_memory` отсутствует в `tools/list`.
5. Negative: попросить `search_memory query=secret password project=...` —
   ответы должны иметь поле `metadata.token = "***redacted***"`, если
   там были такие записи.

### 9. Ротация токена (zero-downtime)
```powershell
uv run python -m memory.cli public-token-rotate
# Старый токен теперь в _PREVIOUS, новый в active. ChatGPT-сторона
# продолжает работать со старым некоторое время — оба валидны.
# Через 24 часа: убедиться через audit log, что запросов со старым нет,
# затем `uv run python -m memory.cli public-token-rotate --drop-previous`
# уберёт старый из keyring.
```

## Подводные камни

1. **Lock-конфликт двух daemon'ов** — у публичного свой
   `data/ai-memory-public-daemon.lock`. SQLite WAL обеспечивает
   concurrent reader+writer; public daemon открывает соединение
   `?mode=ro` — двойная гарантия read-only.
2. **Утечка токена в логи** — `uvicorn(access_log=False)`, в Bearer
   middleware логируется только `ip`, `path`, счётчик отказов; в
   AuditMiddleware — `ip`, `method`, `path`, `status`, `ms`; никогда
   `Authorization`, никогда тело. Проверяется тестом
   `tests/test_public_audit.py`.
3. **OPTIONS preflight** — все middleware пропускают `OPTIONS` и
   `/healthz` без auth и rate-limit, иначе ChatGPT-клиент упрётся.
4. **`keyring` на Windows** — использует Credential Manager. При
   запуске под другим пользователем токен не виден — это нормально и
   намеренно. Daemon должен запускаться от того же пользователя,
   который установил токен.
5. **`tailscale funnel` не идемпотентен** — PS-скрипт сначала
   `tailscale funnel status --json` и сравнение; при дубле выйти 0;
   при конфликте — fail с подсказкой `-Force` (вызывает
   `tailscale funnel reset`).
6. **Built-in ChatGPT Memory** — Developer Mode её выключает. Это
   ожидаемая цена; в `docs/CHATGPT.md` указать явно.
7. **Hard cap на `limit`** — даже если ChatGPT попросит limit=10000,
   tool вернёт максимум 50. Это намеренное защитное поведение от
   массовой экфильтрации.
8. **`process_state` в публичном процессе** — `_CURRENT_STATE` живёт
   per-process; public daemon рапортует только своё состояние.
   Документировать.
9. **Token rotation race** — окно, когда `_PREVIOUS` ещё не записан, а
   `active` уже новый, минимизировано: write-операция в keyring сначала
   копирует active→previous, затем перезаписывает active. При сбое
   между шагами — нужен явный `public-token-rotate --resume`.
10. **`.gitignore`** — `data/logs/ai-memory-public-audit.jsonl*` и
    `data/ai-memory-public-daemon.lock` уже покрываются `data/` glob;
    проверить и при необходимости явно добавить.
11. **`bandit` ложноположительные** — `subprocess`, `random` (мы не
    используем), `assert` в тестах. Использовать `# nosec`-комментарии
    с обоснованием на русском, либо whitelist в `pyproject.toml`
    (`[tool.bandit]`).
12. **Размер audit log** — ротация по 10 MiB, хранится один rotated
    файл `.1`. Если пользователь хочет дольше — задокументировать как
    forwardable в внешнюю систему.
13. **Default-deny по умолчанию неудобен** — без явного
    `AI_MEMORY_PUBLIC_ALLOWED_PROJECTS` ChatGPT получит пустые ответы.
    Это намеренно: пользователь должен осознанно расширить allow-list
    (например, `portfolio,AI-memory`). В `docs/CHATGPT.md` приводим
    happy-path: «начните с одного проекта, расширяйте после
    наблюдения за audit log».
14. **Перекрытие Wave-флага и физическое отсутствие tool** — если
    `AI_MEMORY_PUBLIC_ENABLED_TOOLS` исключает tool, его нет в
    `tools/list`. Это сильнее, чем рантайм-rejection — подменить
    запросом нельзя. Документировать.
15. **Hardening шаг 14 — security gate перед включением Funnel наружу**.
    Без отдельного low-priv пользователя процесс уязвим к escalation,
    если в FastMCP/uvicorn/middleware/зависимостях найдут RCE. Чёткий
    разрыв через Windows-учётку — последняя линия обороны на уровне ОС.
    План не считается выполненным, пока шаг 14 не пройден, если
    Tailscale Funnel предполагается включать публично.
16. **Tailscale Funnel — это публичный ingress**. Funnel ACL не
    ограничивает кто может прийти на URL — поэтому полагаемся на
    Bearer + rate limit + audit. Документировать, что Funnel включается
    только когда нужен ChatGPT, и выключается командой
    `pwsh -File scripts\configure_tailscale_funnel.ps1 -Remove`,
    когда не нужен. Включение/выключение Funnel — это явное действие
    пользователя, не часть автозапуска.
17. **Healthcheck не должен быть источником утечки**. `/healthz` без
    auth отвечает только `{"status":"ok","role":"public-readonly"}` —
    не раскрывает enabled tools, проекты, пути, hostname, версии.
    Полная диагностика только через локальный CLI `public-status` (без
    HTTP), доступна только тому, кто залогинен в OS-сессию основного
    пользователя или low-priv пользователя.
18. **MVP-минимум перед Funnel**: `search`/`fetch` (Wave 0), default-deny
    project allow-list (явный непустой список), read-only DB, Bearer +
    keyring, audit (HTTP + tool), redaction, response cap, локальный
    smoke. Tailscale Funnel включаем **только** после прохождения
    шага 14 (low-priv user). Пока шаг 14 не пройден — daemon виден
    только на 127.0.0.1, ChatGPT к нему не ходит.

## Followup #124 (обнаружено в эксплуатации 2026-05-11): project-local Python runtime

### Контекст пробела

После применения `#123` hardening (`install_public_daemon_service.ps1
-CreateUser -RegisterTask`) выяснилось, что scheduled task падает с
`ERROR_LOGON_TYPE_NOT_GRANTED (0x80070569)` → выдали `SeBatchLogonRight`.
Потом — `cmd` под `ai-memory-public` даёт:

```
No Python at '"C:\Users\Zaxva\AppData\Local\Programs\Python\Python311\python.exe'
```

Причина:

- `.venv\Scripts\python.exe` — это 274 KB **shim/launcher**, не настоящий
  interpreter. `pyvenv.cfg` указывает на
  `C:\Users\Zaxva\AppData\Local\Programs\Python\Python311\python.exe`
  (per-user install в профиль Zaxva).
- `uv` лежит в `C:\Users\Zaxva\.local\bin\uv.exe` — тоже в Zaxva profile.
- uv-managed Python'ы — в `C:\Users\Zaxva\AppData\Roaming\uv\python\*` —
  тоже в Zaxva profile.
- `ai-memory-public` (по принципу #123) **не имеет** и не должна иметь
  доступа в `C:\Users\Zaxva\*`.
- `#123` дал ACL только на `memory/` и `.venv/`, но базовый интерпретатор
  и сам uv остались за пределами проекта.

То есть `#123` hardening **архитектурно неполный**: ACL на проектную
часть есть, а runtime ссылается наружу, в профиль основной учётки.

### Решение (выбрано): Project-local Python и uv

Втянуть весь runtime в проект:

- `D:\GitHub\AI-memory\.python\` — Python 3.11 через `uv python install
  --install-dir`. В `.gitignore`.
- `D:\GitHub\AI-memory\.tools\uv.exe` — копия uv. В `.gitignore`.
- `.venv` пересоздать так, чтобы `pyvenv.cfg` ссылался на `.python\`.
- ACL `ai-memory-public:RX` дополнительно на `.python\` и `.tools\`.
- Scheduled task action остаётся прежним:
  `cmd.exe /c set UV_CACHE_DIR=...&& ".venv\Scripts\python.exe" -m
  memory.cli public-daemon` — но теперь shim разрешается в доступный
  `.python\`.

Это согласуется с AGENTS.md «Project environment» (всё в `.venv` через
`uv`, без system Python; здесь дополнительно — без user-profile Python).

### Команды (для Codex; не запускать вручную)

```powershell
# Из elevated PS под Zaxva
$projectRoot      = "D:\GitHub\AI-memory"
$pythonInstallDir = Join-Path $projectRoot ".python"
$toolsDir         = Join-Path $projectRoot ".tools"

# 1) uv в .tools/
New-Item -ItemType Directory -Force -Path $toolsDir | Out-Null
Copy-Item "$env:USERPROFILE\.local\bin\uv.exe" (Join-Path $toolsDir "uv.exe") -Force

# 2) Python 3.11 в .python/
New-Item -ItemType Directory -Force -Path $pythonInstallDir | Out-Null
& (Join-Path $toolsDir "uv.exe") python install 3.11 --install-dir $pythonInstallDir

# 3) Получить путь только что установленного python.exe
$pyExe = (Get-ChildItem -Path $pythonInstallDir -Recurse -Filter "python.exe" |
          Select-Object -First 1).FullName

# 4) Пересоздать .venv с project-local Python
if (Test-Path "$projectRoot\.venv") {
    Remove-Item -Recurse -Force "$projectRoot\.venv"
}
Set-Location $projectRoot
& (Join-Path $toolsDir "uv.exe") venv --python $pyExe
& (Join-Path $toolsDir "uv.exe") sync --frozen --dev

# 5) ACL для ai-memory-public
icacls $pythonInstallDir /grant "ai-memory-public:RX" /T
icacls $toolsDir         /grant "ai-memory-public:RX" /T
```

### Изменения в скриптах и docs

- [scripts/install_public_daemon_service.ps1](D:/GitHub/AI-memory/scripts/install_public_daemon_service.ps1):
  - Добавить `Grant-PathRule (Join-Path $ProjectRoot ".python") "RX"` и
    `Grant-PathRule (Join-Path $ProjectRoot ".tools") "RX"`.
  - Исправить баг с регистрацией scheduled task: передавать **`-User`**
    и **`-Password`** в `Register-ScheduledTask` явно (сейчас передаётся
    только `-Principal` без пароля, и Windows валит регистрацию с
    «Неверное имя пользователя или пароль», даже если пароль валиден).
  - Добавить раздел про `SeBatchLogonRight` — выдать через `secedit
    /export /cfg ... /areas USER_RIGHTS`, патч SID, `secedit /configure`.
    Без этого права scheduled task с `LogonType Password` не запустится
    (получит `ERROR_LOGON_TYPE_NOT_GRANTED`).
  - В конец скрипта — постусловие: вывести инструкцию пользователю,
    как залогиниться под `ai-memory-public` через `Start-Process
    -Credential` (создаёт user profile + Credential Manager), чтобы
    `keyring`-based токен мог быть сохранён.
- [scripts/configure_windows_startup.py](D:/GitHub/AI-memory/scripts/configure_windows_startup.py) /
  любые runtime-setup скрипты — если они опираются на per-user Python,
  поправить аналогично через project-local `.python\`.
- [AGENTS.md](D:/GitHub/AI-memory/AGENTS.md) (раздел Project environment):
  явно зафиксировать, что для **hardened deployment** (production
  scheduled task под low-priv user) runtime обязан жить в `.python\`
  + `.tools\` внутри репо, а не в профиле основной учётки. Локальное dev
  по-прежнему допускает per-user Python.
- [.gitignore](D:/GitHub/AI-memory/.gitignore) — добавить `.python/` и
  `.tools/`.
- [docs/CHATGPT.md](D:/GitHub/AI-memory/docs/CHATGPT.md) — раздел
  «Hardening gate» дополнить: «Перед запуском
  `install_public_daemon_service.ps1` сначала выполните шаг
  `setup_project_python.ps1` (или эквивалент), чтобы `.python\` и
  `.tools\` существовали и были покрыты ACL».

### Verification

1. `uv run ruff check .` и `uv run python -m unittest discover -s tests`
   остаются зелёными.
2. `Get-Content D:\GitHub\AI-memory\.venv\pyvenv.cfg` показывает
   `home = D:\GitHub\AI-memory\.python\...`.
3. Под `ai-memory-public` (через `Start-Process -Credential`):
   `.venv\Scripts\python.exe -V` → `Python 3.11.x` без ошибок.
4. `schtasks /Run /TN "AI-memory-public-daemon"`; через ~5 секунд
   `Get-NetTCPConnection -State Listen -LocalPort 8767` показывает
   listener; `uv run python -m memory.cli public-healthcheck` →
   `{"status":"ok","role":"public-readonly"}`.

## Дальнейшие расширения (вне scope, на будущее)

- **Запись из ChatGPT через proposal queue** — даже если в будущем Plus
  получит write-tools через MCP (или мы захотим использовать Custom GPT
  Action для записи), наружу публикуется не `store_memory`, а
  `propose_memory(text, project, kind, metadata)`. Tool пишет не в
  основную таблицу `memory`, а в отдельную staging-таблицу
  `memory_proposals` (или append-only JSONL) с
  `proposed_by=chatgpt`, `status=pending`. Локальный reviewer в
  Assistant-UI показывает список pending, пользователь одобряет/правит
  → запись через приватный `store_memory`. Никогда не давать ChatGPT
  прямой write в production memory.
- **PM-MCP в ChatGPT** — повторить тот же паттерн отдельным public
  daemon’ом на `127.0.0.1:8769`, отдельный keyring-токен, путь Funnel
  `/pm-mcp`. Не мультиплексировать через AI-memory.
- **Свой домен** — заменить Tailscale Funnel на Cloudflare named tunnel
  + CNAME (~$8–12/год). Daemon и middleware остаются прежними.
- **Per-tool ACL гранулярнее** — pattern-based ACL на уровне аргументов
  (например, «search только по `kind in (decision,fact)`»), если в
  будущем захочется ещё более узкого scope.
- **Forwarding audit log** — отправка JSONL в внешний SIEM/файловую
  систему через лёгкий forwarder.
- **OAuth2 user flow** — при появлении необходимости персонального
  per-user-of-ChatGPT доступа (несколько людей с разными правами),
  заменить static Bearer на OAuth2 Resource Server. ChatGPT user-flow
  это умеет; client_credentials по-прежнему не поддержит.
- **Standard connector mode совместимость** — добавить тонкий adapter,
  который выставляет MCP tools `search` и `fetch` в схеме, ожидаемой
  ChatGPT deep research (см. [Connectors and MCP servers — OpenAI](https://platform.openai.com/docs/guides/tools-connectors-mcp)).
  Тогда тот же endpoint можно подключить и в standard режиме (без
  Developer Mode), что снимает зависимость от наличия DevMode в Plus.

## Fallback-варианты (если Pre-flight выявит, что в Plus нет Custom MCP)

### Fallback A — Custom GPT с Actions (deprecated, но рабочий)

Поднять отдельный HTTP-эндпоинт в Assistant-UI или в новом
`memory/public_actions.py` с двумя REST-операциями:
`POST /actions/memory/search` и `GET /actions/memory/recent`. OpenAPI
3.1 spec с Bearer auth. В ChatGPT Custom GPT Builder → Configure →
Actions → импорт OpenAPI URL. Все security-слои этого плана
(SecurityHeaders, BodySize, RateLimit, Bearer, Audit, response cap,
redaction, project allow-list, read-only SQLite) применяются ровно
так же. ChatGPT при этом — не основной чат, а отдельный Custom GPT
(нужно явно его выбирать в боковой панели).

### Fallback B — Assistant-UI как chat-клиент через OpenAI API

**Это уже не «ChatGPT Plus связан с AI-memory», а отдельный продуктовый
режим: локальный Assistant-UI использует OpenAI API.** ChatGPT-сайт и
подписка Plus в этом сценарии не задействованы — оплата идёт по запросам
(API key, отдельный биллинг). Пользователь должен это явно осознать,
прежде чем выбрать этот путь.

ChatGPT-сайт не используется. Пользователь общается через свой
[Assistant-UI](D:/GitHub/Assistant-UI) на `127.0.0.1:8000`, провайдер —
OpenAI (`ASSISTANT_LLM_PROVIDER=openai`, ключ в keyring через
`assistant-ui secrets set OPENAI_API_KEY`). Assistant-UI уже умеет
read+write AI-memory через приватный stdio/HTTP клиент. Это самый
управляемый и безопасный вариант: трафик не уходит за пределы
loopback, кроме API-вызова к OpenAI. Никакой Tailscale Funnel,
публичных endpoint’ов и keyring токенов для public daemon — не
нужно.

В этом варианте раздел «public daemon» данного плана **не
имплементируется вообще**; вместо него:
1. Установить `OPENAI_API_KEY` в keyring через
   `uv run assistant-ui secrets set OPENAI_API_KEY`.
2. Установить `ASSISTANT_LLM_PROVIDER=openai` и
   `ASSISTANT_LLM_MODEL=gpt-4.1-mini` (или другую модель).
3. Перезапустить Assistant-UI: `uv run assistant-ui serve`.
4. В UI на `http://127.0.0.1:8000/console` чат пользуется AI-memory
   через `MEMORY_ENABLED=true` и `with_memory_context()`.
