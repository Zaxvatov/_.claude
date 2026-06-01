# Gateway production tasks — drafts для PM-MCP

Создать после восстановления PM-MCP. Все задачи в подсистеме `gateway/`
монорепо `D:\GitHub\AI-Assistant`. Перед `create_task` проверить через
`list_projects`, какой `project_path` зарегистрирован для gateway:
- основной: `D:\GitHub\AI-Assistant\gateway`
- fallback (если gateway ещё не в registry): `D:\GitHub\AI-Assistant\pm-mcp-server`
  с пометкой [gateway/] в начале title

Все 5 задач — приоритет `высокий`, статус `Бэклог`. После создания
сразу `approve_task` (пользователь явно одобрил вариант B), и
`link_task_dependency` по графу ниже.

---

## Задача 8.1 — Подключить реальные backends gateway → ai-memory + pm-mcp-server

**Title:**
[Редизайн архитектуры | Шаг 8.1] Подключить реальные backends в gateway:
убрать _default_backend echo, реализовать HTTP/loopback forwarders на
ai-memory (127.0.0.1:8765 MCP) и pm-mcp-server (loopback MCP). Каждый
route из scope_policy.ALLOWLIST форвардится в соответствующий MCP-tool.
Включить integration тесты, которые поднимают gateway + ai-memory +
pm-mcp loopback и проверяют каждый allowlist-route на ожидаемый формат
ответа.

**Файлы:**
- `gateway/gateway/app.py` — пробросить `backends` через `create_gateway`.
- `gateway/gateway/backends.py` (новый) — forwarders для memory.* и pm.*.
- `gateway/tests/test_integration.py` (новый) — end-to-end сценарии.

**Acceptance:**
- `POST /memory/search` через gateway → возвращает реальные результаты
  из ai-memory `search_memory`.
- `POST /pm/list_work_items` → реальный список задач из pm-mcp-server.
- Все 5 scope/routes из ALLOWLIST покрыты тестами.

**Зависит от:** — (можно начинать сразу).

---

## Задача 8.2 — OAuth Authorization Code flow для ChatGPT Custom GPT Actions

**Title:**
[Редизайн архитектуры | Шаг 8.2] Реализовать полноценный OAuth 2.0
Authorization Code with PKCE flow в gateway: GET /authorize с web login
UI (минимальная HTML-страница с кнопкой "Allow access" + список запрошенных
scopes), CSRF protection, state parameter, redirect_uri allowlist. POST
/token расширить под exchange кода авторизации (сейчас принимает только
готовую пару code_verifier/code_challenge). Поддержать refresh_token
(TTL 30 дней) дополнительно к access_token (TTL 1 час).

**Файлы:**
- `gateway/gateway/oauth_flow.py` (новый) — authorize handler, code
  storage (in-memory с TTL), refresh token rotation.
- `gateway/gateway/app.py` — GET /authorize handler, расширенный POST /token.
- `gateway/templates/authorize.html` (новый) — login page.
- `gateway/tests/test_oauth_flow.py` (новый).

**Acceptance:**
- Полный flow: GET /authorize → user approval → redirect with code →
  POST /token с code+code_verifier → bearer + refresh.
- POST /token с grant_type=refresh_token выдаёт новый access.
- state, redirect_uri валидируются, без них — 400.

**Зависит от:** #8.1 (нужны рабочие backends, чтобы protected route
после OAuth реально что-то делал).

---

## Задача 8.3 — OAuth client registration + secure storage

**Title:**
[Редизайн архитектуры | Шаг 8.3] Реализовать регистрацию OAuth-клиентов
в gateway: CLI `python -m gateway.clients register --name chatgpt-prod
--redirect-uri <url> --scopes memory.read,memory.propose,pm.read,pm.propose`
выдаёт client_id (UUID) и client_secret (random 32 bytes hex). Secret
хранится только в виде argon2/bcrypt-хеша в SQLite
`gateway/data/clients.sqlite3`. CLI команды: register, list, revoke,
rotate-secret.

**Файлы:**
- `gateway/gateway/clients.py` (новый) — SQLite registry + CLI.
- `gateway/gateway/app.py` — `client_id` валидируется в /authorize и /token.
- `gateway/tests/test_clients.py` (новый).

**Acceptance:**
- CLI register выдаёт пару, secret хешируется.
- /authorize отклоняет запрос с неизвестным client_id или
  mismatched redirect_uri.
- /token отклоняет запрос без правильного client_secret (для
  confidential clients) или без PKCE (для public clients как ChatGPT).
- revoke помечает клиента invalid, его токены перестают работать.

**Зависит от:** #8.2.

---

## Задача 8.4 — Production signing secret + rotation policy

**Title:**
[Редизайн архитектуры | Шаг 8.4] Заменить dev-only signing_secret на
required env var: gateway должен ABORT при запуске если
AI_ASSISTANT_GATEWAY_SECRET не задан или содержит "dev-only-change-me".
Секрет генерировать минимум 64 случайных байта (base64url). Документировать
процедуру rotation каждые 90 дней (Windows Scheduled Task или ручная):
старый секрет остаётся валидным для verify_token ещё 2 часа после
смены (overlap window), все новые tokens — с новым секретом.

**Файлы:**
- `gateway/gateway/config.py` — assertion на secret presence/quality.
- `gateway/gateway/auth.py` — multi-secret verify (current + previous).
- `gateway/scripts/rotate-secret.ps1` (новый) — генерация + установка
  в OS keyring или защищённый файл.
- `gateway/README.md` — раздел Secrets Management.

**Acceptance:**
- `uv run python -m gateway.app` без env → exit 1 с понятной ошибкой.
- Rotation script меняет SECRET, существующие tokens продолжают работать
  2 часа, затем отклоняются.

**Зависит от:** — (можно параллельно с #8.1).

---

## Задача 8.5 — OpenAPI schema + ChatGPT Custom GPT Action wiring

**Title:**
[Редизайн архитектуры | Шаг 8.5] Подготовить OpenAPI 3.1 спецификацию
gateway для ChatGPT Custom GPT Actions: все routes из scope_policy.ALLOWLIST,
security scheme OAuth2 authorizationCode с указанием
authorization_url=/authorize, token_url=/token, scopes (memory.read,
memory.propose, pm.read, pm.propose, pm.action). Schema опубликовать
как `gateway/openapi.yaml` и через GET /openapi.yaml endpoint.

**Файлы:**
- `gateway/openapi.yaml` (новый) — спецификация.
- `gateway/gateway/app.py` — GET /openapi.yaml handler (отдаёт файл).
- `gateway/tests/test_openapi.py` (новый) — валидация openapi-spec-validator.

**Acceptance:**
- `gateway/openapi.yaml` проходит swagger/openapi validator.
- GET /openapi.yaml возвращает YAML с правильным Content-Type.
- В ChatGPT (Custom GPT Builder → Actions → Import from URL) спецификация
  загружается без ошибок, видно все endpoints.

**Зависит от:** #8.1, #8.2.

---

## Задача 8.6 — External exposure через Tailscale Funnel + TLS

**Title:**
[Редизайн архитектуры | Шаг 8.6] Настроить external exposure gateway:
PowerShell скрипт `gateway/scripts/expose-tailscale.ps1` запускает
`tailscale serve` (Tailnet-only, для теста) или `tailscale funnel`
(публичный internet, для ChatGPT cloud) на 443 → 127.0.0.1:8780.
Документировать в README выбор между serve и funnel. TLS termination
делает Tailscale автоматически. Health check: после запуска
GET https://laptop.tail19f97f.ts.net/healthz → 200.

**Файлы:**
- `gateway/scripts/expose-tailscale.ps1` (новый).
- `gateway/README.md` — раздел External Exposure.

**Acceptance:**
- Скрипт идемпотентен (повторный запуск не ломает).
- `curl https://laptop.tail19f97f.ts.net/healthz` → `{"ok":true,"role":"gateway"}`.
- POST /memory/search с валидным bearer возвращает реальные данные.

**Зависит от:** #8.4 (production secret должен быть установлен до того,
как gateway виден из интернета).

---

## Граф зависимостей

```
#8.1 backends ────┬───→ #8.2 OAuth flow ────→ #8.3 client registry
                  └───→ #8.5 OpenAPI ←────────┘
#8.4 secret ─────────────────────→ #8.6 external exposure
```

После завершения всех 6 — gateway production-ready. Тогда ChatGPT
Custom GPT Action настраивается через:

- Authentication: OAuth
- Client ID / Secret: из `python -m gateway.clients register --name chatgpt-prod ...`
- Authorization URL: `https://laptop.tail19f97f.ts.net/authorize`
- Token URL: `https://laptop.tail19f97f.ts.net/token`
- Scope: `memory.read memory.propose pm.read pm.propose`
- Schema: импорт из `https://laptop.tail19f97f.ts.net/openapi.yaml`

---

## Команды для создания (после восстановления PM-MCP)

```
list_projects  # узнать актуальный project_path для gateway

# Для каждой задачи:
create_task(project_path, title, priority="высокий")
approve_task(project_path, task_id)  # Бэклог → К выполнению (юзер одобрил вариант B)

# Связи:
link_task_dependency(8.2 → 8.1)
link_task_dependency(8.3 → 8.2)
link_task_dependency(8.5 → 8.1)
link_task_dependency(8.5 → 8.2)
link_task_dependency(8.6 → 8.4)
```
