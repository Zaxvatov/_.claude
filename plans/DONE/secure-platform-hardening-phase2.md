# Secure AI-Agent Platform — Hardening Phase 2

> Статус: **Phase 2 выполнен; план закрыт 2026-06-13**
> Автор: Claude (Principal Security Architect / Staff Architect / Red Team / TPM)
> Дата: 2026-06-11
> PM-MCP: #894 (umbrella, утверждён 2026-06-12). **P0/P1 закрыты (Готово 2026-06-12):** WP-A #895 (gateway), WP-B #896 (ai-memory+budget), WP-B2 #897 (ai-memory). **P2-пул заведён:** WP-C #899 (gateway, →#896), WP-D #900, WP-E #901, WP-F #902, WP-G1 #903 (gateway), WP-G2 #904 (assistant-ui). **P3-пул заведён** (по запросу пользователя, до закрытия P2): WP-H1 #905 (ai-memory), WP-H2 #906 (pm-mcp), WP-H3 #907 (budget) — все →#899; WP-I #908 (gateway, →#900); WP-J #909 (gateway, →#899); WP-K #910 (root, at-rest spike→ADR-0023, без блокера; ADR-0022 уже занят roadmap ADR). At-rest gate #912 и impl-задачи #913/#914 закрыты после SQLCipher POC и реализации ADR-0023. Предыдущая umbrella — #853 (закрыта).
> Предшественник: `DONE/secure-agent-platform-hardening.md` (#853, P0/P1 реализованы коммитом `a620947`)
> Связанные ADR: ADR-0018 (owner-auth), ADR-0019 (service identity + signed service token/backend parity), ADR-0024 (multi-tenant principal/storage boundary), ADR-0023 (data-at-rest; ADR-0022 занят roadmap ADR)
> Codex review round 1 (2026-06-12) учтён: §5 F-3R evidence, §5.1 surface matrix, §5.2 token model, WP-A (оба start-path), WP-C вне P1-критпути, §4 bricks, WP-K at-rest spike, §10.1 rollout checklist, §11 regression. Proposal #16.
> Codex review round 2 (2026-06-12) учтён: §5.2 delivery-модель (editable пакет) + mint/refresh + **asymmetric Ed25519 broker** (claims неподделываемы), §5.1 ai-memory `/openapi`, §5.2 разбиение scopes, WP-H разложен по БД, WP-K факторы spike. Proposal #17.
> Codex review round 3 (2026-06-12) учтён: §5.3 (broker собственная авторизация + caller-registry + owner-scope→owner-auth + key/registry integrity + крипто-профиль EdDSA-only), WP-B OpenAPI positive-path, §10.2 rollback, WP-C — отдельный shared package. Proposal #18.
> Codex review round 4 (2026-06-12) учтён: WP-B2 миграция `ai-memory-proposals :8770` на Ed25519, WP-A route/tool-level scope enforcement в pm-mcp, WP-A per-caller credential/envelope-registry как deploy-артефакт на обоих start-path, WP-H global `#NNN` сохраняются (ADR-0003, tenant ограничивает видимость). Proposal #19.
> Codex review round 5 (2026-06-12) учтён: WP-B2 явно **удаляет OAuth mini-surface proposals-демона** (`/.well-known/*`,`/authorize`,`/token`,HS256), WP-A pm-mcp route-scope **regression-тесты**, §10/§11 включают WP-B2, явные provisioning-тесты (per-caller credentials + `service-callers.json`). Proposal #20.
> Codex review round 6 (2026-06-12) учтён: surface matrix `/events` Ed25519+`pm.read` и отдельная строка OAuth :8770 (target removed/404), §10.1 rollout WP-B2 (propose=200, direct :8770=401, OAuth=404), §10.2 rollback включает :8770. Proposal #20.
> Решения пользователя (2026-06-12): (1) **подписанные короткоживущие service-токены сразу**; (2) **полный объём P3**; (3) **mint-брокер на gateway + asymmetric Ed25519** (бэкенды только verify public-key) для неподделываемых claims и refresh.

---

## 1. Контекст и текущая позиция

Монорепо персонального AI-ассистента (5 подсистем) опубликован в интернет через
Tailscale **Funnel** → `gateway` (:8780) → loopback-бэкенды (`ai-memory` :8765/:8770,
`pm-mcp-server` :8766, `budget` :8767, `assistant-ui` :8000).

План #853 (аудит из 11 находок F-1…F-11) **исполнен в значительной части** коммитом
`a620947 "Harden agent platform security controls"`:

| Находка | Статус (по коду) | Evidence |
|---|---|---|
| F-1 legacy `/token` grant | ✅ закрыто | `gateway/gateway/app.py:268-271,326` → `unsupported_grant_type` |
| F-2 public DCR + consent без owner | ✅ закрыто | `app.py:335` (register требует owner/initial-token), `app.py:803-808,859-864` (`/authorize` owner-only) |
| F-4 `store_memory` обход secret-scan | ✅ закрыто | `ai-memory/memory/storage.py:258-259`, `secret_scan.py` |
| F-5 агентный tool-loop без policy | ✅ закрыто | `assistant-ui/app/agent_loop.py:350-399`, untrusted-content guard |
| /healthz, /openapi.yaml public | ✅ закрыто | `app.py:787-828` → `sensitive_surface_allowed` |

**Однако** свежее ревью выявило, что находка **F-3 (loopback trust) закрыта лишь
частично**, и появилась новая операционная находка по провижну токенов. Эти зазоры —
предмет Phase 2.

## 2. Цель

1. Перевести S2S на **Ed25519 broker-minted короткоживущие токены** (claims:
   iss/aud/sub/tenant_id/scopes/exp/kid; минтит только gateway-брокер, бэкенды только verify) и
   достроить паритет: **все** loopback-бэкенды (оба mount `/mcp` и `/openapi`) проверяют токен
   и имеют DNS-rebinding protection (не только `pm-mcp` и `:8770`). Включая миграцию pm-mcp с
   opaque-bearer (Phase 1) на Ed25519 (migration discipline).
2. Закрыть провижн-зазор на **обоих** путях запуска (NSSM `register_services.ps1` И logon
   `run-user-session.ps1`): Ed25519 keypair (приватный только у брокера, публичный у бэкендов)
   + broker-URL раздаются до того, как fail-closed enforce станет живым (иначе деплой рвёт трафик).
3. Закрыть остаточные P2: claims-based shared policy, tamper-evident audit, pre-parse caps,
   calendar-webhook, assistant-ui localhost-cookie, previous-secret из keyring.
4. **Реализовать** (полный объём, решение пользователя) multi-user/remote/external-MCP слой:
   `tenant_id` сквозной + backfill `self`, sandbox внешних MCP, at-rest encryption (через spike→ADR).

## 3. Ограничения и инварианты

- **Migration discipline** (AGENTS.md Q): новый путь → миграция → удаление; без shim'ов.
- **Asymmetric mint-монополия**: только gateway-брокер держит приватный Ed25519-ключ и минтит;
  все бэкенды держат лишь публичный ключ и verify. Никаких shared mint-секретов (Codex r2 п.3).
- **`service_identity` — editable-пакет** в `pyproject.toml` каждой подсистемы (`tool.uv.sources`),
  не копипаста кода между `.venv` (Codex r2 п.1).
- **ADR-0019 warn→enforce**: enforce включается ТОЛЬКО после провижна всех клиентов.
- **Subsystem boundaries** (ADR-0001): gateway — единственный внешний ingress.
- Каждое Python-изменение: `uv run ruff check .` + тесты затронутой подсистемы.
- Repo files > task status: `#860` помечен «Готово», но код показывает незакрытый остаток
  (см. F-3R) — ground truth = код.

## 4. Tech-stack bricks: применимость и предложения

Проверено против `D:\GitHub\_engineering_rules\tech-stack-choices.md`.

**Используем как есть:** #3 (SQLite+FTS5+WAL), #4 (namespaced env), #17 (runtime verify
Track B), #19 (pwsh7), #22 (secrets keyring-first), #23 (external ingress auth),
#24 (proposal contour), #26 (ruff S SAST).

**Уже покрыто каталогом — НЕ создавать дубль (проверено в коде каталога 2026-06-12):**
- **#3 (line 48)** уже содержит `tenant_id` forward-readiness (колонка заранее + backfill
  `self`) → WP-H реализует существующий forward, **новый brick не нужен**.
- **#5 (lines 64-65)** уже фиксирует «было/стало loopback ≠ trust» и явно говорит, что
  service-identity / S2S оформляется **отдельным кирпичиком ПОСЛЕ реализации+evidence**.
  Значит правка «уточнить #5 trust-by-default» избыточна — каталог уже синхронен с ADR-0019.

**Candidate bricks (создавать ТОЛЬКО после landing+evidence и подтверждения пользователя):**
- candidate **#27** — Ed25519 broker-minted short-lived service token (asymmetric: издатель минтит, verifier только проверяет pub-key; aud/iss/kid/scopes; editable `service_identity` пакет) — после WP-A/WP-B; промоут из «#5 S2S gap».
- candidate **#28** — claims-based central policy `decide()` — после WP-C.
- candidate **#29** — tamper-evident audit (serialized writer + external/WORM anchor) — после WP-D.
- candidate **#30** — security monitoring/alerting (denied/rate/new-client thresholds) — после WP-I.
- candidate **#31** — data-at-rest pattern (по выбору spike) — после WP-K/ADR-0023 и реализации #912/#913/#914.
- candidate **#32** — JWT/crypto-профиль (`PyJWT[crypto]`+`cryptography`, EdDSA-only allowlist,
  запрет `none`/key-confusion, строгий `iss/aud/iat/exp/kid`) — после WP-A; обновляет brick #23
  (там сейчас hand-rolled HS256 `auth.py`).

## 5. Находки Phase 2 (только с code-evidence)

| ID | Sev | Суть | Файлы |
|----|-----|------|-------|
| **F-3R** | 🔴 High | S2S **асимметричен**: `ai-memory` :8765 и `budget` :8767 построены `FastMCP(...)` без inbound-auth и без DNS-rebinding (grep `auth/Bearer/middleware/TransportSecurity` по `ai-memory/memory/mcp_app.py` = 0); gateway/assistant-ui шлют им bearer, который игнорируется. Прямой loopback/SSRF/rebinding-вызов читает память и финансовый ledger и пишет реальные write-tools `store_memory`, `store_memory_batch`, `store_summary`, `confirm_memory_entry` (`mcp_app.py:178-182`) в обход gateway. **NB (Codex):** `delete_memory` — НЕ экспортируемый backend-tool, только gateway-инвариант `DENIED_TOOLS` (`scope_policy.py:48`); не завышать evidence. | `budget/budget/mcp_app.py:205`, `ai-memory/memory/mcp_app.py:178-182,450`, vs `ai-memory/memory/proposals/app.py:433` |
| **N-1** | 🔴 High (deploy-gate) | Провижн-зазор на **ОБОИХ** start-path. `tools/register_services.ps1` НЕ задаёт `AI_ASSISTANT_GATEWAY_INTERNAL_SERVICE_TOKEN(_FILE)`, `PM_MCP_AUTH_TOKEN_PATH`, `ASSISTANT_*_AUTH_TOKEN`. **Второй путь (Codex):** `pm-mcp` и `assistant-ui` стартуют также через logon scheduled task `tools/user-session/run-user-session.ps1` (session-0 vault fix, id 1629), который зеркалит лишь часть env (`:26-43`) и тоже не раздаёт токен. `.secrets/` содержит лишь `gateway-secret.txt` и `ai-memory-proposals-token.txt`. pm-mcp middleware fail-closed (`http_transport.py:77-93`, `auth.py:30-36`). При деплое `a620947` (рестарт любым путём) gateway→pm-mcp и ui→pm-mcp получают 401. Нет warn-фазы (ADR-0019 §3 требует, код её не имеет). | `tools/register_services.ps1:294-372`, `tools/user-session/run-user-session.ps1:26-43`, `pm-mcp-server/app/auth.py:30-36`, `gateway/gateway/config.py:170-174` |
| **N-2** | 🟡 Med | Audit пишет сырые `headers`+`body` в metadata, затем узкая redaction (3 regex + 5 ключей). Obsidian-контент, memory-текст, budget-строки, query оседают в `gateway-audit.jsonl`. (исходно F-6) | `gateway/gateway/app.py:752-758`, `redaction.py:6-10` |
| **N-3** | 🟡 Med | Hash-chain не tamper-evident и не anchored; `_previous_hash()` перечитывает весь файл на каждый append → гонка под `ThreadingHTTPServer`, разрыв цепочки при конкурентной записи. (исходно F-7) | `gateway/gateway/audit.py:37-73` |
| **N-4** | 🟡 Med | Нет pre-parse body-cap/concurrency-cap; `_read_body` читает весь Content-Length в память; rate-limit ключ = subject, под Funnel client_ip всегда 127.0.0.1 → per-IP throttle бессмыслен, pre-auth surface не ограничен. (исходно F-8/F-11) | `gateway/gateway/app.py:930-934`, `rate_limit.py:22-33` |
| **N-5** | 🟡 Med | Каждый backend-вызов = `asyncio.run` + новая MCP-сессия/`initialize` (нет пула) → амплификация и латентность; усиливает N-4. | `gateway/gateway/backends.py:142-162` |
| **N-6** | 🟢→🟡 | Calendar-webhook публичен, форвардит произвольные `X-Goog-*` в `pm-mcp /internal/calendar/notify` до валидации channel-token (gateway его не проверяет; pm-mcp owns). Амплификация к internal route. Частично смягчён (rate-limit + shape-check). (исходно F-11) | `gateway/gateway/app.py:565-634`, `backends.py:98-125` |
| **N-7** | 🟢 Low | assistant-ui: localhost GET авто-выдаёт session cookie без токена → на shared/multi-user хосте отдаёт сессию любому локальному GET. (исходно F-10, fail-open флип исправлен, localhost-cookie остался) | `assistant-ui/app/security.py:99-102` |
| **N-8** | 🟢 Low | previous signing secret читается из plaintext env, не из secrets-слоя. (исходно F-9) | `gateway/gateway/config.py:92` |
| **N-9** | 🟡 Med (arch) | `policy.decide()` живёт только в gateway; бэкенды не консультируют общую policy. `DENIED_TOOLS` enforce только на gateway → прямой loopback-вызов ai-memory обходит её (связано с F-3R). | `gateway/gateway/policy.py`, `scope_policy.py:45-52` |
| **N-10** | 🟡 Med (forward) | `Principal.tenant_id="self"` есть в gateway, но НЕ распространяется в бэкенды и НЕ персистится; схемы memory/tasks/budget без tenant-колонки → multi-user не изолируем. | `gateway/gateway/policy.py:11-16` |
| **N-11** | 🟢 Low | `_healthz_allowed` (loopback+spoofable Tailscale-User) больше не используется для /healthz (заменён `sensitive_surface_allowed`), но остался мёртвым кодом — риск повторного подключения. | `gateway/gateway/app.py:1002-1019` |

### 5.1 Surface matrix (S2S parity: текущее → целевое) — добавлено по Codex

| Daemon | Endpoint | Сейчас | Целевое (Phase 2) |
|--------|----------|--------|-------------------|
| pm-mcp :8766 | `/mcp-streamable/*` | opaque bearer ✅ | **Ed25519 token (claims)** — миграция |
| pm-mcp :8766 | `/internal/calendar/notify` | bearer ✅ (middleware) | Ed25519 token, scope `internal.write` |
| pm-mcp :8766 | `/openapi/*` (OWUI) | opaque bearer ✅ | Ed25519 token; **regression на auth** |
| pm-mcp :8766 | `/health` | public | без изменений |
| pm-mcp :8766 | `/events` (SSE) | bearer | **Ed25519 + `pm.read`** (Codex r6) |
| **ai-memory :8765** | `/mcp` | **нет auth ❌** | **Ed25519 token (`memory.*`) + DNS-rebinding** |
| **ai-memory :8765** | `/openapi/*` (bridge) | **нет auth ❌** | **Ed25519 token + regression** (Codex r2; `openapi_bridge.py`, `daemon.py`) |
| ai-memory :8765 | `/healthz` (опц.) | public | health-only (без claims) |
| ai-memory-proposals :8770 | `/mcp` | ad-hoc token + rebinding ✅ | мигрировать на Ed25519 (scope `memory.propose`) |
| ai-memory-proposals :8770 | `/.well-known/*`, `/authorize`, `/token` (OAuth+HS256) | собственный OAuth-сервер ⚠️ | **target: removed/404** (WP-B2, Codex r6) |
| **budget :8767** | `/mcp` | **нет auth ❌** | **Ed25519 token (`budget.*`) + DNS-rebinding** |
| assistant-ui :8000 | `/api/*`, страницы | cookie/bearer, localhost-GET выдаёт cookie | fail-closed (N-7) |
| gateway :8780 | внешний surface | Phase 1 ✅ | hardening (N-2…N-6,N-8,N-11) |

DNS-rebinding (`TransportSecuritySettings(enable_dns_rebinding_protection=True)`): сейчас
только :8770 → расширить на :8765/:8766/:8767 (образец — `proposals/app.py:433`).

### 5.2 Token model (РЕШЕНО: Ed25519 broker-minted short-lived tokens)

Codex round 1 отметил противоречие «opaque bearer vs principal из claims», round 2 —
что **единый shared HMAC делает claims самоутверждаемыми** (любой caller с секретом
выпишет себе owner-scope) и что **TTL≤15м нельзя класть статичным файлом**. Решение
пользователя: **mint-брокер + asymmetric**. Это включает **миграцию pm-mcp** с opaque
`verify_authorization_header` (Phase 1) на новую модель (migration discipline, без shim).

- **Asymmetric (Ed25519)**: издатель — **gateway как mint-брокер** (он уже выпускает OAuth-
  токены) — держит ПРИВАТНЫЙ ключ в secrets-слое (keyring+restricted-file, запрет
  plaintext-env — закрывает N-8). Бэкенды держат ТОЛЬКО публичный ключ и **verify, мінтить
  не могут** → claims неподделываемы (Codex round 2 п.3).
- **claims** = `{iss:"gateway", aud:"<target-service>", sub:"<caller>", tenant_id, scopes:[...],
  iat, exp(TTL≤15м), kid}`. Verifier проверяет подпись pub-key + `aud`(=свой сервис) + `iss` +
  `kid`(ротация). Никакого `mcp.invoke`-универсала.
- **Scopes по сервису (Codex round 2 п.4)**: `memory.read`, `memory.propose`,
  `memory.write.internal`, `pm.read`, `pm.action`, `budget.read`, `budget.propose`, плюс
  отдельный `owner`/`internal.write`. Backend пускает tool только при наличии нужного scope.
- **Mint/refresh (Codex round 2 п.2)**: короткий TTL ⇒ **runtime mint**. Gateway-брокер даёт
  internal mint-endpoint; клиенты (`assistant-ui/app/mcp_client.py:169-180`, сейчас читает
  bearer статичной строкой :616) переводятся на **refreshing token-provider** (мінт перед
  истечением). Локальные агенты (Claude/Codex) получают токен от брокера, не из статичного файла.
- **Delivery-модель (Codex round 2 п.1)**: общий код — отдельный editable-пакет
  `service_identity/` (verify + claims-схема; mint только у gateway), подключается в каждый
  `pyproject.toml` по образцу `budget` в Assistant-UI (`assistant-ui/pyproject.toml:49-50`
  `[tool.uv.sources] budget = { path="../budget", editable=true }`). Пакет в `dependencies` +
  `tool.uv.sources` у `gateway`, `pm-mcp`, `ai-memory`, `budget`, `assistant-ui`.
- **warn → enforce** флаг на каждом бэкенде: warn = verify+redacted-log (пропускает);
  enforce = 401 missing / 403 invalid-scope (**не 503**). `kid`-ротация active/previous.
- **Сиквенс (зерно Codex round 1)**: P1 бэкенд проверяет подпись+`aud`+scope **локально** —
  НЕ блокируется shared policy; централизованный `decide()` на claims — отдельно в P2 (WP-C).

### 5.3 Mint-брокер: собственная авторизация, целостность ключей, крипто-профиль (Codex round 3)

**Авторизация брокера (Codex r3 п.1)** — иначе shared-HMAC-проблема заменяется на «любой
локальный процесс просит broker выдать owner-scope»:
- **mint endpoint local-only**: loopback bind, **исключён из Funnel и public OpenAPI**; audit
  без секретов (fingerprint `sha256[:8]`, brick #22).
- **caller authentication**: каждый caller (gateway-internal, `assistant-ui`, локальные агенты
  Claude/Codex) предъявляет per-caller bootstrap-credential (ACL-restricted в secrets-слое),
  НЕ анонимный loopback.
- **caller → allowed envelope registry** (декларативно): для каждого caller разрешённые
  `aud`/`sub`/`scopes`; broker отказывает в scope вне envelope.
- **owner-scope требует owner-auth**: мінт `owner`/`internal.write` для чувствительных путей
  требует owner-session/secret (ADR-0018), а не только caller-credential.

**Целостность ключей/registry (Codex r3 п.2)**: не только приватный ключ — public key и
caller-registry под **owner-only write ACL**; verifier пинит fingerprint pub-key и логирует
`kid`+fingerprint; тест на подмену `kid`/pub-key (атакующий не подсунет свою пару).

**Крипто-профиль (Codex r3 п.3)** — в каталоге brick'а нет → **candidate #32**:
- библиотека `PyJWT[crypto]` + `cryptography` (Ed25519); заменяет hand-rolled HS256 (`gateway/auth.py:55`).
- `alg` allowlist **только `EdDSA`**; запрет `none` и HS/RS-key-confusion; строгая валидация
  `iss`/`aud`/`iat`/`exp`/`kid` (минимальный leeway).

## 6. Этапы (Phases)

### P0 — Deploy-gate (до следующего рестарта служб)

**WP-A — `service_identity` пакет (Ed25519) + mint-брокер на gateway + провижн на ОБОИХ start-path + warn-фаза (N-1).** Subsystem: root `service_identity` + `gateway` + `tools` + `pm-mcp`.
- **Editable-пакет `service_identity/`** (§5.2 delivery): `verify(token, *, audience, public_key)`
  + claims-схема + scope-константы; `mint(...)` (Ed25519 private) — используется **только**
  gateway. Подключить в `pyproject.toml` каждой подсистемы по образцу
  `assistant-ui/pyproject.toml:49-50` (`[tool.uv.sources] service_identity = { path="../service_identity", editable=true }`).
- **Ключи**: gateway генерирует Ed25519 keypair; **приватный** — `.secrets/service-identity-ed25519.key`
  (ACL-restrict, `Protect-SecretFile`), читается только gateway; **публичный** — раздаётся
  бэкендам (`.secrets/service-identity-ed25519.pub` / env). `kid` для ротации.
- **Mint-брокер**: gateway internal endpoint выпускает короткие (TTL≤15м) токены с
  `aud`=target-сервис, проверенными `sub`/`scopes`. Никаких статичных long-lived токенов в `*_TOKEN_FILE`.
- **Авторизация брокера + целостность ключей + крипто (§5.3)**: mint endpoint local-only
  (вне Funnel/public-OpenAPI); caller-credential + envelope-registry; owner-scope только за
  owner-auth (ADR-0018); public-key/registry под owner-only ACL с fingerprint-pinning;
  библиотека `PyJWT[crypto]`+`cryptography`, `alg` EdDSA-only, запрет `none`/key-confusion.
- **Миграция pm-mcp**: заменить opaque `verify_authorization_header` (`auth.py:30-36`) на
  `service_identity.verify(audience="pm-mcp", public_key=...)`. Старый opaque-путь удалить.
- **Route/tool-level scope в pm-mcp (Codex r4 п.2)** — `aud=pm-mcp` недостаточно (слишком широко):
  `/internal/calendar/notify` → `internal.write`; MCP tools и `/openapi` → per-tool `pm.read`/
  `pm.action` (read vs write/approve); `/events` (SSE) → `pm.read`. Backend проверяет scope claim
  против карты route/tool, а не только `aud`.
- **Клиенты — refreshing provider (Codex r2 п.2)**: `assistant-ui/app/mcp_client.py` (статичный
  bearer :616) → провайдер, который минтит/обновляет токен у брокера перед `exp`. Локальные
  агенты (Claude/Codex) — токен от брокера, не из файла.
- **Провижн на обоих путях**: и `register_services.ps1`, и `run-user-session.ps1` раздают
  приватный ключ только gateway, публичный — бэкендам/клиентам; настраивают broker-URL.
- **Per-caller credential + envelope-registry как deploy-артефакт (Codex r4 п.3)** — НЕ только
  модель из §5.3: оба регистратора создают per-caller bootstrap-credential (ACL-restrict) для
  gateway-internal/`assistant-ui`/локальных агентов и материализуют envelope-registry
  (`.secrets/service-callers.json`, owner-only) с разрешёнными `aud`/`sub`/`scopes` на каждого caller.
- **warn→enforce флаг** в каждом верификаторе (ADR-0019 §3): warn — verify+redacted-log;
  enforce — 401/403. Дефолт warn до подтверждения провижна.
- Acceptance: keypair создан с restrict-ACL (приватный только у gateway); рестарт **обоими**
  путями даёт рабочую identity; gateway→pm-mcp и ui→pm-mcp проходят с Ed25519-токеном через
  refresh; **подделанный токен с чужим `sub`/owner-scope без приватного ключа отвергается**; warn-лог пуст; затем enforce.
- Тесты: `test_ed25519_mint_verify_roundtrip`, `test_token_expired_rejected`,
  `test_forged_claims_without_private_key_rejected`, `test_wrong_audience_rejected`,
  `test_client_refreshes_before_expiry`, `test_registrar_provisions_keypair`,
  `test_user_session_env_propagation`, `test_pm_mcp_warn_then_enforce`,
  `test_broker_rejects_unauthenticated_caller`, `test_broker_rejects_scope_outside_envelope`,
  `test_owner_scope_requires_owner_auth`, `test_substituted_public_key_rejected`,
  `test_alg_none_and_hs_confusion_rejected`, `test_mint_endpoint_excluded_from_funnel_and_openapi`.

### P1 — High (достроить F-3, БЕЗ зависимости от shared policy)

**WP-B — Backend Ed25519-verify parity для ai-memory :8765 и budget :8767 (F-3R).**
- Добавить `service_identity.verify(audience=<self>, public_key=...)` middleware
  (Ed25519 + per-service scope, warn→enforce) в `ai-memory/memory/mcp_app.py` и
  `budget/budget/mcp_app.py`. **Оба mount (Codex r2 п.5)**: и `/mcp`, и `/openapi`
  (`ai-memory/memory/openapi_bridge.py`+`daemon.py`) — ADR-0019 MCP/OpenAPI parity.
- `TransportSecuritySettings(enable_dns_rebinding_protection=True, allowed_hosts=[...])` на
  :8765/:8766/:8767 (по образцу `:8770` `proposals/app.py:433`).
- **Per-service scopes (не `mcp.invoke`)**: ai-memory read → `memory.read`, propose →
  `memory.propose`; write-tools `store_memory`/`store_memory_batch`/`store_summary`/
  `confirm_memory_entry` требуют `memory.write.internal`+`owner` и недоступны без них даже
  при прямом loopback-вызове. budget read → `budget.read`, propose → `budget.propose`.
  (`delete_memory` остаётся gateway-инвариантом — не backend-tool.)
- **WP-B НЕ ждёт WP-C** (зерно Codex r1): локальная проверка подписи+`aud`+scope закрывает
  открытый бэкенд немедленно; централизованная политика — отдельно в P2.
- **OpenAPI positive-path (Codex r3 п.4)**: не только negative-401 — добавить рабочий путь:
  bearer в generated OpenAPI `securityScheme`, CORS preflight с `Authorization`, реальный вызов
  tool через `/openapi/*` с валидным токеном (ai-memory bridge монтируется в `daemon.py:55`).
- Тесты: `test_aimemory_mcp_requires_token`, `test_aimemory_openapi_requires_token`,
  `test_aimemory_openapi_positive_path_with_token`, `test_openapi_security_scheme_advertises_bearer`,
  `test_openapi_cors_preflight_allows_authorization`,
  `test_budget_requires_token`, `test_backend_invalid_scope_403`,
  `test_backend_wrong_audience_rejected`, `test_backend_dns_rebinding_rejected`,
  `test_direct_store_memory_denied_without_owner_scope`.

**WP-B2 — Миграция `ai-memory-proposals :8770` на Ed25519 (Codex r4 п.1).** Subsystem: `ai-memory`.
- Сейчас :8770 уже аутентифицирован ad-hoc shared-token + DNS-rebinding (`proposals/app.py:433`),
  gateway реально ходит туда для `/memory/propose` (`backends.py` target `ai_memory_proposals`).
  Мигрировать на `service_identity.verify(audience="ai-memory-proposals")`, scope `memory.propose`;
  старый shared-token удалить (migration discipline). Gateway-клиент
  `ai_memory_proposals_bearer_token` → Ed25519.
- **РЕШЕНИЕ по OAuth mini-surface proposals-демона (Codex r5 п.1) — УДАЛИТЬ.** На :8770 есть
  собственный OAuth-сервер: `/.well-known/oauth-authorization-server`,
  `/.well-known/oauth-protected-resource`, `/authorize`, `/token`, HS256 `_issue_access_token`/
  `_verify_hs256_jwt`, legacy-bearer (`ProposalsAuthMiddleware` `app.py:315-354`, `:300-312`).
  Под ADR-0001 gateway — единственный ingress, :8770 loopback-only/вне Funnel, а gateway
  аутентифицируется bearer'ом (→Ed25519), НЕ через этот OAuth-флоу → surface избыточен.
  Удалить все перечисленные роуты + HS256 issue/verify + legacy-bearer; оставить только
  Ed25519-verify (WP-B2) + health/ready. Reversibility: если найдётся прямой OAuth-консьюмер
  (сейчас не существует — surface не достижим извне), пересмотреть.
- Acceptance: `/memory/propose` через gateway работает на Ed25519; старый shared-token и
  **OAuth-роуты (`/authorize`,`/token`,`/.well-known/*`) мертвы (404/удалены)**.
- Тесты: `test_proposals_requires_ed25519_token`, `test_proposals_propose_scope_enforced`,
  `test_proposals_old_shared_token_dead`, `test_proposals_oauth_surface_removed`,
  `test_proposals_hs256_path_removed`.

**WP-C — перенесён в P2** (shared claims-based policy; см. ниже). Причина: не блокировать
fail-closed backend (WP-B) на policy-архитектуре.

### P2 — Medium (shared policy + остаточный gateway hardening)

- **WP-C** Shared claims-based policy (N-9): `decide(principal, tool, resource)` в **отдельном
  shared editable-пакете** (`service_policy/`, по образцу `service_identity` §5.2), а НЕ импортом
  `gateway/gateway/policy.py` в бэкенды (Codex r3 minor — сохраняет subsystem boundaries).
  `gateway/policy.py` мигрирует на этот пакет. `principal` из verified Ed25519-claims. Зависит от WP-B. → candidate brick #28.
- **WP-D** Audit: метаданные/хэши вместо сырых тел/заголовков (N-2); serialized writer +
  WORM/anchor головы цепочки (N-3); retention/ротация. → candidate brick #29.
- **WP-E** Pre-parse body-cap + concurrency-cap + trusted-forwarded client-IP для per-IP
  throttle (N-4); пул MCP-сессий (N-5).
- **WP-F** Calendar-webhook: gateway проверяет channel-token до форварда ИЛИ документирует
  pm-mcp как единственный валидатор + сужает rate-bucket (N-6).
- **WP-G** assistant-ui: не выдавать cookie на неаутентифицированный localhost GET (N-7);
  previous-secret из keyring (N-8); удалить мёртвый `_healthz_allowed` (N-11).

### P3 — Стратегическое (полный объём — реализация, решение пользователя)

- **WP-H** Multi-tenant `tenant_id` (N-10) — разложено по БД (Codex r2 п.6), НЕ «просто колонка».
  Реализует **существующий forward кирпичика #3** (нового brick не нужно). ADR-0024. Для КАЖДОЙ БД:
  - колонка `tenant_id` + индексы под актуальные query-паттерны (composite с существующими фильтрами);
  - backfill `self` (idempotent миграция, `schema_version` bump);
  - **query filters**: каждый read/write-путь фильтрует по `principal.tenant_id` (не только колонка);
  - **unique constraints** пересмотреть на `(tenant_id, ...)` где уникальность была глобальной;
  - **memory.db**: FTS5 rebuild с учётом tenant; lineage/graph-рёбра в пределах tenant;
  - **tasks (pm-mcp)**: **global `#NNN` СОХРАНЯЮТСЯ** (ADR-0003 + AGENTS — не переоткрывать;
    Codex r4 п.4). `tenant_id` ограничивает только видимость/запросы/связи (cross-ref и
    dependency-links валидны лишь внутри tenant), счётчик остаётся глобальным; goals-дерево per-tenant;
  - **budget.db**: ledger/accounts/FX per-tenant, проверка инвариантов балансов после backfill;
  - **calendar**: watch-channels/sync-state per-tenant.
  - **Acceptance — отдельный per-БД** (`test_<db>_tenant_isolation`, `test_<db>_backfill_self_idempotent`,
    `test_<db>_unique_constraint_per_tenant`).
- **WP-I** Security monitoring: метрики denied/rate_limited/new-client_id → пороги →
  уведомление. → candidate brick #30.
- **WP-J** Sandbox/allowlist для внешних MCP и делегированных агентов: ограниченный tool-set
  (read/propose), отдельный trust-tier + rate/audit, provenance-пометка ответа
  (provenance/trust_tier фундамент уже есть в ai-memory).
- **WP-K** At-rest encryption: **СПАЙК сначала** (не сразу SQLCipher — зерно Codex r1). Сравнить
  SQLCipher vs OS-level (EFS/BitLocker) vs field-level vs keyring-wrapped; классификация
  `budget.db` (финансы) / `memory.db` (личное). **Явные факторы оценки (Codex r2 п.7):**
  - daily backup compatibility (зашифрованный файл бэкапится/восстанавливается);
  - WAL/checkpoint поведение под шифрованием (и FTS5-индексы);
  - key recovery (потеря ключа = потеря данных — процедура восстановления/escrow);
  - **session-0 vs logon доступ к ключу** (NSSM session-0 vs logon-task — ключ должен читаться
    в обоих, ср. vault session-0 проблему id 1629);
  - поведение при **недоступном keyring** (fail-closed vs degraded read-only vs отказ старта);
  - модель угроз: live-compromise (шифрование не спасает) vs диск-at-rest/кража носителя (спасает);
  - **критерий выбора** механизма (что именно склоняет к SQLCipher vs EFS/BitLocker vs field-level).
  → **ADR-0023** с выбором (ADR-0022 уже занят) → implementation tasks #912 (root key contract/SQLCipher POC/runbook), #913 (ai-memory memory.db), #914 (budget budget.db); ключи в secrets-слое; миграция БД с verify целостности. → candidate brick #31.

## 7. Недостающие компоненты (кирпичики)

| ID | Компонент | Приоритет | Лечит |
|----|-----------|-----------|-------|
| MC-A | Signed service-token lib + backend parity (auth + rebinding на :8765/:8767) | P0/P1 | F-3R, N-1 |
| MC-B | Token provisioning на обоих start-path + warn→enforce | P0 | N-1 |
| MC-C | Shared claims-based policy engine (gateway+backends) | P2 | N-9 |
| MC-D | Tamper-evident audit pipeline | P2 | N-2, N-3 |
| MC-E | API gateway hardening (caps, pool, forwarded-IP) | P2 | N-4, N-5 |
| MC-F | Security monitoring / alerting | P3 | детект |
| MC-G | Multi-tenant partitioning (tenant_id, реализация) | P3 | N-10 |
| MC-H | External-MCP / delegated-agent sandbox (реализация) | P3 | delegated/external |
| MC-I | Data-at-rest encryption (spike→ADR-0023→#912/#913/#914 реализация) | P3 | budget.db/memory.db |

## 8. Архитектурная эволюция (закладываем сейчас)

1. Service-token несёт `principal` (subject+tenant+scopes) → бэкенды решают по claims, а не
   по сетевой позиции. Снимает «loopback = identity» окончательно.
2. `tenant_id` в схемы немедленно (backfill `self` дёшев сейчас, дорог потом).
3. owner-identity (shared secret/cookie) — seam к реальному OIDC IdP для multi-user; client
   vs owner identity уже разделены (ADR-0018).
4. Mobile: текущий OAuth/PKCE + refresh-rotation готовы; per-device — owner-gated DCR.
5. External MCP: sandbox + provenance trust-tier до первого внешнего сервера.

## 9. Риски

- **R1**: WP-A enforce без полного провижна рвёт внутренний трафик → warn-фаза обязательна,
  enforce только после подтверждённого провижна на **обоих** start-path (NSSM+logon) и пустого warn-лога.
- **R2**: WP-B на живых :8765/:8767 → изолированный verify (brick #17 Track B) до рестарта NSSM.
- **R3**: Рестарт служб активирует ВЕСЬ коммит `a620947` (включая pm-mcp fail-closed) — деплой
  делать после WP-A, единым окном, с проверкой свежего PID (AI-memory id 1529).
- **R4**: tenant_id-миграция БД — backfill + проверка целостности FTS5/индексов.

## 10. Критерии приёмки

- [ ] P0: `service_identity` editable-пакет во всех `pyproject.toml`; Ed25519 keypair (приватный
      только у брокера); **брокер авторизует caller'ов** (credential + envelope-registry,
      owner-scope за owner-auth, endpoint вне Funnel/OpenAPI); public-key/registry под owner-only ACL;
      `PyJWT`+`cryptography`, EdDSA-only; pm-mcp мигрирован opaque→Ed25519; клиенты на refresh-провайдере;
      **подделка токена/подмена pub-key отвергнуты**; провижн на **обоих** start-path; warn-лог пуст;
      enforce включён; деплой `a620947` не рвёт внутренний трафик.
- [ ] P1: ai-memory :8765 (**оба mount /mcp и /openapi**) и budget :8767 fail-closed
      (Ed25519 + per-service scope) + DNS-rebinding; прямой `store_*` без `memory.write.internal`+owner
      отклонён; неверный `aud` отвергнут; **WP-B не блокирован WP-C**.
- [ ] P1 (WP-B2): `ai-memory-proposals :8770` на Ed25519 (`memory.propose`), `/memory/propose`
      через gateway работает, **OAuth mini-surface + HS256 + legacy-bearer удалены**; pm-mcp
      route-scope enforced (`internal.write`/`pm.read`/`pm.action` по route/tool, не только `aud`).
- [ ] P2: claims-based `decide()` в бэкендах; audit без сырых тел/заголовков + tamper-evident;
      pre-parse caps; webhook/UI/secret фиксы (N-6/N-7/N-8/N-11).
- [ ] P3 (полный объём): `tenant_id` сквозной + backfill `self`; sandbox внешних MCP;
      at-rest ADR зафиксирован как ADR-0023; реализация вынесена в #912/#913/#914 после SQLCipher POC.
- [ ] ADR-0019 (signed service token) / ADR-0024 (multi-tenant) / ADR-0023 (at-rest; ADR-0022 занят) написаны;
      ADR-0019 расширен surface-matrix §5.1.
- [ ] Docs-sweep: ARCHITECTURE/README бэкендов не называют loopback security boundary;
      grep `без auth`/`trust-by-default`/`loopback` чист (brick #5 уже синхронен — не трогаем).
- [ ] Rollout checklist (§10.1) пройден; **rollback-процедура (§10.2) отрепетирована** (enforce→warn без возврата к opaque); Security regression suite (§11) зелёный.
- [ ] Candidate bricks #27–#32 промоутятся только после landing+evidence (с подтверждением).
- [ ] Per-WP work items заведены по subsystem (absolute `project_path` через `list_projects`) с `link_task_dependency`.

## 10.1 Rollout checklist (обязателен перед enforce; добавлено по Codex)

- [ ] verify **fresh PID** после рестарта (status Running ≠ свежий код — AI-memory id 1529).
- [ ] smoke `gateway → pm-mcp/ai-memory/budget` **с** валидным signed-токеном = 200.
- [ ] smoke `assistant-ui → pm-mcp` **с** токеном = 200 (оба start-path: NSSM и logon).
- [ ] negative: тот же вызов **без** токена = 401 (не 503, не pass) на :8765/:8766/:8767.
- [ ] **WP-B2 (Codex r6)**: smoke `gateway → /memory/propose` с Ed25519 = 200; direct :8770
      без токена/со старым shared-token = 401; OAuth-роуты на :8770 (`/authorize`,`/token`,`/.well-known/*`) = 404.
- [ ] warn-фаза: warn-лог redacted и **пуст** (никто не ходит без токена) перед enforce.
- [ ] env/task fingerprints совпадают на NSSM и logon-path, **без вывода значений секретов**.

## 10.2 Rollback после enforce (Codex round 3 п.5)

Аварийный откат **НЕ возвращает** к opaque/static-token модели (она удалена, migration discipline) —
только enforce→warn в рамках Ed25519:
1. Переключить флаг каждого бэкенда `enforce → warn` (env/flag list: `*_S2S_ENFORCE=0`),
   токены продолжают verify, но отсутствие/невалидность не блокирует — трафик восстановлен.
2. Порядок рестарта: бэкенды (:8765/**:8770**/:8767/:8766) → gateway-брокер; проверить **fresh PID**
   каждого (id 1529) и пустой error-лог. **Не забыть proposals :8770** (Codex r6).
3. Smoke: gateway→каждый бэкенд (включая `/memory/propose` на :8770) и ui→pm-mcp с валидным
   токеном = 200; затем диагностировать причину.
4. Если скомпрометирован ключ — ротация `kid` (active/previous), а не откат к warn.
- Тесты: `test_enforce_to_warn_rollback_keeps_ed25519` (откат не реактивирует opaque-путь),
  `test_rollback_includes_proposals_daemon` (:8770 в порядке рестарта и smoke).

## 11. Security regression suite (red-team, Phase 2)

- `test_ed25519_mint_verify_roundtrip`, `test_token_expired_rejected` — mint/verify Ed25519 claims.
- `test_forged_claims_without_private_key_rejected` — без приватного ключа нельзя выписать `sub`/owner-scope (Codex r2 п.3).
- `test_wrong_audience_rejected` — токен с `aud` другого сервиса отвергается verifier'ом.
- `test_client_refreshes_before_expiry` — клиент минтит новый токен до `exp` (Codex r2 п.2).
- `test_aimemory_mcp_requires_token`, `test_aimemory_openapi_requires_token`, `test_budget_requires_token` — :8765(/mcp,/openapi)/:8767 без токена → 401.
- `test_backend_invalid_scope_403` — токен без нужного per-service scope → 403 (не 503).
- `test_direct_store_memory_denied_without_owner_scope` — прямой `store_*` без `memory.write.internal`+owner отклонён.
- `test_backend_dns_rebinding_rejected` — Host-spoof на loopback daemon отклонён.
- `test_registrar_provisions_keypair`, `test_user_session_env_propagation` — провижн (приватный только gateway) на обоих путях.
- `test_pm_mcp_warn_then_enforce` — warn пропускает+логирует, enforce 401.
- `test_pm_mcp_openapi_mount_authenticated` — pm-mcp `/openapi` не отдаёт tools без токена.
- `test_broker_rejects_unauthenticated_caller`, `test_broker_rejects_scope_outside_envelope` — авторизация брокера (Codex r3 п.1).
- `test_owner_scope_requires_owner_auth` — owner-scope мінтится только за owner-auth (ADR-0018).
- `test_substituted_public_key_rejected` — подмена pub-key/`kid` отвергается (Codex r3 п.2).
- `test_alg_none_and_hs_confusion_rejected` — только `EdDSA`, запрет `none`/HS-confusion (Codex r3 п.3).
- `test_aimemory_openapi_positive_path_with_token` — рабочий вызов tool через `/openapi/*` с токеном (Codex r3 п.4).
- `test_enforce_to_warn_rollback_keeps_ed25519` — откат не реактивирует opaque-путь (Codex r3 п.5).
- `test_audit_no_raw_body_headers` — в audit нет сырых body/headers.
- `test_audit_concurrent_append_chain_intact` — конкурентный append не рвёт hash-chain.
- `test_request_cap_before_parse` — body-cap срабатывает ДО парсинга JSON.
- `test_assistant_ui_fail_closed_without_token`, `test_localhost_get_no_session_cookie` — N-7.
- `test_previous_secret_not_from_env` — N-8.
- **pm-mcp route-scope (Codex r5 п.2)**: `test_pm_calendar_notify_requires_internal_write`,
  `test_pm_events_requires_pm_read`, `test_pm_read_tool_denied_with_only_pm_action`,
  `test_pm_action_tool_denied_with_only_pm_read`.
- **WP-B2 proposals (Codex r5 п.3)**: `test_proposals_propose_via_gateway_ed25519`,
  `test_proposals_old_shared_token_dead`, `test_proposals_wrong_scope_rejected`,
  `test_proposals_oauth_surface_removed`, `test_proposals_hs256_path_removed`.
- **Provisioning (Codex r5 п.4)**: `test_registrar_provisions_caller_credentials`,
  `test_envelope_registry_materialized_both_paths` (per-caller creds + `.secrets/service-callers.json`
  на NSSM и logon).
- `test_tenant_isolation` — principal tenant A не видит rows tenant B.

## 12. Статус реализации (2026-06-12, Codex)

- PM-MCP #895 (WP-A): реализованы `service_identity` editable-пакет, Ed25519 service-токены, провижн обоих start-path, миграция pm-mcp на signed-token verify и route/tool-scope enforcement.
- PM-MCP #896 (WP-B): реализован Ed25519 verify parity для `ai-memory` :8765 и `budget` :8767, включая `/mcp`, `/openapi`, per-service scopes и DNS-rebinding protection.
- PM-MCP #897 (WP-B2): `ai-memory-proposals` :8770 переведён на Ed25519 `memory.propose`; legacy shared-token, HS256 и OAuth mini-surface удалены.
- Проверки зелёные: `service_identity` pytest; `gateway` unittest; `pm-mcp-server` pytest; `ai-memory` unittest; `budget` pytest; `assistant-ui` pytest; `ruff check .` во всех затронутых Python-подсистемах.
- План закрыт: P0/P1/P2/P3 (`WP-A…WP-K`) и follow-up #912/#913/#914 в PM-MCP имеют статус `Готово`; ADR-0023 и runbook обновлены; outcome записан в AI-memory.

Pre-close engineering retrospective:

| Axis | Verdict | Note |
|---|---|---|
| `tech-stack-choices.md` | follow-up-task | Возник reusable pattern `service_identity`/Ed25519 service tokens; создан PM-MCP #898 на согласование промоута brick'а. |
| Design-system | no-change | UI/дизайн-система не затрагивались. |
| Skills | no-change | Повторяемый workflow не требует нового skill сверх текущих `central-plan-workflow`, `pm-mcp-task-flow`, `ai-memory-capture`. |
| Hooks | no-change | Новый детерминированный guard/hook не выявлен. |
## References

- Предшественник: `DONE/secure-agent-platform-hardening.md` (#853), коммит `a620947`.
- ADR-0018 owner-auth, ADR-0019 service identity layer / signed token,
  ADR-0024 (multi-tenant), ADR-0023 (at-rest, после spike; ADR-0022 занят roadmap ADR).
- Codex review «новой редакции» (2026-06-12), раунды 1–6 — AI-memory proposals #16–#20
  (round 1 #16; round 2 #17; round 3 #18; round 4 #19; rounds 5–6 #20). #16: direct
  `store_memory_batch` был заблокирован guard'ом формата batch-входа.
- AI-memory: id 1654 (Funnel loopback ≠ identity), 1660 (#858/#859/#863/#865 closed),
  1630 (DCR scope иммутабелен), 1529 (Running ≠ свежий PID), 1629 (logon-task start-path).
- Tech-stack: bricks #3 (tenant_id forward, line 48), #5 (loopback≠trust, lines 64-65),
  #22/#23/#24/#26 (`_engineering_rules/tech-stack-choices.md`).





