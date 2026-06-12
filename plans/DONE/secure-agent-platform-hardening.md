# Secure AI-Agent Platform Hardening

> Статус: **draft / на согласовании с Codex**
> Автор аудита: Claude (Principal Security Architect / Red Team)
> Дата: 2026-06-09
> PM-MCP: #853 (umbrella планa; per-subsystem задачи создаются после утверждения направления)
> Связанные ADR: ADR-0001 (target architecture), кандидаты ADR-0018 (owner-auth + удаление legacy-grant), ADR-0019 (service identity layer)

---

## 1. Контекст

Монорепо персонального AI-ассистента (5 подсистем) выставлен в публичный интернет
через Tailscale **Funnel** → `gateway` (:8780) → loopback-бэкенды
(`ai-memory` :8765/:8770, `pm-mcp-server` :8766, `budget` :8767, `assistant-ui` :8000).
Funnel = публичный ingress (`gateway/scripts/install-funnel-task.ps1`, режим по умолчанию `funnel`).

Проведён полный security-аудит (architecture discovery → threat model → code review).
Найдено 11 находок, из них 2 критических на пути аутентификации gateway и
3 high по доверию loopback / памяти / агентному runtime. Полный отчёт вынесен в
durable-артефакт `docs/security/audit-2026-06.md` (threat model, exploit-пробы по
каждой находке, affected endpoints, severity rationale, regression mapping; создаётся
в P0). Ключевые находки сведены в §5 и в таблице ниже.

Цель этого плана — превратить репозиторий в production-grade secure AI-agent
platform: закрыть критические дыры немедленно и заложить слой идентичности/политик,
который снимает «localhost-предположение» и готовит систему к multi-user / remote /
делегированным агентам / внешним MCP.

## 2. Цель

1. Закрыть полный обход аутентификации gateway (F-1, F-2) **до любого следующего включения/перезапуска Funnel или Gateway (gate от 2026-06-11)**.
2. Заменить «loopback = доверие» на явную идентичность сервисов с fail-closed (F-3).
3. Закрыть прямое отравление памяти и неконтролируемый агентный tool-loop (F-4, F-5).
4. Создать недостающие security-кирпичики (identity, policy, secrets, audit, monitoring).
5. Заложить контрактные изменения (principal / tenant_id) сейчас, чтобы не рефакторить под multi-user позже.

## 3. Ограничения и инварианты

- **Migration discipline** (AGENTS.md Q.1): новый путь → миграция зависимостей → удаление старого. Никаких backwards-compat shim'ов. Особенно касается F-1 (legacy-grant удаляется целиком, а не «латается»).
- **Subsystem boundaries** (ADR-0001): gateway — единственный внешний ingress; бэкенды loopback; смена контракта между подсистемами → ADR + `link_task_dependency`.
- **Один атомарный коммит** на логическую cross-subsystem единицу (AGENTS.md J.4).
- Каждое изменение Python проходит `uv run ruff check .` + тесты затронутой подсистемы (AGENTS.md M.2/M.3).
- Не ломать живые ChatGPT-коннекторы без явного rollout-плана (DCR scope иммутабелен — AI-memory id 1630).

## 4. Tech-stack bricks: применимость и отклонения

Проверено против `D:\GitHub\_engineering_rules\tech-stack-choices.md`.

**Релевантные кирпичики (используем как есть):**
- #3 SQLite+FTS5+WAL — для новых таблиц identity/policy/audit.
- #4 namespaced env vars — для конфигов секретов/политик.
- #5 FastMCP + runtime_contract + fail-open MCP client — расширяем, см. отклонение ниже.
- #7 uv + per-subsystem .venv.
- #8 singleton-guard — без изменений.
- #17 runtime verification (Track B) — для контрактных тестов auth.
- #19 pwsh 7 — для rollout-скриптов секретов.

**⚠ Явное отклонение от кирпичика #5 (Hybrid trust model):**
Кирпичик #5 фиксирует «loopback (127.0.0.1) — trust-by-default, **без auth**».
Находка F-3 показывает, что это предположение — единственное, на чём держится вся
безопасность бэкендов, и оно несовместимо с заявленным будущим (multi-user, remote,
делегированные агенты, внешние MCP). План **сознательно отклоняется**: вводит
**fail-closed S2S-аутентификацию на loopback** (подписанные service-токены / общий
bearer на переходный период) + DNS-rebinding protection на :8765/:8766/:8767.

Причина отклонения: «loopback ≠ trust» как только появляется второй хост/пользователь
или SSRF/DNS-rebinding-вектор; явная идентичность работает одинаково локально и
распределённо. Локальные агенты (Claude/Codex/Ollama) получают service-токен через
общий secrets-слой.

✅ Кирпичики каталога приведены в соответствие (2026-06-09): #5 обновлён («было/стало» loopback≠trust),
#3/#6 уточнены, добавлены #22–#25. См. `D:\GitHub\_engineering_rules\tech-stack-choices.md`
и AI-memory id 1633. Ниже — только оставшаяся реализационная работа (WP-*).

## 5. Находки (сводка)

| ID | Sev | Суть | Файлы |
|----|-----|------|-------|
| F-1 | 🔴 Critical | `/token` без grant_type выпускает JWT с любыми scope (самосогласованный PKCE) | `gateway/gateway/app.py:266-300`, `auth.py:46-83` |
| F-2 | 🔴 Critical | Нет аутентификации владельца ресурса: открытая DCR + декоративный consent | `gateway/gateway/oauth_flow.py:43-78`, `app.py:357-390` |
| F-3 | 🟠 High | Loopback-бэкенды без auth (incl. destructive); `verify_authorization_header` fail-open; MCP-mount исключён | `pm-mcp-server/app/http_transport.py:77-97`, `app/auth.py:30-36`, `ai-memory/memory/mcp_app.py:118-168` |
| F-4 | 🟠 High | `store_memory` в обход secret-scan/quota/review → отравление памяти | `ai-memory/memory/mcp_app.py:118-168` vs `proposals/intake.py` |
| F-5 | 🟠 High | Агентный tool-loop авто-исполняет всё (подтверждение у 2 tools); injection из retrieved-content | `assistant-ui/app/agent_loop.py:30-32,110-219` |
| F-6 | 🟡 Med | Тела запросов в аудите в открытом виде; redaction узкий | `gateway/gateway/audit.py:51-73`, `redaction.py` |
| F-7 | 🟡 Med | Hash-chain аудита не tamper-evident + гонка под потоками | `gateway/gateway/audit.py:37-73` |
| F-8 | 🟡 Med | Нет concurrency-cap; per-call asyncio.run + новая MCP-сессия; rate-limit обходится сменой subject | `gateway/gateway/backends.py:128-150`, `rate_limit.py` |
| F-9 | 🟢 Low | previous signing secret из plaintext env | `gateway/gateway/config.py:83` |
| F-10 | 🟢 Low | assistant-ui auth fail-open + localhost-GET выдаёт сессию | `assistant-ui/app/security.py:43-46,93-122` |
| F-11 | 🟢 Low | Calendar-webhook интернет-доступен, амплификация до валидации | `gateway/gateway/app.py:550-619`, `calendar_watch.py:76-109` |

## 6. Этапы (Phases)

### P0 — Critical (до любого следующего включения/перезапуска Funnel или Gateway; gate от 2026-06-11)

**WP-0 — Emergency containment (немедленно, до WP-1/WP-2).** Subsystem: `gateway`.
Быстрый gate против активной эксплуатации; чистую реализацию формализуют WP-1/WP-2.
- **a.** Hotfix-блок legacy-ветки `/token` (убрать `if not grant_type → legacy_token`).
- **b.** Временно закрыть `/register` (public DCR deny) до owner-gate WP-2.
- **c.** **Hard rotation** gateway signing secret через secrets-слой — **БЕЗ `previous_secret`-окна**: previous-window сохранил бы проход legacy-minted токенов и противоречит цели containment. Graceful previous-window допустим только для плановой ротации позже, не здесь. Цена: легитимные коннекторы переживают пересозданием (R3; DCR иммутабелен — id 1630).
- **d.** Инвентаризация и revoke неизвестных OAuth-клиентов (`gateway/data/clients.sqlite3` → таблица `oauth_clients`).
- **e.** Проверить Funnel-поверхность (`tailscale serve status`); зафиксировать находки в `docs/security/audit-2026-06.md`.
- **Manual/elevated — агент НЕ выполняет сам, готовит snippet пользователю**: rotation секрета и `nssm restart AI-Assistant-Gateway` — admin-операции (см. §15 Containment runbook).
- **Acceptance**: `test_legacy_minted_token_dead_after_rotation` зелёный; неизвестные clients revoked; анонимный `/register` отклонён; Funnel surface подтверждён и записан.

**WP-1 — Удалить legacy token-grant (F-1).**
- Удалить ветку `if not grant_type` и метод `legacy_token` в `gateway/gateway/app.py`.
- `/token` принимает только `authorization_code` / `refresh_token`.
- Acceptance: `POST /token` без grant_type → 400 `unsupported_grant_type`; самосогласованный PKCE-пейр больше не выпускает токен; authorization_code-флоу ChatGPT работает.
- Тест: `test_token_legacy_grant_removed`.

**WP-2 — Resource-owner authentication + политика DCR (F-2).** Subsystem: `gateway`.
- Новый `gateway/gateway/owner_auth.py`: `require_owner_session()` принимает ТОЛЬКО явный owner-факт — owner-session cookie / owner-secret / one-time initial access token (через secrets-слой).
- **⚠ Funnel-loopback principle (Codex round 4, blocking)**: **НЕ переиспользовать `_healthz_allowed`** для owner-auth. Под Funnel все внешние запросы приходят с `client_address=127.0.0.1` (proxy на loopback), поэтому loopback ≠ identity; а `Tailscale-User-Login` — клиентский заголовок, подделываем без trusted-proxy контракта (`app.py:967`). Если перенести этот предикат на `/authorize`/`/register`, F-2 открывается заново. Tailnet-identity, если нужна, — отдельный валидатор с trusted-proxy проверкой, отвергающий spoofed headers и не доверяющий голому loopback. Тот же корень касается `/healthz` (WP-3) и per-IP throttle (WP-9).
- **Решение по DCR**: public DCR запрещён. `/register` требует initial access token (RFC 7591 §3, one-time, генерится owner через CLI) ИЛИ активной owner-сессии.
- **Spike (обязателен до фиксации флоу)**: проверить, поддерживает ли ChatGPT-коннектор protected DCR (передачу initial access token на `/register`). **Fallback, если нет**: owner регистрирует public-клиента локально и отдаёт `client_id` для ручного ввода в ChatGPT (PKCE, без секрета); `/register` остаётся закрытым.
- **CLI-gap (Codex round 3, blocking)**: текущий CLI `register` (`gateway/gateway/clients.py:414`) НЕ принимает auth-method и зовёт `register_client` → всегда `client_secret_post` (confidential, `clients.py:98`). Public-fallback неисполним as-is. WP-2 включает: расширить CLI флагом `--token-endpoint-auth-method none` → вызов `register_dynamic_client(..., token_endpoint_auth_method="none")` (уже поддерживает, `clients.py:345`). Альтернатива — fallback на confidential client с `client_secret`, если spike покажет, что ChatGPT его принимает.
- `/authorize`: consent только в owner-сессии (не «кнопка для всех»).
- Acceptance: анонимный `GET /authorize` → 401/redirect на `/login`; `POST /register` без initial-token/owner → 401; CLI регистрирует **public** client (`token_endpoint_auth_method=none`) и он работает коннектором; owner-флоу выпускает рабочий коннектор; уже выпущенные клиенты продолжают `/token`.
- Тесты: `test_authorize_requires_owner`, `test_register_requires_initial_token`, `test_cli_registers_public_client`, `test_owner_auth_rejects_spoofed_tailscale_user_header`, `test_owner_auth_rejects_funnel_loopback_without_owner_cookie`.

**WP-3 — Public surface lockdown, endpoint-by-endpoint (F-2 defense-in-depth).** Subsystem: `gateway` (+ `tools` Funnel).
Зафиксировать поверхность таблицей; `/.well-known/*` НЕ закрывать (иначе ломается ChatGPT discovery):

| Endpoint | Доступ | Примечание |
|----------|--------|-----------|
| `/token` | public (rate-limited) | выпущенные клиенты + code/refresh |
| `/mcp`, `/mcp/*` | public + Bearer JWT + scope | основной surface |
| `/.well-known/oauth-authorization-server` | **public — оставить** | ChatGPT discovery |
| `/.well-known/oauth-protected-resource` | **public — оставить** | ChatGPT discovery |
| `/authorize` | owner-session only | consent |
| `/register` | initial-access-token / owner | public DCR закрыт (WP-2) |
| `/healthz` | исключить из Funnel ИЛИ service-bearer | текущий `_healthz_allowed` слаб под Funnel (loopback-trust, `app.py:967`) — не security boundary |
| `/openapi.yaml` | исключить из Funnel ИЛИ owner/service-bearer | НЕ через `_healthz_allowed` (loopback слаб под Funnel); сейчас public GET |
| `/webhooks/google-calendar` | public, token-валидируется в backend | F-11 |

### P1 — High

**WP-4 — Fail-closed S2S identity на бэкендах (F-3).** ⚠ deviation от brick #5. Subsystems: `gateway`+`pm-mcp-server`+`ai-memory`+`budget`+`assistant-ui`.
- **Token provisioning** всем internal-клиентам: gateway→backends, assistant-ui→backends, ai-memory stdio-bridge, budget, pm-mcp internal (calendar-webhook путь). Единый secrets-слой (WP-7).
- **Коды (исправление)**: invalid/missing token → **401 (missing) / 403 (invalid scope)**, **НЕ 503** (503 = unavailable и вводит клиента в заблуждение; fail-closed = deny).
- **warn → enforce** через фичефлаг: фаза 1 — log-only (пропускает, предупреждает), фаза 2 — enforce. Rollout-чеклист: провижн ВСЕХ клиентов до enforce.
- **rotation active/previous** (окно), как у proposals.
- Снять исключение из `pm-mcp-server/app/http_transport.py` auth_middleware для **обоих** mount: `/mcp-streamable/*` И `/openapi/*` (сейчас исключены оба).
- **OpenAPI / Open WebUI (явное решение, в ADR-0019)**: предпочесть (i) OWUI предъявляет service-bearer (header задокументирован) ЛИБО (iii) `/openapi`-mount temporarily disabled. Вариант «loopback-only без enforce» — ТОЛЬКО как временное ADR-0019-исключение **с датой удаления** + exact-origin CORS + read-only allowlist + DNS-rebinding (иначе ослабляет весь принцип «loopback ≠ trust»).
- `TransportSecuritySettings(enable_dns_rebinding_protection=True)` на :8765/:8766/:8767 (по образцу :8770).
- Тесты: `test_backend_requires_bearer`, `test_mcp_mount_authenticated`, `test_openapi_mount_authenticated`, `test_missing_token_returns_401_not_503`, rebinding-тест.

**WP-5 — Sanitize `store_memory` (F-4).**
- Вынести `SECRET_PATTERNS`/`find_secret_patterns` из `proposals/intake.py` в общий модуль; применить в `store()`.
- Добавить `provenance`/`trust_tier`; недоверенный источник → только `propose`; учесть trust в ранжировании search.
- Тесты: `test_store_memory_rejects_secrets`, `test_store_memory_provenance`.

**WP-6 — Tool capability gating + approval (F-5).**
- Реестр capability (read/propose/act/destructive) для MCP-tools.
- В `agent_loop.py` заменить хардкод `CONFIRMATION_TOOL_NAMES` на `capability ∈ {act, destructive} or is_write`.
- Помечать сообщения с retrieved-контентом; запрет инициировать `act` из них без аппрува.
- Тесты: `test_act_tool_requires_confirmation`, `test_injected_content_cannot_act`.

**WP-7 — Service Identity Layer (MC-3) + унификация secrets (MC-4).**
- Консолидировать `*/secrets.py` в единый интерфейс read/write/rotate (keyring + restricted-file, запрет env-секретов в проде).
- Короткоживущие подписанные service-токены (mint/verify); основа для multi-host.
- Отдельный ADR-0019.

**WP-8 — Централизованный policy-движок (MC-2).**
- Свести `ALLOWLIST` + `MCP_PROFILES` + `DENIED_TOOLS` + схемы в один декларативный модуль `decide(principal, tool, resource) -> allow/deny/needs_approval`, общий для gateway и бэкендов.
- Зависит от WP-7.

### P2 — Medium

- **WP-9** Gateway hardening: concurrency-cap, per-IP throttle до scope-лимита, пул MCP-сессий, cap размера тела (F-8, F-11). NB: под Funnel все запросы — `127.0.0.1`, поэтому per-IP throttle требует реальный client IP из trusted forwarded-заголовка (тот же Funnel-loopback корень, что и owner-auth WP-2).
- **WP-10** Audit: метаданные/хэши вместо сырых тел, сериализация записи, внешний/WORM-sink + якорь головы цепочки, retention/ротация (F-6, F-7).
- **WP-11** Security monitoring: метрики denied/rate_limited/новые client_id → пороги → уведомление (MC-10).
- **WP-12** previous-secret из keyring (F-9); assistant-ui fail-closed + не выдавать cookie на неаутентиф. GET (F-10).

### P3 — Стратегическое (схему закладывать сейчас)

- **WP-13** Multi-tenant: сквозной `tenant_id` во всех БД (memory/tasks/budget/calendar) + backfill `tenant=self`; политика по tenant (MC-11).
- **WP-14** Sandbox/allowlist для внешних MCP и делегированных агентов.
- **WP-15** Шифрование чувствительных БД at-rest (budget.db, memory.db).

## 7. Недостающие security-компоненты (кирпичики)

| ID | Компонент | Приоритет | Лечит |
|----|-----------|-----------|-------|
| MC-1 | Resource-owner authentication (identity) | P0 | F-2 |
| MC-2 | Централизованный policy engine (RBAC/ABAC) | P1 | размазанная политика |
| MC-3 | Service identity layer (S2S auth/mTLS) | P1 | F-3 |
| MC-4 | Унифицированный secrets manager | P1 | F-9, разнобой |
| MC-5 | Tool permission system на бэкендах | P1 | F-3, F-5 |
| MC-6 | Memory access controls | P1 | F-4 |
| MC-7 | Approval workflow для агентных действий | P1 | F-5 |
| MC-8 | API gateway hardening | P2 | F-8, F-11 |
| MC-9 | Tamper-evident audit pipeline | P2 | F-6, F-7 |
| MC-10 | Security monitoring / alerting | P2 | детект |
| MC-11 | Multi-tenant data partitioning | P3 (схему сейчас) | multi-user readiness |

## 8. Архитектурная эволюция (закладываем сейчас)

1. Разделить client-identity и owner-identity в gateway (WP-2) — точка роста к OIDC multi-user.
2. Ввести `principal` в сквозной контракт всех вызовов (даже если сейчас `principal=self`).
3. Заложить `tenant_id` в схемы БД немедленно (backfill `self` дёшев сейчас, дорог потом).
4. Заменить loopback-доверие явной идентичностью сервисов до первого remote-сценария (WP-7).
5. Единый policy-движок до роста числа правил (WP-8).
6. Граница недоверенного контента (provenance/trust_tier) — фундамент для sandbox внешних MCP.

## 9. Риски

- **R1**: WP-4 (fail-closed bearer) может оборвать локальные агенты и calendar-webhook, если не выдать токены. → фичефлаг warn→enforce, rollout-чеклист.
- **R2**: WP-2 может сломать ChatGPT-коннектор, если owner-gate перекроет `/token` уже выпущенных клиентов. → публичными оставить только `/token`+`/mcp`.
- **R3**: Изменение scope/owner-флоу → пересоздание ChatGPT-коннектора (DCR иммутабелен, AI-memory id 1630). → задокументировать в rollout.
- **R4**: P0-фиксы трогают подписанный auth-путь в проде → обязательный изолированный verify (brick #17 Track B) перед рестартом NSSM.

## 10. Критерии приёмки (плана в целом)

- [ ] P0 закрыт: `/token` без grant_type отклоняется; `/authorize` за owner-session, `/register` за initial-access-token/owner (public DCR закрыт); previous-secret снят после hard-rotation; verified изолированным прогоном.
- [ ] P1 закрыт: бэкенды fail-closed с S2S-токеном + rebinding protection; `store_memory` санируется; `act`/write из агента за подтверждением.
- [ ] Контрактные изменения (principal/tenant_id) внесены в схемы.
- [ ] ADR-0018/0019 написаны; кирпичик #5 обновлён по подтверждению.
- [ ] Документация (root/subsystem AGENTS.md, README, ARCHITECTURE) описывает только новый путь (migration discipline).
- [ ] Живые ChatGPT-коннекторы работают (или есть rollout-инструкция по пересозданию).
- [ ] Durable security report `docs/security/audit-2026-06.md` создан (threat model, exploit-пробы, severity rationale, regression mapping).
- [ ] Docs-sweep после ADR-0019: grep по `trust-by-default`, `without auth`, `без auth`, `loopback`, `registration_endpoint`, `store_memory` в README/ARCHITECTURE/AGENTS; обновлены подтверждённые конфликты `pm-mcp-server/ARCHITECTURE.md:292` и `pm-mcp-server/README.md:228` под fail-closed модель.
- [ ] Security regression suite (§14) зелёный.
- [ ] Per-WP work items заведены по subsystem (§13 Task matrix) с `link_task_dependency`.

## 11. Разделение работ Claude / Codex

- **Claude**: аудит (готов), этот план, ADR-черновики, ревью реализации Codex, threat-model regression.
- **Codex**: реализация WP-1…WP-15 по согласованным приоритетам, тесты, rollout-скрипты.
- Координация — через PM-MCP work items (по одному на затронутую подсистему, K.4), зависимости через `link_task_dependency` (P0 → P1 → P2). Umbrella — #853.
- **Codex review round 1–4 (2026-06-11) учтён**: WP-0 containment (hard rotation + снятие previous-secret), owner-auth/DCR + spike/fallback, Funnel-loopback principle (owner-auth не на loopback/`_healthz_allowed`), S2S 401/403 + OpenAPI/OWUI решение, durable report, task matrix (§13, subsystem→absolute project_path), regression suite (§14), containment runbook (§15), docs-sweep, уточнённый gate-срок.

## 12. Portfolio-wide preventive layer (шире AI-Assistant)

Профилактика для ВСЕХ будущих проектов `D:\GitHub\` — прививается через существующие механизмы
(`_project_template`, кирпичики, `_engineering_rules/hooks`, skills, ruff-gate). Эти пункты выходят
за пределы AI-Assistant; затрагивают `_engineering_rules` / `_project_template` (отдельные
project_path — при необходимости вынести в отдельные work items, K.4).

**Уже сделано в этой сессии (2026-06-09):**
- ✅ `_project_template/AGENTS.md` §H «Security defaults (fail-closed)» — секьюр-дефолты наследуются новыми проектами.
- ✅ Кирпичик **#26** (ruff + flake8-bandit `S` как security-gate) в `tech-stack-choices.md`.

**Backlog (приоритет = порядок):**

- **PB-1 — ruff `S` rollout** (effort S). Включить набор `S` (flake8-bandit) в ruff-конфиг всех Python-проектов портфеля + в `_project_template/pyproject`. Глушить ложные per-line `# noqa: S### — причина` / per-path. Кодифицировано брик #26. Ловит ≈половину классов аудита автоматически (hardcoded secrets, shell=True, assert, слабая крипта, bind 0.0.0.0, небезопасная десериализация).
- **PB-2 — Secret-scan shared hook** (effort M). Вынести `find_secret_patterns` (16 паттернов, `ai-memory/memory/proposals/intake.py`) в `_engineering_rules/hooks/secret_scan_guard.py` (PreToolUse Write/Edit + pre-commit), по образцу существующих `*_guard.py`. Блокирует секрет ДО диска/git во всех репо. Тесты — `hooks/tests`. Оформить через skill `hook-authoring`.
- **PB-3 — No-public-bind guard hook** (effort S). Блок литералов `0.0.0.0`/`::` без явного allow-флага. Подкрепляет §H.1 и ruff S104.
- **PB-4 — Auth-touch reminder hook** (effort S). Правка `*auth*`/`*oauth*`/`*secret*`/`scope_policy` → напоминание «перечитай кирпичики #22/#23, проверь fail-closed».
- **PB-5 — Route↔scope parity guard** (effort S-M). Добавление route в gateway `backends.py`/`ROUTES` без записи в `scope_policy.ALLOWLIST` → блок (класс, родственный F-1/F-2 «поверхность без политики»).
- **PB-6 — AI tool capability classification** (effort M, → кирпичик после WP-6). Стандарт read/propose/act/destructive для ЛЮБОГО MCP-сервера портфеля, не только AI-Assistant. Сейчас gap (нет evidence) → кирпичик создаётся после реализации.
- **PB-7 — Untrusted-content trust-tier** (effort M, → кирпичик после WP-5/WP-6). Память/файлы/web/Obsidian/календарь = недоверенный ввод; provenance-пометка + запрет инициировать `act` из retrieved-content. Защита от lethal-trifecta во всех агентных проектах.
- **PB-8 — Dependency/supply-chain audit** (effort S). `pip-audit` / `uv pip audit` как локальный pre-push шаг (CI нет — через `uv run`); осторожно с `uvx <untrusted>` в прод-путях (pin в `.venv`); SBOM для shippable (Verua client zip). Прецедент bandit/security-gate — AI-memory id 1019.
- **PB-9 — Data-at-rest** (effort M, → кирпичик после реализации). Классификация sensitive БД (budget.db финансы, memory.db личное) + шифрование at-rest (SQLCipher/OS-level); портфельный стандарт, расширяет WP-15.
- **PB-10 — `/security-baseline` skill** (effort M). Через `skill-creator`: чеклист (bind, fail-open/closed, источник секретов, scope/permission-модель, наличие audit, dep-audit, ruff S) → gap-report. Security-двойник `repo-automation-audit`; кадэнс раз в месяц по активным репо.
- **PB-11 — Threat-model чеклист на trust boundary** (effort S). Пара вопросов в `_project_template/AGENTS.md` (или skill-триггер), когда фича вводит новый ingress / внешнюю интеграцию / новый write-путь.

## 13. Task matrix (work items по subsystem)

Для #853 заводятся отдельные work items по затронутой подсистеме (K.4), чтобы umbrella не была «в работе без ответственных».
**Перед `create_task` (Codex round 4)**: значения колонки `subsystem` — НЕ валидные PM-MCP `project_path` (`project_path="gateway"` → project_not_found). Резолвить точный absolute path через PM-MCP `list_projects`: `gateway` → `D:\GitHub\AI-Assistant\gateway`, `ai-memory` → `D:\GitHub\AI-Assistant\ai-memory`, и т.д.; `root *` → `D:\GitHub\AI-Assistant`.

✅ **Пул задач создан 2026-06-11** (assignee `codex`, umbrella #853): WP-0 **#854** (К выполнению), WP-1 **#855**, WP-2 **#856**, WP-3 **#857**, security report **#858** (К выполнению), ADR-0018 **#859** (К выполнению), WP-4 **#860**, WP-5 **#861**, WP-6 **#862**, WP-7 **#863**, WP-8 **#864**, ADR-0019 **#865**. Зависимости: #855→#854, #856→#855+#859, #857→#856, #863→#865, #860→#863, #864→#860. **P2/P3 (WP-9…WP-15) НЕ заведены** — создаются после закрытия P1, чтобы не плодить stale-бэклог.

| WP | subsystem | Приоритет |
|----|-----------|-----------|
| WP-0 containment | `gateway` | P0 |
| WP-1 legacy-grant removal | `gateway` | P0 |
| WP-2 owner-auth/DCR (+spike) | `gateway` | P0 |
| WP-3 surface lockdown | `gateway` (+ `tools`) | P0 |
| Durable security report | root `docs/security` | P0 |
| ADR-0018 owner-auth | root `docs/adrs` | P0 |
| WP-4 S2S | `gateway`+`pm-mcp-server`+`ai-memory`+`budget`+`assistant-ui` | P1 |
| WP-5 sanitize store_memory | `ai-memory` | P1 |
| WP-6 capability gating | `assistant-ui` | P1 |
| WP-7 secrets/identity | root + все (secrets-слой) | P1 |
| WP-8 policy engine | `gateway` + backends | P1 |
| ADR-0019 service identity | root `docs/adrs` | P1 |
| WP-9 gateway hardening | `gateway` | P2 |
| WP-10 audit pipeline | `gateway` | P2 |
| WP-11 monitoring | `gateway` | P2 |
| WP-12 secret/UI fixes | `gateway` (F-9) + `assistant-ui` (F-10) | P2 |
| WP-13 multi-tenant | `ai-memory`+`pm-mcp-server`+`budget` | P3 |
| WP-14 sandbox ext MCP | `gateway` | P3 |
| WP-15 at-rest encryption | `budget`+`ai-memory` | P3 |

Зависимости: WP-0 → WP-1 → WP-2 → WP-3; WP-7 → WP-4 → WP-8. Umbrella — #853.

## 14. Security regression suite (red-team)

Единый набор; обязателен в Acceptance (§10):

- `test_token_no_grant_type_rejected` — `/token` без grant_type не выпускает JWT.
- `test_unauth_register_rejected` — анонимный `/register` отклонён.
- `test_legacy_minted_token_dead_after_rotation` — токен, выпущенный legacy-путём, мёртв после hard-rotation секрета.
- `test_backend_missing_bearer_rejected` — отсутствие bearer на backend MCP → 401 (не 503, не pass).
- `test_openapi_mount_authenticated` — `/openapi`-mount не отдаёт tools без авторизации (по решению WP-4).
- `test_injected_memory_cannot_act` — `act`/write-tool не вызывается из retrieved-content (prompt injection).
- `test_audit_no_raw_secrets` — в audit нет raw Bearer/JWT/секретов.
- `test_request_cap_before_parse` — body/size cap срабатывает ДО парсинга тела.
- `test_owner_auth_rejects_spoofed_tailscale_user_header` — owner-auth не принимает подделанный `Tailscale-User-Login`.
- `test_owner_auth_rejects_funnel_loopback_without_owner_cookie` — owner-auth не пускает Funnel-loopback без owner-cookie/secret/initial-token.

## 15. Containment runbook (WP-0) и rollback

**Порядок apply:**
1. Deploy hotfix-блок legacy `/token` + deny `/register` (Edit + изолированный verify, brick #17 Track B, ephemeral-порт, не против живого :8780).
2. **Manual/elevated — выполняет пользователь в admin pwsh 7 (агент только готовит snippet, сам admin-операции не запускает):** ротация signing secret + рестарт gateway, проверка свежего PID (AI-memory id 1529 — статус Running ≠ свежий код):

```powershell
# 1) записать новый signing secret (64+ случайных байт, base64url) в keyring/secret-file
# 2) ОБЯЗАТЕЛЬНО снять previous-secret (иначе legacy-minted токены пройдут — config.py:83):
#    очистить AI_ASSISTANT_GATEWAY_PREVIOUS_SECRET и AI_ASSISTANT_GATEWAY_PREVIOUS_SECRET_EXPIRES_AT
#    в NSSM env (nssm set ... AppEnvironmentExtra) / secrets-слое
# 3) рестарт и проверка нового PID:
Stop-Service AI-Assistant-Gateway -Force
Start-Service AI-Assistant-Gateway
Get-CimInstance Win32_Process -Filter "Name='python.exe'" | Select-Object ProcessId, CreationDate
```
- **Проверка после ротации (Codex round 3, important)**: `_verification_secrets()` (`gateway/gateway/app.py:703`) содержит ТОЛЬКО новый секрет — previous отсутствует/истёк. Покрыто `test_legacy_minted_token_dead_after_rotation`.

3. Инвентаризация/revoke клиентов (можно не-admin, в gateway venv):

```powershell
cd D:\GitHub\AI-Assistant\gateway
uv run python -m gateway.clients list
uv run python -m gateway.clients revoke <client_id>
```

4. `tailscale serve status` → подтвердить поверхность; записать в `docs/security/audit-2026-06.md`.

**Rollback** (если owner-gate/ротация рвут легитимный коннектор): пересоздать коннектор в ChatGPT (DCR иммутабелен по scope — id 1630); hard-rotation намеренно НЕ имеет previous-window, поэтому «мягкого» отката токенов нет — это осознанная цена containment (Codex round 2, пункт 1). Откат hotfix-блока — только при подтверждённом отсутствии активной эксплуатации.

## References

- Внешний фон по MCP OAuth/DCR-рискам: arXiv 2605.22333.
- Codex review round 1–4 (2026-06-11) — учтён в §6/§10/§11/§13–§15; review-контекст в AI-memory proposal #15.
- Прецеденты: NSSM restart vs свежий PID — AI-memory id 1529; DCR scope иммутабелен — id 1630; bandit security-gate — id 1019.
