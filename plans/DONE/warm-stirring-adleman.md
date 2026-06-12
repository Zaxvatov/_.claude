# Obsidian read-only bridge для ChatGPT / Open WebUI

> **Изменение скоупа**: предыдущий план refactor'а lazy classification +
> bidirectional sync (история — `git log C:\Users\Zaxva\.claude\plans\warm-stirring-adleman.md`)
> отложен в backlog. Текущая задача — минимальная read-only интеграция:
> ChatGPT (через `https://laptop.tail19f97f.ts.net/mcp/read`) и Open WebUI
> читают `.md` файлы из Obsidian vault'а.

## Context

Сейчас (на основании Explore round 1):

- Gateway `/mcp/read` отдаёт 4 tools: `memory_search`, `memory_fetch`, `pm_list_work_items`, `pm_get_work_item` ([gateway/app.py:68-70](D:\GitHub\AI-Assistant\gateway\gateway\app.py)).
- Obsidian-код существует, но **узкий**: `pm-mcp-server/app/adapters/obsidian.py::read_obsidian_work_items` парсит только `- [ ]` tasks + `idea:` markers для `list_work_items`. Общего read для произвольных `.md` нет.
- Vault path только в env: `PM_MCP_OBSIDIAN_VAULT_PATH` + fallback `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` ([pm-mcp-server/app/config.py:14-16](D:\GitHub\AI-Assistant\pm-mcp-server\app\config.py), [assistant-ui/app/main.py:808](D:\GitHub\AI-Assistant\assistant-ui\app\main.py)). Default `None`. Менять — только рестартом с новым env.
- ADR-0014 ([docs/adrs/0014-openwebui-dual-protocol.md](D:\GitHub\AI-Assistant\docs\adrs\0014-openwebui-dual-protocol.md)) фиксирует: Open WebUI идёт через OpenAPI (`/openapi/{tool}`), не raw MCP. У pm-mcp-server есть hard-coded allowlist для `/openapi/...`.
- ADR-0001 D-3: gateway = единственный внешний ingress, scope tree жёсткий, новые scopes требуют ADR.
- Реальный vault: `G:\Мой диск\Obsidian\Vault\` (`.obsidian/` + ~255 `.md`).

**Решения пользователя:**

1. **Vault path** — через UI Settings (Настройки → Параметры), не env-only. Env остаётся как initial / fallback.
2. **Gateway scope** — новый `obsidian.read` через микро-ADR (ADR-0015). Clean separation от `memory.read`.
3. **Tool set** — full: `obsidian_list_files`, `obsidian_read_file`, `obsidian_search_text`, `obsidian_get_note`.

## Цели

1. ChatGPT через `https://laptop.tail19f97f.ts.net/mcp/read` (после re-auth с явно запрошенным `obsidian.read` scope) видит 4 read-tools для vault'а.
2. Open WebUI видит те же 4 tools через локальный OpenAPI bridge на pm-mcp-server (`http://127.0.0.1:8766/openapi`), как зафиксировано в ADR-0014. **Не через gateway** — gateway exposure для Open WebUI — отдельный future ADR.
3. Vault path конфигурируется через существующий UI `/settings` (новая вкладка Obsidian в табах [Счета/Категории/Параметры/Списки]) и применяется без рестарта сервисов.
4. **Read-only**: никакого write, никакого sync, никакого watchdog. Никаких новых daemon'ов.

## Целевая модель

| Клиент | Транспорт | Endpoint | Auth | Tools |
|---|---|---|---|---|
| ChatGPT | MCP через gateway | `https://laptop.tail19f97f.ts.net/mcp/read` | OAuth/PKCE + scopes `memory.read,pm.read,obsidian.read` (последний — **explicit opt-in** при регистрации клиента) | 4 + existing 4 |
| Open WebUI | OpenAPI loopback | `http://127.0.0.1:8766/openapi/obsidian_*` | local trust (loopback, ADR-0014) | 4 |
| Local Claude/Codex | MCP loopback | `127.0.0.1:8766` | local trust | все pm-mcp tools |

**Открытое для следующего ADR**: Open WebUI через gateway (если когда-нибудь понадобится unified URL). Сейчас local OpenAPI bridge — рабочий рабочий путь по ADR-0014.

## Architectural decisions

| # | Решение | Обоснование |
|---|---|---|
| 1 | Tools живут в **pm-mcp-server** (не новый MCP-сервер, не ai-memory) | Уже имеет Obsidian adapter ([pm-mcp-server/app/adapters/obsidian.py](D:\GitHub\AI-Assistant\pm-mcp-server\app\adapters\obsidian.py)) + vault env vars + OpenAPI surface. ADR-0001 D-8 (минимизация daemon'ов). Не размывает ai-memory тематически |
| 2 | Новый scope `obsidian.read` (ADR-0015) | Clean separation. Obsidian = knowledge source, не «память». ChatGPT в OAuth flow явно видит чему даёт доступ |
| 3 | Vault path через **persistent settings**, не только env | Запрос пользователя. Env остаётся fallback для initial bootstrap |
| 4 | Persistent storage — **shared JSON-файл** по computed path `<repo_root>/data/obsidian-settings.json` (см. решение #12, env override через `AI_ASSISTANT_OBSIDIAN_SETTINGS_PATH`) | Простота. И pm-mcp-server, и assistant-ui читают его. `mtime`-cache в pm-mcp-server для дешёвого re-read at each tool call. Не SQLite — оверкилл для одного значения. **Осознанное отклонение от tech-stack #4** (namespaced env vars, не JSON-config-file) — фиксируется в Ретроспективе как candidate brick |
| 5 | Path-traversal protection: tools принимают relative path, **resolve + is_relative_to(vault_path)** check (как в [assistant-ui/app/obsidian.py](D:\GitHub\AI-Assistant\assistant-ui\app\obsidian.py) для export) | Защита от `../`, абсолютных путей, symlink escape |
| 6 | Max file size — **1 MB** на `read_file`/`get_note`, **100 results** на `list_files`/`search_text` | Защита от response bloat (gateway имеет response cap 1MB по ADR-0001) |
| 7 | `search_text` — **простой substring grep**, не FTS | Минимальная реализация; vault ~255 файлов, sequential acceptable. FTS — отдельный план если понадобится |
| 8 | `get_note` возвращает **`{frontmatter: dict, body: str}`** через `yaml.safe_load` | Convenience для агента. Дополняет (не дублирует) `read_file`, который отдаёт raw markdown |
| 9 | Open WebUI работает **только через локальный OpenAPI bridge** `http://127.0.0.1:8766/openapi` (ADR-0014). Gateway exposure для Open WebUI — отдельный future ADR, в этом плане **не вводится** | Codex round 2 #2 + round 3 #1: ADR-0014 фиксирует local-only путь; внешний URL для Open WebUI — отдельная архитектурная работа |
| 10 | UI Settings — **расширение существующего `/settings`** ([assistant-ui/app/main.py:1457](D:\GitHub\AI-Assistant\assistant-ui\app\main.py)). Новый таб «Obsidian» в существующих [Счета/Категории/Параметры/Списки] ([settings.html](D:\GitHub\AI-Assistant\assistant-ui\app\templates\settings.html)). JS — отдельный `app/static/settings.js`, не inline. | Codex round 2: `/settings` уже существует; правило про разделение JS от шаблонов |
| 11 | `obsidian.read` НЕ добавляется в `DEFAULT_DCR_SCOPES` автоматически — explicit opt-in | Codex round 2 #3: Obsidian vault потенциально чувствительнее текущих PM summary tools. OAuth metadata публикует scope, README даёт команду регистрации с явным `--scopes memory.read,pm.read,obsidian.read` |
| 12 | Shared config path — через env override `AI_ASSISTANT_OBSIDIAN_SETTINGS_PATH` + computed default `<repo_root>/data/obsidian-settings.json` (relative path вычисляется через `Path(__file__).resolve().parents[N]`) | Codex round 2 #4: hardcoded абсолютный путь нарушает root AGENTS.md E.3 |
| 13 | Security hardening: only `*.md`, exclude `.obsidian/`/`.trash/`/hidden (`.*`); запрет absolute/UNC/drive paths и null-byte; `resolve(strict=True) + relative_to(vault_resolved)` перед каждым result | Codex round 2 #5: vault содержит sensitive структуры (`.obsidian/workspace`, `.obsidian/plugins`), внешний клиент не должен их видеть |
| 14 | Read budget: `read_file`/`get_note` default `128 KiB`, hard max `1 MiB`, `truncated:true` в response; `search_text` budget `max_files=500`, `max_total_bytes_scanned=10 MiB`, `timeout_ms=5000` | Codex round 2: OpenAPI cap `RESPONSE_CAP_BYTES = 256 * 1024` ([openapi_bridge.py:15](D:\GitHub\AI-Assistant\pm-mcp-server\app\openapi_bridge.py)); 1 MiB default будет молча резаться |
| 15 | mtime cache invalidation в `_load_vault_path()` — `(st_mtime_ns, st_size)` tuple, atomic compare-and-swap; не hash | Codex round 2: `st_mtime_ns` точнее, hash оверкилл для одного JSON-файла с atomic write |
| 16 | Frontmatter в `get_note`: dict-only check, лимит размера frontmatter (16 KiB), JSON-compatible нормализация дат/объектов; malformed YAML → fail-soft (`frontmatter:null`, `body=raw`) | Codex round 2: защита от размерного abuse и непредсказуемой YAML-сериализации |
| 17 | Calendar prior art: реализация следует pattern'у плана `buzzing-jumping-goose.md` (Google Calendar read-only) — `app/adapters/<domain>.py` DI-pattern, WorkItemDomain registration, env allowlist через существующий `PM_MCP_OPENAPI_TOOLS` | AI-memory recall ID 1603 (2026-06-05): ровно тот же паттерн уже реализован для calendar |

## План на 4 коммита

### C1 — ADR-0015: obsidian.read scope

**Цель**: зафиксировать архитектурное решение до кода.

**Файлы:**
- `D:\GitHub\AI-Assistant\docs\adrs\0015-obsidian-read-scope.md` (новый, **English**):
  - Status: accepted; Date: today.
  - Related: ADR-0001 (gateway scope tree), ADR-0014 (Open WebUI dual protocol).
  - **Context**: ChatGPT and Open WebUI need read-only access to user's Obsidian vault. Current gateway scopes (`memory.*`, `pm.*`) don't fit semantically (Obsidian is knowledge source, not memory or PM).
  - **Decision**:
    1. New scope `obsidian.read` with 4 tools: `obsidian_list_files`, `obsidian_read_file`, `obsidian_search_text`, `obsidian_get_note`.
    2. Tools hosted in `pm-mcp-server` (reuses existing Obsidian adapter and env). No new MCP server.
    3. Vault path: persistent setting via UI, env as bootstrap fallback.
    4. Read-only. No write tools in this scope. Future write needs separate scope + ADR.
  - **Alternatives considered**:
    - A) Extend `memory.read`. Rejected: semantic pollution; ChatGPT user can't distinguish what's accessed.
    - B) Generic `knowledge.read` (Notion/Obsidian/bookmarks). Rejected: premature abstraction; introduce when second source appears.
    - C) New `obsidian-mcp` daemon. Rejected: ADR-0001 D-8 (no new daemons unless justified).
  - **Consequences**: OAuth metadata `scopes_supported` publishes `obsidian.read`; `DEFAULT_DCR_SCOPES` does NOT include it (explicit opt-in only). ChatGPT re-auth needed once with explicit `--scopes obsidian.read` to acquire new scope.
- `D:\GitHub\AI-Assistant\docs\adrs\README.md` — index.

**Acceptance**: ADR-0015 формат соответствует ADR-0001..0014. README обновлён. Изменений кода нет.

### C2 — pm-mcp-server: 4 MCP tools + OpenAPI allowlist + path-traversal protection

**Цель**: реализовать read-tools в pm-mcp-server.

**Файлы:**
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\adapters\obsidian.py` — расширение существующего модуля. **Reference implementation**: следовать pattern'у `app/adapters/calendar.py` из плана `buzzing-jumping-goose.md` (AI-memory ID 1603) — DI-adapter, lifespan integration, EVENT_BUS НЕ нужен (static read).
  - Константы:
    - `EXCLUDED_TOP_DIRS = frozenset({".obsidian", ".trash"})` — никогда не возвращаются и не сканируются.
    - `ALLOWED_EXTENSIONS = frozenset({".md"})` — `read_file`/`get_note` отказывают на других расширениях с error `unsupported_file_type`.
    - `READ_DEFAULT_BYTES = 128 * 1024`, `READ_HARD_MAX_BYTES = 1 * 1024 * 1024`.
    - `SEARCH_MAX_FILES = 500`, `SEARCH_MAX_TOTAL_BYTES = 10 * 1024 * 1024`, `SEARCH_TIMEOUT_MS = 5000`.
    - `FRONTMATTER_MAX_BYTES = 16 * 1024`.
  - Helper `_resolve_safe(vault: Path, relative: str) -> Path`:
    1. Reject empty `relative`, любые null-байты (`\x00`), `\r`, `\n` в строке.
    2. Reject absolute paths (`os.path.isabs`), Windows drive letters (regex `^[A-Za-z]:`), UNC paths (`startswith(("\\\\", "//"))`).
    3. Normalize separators, запрет `..` сегментов после split.
    4. Проверка первого сегмента: если в `EXCLUDED_TOP_DIRS` или начинается с `.` → reject `excluded_path`.
    5. `candidate = (vault / relative).resolve(strict=True)` (strict — файл должен существовать; для list-операций — отдельный `_resolve_safe_dir` без strict).
    6. `candidate.relative_to(vault.resolve())` — ValueError → reject `path_escape`.
    7. Возвращает absolute Path.
  - Helper `_load_vault_path() -> Path | None`:
    - Источник конфига: env `AI_ASSISTANT_OBSIDIAN_SETTINGS_PATH` или default `_repo_root() / "data" / "obsidian-settings.json"` (где `_repo_root()` вычисляется через `Path(__file__).resolve().parents[N]` до root монорепо).
    - Cache invalidation: tuple `(st_mtime_ns, st_size)` через `os.stat`, не hash. Threadsafe через `threading.Lock`.
    - Чтение: `json.loads`; ожидается dict с ключом `vault_path` (string). Invalid JSON → log error + return None (НЕ молча падать в env fallback).
    - Fallback: env `PM_MCP_OBSIDIAN_VAULT_PATH`; final `None`.
    - Validation возвращаемого Path: `is_dir()` (не `is_file()`), warning в лог если нет `.obsidian/`.
  - Allowlist через env: переиспользовать существующий `OPENAPI_TOOLS_ENV = "PM_MCP_OPENAPI_TOOLS"` ([openapi_bridge.py:14](D:\GitHub\AI-Assistant\pm-mcp-server\app\openapi_bridge.py)) — добавить 4 tools в `DEFAULT_READ_TOOLS` (line 17-50). НЕ создавать `PM_MCP_OBSIDIAN_READ_TOOLS` отдельно (избегаем дублирования pattern'а).
  - `obsidian_list_files(folder: str | None = None, pattern: str = "*.md", limit: int = 100) -> dict`:
    - Только `*.md` (если pattern не `*.md`/`**/*.md` — reject `unsupported_pattern`).
    - Если `folder` задан → `_resolve_safe_dir(vault, folder)` (валидация без strict — каталог может не существовать).
    - `Path.rglob("*.md")` под scope; фильтр первого сегмента через `EXCLUDED_TOP_DIRS`/hidden.
    - Returns `{ok:true, files:[{path:relative, size, mtime:iso8601}], count, vault, truncated, vault_configured:true}`.
    - Если vault не сконфигурирован → `{ok:false, error:"vault_not_configured"}`.
    - Sort `mtime desc`, hard cap `limit=500`.
  - `obsidian_read_file(relative_path: str, max_bytes: int = READ_DEFAULT_BYTES) -> dict`:
    - `max_bytes` clamped в `[1024, READ_HARD_MAX_BYTES]`.
    - `_resolve_safe` + проверка `.md` extension.
    - **Streaming read префикса**: `open(path, "rb")` + `f.read(clamped_max_bytes + 1)`, не читать файл целиком. Если прочитано > `clamped_max_bytes` байт → `truncated:true`, content обрезается до `clamped_max_bytes`.
    - `size` берётся из `path.stat().st_size` (real file size, для UX).
    - UTF-8 decode с `errors="replace"`.
    - Returns `{ok:true, path, content, size, encoding:"utf-8", truncated}` — **никогда не отказывает на больших файлах**; контракт всегда возвращает доступный префикс (Codex round 3 #6: hard max — лимит ответа, не файла).
    - Errors (только path/типа): `unsupported_file_type`, `not_found`, `path_escape`, `excluded_path`, `vault_not_configured`. **Никаких `file_too_large`** — большой файл всегда возвращается с `truncated:true`.
  - `obsidian_search_text(query: str, folder: str | None = None, limit: int = 50, snippet_chars: int = 200) -> dict`:
    - Reject пустой query, query > 256 chars (`invalid_query`).
    - Budget: остановка scan если `scanned_files >= SEARCH_MAX_FILES` ИЛИ `scanned_bytes >= SEARCH_MAX_TOTAL_BYTES` ИЛИ `elapsed_ms >= SEARCH_TIMEOUT_MS`.
    - Case-insensitive substring через `str.casefold()`.
    - Returns `{ok:true, matches:[{path, line_number, snippet, mtime}], count, truncated, budget:{scanned_files, scanned_bytes, elapsed_ms}}`.
  - `obsidian_get_note(relative_path: str) -> dict`:
    - Использует `read_file` internally с `max_bytes=READ_HARD_MAX_BYTES` (чтобы frontmatter точно поместился, body может truncate'нуться).
    - Разделение frontmatter/body: regex `\A---\n(.*?\n)---\n` (multiline, non-greedy). Если frontmatter блок > `FRONTMATTER_MAX_BYTES` → fail-soft (`frontmatter:null`).
    - `yaml.safe_load`; если результат не `dict` → fail-soft.
    - Нормализация: datetime → ISO string, sets → lists, не-JSON-serializable значения → str() с warning.
    - Returns `{ok:true, path, frontmatter:dict|null, body:str, size, frontmatter_truncated:bool, body_truncated:bool}`.

- `D:\GitHub\AI-Assistant\pm-mcp-server\server.py` — регистрация 4 MCP tools через `@mcp.tool()` (по pattern existing tools).

- OpenAPI allowlist (ADR-0014, [pm-mcp-server/app/openapi_bridge.py:17-50](D:\GitHub\AI-Assistant\pm-mcp-server\app\openapi_bridge.py) `DEFAULT_READ_TOOLS`) — добавить 4 tool names.

- Errors: общий error code dictionary в этом файле или helper:
  - `vault_not_configured` (если `_load_vault_path()` → None).
  - `not_found` (404).
  - `path_escape` (400).
  - `excluded_path` (400, попытка обращения к `.obsidian/`/hidden).
  - `unsupported_file_type` (400, не `.md`).
  - `unsupported_pattern` (400, в `list_files`).
  - `invalid_query` (400, пустой query в search).

- `D:\GitHub\AI-Assistant\pm-mcp-server\pyproject.toml` + `uv.lock` — проверить, что `PyYAML` уже dependency (используется для frontmatter в `get_note`). Если нет — добавить `pyyaml>=6.0`.

**Тесты** `pm-mcp-server/tests/test_obsidian_read_tools.py` (новый):
- `obsidian_list_files`:
  - happy, subfolder filter, truncation, sort by mtime.
  - vault not configured → `ok:false, error:"vault_not_configured"`.
  - `.obsidian/`, `.trash/`, hidden directories — НЕ возвращаются.
  - non-.md pattern → `unsupported_pattern`.
- `obsidian_read_file`:
  - happy: файл < 128 KiB → `truncated:false`, content полный.
  - default cap: файл > 128 KiB → `truncated:true`, content обрезан до 128 KiB, `size` показывает real file size.
  - explicit `max_bytes=1_048_576` (hard max) → файл ≤ 1 MiB полный; > 1 MiB → `truncated:true`.
  - `max_bytes=2_000_000` → clamped до 1 MiB, поведение как выше.
  - `max_bytes=500` → clamped вверх до 1024 (нижний bound).
  - **Файл 100 MiB**: streaming read, **не падает** на memory, возвращает префикс 128 KiB с `truncated:true`.
  - non-.md file (например `.png`) → `unsupported_file_type`.
  - not_found.
- `obsidian_search_text`:
  - happy с match, no match.
  - empty/oversized query → `invalid_query`.
  - budget: vault > 500 файлов — scan останавливается, `budget.scanned_files==500`.
  - budget: timeout — synthetic large file → `elapsed_ms >= 5000`, partial results.
  - snippet correctness вокруг match position.
- `obsidian_get_note`:
  - happy с frontmatter (string/int/list/date).
  - no frontmatter → `frontmatter:null`, body=raw.
  - malformed YAML → fail-soft.
  - oversized frontmatter (> 16 KiB) → fail-soft.
  - frontmatter: non-dict (например `---\n- item1\n---`) → fail-soft.
- **Path escape comprehensive** (отдельный test class):
  - `relative_path="../../../etc/passwd"` → `path_escape`.
  - `relative_path="C:\\Windows\\System32"` (absolute Windows) → `path_escape`.
  - `relative_path="\\\\server\\share\\file.md"` (UNC) → `path_escape`.
  - `relative_path="file\x00.md"` (null-byte) → `path_escape`.
  - `relative_path=".obsidian/workspace"` → `excluded_path`.
  - `relative_path=".hidden/secret.md"` → `excluded_path`.
  - symlink escape (Unix-only через `os.symlink`, Windows skip): symlink в vault указывает наружу → `_resolve_safe` через `resolve(strict=True)` ловит, `relative_to(vault)` fails.
  - Windows junction (если есть admin) → аналогично.
- **OpenAPI bridge integration**:
  - 4 новых tools видны в `GET /openapi/openapi.json` после env `PM_MCP_OPENAPI_TOOLS=obsidian_list_files,obsidian_read_file,obsidian_search_text,obsidian_get_note` ИЛИ после добавления в `DEFAULT_READ_TOOLS`.
  - Write-tool (любой не в allowlist) через `/openapi/<name>` → 404.
  - CORS: запрос с `Origin: http://evil.com` → отклоняется (exact-origin); `http://127.0.0.1:8080` (default allowed) → проходит.
  - Response cap: synthetic 300 KiB response → `truncated:true` в payload (because `RESPONSE_CAP_BYTES = 256 * 1024`).
- Все tests через `tmp_path` с fake vault structure (`.md` files + `.obsidian/workspace` + `.trash/old.md` + hidden `.dotfile/secret.md`).

**Acceptance:**
- 4 MCP tools работают через ai-memory/pm-mcp-server loopback.
- OpenAPI: `GET /openapi/openapi.json` показывает 4 новых endpoints; `POST /openapi/obsidian_list_files` возвращает list.
- Path-traversal попытки отклонены тестами.
- `uv run pytest && uv run ruff check .` зелёные в `pm-mcp-server/`.

### C3 — gateway: obsidian.read scope + 4 routes + DCR scopes + OAuth metadata

**Цель**: экспонировать 4 tools через `/mcp/read`.

**Файлы:**
- `D:\GitHub\AI-Assistant\gateway\gateway\scope_policy.py`:
  - Добавить в `ALLOWLIST`:
    ```python
    "obsidian.read": frozenset({
        "/obsidian/list_files",
        "/obsidian/read_file",
        "/obsidian/search_text",
        "/obsidian/get_note",
    }),
    ```
  - `DENIED_TOOLS` — без изменений (никаких write для obsidian).

- `D:\GitHub\AI-Assistant\gateway\gateway\app.py`:
  - В `MCP_TO_ROUTE` добавить 4 entries: `obsidian_list_files` → `/obsidian/list_files` etc.
  - В `MCP_PROFILES["read"]` добавить 4 tool names (то же что в `obsidian.read` scope, но в read-профиле).
  - **Если есть отдельный profile per scope** — добавить `obsidian.read` profile.

- `D:\GitHub\AI-Assistant\gateway\gateway\backends.py`:
  - Routes `/obsidian/*` → handler, проксирующий на `pm_mcp` backend (loopback 8766) с tool names `obsidian_list_files` / `obsidian_read_file` / `obsidian_search_text` / `obsidian_get_note`.

- DCR (Dynamic Client Registration) — **`obsidian.read` НЕ добавляется в `DEFAULT_DCR_SCOPES`** (Codex round 2 #3). Default остаётся как сейчас: `memory.read`, `memory.propose`, `pm.read`, `pm.propose`, `pm.action` (sensitive scopes тоже не auto-grant). Регистрация клиента с явным opt-in:
  - `gateway/app.py` уже имеет explicit `DEFAULT_DCR_SCOPES` (не `frozenset(ALLOWLIST)`). **Сохранить existing default set без `obsidian.read`** — не добавлять новый scope автоматически. Это фиксирует паттерн для будущих sensitive scopes: `obsidian.read` остаётся opt-in.
  - Проверить любые места, где scopes-сет расширяется автоматически из `ALLOWLIST` (если такие есть) — исключить `obsidian.read`.

- OAuth metadata (`.well-known/oauth-authorization-server`): `scopes_supported` **публикует** `obsidian.read` (клиент может его запросить), но `register_client` без `--scopes obsidian.read` его не выдаст.

- `D:\GitHub\AI-Assistant\gateway\README.md`: добавить секцию «Sensitive scopes (explicit opt-in)» с командой:
  ```bash
  curl -X POST https://laptop.tail19f97f.ts.net/register \
    -H "Content-Type: application/json" \
    -d '{"client_name":"ChatGPT","scopes":["memory.read","pm.read","obsidian.read"]}'
  ```

**Тесты** `gateway/tests/test_obsidian_scope.py` (новый):
- `ensure_allowed("/obsidian/list_files", scopes={"obsidian.read"})` → ok.
- `ensure_allowed("/obsidian/list_files", scopes={"memory.read"})` → ScopeError.
- `ensure_allowed("/obsidian/list_files", scopes={"pm.read"})` → ScopeError.
- `ensure_allowed("/memory/search", scopes={"obsidian.read"})` → ScopeError (изоляция в обе стороны).
- DCR default: `register_client` без явных scopes → token не содержит `obsidian.read`.
- DCR explicit: `register_client(scopes=["obsidian.read"])` → token содержит scope.
- Rate-limit applied: 100+ запросов в минуту к `/obsidian/list_files` → 429.
- Redaction: response от `obsidian_read_file` содержит `sk-XXX` или email — gateway вырезает в audit log.
- Audit: каждый успешный `/obsidian/*` запрос → запись в audit log с hash chain.
- E2E с реальным backend mock: token с `obsidian.read` → POST `/obsidian/list_files` → 200 проксируется в pm-mcp.
- **Tool descriptor advertising vs scope enforcement** (Codex round 4 #4): `tools/list` на `/mcp/read` рекламирует Obsidian tools (через `MCP_PROFILES["read"]`) даже клиенту без `obsidian.read`. Это **ожидаемое поведение** (advertising ≠ access; защита через scope-enforcement при tool/call). Тест: клиент с `memory.read` только → `tools/list` показывает Obsidian tools в descriptor; `tools/call` для `obsidian_list_files` → 403 `insufficient_scope`. Альтернатива (отдельный `/mcp/obsidian/read` profile) **отклонена** — увеличивает MCP surface area без security gain.

**Acceptance:**
- `curl -H "Authorization: Bearer <token-with-obsidian.read>" -X POST https://laptop.tail19f97f.ts.net/obsidian/list_files -d '{}'` → 200.
- Token без `obsidian.read` → 403 `insufficient_scope`.
- `register_client` без явного `--scopes obsidian.read` → выданный токен НЕ имеет scope; `/obsidian/*` → 403.
- `.well-known/oauth-authorization-server` показывает `obsidian.read` в `scopes_supported` (доступен для запроса).
- README содержит инструкцию opt-in регистрации.
- `uv run pytest` зелёный в `gateway/`.

### C4 — assistant-ui: расширить существующий /settings + shared vault config + bootstrap env→config

**Цель**: дать пользователю UI для смены vault path без рестарта. **Расширение существующего `/settings`** ([main.py:1457](D:\GitHub\AI-Assistant\assistant-ui\app\main.py)), не новая страница.

**Файлы:**

- [assistant-ui/app/templates/settings.html](D:\GitHub\AI-Assistant\assistant-ui\app\templates\settings.html):
  - В `<md-tabs class="settings-tabs">` (line 23-28) добавить таб: `<md-primary-tab id="tab-obsidian" aria-controls="panel-obsidian">Obsidian</md-primary-tab>`.
  - Добавить `<section id="panel-obsidian" class="settings-panel" role="tabpanel" aria-labelledby="tab-obsidian" hidden>` с формой:
    - `<md-outlined-text-field name="vault_path" label="Путь к vault'у Obsidian">` (текущее значение из server-rendered context).
    - Кнопка `<md-filled-button>Сохранить</md-filled-button>`.
    - Status badge: `from settings.json` / `from env (default)` / `not configured`.
    - Validation hint: «Каталог должен содержать `.obsidian/`».
  - НЕ inline `<script>` — поведение в `app/static/settings.js`.

- `D:\GitHub\AI-Assistant\assistant-ui\app\static\settings.js` (новый):
  - `initObsidianTab()` — fetch текущего состояния через `GET /api/settings/obsidian`, рендер.
  - `submitObsidianVaultPath(event)` — XHR с `X-CSRF-Token` header → `POST /api/settings/obsidian`.
  - Loading/error states.

- `D:\GitHub\AI-Assistant\assistant-ui\app\main.py` (расширение существующих `/api/settings/*` endpoints, см. existing `/api/settings/project-colors` line 529 и `/api/settings/budget` line 1682):
  - `GET /api/settings/obsidian` — JSON: `{vault_path:str|null, source:"file"|"env"|"none", vault_exists:bool, is_valid_vault:bool, settings_file_path:str}`.
  - `POST /api/settings/obsidian` body `{vault_path:str}` — atomic write через tmp+rename в `obsidian_settings_path()`.
    - Validate: path может быть relative — convert to absolute через `Path(value).expanduser().resolve()`. **Не блокируем** если path сейчас не существует (network drive может быть offline) — возвращаем `vault_exists:false` для UX feedback, но сохраняем.
    - Reject: пустая строка, null-byte, control chars → 400 `invalid_path`.
  - В существующей рендеринг-логике `GET /settings` (line 1457) пробрасывать `obsidian_settings` контекст для server-rendered initial state.
  - При startup приложения — bootstrap migration (если файла нет и env задан — создать). Лог `"Created obsidian-settings.json from PM_MCP_OBSIDIAN_VAULT_PATH"`.

- `D:\GitHub\AI-Assistant\assistant-ui\app\security.py`: `/api/settings/obsidian` подпадает под существующий wildcard `/api/settings/*`, явная регистрация не нужна; проверить, что он уже включён.

- Shared config helper (новый модуль `D:\GitHub\AI-Assistant\assistant-ui\app\obsidian_settings.py`):
  - `obsidian_settings_path() -> Path`:
    - Env `AI_ASSISTANT_OBSIDIAN_SETTINGS_PATH` если задан.
    - Иначе computed: `Path(__file__).resolve().parents[2] / "data" / "obsidian-settings.json"` (= `<repo_root>/data/obsidian-settings.json`).
    - Никаких hardcoded абсолютных путей в коде.
  - `read_obsidian_settings() -> dict | None` с `(st_mtime_ns, st_size)` cache.
  - `write_obsidian_settings(vault_path: str) -> None` — atomic via tmp+rename.
  - **Тот же helper переиспользуется в pm-mcp-server** (импорт через прямой path или дублирование с identical контрактом — выбор по простоте; рекомендую дублирование, так как cross-subsystem import — отдельная архитектурная нагрузка).

- Shared config-файл: путь по `obsidian_settings_path()` (default `<repo_root>/data/obsidian-settings.json`). Структура:
  ```json
  {"vault_path": "G:\\Мой диск\\Obsidian\\Vault", "updated_at": "2026-06-05T12:00:00Z", "updated_by": "settings-ui"}
  ```

- Migration env → config (по [migration-discipline](D:\GitHub\_engineering_rules\skills\migration-discipline)):
  - Bootstrap migration на первом старте (как описано выше).
  - Env-vars **остаются** как fallback. Документация (AGENTS.md) объясняет иерархию: file → env.
  - **Не deprecate** env сразу.

**Тесты** `assistant-ui/tests/test_settings_obsidian.py` (новый):
- `GET /api/settings/obsidian` без файла + без env → `{vault_path:null, source:"none"}`.
- `GET /api/settings/obsidian` с env → `source:"env"`.
- `POST /api/settings/obsidian` valid → файл создан, `source:"file"`.
- `POST` с nonexistent path → 200 (сохраняем), `vault_exists:false`.
- `POST` с null-byte/control chars → 400 `invalid_path`.
- **CSRF**: POST без `X-CSRF-Token` → 403.
- **Anonymous**: GET/POST без сессии → 302 redirect на login.
- **Invalid JSON config**: файл содержит broken JSON → `_load_vault_path()` НЕ молча падает в env fallback (silent exposure), а логирует error и возвращает None.
- Bootstrap migration: файла нет + env есть → файл создан с правильным содержимым.
- Env override `AI_ASSISTANT_OBSIDIAN_SETTINGS_PATH`: файл пишется по указанному пути, не по дефолту.

**Acceptance:**
- `/settings` показывает новый таб «Obsidian» с текущим состоянием.
- Изменение path в UI → следующий MCP-вызов `obsidian_list_files` использует новый path без рестарта (mtime cache invalidates).
- Bootstrap migration работает однократно.
- `uv run pytest && uv run ruff check .` зелёные в `assistant-ui/`.

## Verification — isolated tests (обязательная automated часть)

Эти проверки гоняются в CI / локально через `uv run pytest` на изолированных `tmp_path` vault'ах. Не зависят от живых сервисов, реальных портов или реального vault'а.

1. **C2 unit tests** (`pm-mcp-server/tests/test_obsidian_read_tools.py`): все кейсы из секции «Тесты» C2 (happy/edge для 4 tools, OpenAPI bridge integration, path escape comprehensive с symlink/junction где доступно).
2. **C3 gateway tests** (`gateway/tests/test_obsidian_scope.py`): все кейсы из секции «Тесты» C3 (scope isolation двусторонняя, DCR default/explicit, rate-limit/redaction/audit, E2E с backend mock).
3. **C4 assistant-ui tests** (`assistant-ui/tests/test_settings_obsidian.py`): все кейсы из секции «Тесты» C4 (GET/POST, CSRF, anonymous, invalid JSON config, bootstrap migration, env override).
4. **Acceptance**: `uv run pytest && uv run ruff check .` зелёные в pm-mcp-server, gateway, assistant-ui.

## Verification — manual live smoke (rollout, не automated)

Эти шаги выполняются вручную при первом deploy / smoke-проверке окружения. Зависят от реальных сервисов на canonical портах и реального vault'а пользователя.

1. Поднять `ai-memory` (8765), `pm-mcp-server` (8766), `assistant-ui` (8000), `gateway` (8780).
2. Установить `PM_MCP_OBSIDIAN_VAULT_PATH=G:\Мой диск\Obsidian\Vault` (или другой реальный путь). Запустить assistant-ui — должен создаться `<repo_root>/data/obsidian-settings.json` (bootstrap migration).
3. **UI Settings**: `GET /settings` → видна секция Obsidian с `vault_path` и `source:"file"`. Изменить путь, сохранить.
4. **MCP loopback (Claude Code/Codex)**: `mcp__PM-MCP-server__obsidian_list_files` (после регистрации) → list файлов. Без рестарта pm-mcp.
5. **Search**: `obsidian_search_text({query: "OAuth"})` → matches в `.md` файлах с snippet'ами.
6. **Get note с frontmatter**: `obsidian_get_note({relative_path: "Inbox/idea-X.md"})` → `{frontmatter:{...}, body:"..."}`.
7. **ChatGPT через gateway**:
   - Re-register client с явным `--scopes memory.read,pm.read,obsidian.read` (см. README C3).
   - `curl -H "Authorization: Bearer <token>" -X POST https://laptop.tail19f97f.ts.net/obsidian/list_files -d '{}'` → 200.
   - Default DCR (без явного opt-in) → token без `obsidian.read`, любые `/obsidian/*` → 403.
8. **Open WebUI напрямую** (loopback, ADR-0014; **НЕ через gateway**):
   - `curl http://127.0.0.1:8766/openapi/openapi.json` → 4 новых tools в schema.
   - `POST http://127.0.0.1:8766/openapi/obsidian_list_files` → 200.
9. **ChatGPT в чате**: «Покажи мне идеи из Obsidian про OAuth» → агент вызывает `obsidian_search_text` → возвращает релевантные snippet'ы из vault'а.
10. **Settings UI live update**: изменить vault path через `/settings` → следующий `obsidian_list_files` (без рестарта pm-mcp) использует новый path в течение секунд.

Manual smoke не входит в automated CI (используется реальный vault и canonical ports — конфликтует с runtime verification brick и параллельным dev). Запускается вручную при deploy/rollout.

## Tech-stack кирпичики

(см. [tech-stack-choices.md](D:\GitHub\_engineering_rules\tech-stack-choices.md))

- **#1 NSSM Windows daemons** — без изменений (никаких новых сервисов).
- **#3 SQLite + FTS5** — не используется (search через простой grep по ~255 файлам acceptable).
- **#4 namespaced env vars** + **persistent JSON-config** — env остаётся для bootstrap; новый `obsidian-settings.json` для runtime override (UI). Иерархия: file → env → None.
- **#5 FastMCP loopback + Hybrid trust** (см. подсекцию Hybrid trust в #5) — 4 новых MCP tools через `@mcp.tool()` в pm-mcp-server loopback. External — через gateway scope `obsidian.read`.
- **#6 FastAPI + Jinja2 + Material Web** — Settings UI на тех же примитивах.
- **#10 Design-system** — Settings секция использует MD3 tokens + `<md-*>` components.
- **#13 YAML frontmatter через safe_load** — используется в `obsidian_get_note` для парсинга frontmatter.

**Не используются** в этом плане: #2 (embeddings), #7 (uv для нового сервиса), #8 (singleton-guard для нового daemon), #9 (xlsx), #11 (watchdog — explicitly excluded), #12 (background tasks — explicitly excluded).

**Prior art reference**: план `C:\Users\Zaxva\.claude\plans\buzzing-jumping-goose.md` (Google Calendar read-only, AI-memory ID 1603) — ровно тот же паттерн (read-only scope в pm-mcp-server, `app/adapters/<domain>.py` DI-pattern, env allowlist через `PM_MCP_OPENAPI_TOOLS`, OpenAPI bridge). Реиспользовать структуру при реализации C2/C3.

**Potential new brick (для ретроспективы)**: runtime JSON settings для shared конфигурации между subsystems (vault path в этом плане — первый use case). Если паттерн повторится 2+ раза — оформить как brick. Фиксируется в ретроспективе перед close_task (см. ниже).

**Отклонения от каталога**: persistent runtime JSON settings (`obsidian-settings.json`) вместо env-only config (#4) — осознанное отклонение, допустимо для no-restart UI override. Фиксируется в Ретроспективе как `needs-user-confirmation` или `follow-up-task` (candidate brick для tech-stack-choices если повторится 2+ раза).

## Cross-subsystem PM-MCP задачи

По правилу J.4 root AGENTS.md:

- **C1** → задача в `D:\GitHub\AI-Assistant` (ADR-0015 в `docs/adrs/`, root). Без зависимостей.
- **C2** → задача в `D:\GitHub\AI-Assistant\pm-mcp-server`. Зависит от C1.
- **C3** → задача в `D:\GitHub\AI-Assistant\gateway`. Зависит от C2.
- **C4** → задача в `D:\GitHub\AI-Assistant\assistant-ui`. Зависит от C2 (для общего contract на vault path).

Связи через `mcp__PM-MCP-server__link_task_dependency` с `dependency_project_path`. Использовать [pm-mcp-task-flow](D:\GitHub\_engineering_rules\skills\pm-mcp-task-flow) skill.

## Критичные файлы

**docs:**
- `D:\GitHub\AI-Assistant\docs\adrs\0015-obsidian-read-scope.md` (новый, English).
- `D:\GitHub\AI-Assistant\docs\adrs\README.md` — index.

**pm-mcp-server:**
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\adapters\obsidian.py` — `_resolve_safe`, `_load_vault_path`, 4 функции.
- `D:\GitHub\AI-Assistant\pm-mcp-server\server.py` — регистрация 4 MCP tools.
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\openapi.py` (или эквивалент) — OpenAPI allowlist update.
- `D:\GitHub\AI-Assistant\pm-mcp-server\pyproject.toml` + `uv.lock` — pyyaml если ещё не dep.
- `D:\GitHub\AI-Assistant\pm-mcp-server\tests\test_obsidian_read_tools.py` (новый).

**gateway:**
- `D:\GitHub\AI-Assistant\gateway\gateway\scope_policy.py` — `obsidian.read` scope.
- `D:\GitHub\AI-Assistant\gateway\gateway\app.py` — `MCP_TO_ROUTE` + `MCP_PROFILES["read"]`.
- `D:\GitHub\AI-Assistant\gateway\gateway\backends.py` — routes `/obsidian/*`.
- `D:\GitHub\AI-Assistant\gateway\README.md` — scope tree update.
- `D:\GitHub\AI-Assistant\gateway\tests\test_obsidian_scope.py` (новый).

**assistant-ui:**
- `D:\GitHub\AI-Assistant\assistant-ui\app\templates\settings.html` (новый или extension).
- `D:\GitHub\AI-Assistant\assistant-ui\app\main.py` — 3 новых endpoints + bootstrap migration в startup.
- `D:\GitHub\AI-Assistant\assistant-ui\app\security.py` — protected paths.
- `D:\GitHub\AI-Assistant\assistant-ui\tests\test_settings_obsidian.py` (новый).

**shared:**
- `D:\GitHub\AI-Assistant\data\obsidian-settings.json` — создаётся в runtime, в .gitignore.

## Соответствие правилам репозитория

- ADR-0015 — на English (root AGENTS.md).
- Code comments / commit messages / task statuses — на русском.
- Каждый коммит атомарен в `main` (J.4) и закрывает свою PM-MCP задачу.
- Никаких новых daemon'ов: tools в существующем pm-mcp-server (ADR-0001 D-8).
- Никаких write tools в `obsidian.read` scope (ADR-0001 D-3 — proposal-flow для write остаётся, в этом плане — N/A).
- [migration-discipline](D:\GitHub\_engineering_rules\skills\migration-discipline): bootstrap-migration env → JSON-config однократно при первом старте; env **не deprecate'итcя** (fallback bootstrap), но иерархия `file → env` явно документируется в AGENTS.md (раздел Project environment).
- [tech-stack-choices.md](D:\GitHub\_engineering_rules\tech-stack-choices.md) сверен — отклонений нет.

## Точки для следующего Codex review (round 3)

Раунд 2 закрыл 5 блокеров + основные оптимизации. Открытые точки:

1. **Frontmatter нормализация**: для `obsidian_get_note` дат и nested-objects — выбрано «datetime → ISO string, sets → lists, не-JSON-serializable → str() с warning». Достаточно ли это безопасно или нужна строгая JSON-валидация с reject?
2. **Bootstrap migration idempotency**: что если конфиг-файл создан вручную с пустым `{}`, а env задан? Migration должна override'ить или нет? Сейчас в плане — игнорировать env если файл существует. Подтвердить.
3. **DCR explicit opt-in** для `obsidian.read`: правильный ли механизм через `--scopes` параметр при регистрации, или нужен более продвинутый OAuth flow (например, incremental scope grant)?
4. **OpenAPI bridge cap 256 KiB** vs наш default 128 KiB на read: 128 KiB + frontmatter ~16 KiB = ~144 KiB, плюс JSON envelope ~ comfortable. Подтвердить, что мы не упираемся.
5. ~~**Cross-subsystem helper**~~ **Решено (Codex round 4 #5)**: дублирование `obsidian_settings.py` в обоих сервисах с identical контрактом. Shared package (`D:\GitHub\AI-Assistant\shared\`) НЕ вводится — это отдельное архитектурное решение, не нужно ради одного JSON-файла. Если паттерн повторится 2+ раза — отдельный план.
6. **Symlink/junction policy**: `resolve(strict=True)` ловит escape, но что если пользователь legitimate использует symlink внутри vault'а на другой каталог внутри vault'а (для cross-folder linking)? Сейчас план запрещает любые resolve которые приводят за пределы vault.resolve() — это правильно (legitimate intra-vault symlink остаётся valid), но стоит явно протестировать.

## Связь с backlog

Прежняя редакция этого же файла (`warm-stirring-adleman.md`) содержала план
refactor'а **lazy classification + Obsidian bidirectional sync**. Текущая
редакция (read-only bridge) **полностью заменяет** прежнюю по требованию
пользователя «упростим постановку».

Если в будущем понадобится:
- bidirectional sync (write half) — отдельный план с новым slug.
- lazy classification refactor существующего `capture_idea` — отдельный план.

Прежний контент не сохраняется в архиве в рамках этого плана; если нужно
восстановить — см. git history `C:\Users\Zaxva\.claude\plans\warm-stirring-adleman.md`.

Этот план **частично пересекается** с пост-MVP частью плана 590
(`D:\GitHub\AI-Assistant\docs\plans\DONE-590-thought-development-system.md`
если он там есть; центральный каталог `D:\GitHub\_engineering_rules\Plans\`
не существует). Read половина реализуется здесь; write — остаётся в
открытых вопросах.

## Central-plan-workflow contract

Актуальный contract (см. `D:\GitHub\_engineering_rules\skills\central-plan-workflow\SKILL.md`):

- Каталог планов — **`C:\Users\Zaxva\.claude\plans\`** (он же git-репозиторий
  и storage; `D:\GitHub\_engineering_rules\Plans\` **не существует** — это
  была моя галлюцинация в ранних правках плана).
- Slug — kebab-case ≤ 40 символов. **Имя файла не меняется** после создания.
  Текущий slug `warm-stirring-adleman` (auto-generated harness) остаётся.
- После approve и создания PM-MCP задач — записать строку `PM-MCP: #<id>`
  **внутри** плана (одна строка на C1, C2, C3, C4 — все ID в этой секции).
- Точечные правки через **Edit**, не Write/Create.
- Архивация: переместить файл в `C:\Users\Zaxva\.claude\plans\DONE\` (имя
  сохраняется, slug без префиксов) когда одновременно:
  1. PM-MCP задачи закрыты (все 4);
  2. outcome записан в AI-memory (через `ai-memory-capture` skill);
  3. архитектурные причины перенесены в ADR-0015;
  4. уникальные acceptance criteria не остались только в плане.

Перед закрытием PM-MCP задач — ретроспектива по 4 осям (tech-stack-choices,
Design-system, Skills, Hooks) с вердиктом `no-change | needs-user-confirmation
| follow-up-task`. См. секцию «Ретроспектива (заполнить перед close_task)».

## Cross-subsystem PM-MCP задачи (план создания)

По правилу J.4 root AGENTS.md:

- **Meta** → задача под `D:\GitHub\AI-Assistant` («Доработка плана warm-stirring-adleman.md по Codex round 2»). PM-MCP: **#826** (создана для unblock'а hook `require_active_task.py`, закрывается после approve и создания C1-C4)
- **C1** → задача под `D:\GitHub\AI-Assistant` («Принять ADR-0015 obsidian.read scope»). PM-MCP: **#827**
- **C2** → задача под `pm-mcp-server` («Реализовать 4 read-tools Obsidian + расширить OpenAPI allowlist»). Зависит от C1 (#827). PM-MCP: **#828**
- **C3** → задача под `gateway` («Добавить obsidian.read scope + 4 routes»). Зависит от C2 (#828). PM-MCP: **#829**
- **C4** → задача под `assistant-ui` («Расширить /settings новой вкладкой Obsidian + shared vault config»). Зависит от C2 (#828). PM-MCP: **#830**

Dependencies зарегистрированы через `link_task_dependency` с `dependency_project_path` (cross-repo). Все задачи в статусе `Бэклог`, готовы к подъёму Codex'ом начиная с C1 (#827).

Связи через `mcp__PM-MCP-server__link_task_dependency` с обязательным
`dependency_project_path` для cross-repo (C2→C1 в root vs pm-mcp-server,
C3 и C4 → C2 cross-repo). Использовать
[pm-mcp-task-flow](D:\GitHub\_engineering_rules\skills\pm-mcp-task-flow) skill.

## Ретроспектива (заполнить перед close_task)

| Ось | Изменения | Вердикт |
|---|---|---|
| tech-stack-choices.md | Runtime JSON settings для vault path реализованы как осознанное отклонение от env-only config; одноразовый use case, новый brick пока не нужен. | `no-change` |
| Design-system | `/settings#obsidian` использует существующие MD3 components и текущие tokens; новых primitives/design bricks не потребовалось. | `no-change` |
| Skills | Использованы существующие `central-plan-workflow`, `pm-mcp-task-flow`, `ai-memory-recall`, `impeccable`, `frontend-verification`; нового повторяемого workflow не появилось. | `no-change` |
| Hooks | `require_active_task.py` не имеет exception для plan-файлов в активном каталоге → catch-22 при первой правке плана. **Уже существует задача #708 [P0]** в `_engineering_rules` («проверить надёжность active-task guard», создана 2026-05-31, AI-memory ID 1483/1484, блокер для #709 консолидации общего слоя hook'ов). Не дублировать. Добавить evidence от 2026-06-05: тот же сценарий воспроизводится 100% — Edit заблокирован → `create_task` + `start_task` через MCP unblock'нет hook → дальше Edit'ы проходят. | `follow-up-task` (привязан к существующему #708 в `_engineering_rules`) |

