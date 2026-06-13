# Secure AI-Agent Platform — Hardening Phase 3

> Статус: **DRAFT (ожидает утверждения пользователя)**
> Автор: Claude (Principal Security Architect / Staff Architect / Red Team / TPM)
> Дата: 2026-06-13
> PM-MCP: #917 (audit+plan, В работе). Implementation work items — после утверждения.
> Предшественники: `DONE/secure-agent-platform-hardening.md` (#853, P0/P1),
> `secure-platform-hardening-phase2.md` (#894, P0/P1 закрыты; P2/P3 частично).
> Связанные ADR: ADR-0018 (owner-auth), ADR-0019 (service identity), ADR-0024 (multi-tenant),
> ADR-0023 (at-rest). **Новый требуемый ADR: ADR-0025 (mint-broker / key-custody model).**

---

## 1. Контекст и метод

Полный read-only аудит репозитория (5 подсистем + `service_identity` + `gateway/service_policy`)
по фактическому коду на 2026-06-13. Проверены: gateway ingress, S2S identity-слой, бэкенды
ai-memory/budget/pm-mcp, agent loop assistant-ui, проводка ключей (`register_services.ps1`,
`run-user-session.ps1`), фактические ACL `.secrets/` на диске (`icacls`).

**Главный вывод:** внешний периметр (OAuth/PKCE/DCR/owner-auth) — действительно закрыт и
сделан грамотно. Но **S2S identity-слой, в который Phase 1/2 вложили основную работу, в
реализованном виде не даёт изоляции**, потому что приватный Ed25519-ключ стал де-факто общим
секретом и физически читается любым локальным пользователем. Это аннулирует целевое свойство
«компрометация одной подсистемы не эскалирует на другие». Дополнительно Assistant-UI в
деплой-конфигурации остаётся без аутентификации, что разрушает контур approval-проверки.

## 2. Цель

1. Восстановить реальное свойство S2S-изоляции: claims неподделываемы И непривилегированный
   локальный процесс/скомпрометированная подсистема НЕ может выписать owner/cross-service токен.
2. Закрыть контур approval: Assistant-UI fail-closed по аутентификации в деплое.
3. Обеспечить целостность ключевого материала на диске (Windows ACL), на обоих start-path.
4. Подготовить платформу к multi-user/internet: tenant из verified-claims (не из аргумента),
   backend-side policy, egress-контроль.

## 3. Ограничения и инварианты

- **Migration discipline** (AGENTS.md Q): новый путь → миграция → удаление; без shim'ов.
- **Repo files > task status**: phase2 §12 объявил WP-B2 «OAuth surface удалён», но код
  (`ai-memory/memory/proposals/app.py:99-106`) ещё содержит bypass-allowlist OAuth-путей —
  ground truth = код.
- **Subsystem boundaries** (ADR-0001): gateway — единственный внешний ingress.
- Каждое Python-изменение: `uv run ruff check .` + тесты затронутой подсистемы.
- Новые tech-stack bricks/hooks — только после явного подтверждения пользователя.

## 4. Находки (только с code-evidence)

| ID | Sev | Суть | Evidence |
|----|-----|------|----------|
| **F31** | 🔴 Critical | **Общий приватный ключ аннулирует S2S.** Один Ed25519 приватный ключ — единственный издатель для всех сервисов; раздаётся gateway+assistant-ui через env и читается ai-memory по default-path; на диске **world-readable** (`BUILTIN\Users:(RX)`, `Authenticated Users:(M)`). Verifier НЕ связывает caller→subject→scope (envelope проверяется только client-side в провайдере). ⇒ любой локальный процесс/скомпрометированная подсистема минтит `sub=gateway`, любые scopes, любой `aud`, любой `tenant_id`. | `service_identity/auth.py:142-208` (verify без subject↔scope), `provider.py:42-48` (envelope client-side), `config.py:165-170` (`_protect_owner_only` no-op на `nt`), `register_services.ps1:355,370`, `backends.py:66`, on-disk `icacls .secrets/service-identity/ed25519-private.key` |
| **F32** | 🔴 Critical | **Assistant-UI без auth в деплое.** `auth_enabled()` = False если `~/.assistant-os/auth.token` отсутствует; ни `register_services.ps1`, ни `run-user-session.ps1` его не создают и не задают `ASSISTANT_UI_AUTH_TOKEN_PATH`. UI управляет agent loop, читает memory/budget и **апрувит budget/memory proposals** (контур approval). Нет Host/DNS-rebinding-проверки на :8000. | `assistant-ui/app/security.py:43-46`, `register_services.ps1:350-362` (нет token env) |
| **F33** | 🟠 High | **Tenant из аргумента, не из claims.** memory/budget tools берут `tenant_id` из аргумента инструмента и игнорируют `claims.tenant_id`; pm-mcp calendar_notify берёт `x-pm-tenant-id` из заголовка. ⇒ при втором tenant — тривиальный cross-tenant read/write. Forward-блокер для multi-user/internet. | `ai-memory/memory/storage.py:327-341`, `ai-memory/memory/mcp_app.py:195-258`, `pm-mcp-server/app/http_transport.py:125-133` |
| **F34** | 🟠 High | **Budget: scope-сегрегация сломана.** middleware budget использует статичный `required_scopes={budget.read,budget.propose}` (intersection) без per-tool resolver ⇒ токен `budget.read` вызывает `budget_propose_*`. ai-memory делает правильно (per-tool `_required_scopes`). | `budget/budget/mcp_app.py:241-247` vs `ai-memory/memory/daemon.py:61-71` |
| **F35** | 🟠 High | **Целостность public-key/caller-registry не защищена на Windows.** `ed25519-public-keys.json` и `service-callers.json` имеют тот же inherited ACL (Authenticated Users: Modify). ⇒ атакующий добавляет свой `kid`→pubkey в реестр и минтит своим приватным ключом, проходя verify. | on-disk `icacls .secrets/service-identity/*`, `config.py:165-170` |
| **F36** | 🟡 Medium | **Нет backend-side policy (N-9).** `store_memory`/`store_summary`/`confirm_memory_entry` — реальные backend write-tools под scope `memory.write.internal` (минтится общим ключом); `DENIED_TOOLS` enforce только на gateway. ⇒ memory-poisoning прямым loopback-вызовом в обход proposal/secret-review. (Secret-scan на direct store работает — F-4 закрыт.) | `ai-memory/memory/mcp_app.py:121-188`, `gateway/service_policy/policy.py:71-78` |
| **F37** | 🟡 Medium | **Audit anchor не WORM/external (остаток N-3).** Serialized writer + anchor `*.head.json` в той же директории с тем же ACL ⇒ атакующий с FS-write переписывает лог и anchor согласованно. | `gateway/gateway/audit.py:58-107` |
| **F38** | 🟡 Medium | **Нет пула MCP-сессий (N-5).** Каждый backend-вызов = `asyncio.run` + новая сессия/`initialize`; усиливает DoS pre-auth. | `gateway/gateway/backends.py:174-200` |
| **F39** | 🟡 Medium | **Calendar webhook форвардит произвольные `X-Goog-*` до валидации.** gateway не проверяет channel-token, форвардит в pm-mcp `/internal/calendar/notify` (там валидация). Pre-validation амплификация к internal route. | `gateway/gateway/app.py:597-671`, `backends.py:123-152` |
| **F310** | 🟢 Low | **Незавершённая миграция WP-B2.** `ProposalsAuthMiddleware` bypass-allowlist всё ещё содержит `/authorize`,`/token`,`/.well-known/*` (handlers удалены, allowlist — мёртвая конфигурация, противоречит «removed»). | `ai-memory/memory/proposals/app.py:99-106` |
| **F311** | 🟢 Low | **OAuthFlow in-memory pending/codes без TTL-эвикции.** `_pending_by_csrf`/`_codes` растут при незавершённых authorize; эвикция только на consume. Мелкий memory-DoS. | `gateway/gateway/oauth_flow.py:40-41` |

**Перепроверено как уже закрытое (не завышать):** F-1 legacy `/token` grant (app.py:293-295,350),
F-2 public DCR/consent (app.py:359, owner-gated authorize), F-4 secret-scan на direct store
(storage.py:341), F-5 capability-gating agent loop (agent_loop.py:397-430, untrusted-content guard),
N-2 raw body/headers в audit (теперь sha256+keys, app.py:1186-1207), N-8 previous-secret
(теперь keyring, `secrets.py:31-36`), N-11 dead `_healthz_allowed` (удалён). DNS-rebinding
protection присутствует на :8765/:8767/:8770.

## 5. Недостающие кирпичики (Phase 6)

| ID | Компонент | Зачем | Лечит |
|----|-----------|-------|-------|
| MC-1 | Реальный mint-broker ИЛИ per-service keypairs ИЛИ server-side envelope-binding в verifier | claims неподделываемы + caller не может выписать чужой scope | F31, F35 |
| MC-2 | Secrets-custody с принудительным Windows ACL (DPAPI/keyring) для всего key-material на обоих start-path | приватный ключ/registry не world-readable | F31, F35 |
| MC-3 | Assistant-UI identity provisioning + fail-closed + OIDC seam | контур approval защищён; задел под multi-user | F32 |
| MC-4 | Tenant context propagation (principal→backend, enforced из claims) | реальная multi-tenant изоляция | F33 |
| MC-5 | Backend-side shared policy (`service_policy` реально общий, бэкенды консультируют `decide()`) | DENIED_TOOLS/policy и на loopback | F36 |
| MC-6 | Tamper-evident external/WORM audit anchor | audit неизменяем при FS-компрометации | F37 |
| MC-7 | Security monitoring delivery (пороги → реальное уведомление, не только файл) | детект mint-аномалий/denied-всплесков | детект F31/F32 |
| MC-8 | External-MCP / delegated-agent sandbox с backend-enforcement trust-tier | внешние MCP/делегаты ограничены на бэкенде, не только в gateway-профиле | future |
| MC-9 | Data-at-rest encryption (#912/#913/#914, блок SQLCipher) | budget.db/memory.db at-rest | future |
| MC-10 | Egress/SSRF-контроль на бэкендах с внешними fetch (calendar/obsidian/google) | ограничить исходящие из loopback-демонов | F39, future |

## 6. Этапы

### P0 — Critical (до следующего рестарта/любого расширения доступа)

**WP-A — Восстановить S2S-изоляцию (F31, F35).** Subsystem: `service_identity` + все бэкенды + `tools`.
Выбрать ОДИН целевой путь (решение пользователя, см. §9 «Открытые решения»):
- **Вариант A1 (брокер):** реально реализовать mint-endpoint на gateway (loopback-only, вне Funnel/OpenAPI);
  приватный ключ ТОЛЬКО у gateway; assistant-ui/ai-memory/локальные агенты получают токен по
  per-caller bootstrap-credential; verifier как есть. Убрать `private_key_file` env у assistant-ui;
  убрать чтение приватного ключа из ai-memory (перевести на брокер).
- **Вариант A2 (server-side envelope binding):** оставить распределённый минт, НО verifier
  дополнительно проверяет `(subject, aud, scopes)` против owner-only signed envelope-registry
  (caller не может выписать scope вне своего envelope, даже имея ключ). Требует подписи реестра.
- **Вариант A3 (per-service keypairs):** у каждого caller свой приватный ключ; verifier пинит
  `kid`→subject; компрометация одного не даёт impersonate другого.
- Во ВСЕХ вариантах: `ensure_key_material` ДОЛЖЕН выставлять owner-only ACL на Windows
  (заменить no-op `_protect_owner_only`: `icacls`/`SetAccessRuleProtection` через ctypes или
  обязательный вызов `Protect-SecretFile` до старта демонов); запрет старта при world-readable ключе.
- Acceptance: `icacls` приватного ключа = только owner+Admins+SYSTEM+service-account; локальный
  непривилегированный процесс НЕ может выписать owner/cross-service токен (red-team тест);
  подмена `kid`/pubkey в реестре отвергается.
- Тесты: `test_private_key_not_world_readable`, `test_unprivileged_process_cannot_mint_owner_scope`,
  `test_substituted_public_key_rejected`, `test_caller_cannot_request_scope_outside_envelope_serverside`.

**WP-B — Assistant-UI fail-closed (F32).** Subsystem: `assistant-ui` + `tools`.
- `auth_enabled()` → fail-closed: если token не сконфигурирован в деплой-профиле — отказ старта
  (не «auth off»). `register_services.ps1` и `run-user-session.ps1` провижнят `auth.token`
  (ACL-restrict) и `ASSISTANT_UI_AUTH_TOKEN_PATH`.
- Host/DNS-rebinding allowlist на :8000 (как на бэкендах).
- Acceptance: запрос к `/api/*` и `/budget/proposals` без токена = 401 на свежем деплое;
  rebinding Host-spoof отклонён.
- Тесты: `test_assistant_ui_fail_closed_when_token_unprovisioned`, `test_ui_rejects_foreign_host_header`.

### P1 — High

**WP-C — Tenant из verified-claims (F33).** Subsystem: `ai-memory` + `budget` + `pm-mcp`.
- Бэкенды берут `tenant_id` из `scope["service_identity"].tenant_id` (verified claim), а аргумент
  инструмента/заголовок — игнорируют или валидируют на равенство claim. Реализует существующий
  forward brick #3 (колонки уже есть: budget.db tenant_id присутствует). ADR-0024.
- Acceptance: tool-call с `tenant_id=другой` при claim `self` — игнор/деny; cross-tenant read/write невозможен.
- Тесты: `test_tenant_from_claim_not_argument`, `test_cross_tenant_write_denied`.

**WP-D — Budget per-tool scope (F34).** Subsystem: `budget`.
- Добавить `scope_resolver` (как в `ai-memory/memory/daemon.py:61-71`): read-tools→`budget.read`,
  propose-tools→`budget.propose`. Убрать статичный `required_scopes` intersection.
- Тесты: `test_budget_read_token_cannot_propose`, `test_budget_propose_token_cannot_read_transactions`.

**WP-E — Backend-side policy (F36).** Subsystem: `service_policy` (сделать реально общим editable-пакетом
с `pyproject.toml`) + бэкенды.
- Перенести `gateway/service_policy/` в корневой `service_policy/` (сейчас пустой) как editable-пакет;
  gateway и бэкенды консультируют `decide(principal, tool, resource)` из claims. `DENIED_TOOLS`/owner-gating
  работают и на прямой loopback-вызов.
- Тесты: `test_direct_store_memory_denied_without_owner_on_backend`, `test_backend_consults_shared_policy`.

### P2 — Medium

- **WP-F** Tamper-evident audit: external/WORM anchor (вне директории лога, иной ACL/носитель) (F37).
- **WP-G** MCP session pool в gateway backends (F38).
- **WP-H** Calendar webhook: gateway валидирует channel-token до форварда ИЛИ документирует pm-mcp как
  единственный валидатор + сужает rate-bucket (F39).
- **WP-I** Cleanup миграции WP-B2: убрать OAuth-пути из proposals bypass-allowlist (F310);
  TTL-эвикция OAuthFlow pending/codes (F311).

### P3 — Стратегическое (multi-user / internet / external-MCP)

- **WP-J** External-MCP/delegated sandbox с backend-enforcement trust-tier (MC-8): профили
  `sandbox`/`delegated` уже есть в gateway (`app.py:113-123`), но бэкенды не различают trust-tier —
  довести enforcement до бэкенда + provenance-пометку.
- **WP-K** Security monitoring delivery (MC-7): `monitoring.py` пишет alert-файл; добавить порог→доставку.
- **WP-L** Egress/SSRF-контроль на бэкендах с внешними fetch (MC-10).
- **WP-M** Data-at-rest encryption (#912/#913/#914, MC-9) — после разблокировки SQLCipher POC.
- **WP-N** OIDC seam для owner→multi-user (MC-3 продолжение, ADR-0024).

## 7. Кандидаты в tech-stack bricks (промоут ТОЛЬКО после landing+evidence и подтверждения)

- candidate **#27** — service-token custody model (выбранный вариант A1/A2/A3) + обязательный
  Windows-ACL на key-material. Обновляет/заменяет наброски brick #23/#5.
- candidate **#28** — backend-side shared policy package `service_policy` (`decide()` из claims).
- candidate **#29** — tamper-evident external audit anchor.
- candidate **#30** — tenant-from-claim enforcement pattern (дополняет brick #3 forward).
- candidate **#31** — fail-closed local UI auth provisioning (assistant-ui).
- Hook-кандидат: pre-deploy guard, проверяющий ACL `.secrets/**` (world-readable = fail) —
  оформить через `hook-authoring` при подтверждении.

## 8. Риски

- **R1**: WP-A меняет проводку ключей — изолированный verify + warn-фаза до рестарта; деплой единым окном.
- **R2**: WP-B fail-closed может заблокировать UI при отсутствии провижна — провижн в том же
  коммите, что и флаг; rollback = warn.
- **R3**: WP-C/WP-E трогают живые БД/политику — backfill `self` идемпотентно, regression на cross-tenant.
- **R4**: смена `service_policy` location — атомарный коммит (migration discipline), удалить пустой корневой stub.

## 9. Открытые решения (нужно подтверждение пользователя)

1. **WP-A вариант**: A1 (реальный брокер) / A2 (server-side envelope) / A3 (per-service keypairs)?
   Рекомендация: **A3 для минимального blast-radius** + A2-binding как defense-in-depth; A1 дороже
   операционно (нужен живой mint-endpoint в критпути).
2. **Объём P3 сейчас**: заводить P3 work items сразу или после закрытия P0/P1 (дисциплина #853 — не плодить stale).
3. **Промоут candidate bricks #27–#31 и hook ACL-guard** — после landing.

## 10. Критерии приёмки

- [ ] P0: приватный ключ не world-readable на обоих start-path; непривилегированный процесс не минтит
      owner/cross-service токен; assistant-ui fail-closed + провижн token; rebinding на :8000.
- [ ] P1: tenant из claims (cross-tenant denied); budget per-tool scope; backend консультирует shared policy.
- [ ] P2: audit external-anchored; MCP session pool; webhook pre-validation/rate-bucket; WP-B2 cleanup.
- [ ] ADR-0025 (key-custody) написан; ADR-0024 (multi-tenant) расширен tenant-from-claim.
- [ ] Docs-sweep: tech-stack brick #5/#23 синхронизированы с фактической custody-моделью.
- [ ] Security regression suite (§6 тесты) зелёный; rollout/rollback отрепетированы.
- [ ] Per-WP work items заведены по subsystem `project_path` с `link_task_dependency` (после утверждения).
