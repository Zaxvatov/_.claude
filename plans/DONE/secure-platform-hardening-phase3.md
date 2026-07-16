# Secure AI-Agent Platform — Hardening Phase 3

> Статус: **DONE (implementation closed; archived 2026-06-14)**
> Автор: Claude (Principal Security Architect / Staff Architect / Red Team / TPM)
> Дата: 2026-06-13
> Обновлено: 2026-06-14
> PM-MCP: #917 (audit+plan, Готово). Implementation work items: #928-#938;
> incident follow-ups: #944/#945; archive task: #948 (см. §11).
> Предшественники: `DONE/secure-agent-platform-hardening.md` (#853, P0/P1),
> `secure-platform-hardening-phase2.md` (#894, P0/P1 закрыты; P2/P3 частично).
> Связанные ADR: ADR-0018 (owner-auth), ADR-0019 (service identity), ADR-0024 (multi-tenant),
> ADR-0023 (at-rest). **Новый требуемый ADR: ADR-0025 (S2S key-custody model).**

---

## 1. Контекст и метод

Полный read-only аудит репозитория (5 подсистем + `service_identity` + `gateway/service_policy`)
по фактическому коду на 2026-06-13. Проверены: gateway ingress, S2S identity-слой, бэкенды
ai-memory/budget/pm-mcp, agent loop assistant-ui, проводка ключей (`register_services.ps1`,
`run-user-session.ps1`), фактические ACL `.secrets/` на диске (`icacls`).

Дополнительный runtime-факт после review: production services AI-Assistant работают как NSSM
services под `LocalSystem`, а user-session tasks простаивают. Поэтому keyring пользователя `Zaxva`
не является service-secret storage. Секреты сервисов должны лежать в отдельной
`C:\ProgramData\AI-Assistant\secrets` / `.secrets/service-identity/**` custody с ACL
`SYSTEM`+`Administrators`+конкретный service account, либо в keyring самого `LocalSystem`.
Запирать общие `data/` директории запрещено: это ломает доступ к SQLite/WAL для локальных tools.

ADR-0023 data-at-rest больше не является Phase 3 work package: #912, #913 и #914 проверены в
PM-MCP как `Готово` 2026-06-13. Phase 3 только учитывает этот контракт как предшественник и не
дублирует SQLCipher work items.

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
5. Отозвать/ротировать старый общий S2S key material и запретить возврат к shared-private-key
   модели без нового ADR.

## 3. Ограничения и инварианты

- **Migration discipline** (AGENTS.md Q): новый путь → миграция → удаление; без shim'ов.
- **Repo files > task status**: phase2 §12 объявил WP-B2 «OAuth surface удалён», но код
  (`ai-memory/memory/proposals/app.py:99-106`) ещё содержит bypass-allowlist OAuth-путей —
  ground truth = код.
- **Subsystem boundaries** (ADR-0001): gateway — единственный внешний ingress.
- **Service runtime topology**: deploy-сервисы работают под `LocalSystem`; user keyring и
  user-profile paths не считаются production custody.
- **ADR-0023 закрыт для Phase 3**: #912/#913/#914 done; data-at-rest не заводится повторно.
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
| MC-1 | Per-service keypairs (A3) + server-side envelope-binding в verifier (A2) | claims неподделываемы + caller не может выписать чужой scope | F31, F35 |
| MC-2 | Secrets-custody с принудительным Windows ACL для `ProgramData`/`.secrets/service-identity/**` на обоих start-path | приватный ключ/registry не world-readable и не user-keyring-only | F31, F35 |
| MC-3 | Assistant-UI identity provisioning + fail-closed + OIDC seam | контур approval защищён; задел под multi-user | F32 |
| MC-4 | Tenant context propagation (principal→backend, enforced из claims) | реальная multi-tenant изоляция | F33 |
| MC-5 | Backend-side shared policy (`service_policy` реально общий, бэкенды консультируют `decide()`) | DENIED_TOOLS/policy и на loopback | F36 |
| MC-6 | Tamper-evident external/WORM audit anchor | audit неизменяем при FS-компрометации | F37 |
| MC-7 | Security monitoring delivery (пороги → реальное уведомление, не только файл) | детект mint-аномалий/denied-всплесков | детект F31/F32 |
| MC-8 | External-MCP / delegated-agent sandbox с backend-enforcement trust-tier | внешние MCP/делегаты ограничены на бэкенде, не только в gateway-профиле | future |
| MC-9 | Emergency key-rotation/containment runbook для S2S material | старый общий ключ отозван; compromised key path закрыт | F31, F35 |
| MC-10 | Egress/SSRF-контроль на бэкендах с внешними fetch (calendar/obsidian/google) | ограничить исходящие из loopback-демонов | F39, future |

## 6. Этапы

### P0 — Critical (до следующего рестарта/любого расширения доступа)

**WP-0 — Emergency containment + ADR-0025.** Task: #928. Subsystem: root docs + `service_identity` + `tools`.
- Зафиксировать ADR-0025: целевая custody-модель = **A3 per-service keypairs + A2 server-side
  envelope-binding**. A1 mint-broker не реализуется в Phase 3 без отдельного ADR.
- Runbook: inventory текущего shared key material, emergency rotation, запрет старого `active` shared
  `kid`, проверка ACL, список сервисов, которые нужно рестартовать после rotation.
- Deploy custody: service secrets под `LocalSystem` лежат в отдельной SYSTEM/Admins-only директории
  (`C:\ProgramData\AI-Assistant\secrets` или `.secrets/service-identity/**` с явным ACL); user keyring
  `Zaxva` не считается production storage.

**WP-A — Восстановить S2S-изоляцию (F31, F35).** Task: #929, зависит от #928.
Subsystem: `service_identity` + все бэкенды + `tools`.
- **A3:** у каждого caller свой приватный ключ; verifier пинит `kid`→subject/caller. Компрометация
  одного private key не даёт impersonate другой сервис.
- **A2:** verifier дополнительно проверяет `(issuer, subject, aud, scopes)` против server-side
  envelope-registry. Client-side `ServiceTokenProvider` остаётся convenience layer, но не является
  security boundary.
- Public-key registry и caller-registry должны иметь owner-only ACL и fail-closed integrity checks.
- `ensure_key_material` ДОЛЖЕН выставлять owner-only ACL на Windows или явно падать с инструкцией
  operator action; no-op `_protect_owner_only` недопустим для deploy.
- Acceptance: `icacls` приватного ключа = только owner+Admins+SYSTEM+service-account; локальный
  непривилегированный процесс НЕ может выписать owner/cross-service токен (red-team тест);
  подмена `kid`/pubkey в реестре отвергается; старый shared key отозван.
- Тесты: `test_private_key_not_world_readable`, `test_unprivileged_process_cannot_mint_owner_scope`,
  `test_substituted_public_key_rejected`, `test_caller_cannot_request_scope_outside_envelope_serverside`.

**WP-B — Assistant-UI fail-closed (F32).** Task: #930, зависит от #928.
Subsystem: `assistant-ui` + `tools`.
- `auth_enabled()` → fail-closed только для deploy/service profile: если token не сконфигурирован —
  отказ старта/401, не «auth off». `dev-local` может иметь explicit opt-out только через отдельный
  env flag.
- `register_services.ps1` и `run-user-session.ps1` провижнят `auth.token` в service-secret storage
  (ACL-restrict) и `ASSISTANT_UI_AUTH_TOKEN_PATH`.
- Host/DNS-rebinding allowlist на :8000 (как на бэкендах).
- Acceptance: запрос к `/api/*` и `/budget/proposals` без токена = 401 на свежем деплое;
  rebinding Host-spoof отклонён.
- Тесты: `test_assistant_ui_fail_closed_when_token_unprovisioned`, `test_ui_rejects_foreign_host_header`.

### P1 — High

**WP-C — Tenant из verified-claims (F33).** Tasks: #932 (ai-memory), #933 (budget), #934 (pm-mcp),
зависят от #929 и #931. Subsystem: `ai-memory` + `budget` + `pm-mcp`.
- Бэкенды берут `tenant_id` из `scope["service_identity"].tenant_id` (verified claim), а аргумент
  инструмента/заголовок — игнорируют или валидируют на равенство claim. Реализует существующий
  forward brick #3 (колонки уже есть: budget.db tenant_id присутствует). ADR-0024.
- Acceptance: tool-call с `tenant_id=другой` при claim `self` — игнор/деny; cross-tenant read/write невозможен.
- Тесты: `test_tenant_from_claim_not_argument`, `test_cross_tenant_write_denied`.

**WP-D — Budget per-tool scope (F34).** Task: #933. Subsystem: `budget`.
- Добавить `scope_resolver` (как в `ai-memory/memory/daemon.py:61-71`): read-tools→`budget.read`,
  propose-tools→`budget.propose`. Убрать статичный `required_scopes` intersection.
- Тесты: `test_budget_read_token_cannot_propose`, `test_budget_propose_token_cannot_read_transactions`.

**WP-E — Backend-side policy (F36).** Task: #931, зависит от #929. Subsystem: `service_policy`
(сделать реально общим editable-пакетом с `pyproject.toml`) + бэкенды.
- Перенести `gateway/service_policy/` в корневой `service_policy/` (сейчас пустой) как editable-пакет;
  gateway и бэкенды консультируют `decide(principal, tool, resource)` из claims. `DENIED_TOOLS`/owner-gating
  работают и на прямой loopback-вызов.
- Явно описать trusted-local direct-write contour, чтобы `ai-memory-capture` и stdio bridge не сломались:
  прямые writes разрешены только subject/scopes, утверждённым server-side policy.
- Тесты: `test_direct_store_memory_denied_without_owner_on_backend`, `test_backend_consults_shared_policy`.

### P2 — Medium

- **WP-F** Task #935: Tamper-evident audit: external/WORM anchor (вне директории лога, иной ACL/носитель) (F37).
- **WP-G** Task #935: MCP session pool в gateway backends (F38).
- **WP-H** Task #936: Calendar webhook: gateway валидирует channel-token до форварда ИЛИ документирует pm-mcp как
  единственный валидатор + сужает rate-bucket (F39).
- **WP-I** Tasks #936/#937: убрать OAuth-пути из proposals bypass-allowlist (F310);
  TTL-эвикция OAuthFlow pending/codes (F311).

### P3 — Стратегическое (multi-user / internet / external-MCP)

- **WP-J** External-MCP/delegated sandbox с backend-enforcement trust-tier (MC-8): профили
  `sandbox`/`delegated` уже есть в gateway (`app.py:113-123`), но бэкенды не различают trust-tier —
  довести enforcement до бэкенда + provenance-пометку.
- **WP-K** Security monitoring delivery (MC-7): `monitoring.py` пишет alert-файл; добавить порог→доставку.
- **WP-L** Egress/SSRF-контроль на бэкендах с внешними fetch (MC-10).
- **WP-M** OIDC seam для owner→multi-user (MC-3 продолжение, ADR-0024).
- P3 work items не заводятся до закрытия P0/P1 и отдельного review, чтобы не плодить stale backlog.

## 7. Кандидаты в tech-stack bricks (промоут ТОЛЬКО после landing+evidence и подтверждения)

- candidate **#27** — service-token custody model (**A3 per-service keypairs + A2 server-side
  envelope-binding**) + обязательный Windows-ACL на key-material. Обновляет/заменяет наброски brick #23/#5.
- candidate **#28** — backend-side shared policy package `service_policy` (`decide()` из claims).
- candidate **#29** — tamper-evident external audit anchor.
- candidate **#30** — tenant-from-claim enforcement pattern (дополняет brick #3 forward).
- candidate **#31** — fail-closed local UI auth provisioning (assistant-ui).
- Hook-кандидат: pre-deploy guard, проверяющий ACL `C:\ProgramData\AI-Assistant\secrets` и
  `.secrets/service-identity/**` (world-readable = fail) — оформить через `hook-authoring` при подтверждении.

## 8. Риски

- **R1**: WP-A меняет проводку ключей — сначала ADR/runbook и test fixtures, затем isolated verify,
  rotation old shared key и деплой единым окном.
- **R2**: WP-B fail-closed может заблокировать UI при отсутствии провижна — провижн в том же
  коммите, что и флаг; rollback = warn.
- **R3**: WP-C/WP-E трогают живые БД/политику — backfill `self` идемпотентно, regression на cross-tenant.
- **R4**: смена `service_policy` location — атомарный коммит (migration discipline), удалить gateway-local copy
  после переноса imports; сохранить trusted-local direct-write contour.

## 9. Итоговые решения

1. **P3 scope**: не входит в этот Phase 3 landing; любые новые P3 work items заводятся только
   после отдельного review.
2. **Промоут custody bricks и hook ACL-guard**: выполнен follow-up #938 после landing #929/#931.
3. **External/WORM audit anchor**: обработан в #935 как часть gateway audit-anchor hardening и
   threat-model выбора; отдельного блокера в этом плане не осталось.

## 10. Критерии приёмки

- [x] P0: ADR-0025 принят; старый shared S2S key revoked/rotated; приватные ключи не world-readable
      на обоих start-path; непривилегированный процесс не минтит owner/cross-service токен;
      assistant-ui fail-closed + провижн token; rebinding на :8000.
- [x] P1: tenant из claims (cross-tenant denied); budget per-tool scope; backend консультирует shared policy.
- [x] P2: audit external-anchored или документирован threat model для выбранного anchor; MCP session pool;
      webhook pre-validation/rate-bucket; WP-B2 cleanup.
- [x] ADR-0025 (key-custody) написан; ADR-0024 (multi-tenant) расширен tenant-from-claim.
- [x] Docs-sweep: tech-stack brick #5/#23 синхронизированы с фактической custody-моделью.
- [x] Security regression suite (§6 тесты) зелёный; rollout/rollback отрепетированы.
- [x] Per-WP work items заведены по subsystem `project_path` с `link_task_dependency`: #928-#937.

## 11. Task matrix

| Task | Project path | Scope | Depends on | Final status |
|------|--------------|-------|------------|--------------|
| #928 | `D:\GitHub\AI-Assistant` | P0 emergency S2S containment + ADR-0025 key-custody decision | #917 | Готово |
| #929 | `D:\GitHub\AI-Assistant` | P0 per-service S2S keypairs + server-side envelope binding | #928 | Готово |
| #930 | `D:\GitHub\AI-Assistant\assistant-ui` | P0 Assistant-UI deploy auth fail-closed + Host allowlist | #928 | Готово |
| #931 | `D:\GitHub\AI-Assistant` | P1 shared `service_policy` package + backend policy contract | #929 | Готово |
| #932 | `D:\GitHub\AI-Assistant\ai-memory` | P1 tenant-from-claims + backend policy for AI-memory writes | #929, #931 | Готово |
| #933 | `D:\GitHub\AI-Assistant\budget` | P1 budget per-tool scopes + tenant-from-claims | #929, #931 | Готово |
| #934 | `D:\GitHub\AI-Assistant\pm-mcp-server` | P1 tenant-from-claims on PM-MCP internal/MCP surfaces | #929, #931 | Готово |
| #935 | `D:\GitHub\AI-Assistant\gateway` | P2 audit anchor hardening + MCP session pooling | #929 | Готово |
| #936 | `D:\GitHub\AI-Assistant\gateway` | P2 calendar webhook edge hardening + OAuthFlow TTL eviction | #929 | Готово |
| #937 | `D:\GitHub\AI-Assistant\ai-memory` | P2 remove stale AI-memory proposals OAuth bypass allowlist | #917 | Готово |
| #938 | `D:\GitHub\AI-Assistant` | Post-landing custody bricks + service-identity ACL pre-deploy guard | #929, #931 | Готово |
| #944 | `D:\GitHub\AI-Assistant\pm-mcp-server` | Incident follow-up: dashboard/roadmap task visibility regression after #934 | #934 | Готово |
| #945 | `D:\GitHub\AI-Assistant\assistant-ui` | Incident follow-up: dashboard initial task render latency after #944 | #944 | Готово |
| #948 | `D:\GitHub\AI-Assistant` | Final central-plan status update and archive | #917, #928-#938, #944, #945 | Готово |

## 12. Pre-close retrospective for #917

| Axis | Verdict | Notes |
|------|---------|-------|
| `tech-stack-choices.md` | `no-change` | Custody bricks and ACL guard promotion completed in #938; no remaining tech-stack follow-up from this plan. |
| Design-system | `no-change` | Security plan and backend hardening do not introduce UI primitives or shared visual tokens. |
| Skills | `no-change` | `central-plan-workflow`, `pm-mcp-task-flow`, `ai-memory-recall` cover this workflow; plan-mode wording clarified separately during #938. |
| Hooks | `no-change` | ACL/pre-deploy guard implemented in #938 and verified against service registration hook simulation. |

## 13. Closure note (2026-06-14)

- PM-MCP tasks #917, #928-#938, #944, #945 and #948 are `Готово`.
- #944/#945 are included because they were discovered while executing Phase 3 and were required to keep
  `/dashboard` and `/roadmap` usable after #934.
- AI-memory has final outcome records for #930, #931/#932/#933/#934/#935/#936/#937, #938, #944 and #945.
- The active plan is archived to `DONE/secure-platform-hardening-phase3.md`; no unique open acceptance
  criteria remain only in the active plan file.
