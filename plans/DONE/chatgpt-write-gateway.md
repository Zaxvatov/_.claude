# План: запись из ChatGPT Plus в AI-memory через proposal queue (write-gateway 8770)

## 1. Context

**Проблема.** ChatGPT Plus умеет читать AI-memory через public read-only daemon `127.0.0.1:8767` (OAuth 2.0 + PKCE, Tailscale Funnel). Пользователь хочет, чтобы ChatGPT также фиксировал новый контекст (факты, заметки), а локальные агенты (Codex, Claude) этим контекстом потом пользовались.

**Что в репозитории уже зафиксировано (нельзя нарушать):**
- `D:\GitHub\AI-memory\AGENTS.md:270`: «The public daemon must never register write tools, even if environment variables request them».
- `docs/CHATGPT.md:56-58`: «Secrets must not be written to AI-memory at all. Redaction is a last-resort outbound guardrail, not a storage policy».
- PM-MCP work item **#141** (Бэклог, низкий приоритет) уже описывает proposal-queue архитектуру: «не давать прямой `store_memory` — реализовать `propose_memory(text, project, kind, metadata)`, который пишет в отдельную staging-таблицу `memory_proposals`… При approve — запись через приватный `store_memory` с `agent='chatgpt'`. Это сохраняет hardening: ChatGPT не может писать в production memory напрямую, даже при компрометации JWT/Bearer».
- PM-MCP work item **#139** (Бэклог, средний приоритет) резервирует порт `127.0.0.1:8769` под отдельный public daemon **PM-MCP**. Поэтому write-gateway для AI-memory занимает следующий свободный порт **8770**.

**Решение.** Реализовать архитектуру из #141. Отдельный AI-memory **Write Gateway** на `127.0.0.1:8770` принимает `propose_memory` (не `store_memory`), записывает в **отдельную таблицу** `memory_proposals` со статусом `proposed`. Production memory (`memory`) остаётся незатронутой до явного approve. Approve приводит к вызову приватного `memory.storage.store()` с `agent='chatgpt'`. До approve `search_memory` других агентов **не видит** этих записей.

**Желаемый исход.** ChatGPT Plus в Developer Mode подключается к двум remote MCP серверам: read (`8767`, без изменений) и propose (`8770`, новый, scope `ai-memory.propose`). Любое предложение от ChatGPT идёт в staging-таблицу, проходит синхронный intake-фильтр и две отложенные проверки (Stage 2a через час, Stage 2b следующей ночью). Финально запись становится memory только после **ручного approve** через CLI (`proposals-approve <id>`). Авто-approve в этом плане **не включён** — это отдельная будущая задача с собственным security review. PM-MCP в этот план не входит — задача #139 решается отдельно.

## 2. Архитектура

```text
ChatGPT Plus (Developer Mode + remote MCP, два connector)
  ├── READ connector ──→ existing public daemon 8767 (без изменений)
  │
  └── WRITE connector ──→ chatgpt-write-gateway 8770 (новый)
        │
        ▼
   AuditHttpMiddleware → BearerAuthMiddleware → RateLimitMiddleware →
   BodySizeMiddleware → SecurityHeadersMiddleware
        │  (отдельный JWT secret, отдельный keyring scope; 20 req/min)
        ▼
   Tool propose_memory  (jsonschema-locked; см. п. 4)
        │
        ▼
   Input validation chain:
     - strict schema: text, kind, project, tags?, reason?
     - text length ≤ 4000, tags ≤ 10, tag ≤ 40 chars
     - kind ∈ {fact, note}  (enum в schema + revalidation)
     - project ∈ AI_MEMORY_WRITE_GATEWAY_PROJECT_ALLOWLIST (enum в schema)
     - intake regex barrier (SECRET_PATTERNS, см. п. 7)
        │  на отказ: 400 + audit (sha256 + length + pattern), payload не пишется
        ▼
   Server-fills (клиент эти поля передать НЕ может — extra-fields в payload
   отклоняются schema с 400, не «игнорируются»):
     proposed_by="chatgpt", source="chatgpt", proposed_at,
     client_fingerprint = sha256(client_id || oauth_sub || ip_prefix_2_octets),
     oauth_scope="ai-memory.propose", tool_version,
     stage2a_at=T+1h, stage2b_at=next-night-after-T+25h
        │
        ▼
   INSERT INTO memory_proposals (status='proposed')
        │
        ├──→ append-only audit: data/logs/write-gateway-proposals.jsonl
        │    (proposal_id, sha256(text), length, kind, project,
        │     matched_patterns, client_fingerprint, decision)
        │
        └──→ возврат ChatGPT: {proposal_id, status: "proposed"}
                              (ChatGPT НЕ получает обратно текст)

Scheduled task `AI-memory-write-review` (every 1h, account ai-memory-write-gateway):
   ReviewWorker.run_due_pending(now):
     - proposals where stage2a_at ≤ now AND stage2a_decision IS NULL → Stage 2a
         clean       → stage2a_decision='clean' (status остаётся 'proposed')
         suspicious  → status='reviewed_suspicious', review_reason="stage2a:<pat>"
     - proposals where stage2b_at ≤ now AND stage2a_decision='clean' AND
                       stage2b_decision IS NULL → Stage 2b
         clean       → stage2b_decision='clean', status='reviewed_clean'
                       (auto-approve вне MVP, см. раздел 3)
         suspicious  → status='reviewed_suspicious', review_reason="stage2b:<pat>"

Human reviewer (единственный путь к promotion в MVP):
   memory.cli proposals-list [--status proposed|reviewed_clean|reviewed_suspicious]
   memory.cli proposals-show <id>      ─ показывает полный text + metadata
   memory.cli proposals-approve <id>   ─ status='approved' + idempotent promote в memory
   memory.cli proposals-reject <id> [--reason]   ─ status='rejected'

Retention (отдельный scheduled task, daily 04:00):
   - rejected/reviewed_suspicious старше 30 дней → DELETE FROM memory_proposals
   - approved старше 7 дней → sanitize: text=NULL, sanitized_at=now,
     sanitized_reason='retention:promoted'. promoted_memory_id и text_sha256
     сохраняются для traceability metadata.proposal_id
   - audit JSONL ротируется ежедневно (daily file), retention 90 дней
```

## 3. Зафиксированные дизайн-решения

| Параметр | Значение | Источник |
|---|---|---|
| Архитектура | Proposal queue (отдельная таблица), production memory не загрязняется | Codex feedback + PM-MCP #141 |
| Tool name | `propose_memory` (не `store_memory`) | Codex feedback п.3 + #141 |
| Where proposals live | Новая SQLite таблица `memory_proposals` в `data/memory.db` + миграция схемы. Append-only JSONL только для audit | Codex feedback п.1 |
| Production memory write | Только через `memory.storage.store()` после approve. ChatGPT никогда не пишет туда напрямую | #141 |
| Порт gateway | `127.0.0.1:8770` (8769 зарезервирован под PM-MCP #139) | #139 |
| Existing 8767 read daemon | НЕ меняется | AGENTS.md:270 |
| Разрешённые kinds | `fact`, `note` | Ответ пользователя |
| Tagging | server-side forced: `proposed_by=chatgpt`, `source=chatgpt`. Клиент НЕ передаёт agent/source/timestamps | Codex feedback п.2 |
| Stage 2 расписание | 2a: T+1h. 2b: ближайшие 03:00 на дату ≥ T+25h. Worker раз в час | Ответ пользователя |
| Auto-approve | **Вне MVP**. `AUTO_APPROVE_CLEAN` всегда `false` в этом плане. Включение — отдельная задача PM-MCP после security review и недели наблюдений | Codex v2 п.4 |
| Действие при Stage 2 suspect | `status='reviewed_suspicious'`, ждёт ручного `proposals-reject` через CLI | Codex v1 п.6 |
| Classifier | regex/heuristics only | Ответ пользователя |
| Input limits | `text 8…4000 chars`, `tags ≤ 10`, `tag ≤ 40 chars`, `reason ≤ 200 chars` | Codex v1 п.5 |
| MCP schema | jsonschema enum для `kind` и `project` + server revalidation; `additionalProperties:false` | Codex v1 п.4 |
| Rate limit | 20 req/min per-IP + **daily quota** 100 proposals/day per client_fingerprint (429 quota_exceeded) | Codex v1 п.5 + v2 п.5 |
| OAuth scope | Отдельный scope `ai-memory.propose`. Read tokens (`ai-memory.read`) к write daemon не подходят | Codex v2 п.8 |
| Tags normalization | lowercase, trim, dedupe, запрет whitespace/slashes/control chars | Codex v2 п.11 |
| Audit content | Только sha256(text), length, kind, project, matched patterns, client_fingerprint, scope. Никогда полный текст в логах | Codex v1 п.7 + v2 п.8 |
| Promotion idempotency | `promote_proposal(id)` сначала проверяет `promoted_memory_id IS NOT NULL` → возвращает существующий memory_id, не дублирует. UNIQUE-индекс на колонке | Codex v2 п.6, п.7 |
| Retention для approved | Не удалять row целиком. Через 7 дней — sanitize: `text=NULL, sanitized_at, sanitized_reason='retention:promoted'`. Сохраняет traceability `metadata.proposal_id → memory_proposals.id` | Codex v2 п.3 |
| Health probes | `/healthz` = процесс жив. `/readyz` = enabled, DB OK, project allowlist задан, keyring доступен | Codex v2 п.9 |
| Empty project allowlist | Gateway отказывается стартовать с пустым `AI_MEMORY_WRITE_GATEWAY_PROJECT_ALLOWLIST`. `/readyz` возвращает 503 при пустом списке | Codex v2 п.10 |
| Tool description | Явно: «After calling, tell the user: 'Submitted for review'. Do NOT say 'I remembered' — entry appears in memory only after approval» | Codex v2 п.13 |
| Channel | ChatGPT **Developer Mode** + remote MCP. GPT Actions вне этого плана | Codex v1 п.8 |
| Confirmation в ChatGPT UI | Не считается security boundary — сервер валидирует независимо | Codex v1 п.9 |
| Data retention | rejected/suspicious 30 дней (DELETE), approved 7 дней (sanitize text), audit JSONL 90 дней | Codex v1 п.11 + v2 п.3 |
| Rollout | loopback → OAuth local → Funnel disabled → ingress; ENV-флаг для отключения в любой момент | Codex v1 п.12 |
| Rollback CLI | `proposals-reject-all --since`, `archive-by-agent --agent chatgpt --since` | Codex v1 п.10 |
| Связь с PM-MCP | Этот план реализует work item **#141** в проекте AI-memory | PM-MCP |

## 4. Контракт `propose_memory` (MCP tool schema)

```json
{
  "name": "propose_memory",
  "description": "Submit a memory proposal for human review. The proposal enters a staging queue and does NOT appear in search_memory until a human reviewer approves it through CLI. After calling this tool, tell the user 'Submitted for review' or similar — do NOT say 'I remembered' or 'I saved that', because the entry is not yet in memory and may be rejected.",
  "inputSchema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["text", "kind", "project"],
    "properties": {
      "text":    {"type": "string", "minLength": 8, "maxLength": 4000,
                  "description": "The fact or note to propose. Self-contained sentence. No secrets, tokens, passwords, payment cards."},
      "kind":    {"type": "string", "enum": ["fact", "note"]},
      "project": {"type": "string", "enum": ["<filled from AI_MEMORY_WRITE_GATEWAY_PROJECT_ALLOWLIST>"]},
      "tags":    {"type": "array", "maxItems": 10,
                  "items": {"type": "string", "minLength": 1, "maxLength": 40},
                  "description": "Optional. Server normalizes: lowercase, trim, dedupe; whitespace/slashes/control chars rejected."},
      "reason":  {"type": "string", "maxLength": 200,
                  "description": "Optional. Why this is worth remembering."}
    }
  },
  "annotations": {"readOnlyHint": false, "destructiveHint": false, "idempotentHint": false, "openWorldHint": false}
}
```

Сервер **повторно** валидирует на каждом запросе (jsonschema может быть отключён клиентом). При `additionalProperties: false` любые поля сверх перечисленных — `agent`, `source`, `proposed_at`, `proposed_by`, `client_fingerprint`, `tool_version`, `stage2a_at`, `stage2b_at` и т. п. — **отклоняются** с `400 schema_validation`. Server-side fields проставляются сервером после успешной валидации; от клиента эти поля никогда не принимаются.

Успешный ответ: `{"proposal_id": <int>, "status": "proposed", "stage2b_at": "<iso8601>"}`. Текст обратно не возвращается.

## 4a. State machine

```text
                                  ┌────────────────────────────────────────┐
                                  ▼                                        │
   [client]                  proposed                                      │
   propose_memory  ──────►   stage2a_decision=NULL                         │
                             stage2b_decision=NULL                         │
                                  │                                        │
                                  │  Stage 2a worker (T+1h)                │
                                  │                                        │
                ┌─────────────────┼─────────────────┐                      │
                │ clean           │                 │ suspicious           │
                ▼                                   ▼                      │
            proposed                          reviewed_suspicious          │ proposals-reject
            stage2a_decision='clean'          stage2a_decision='suspicious'│   ──────►  rejected
                │                             review_reason="stage2a:..."  │
                │                                                          │
                │  Stage 2b worker (next night after T+25h)                │
                │                                                          │
        ┌───────┼───────────┐                                              │
        │ clean             │ suspicious                                   │
        ▼                   ▼                                              │
   reviewed_clean       reviewed_suspicious ─────────────────────────────► (см. справа)
   stage2b_decision='clean'  review_reason="stage2b:..."
        │
        │  proposals-approve <id>   (manual)
        ▼
    approved
    promoted_memory_id=<memory.id>
    promoted_at=<now>
        │
        │  retention worker (after 7 days)
        ▼
    approved (sanitized)
    text=NULL, sanitized_at=<now>, sanitized_reason='retention:promoted'
    (row хранится навсегда для traceability metadata.proposal_id)

Terminal: approved (sanitized хранится навсегда), rejected (удаляется через 30 дней),
          reviewed_suspicious (удаляется через 30 дней; вручную можно только REJECT).

Manual approve в MVP доступен ТОЛЬКО из:
  - 'proposed' со stage2a_decision='clean'
  - 'reviewed_clean'
Из 'reviewed_suspicious' approve НЕДОСТУПЕН в MVP.
Если пользователь считает что это false positive — он создаёт запись вручную через
приватный CLI 'memory.cli store ...' или ждёт будущей команды 'approve --force'
(вне scope этого плана). Это убирает риск что reviewer случайно «зальёт» suspect.

Manual reject доступен из любого open-статуса (proposed / reviewed_clean / reviewed_suspicious).
```

## 5. Schema миграции

Новая ревизия схемы в `memory/db.py` (следующая после v6, например v7). Таблица `memory_proposals` живёт в той же `memory.db`.

```sql
CREATE TABLE memory_proposals (
    id                    INTEGER PRIMARY KEY AUTOINCREMENT,
    proposed_at           TEXT NOT NULL,                    -- ISO 8601 UTC
    text                  TEXT,                             -- NULL после retention sanitize (для approved)
    text_sha256           TEXT NOT NULL,                    -- сохраняется даже после sanitize
    text_length           INTEGER NOT NULL,                 -- сохраняется даже после sanitize
    kind                  TEXT NOT NULL,                    -- 'fact' | 'note'
    project               TEXT NOT NULL,
    tags_json             TEXT NOT NULL DEFAULT '[]',       -- нормализованные tags после server-side обработки
    reason                TEXT,
    proposed_by           TEXT NOT NULL,                    -- всегда 'chatgpt' в Wave 1
    source                TEXT NOT NULL,                    -- всегда 'chatgpt'
    client_fingerprint    TEXT NOT NULL,                    -- sha256(client_id|oauth_sub|ip-prefix)
    oauth_scope           TEXT NOT NULL,                    -- 'ai-memory.propose'
    tool_version          TEXT NOT NULL,                    -- версия gateway
    stage2a_at            TEXT NOT NULL,                    -- ISO 8601
    stage2a_decision      TEXT,                             -- NULL | 'clean' | 'suspicious'
    stage2a_patterns_json TEXT,                             -- ["aws_access_key", ...] при suspect
    stage2a_done_at       TEXT,
    stage2b_at            TEXT NOT NULL,
    stage2b_decision      TEXT,
    stage2b_patterns_json TEXT,
    stage2b_done_at       TEXT,
    status                TEXT NOT NULL DEFAULT 'proposed', -- proposed | reviewed_clean |
                                                           --   reviewed_suspicious | approved | rejected
    review_reason         TEXT,                             -- человекочитаемая причина reject/suspicion
    promoted_memory_id    INTEGER,                          -- FK на memory.id (после approve)
    promoted_at           TEXT,
    sanitized_at          TEXT,                             -- ISO 8601 — когда text стёрт по retention
    sanitized_reason      TEXT,                             -- например 'retention:promoted'
    intake_patterns_json  TEXT NOT NULL DEFAULT '[]'        -- паттерны, сработавшие на intake-этапе
);

CREATE INDEX idx_memory_proposals_status      ON memory_proposals(status);
CREATE INDEX idx_memory_proposals_stage2a_at  ON memory_proposals(stage2a_at)
    WHERE stage2a_decision IS NULL;
CREATE INDEX idx_memory_proposals_stage2b_at  ON memory_proposals(stage2b_at)
    WHERE stage2b_decision IS NULL AND stage2a_decision = 'clean';
CREATE INDEX idx_memory_proposals_proposed_at ON memory_proposals(proposed_at);
CREATE INDEX idx_memory_proposals_fingerprint ON memory_proposals(client_fingerprint, proposed_at);

-- Идемпотентность promotion: один proposal — максимум одна запись в memory
CREATE UNIQUE INDEX uniq_memory_proposals_promoted
    ON memory_proposals(promoted_memory_id)
    WHERE promoted_memory_id IS NOT NULL;

-- DB-level guardrails (поверх Python-валидации; SQLite поддерживает CHECK)
-- Если в текущем проекте уже используются CHECK constraints в других таблицах —
-- следовать существующему стилю; иначе добавить как новую практику и упомянуть
-- в ARCHITECTURE.md.
ALTER TABLE memory_proposals ADD CONSTRAINT chk_kind
    CHECK (kind IN ('fact', 'note'));
ALTER TABLE memory_proposals ADD CONSTRAINT chk_status
    CHECK (status IN ('proposed', 'reviewed_clean', 'reviewed_suspicious',
                       'approved', 'rejected'));
ALTER TABLE memory_proposals ADD CONSTRAINT chk_stage2a
    CHECK (stage2a_decision IS NULL OR stage2a_decision IN ('clean', 'suspicious'));
ALTER TABLE memory_proposals ADD CONSTRAINT chk_stage2b
    CHECK (stage2b_decision IS NULL OR stage2b_decision IN ('clean', 'suspicious'));
ALTER TABLE memory_proposals ADD CONSTRAINT chk_proposed_by
    CHECK (proposed_by = 'chatgpt');                -- в Wave 1 только chatgpt
```
SQLite формально не поддерживает `ALTER TABLE … ADD CONSTRAINT` — CHECK задаются в `CREATE TABLE`. Реализация: добавить CHECK прямо в DDL миграции v7. Выше показано для читаемости — мигратор перепишет.

Read-only handle public daemon 8767 **не получает** доступ к таблице `memory_proposals`. Таблица не fts-индексируется и не появляется в `search_memory`. Production agents (Codex, Claude) не видят proposal до явного `proposals-approve`. Тест в `tests/test_public_app.py` фиксирует это (см. п. 13 open question 3).

## 6. Новые файлы

| Файл | Назначение |
|---|---|
| `memory/write_gateway/__init__.py` | Пакет. |
| `memory/write_gateway/runtime.py` | Константы: `WRITE_GATEWAY_PORT=8770`, paths, новые keyring service strings (отличные от 8767), `WRITE_GATEWAY_OAUTH_SCOPE="ai-memory.propose"`. |
| `memory/write_gateway/app.py` | Starlette app: OAuth-endpoints (повтор контракта public_app, но свой keyring, scope `ai-memory.propose`), `/mcp` с регистрацией одного tool `propose_memory`, `/healthz` (liveness), `/readyz` (readiness: enabled + DB OK + allowlist непуст + keyring доступен). Импортирует middleware из `memory/public_security.py`. |
| `memory/write_gateway/daemon.py` | uvicorn runner, single-instance lock (`data/ai-memory-write-gateway.lock`). При старте: проверка `AI_MEMORY_WRITE_GATEWAY_PROJECT_ALLOWLIST` непуст; иначе fail-fast с осмысленной ошибкой. |
| `memory/write_gateway/intake.py` | Tool wrapper: `with_strict_schema`, `with_tag_normalization`, `with_server_side_fields`, `with_daily_quota`, `with_intake_secret_filter`. SECRET_PATTERNS константа (см. п. 7). |
| `memory/write_gateway/proposals.py` | Repository слой над таблицей `memory_proposals`: `insert_proposal(...)`, `list_proposals(status, limit, fingerprint?)`, `get_proposal(id)`, `update_stage2a(id, decision, patterns)`, `update_stage2b(...)`, `mark_approved(id, memory_id)` (с idempotency check), `mark_rejected(id, reason)`, `count_today_by_fingerprint(fp)` (для daily quota), `purge_expired(now, retention_rules)`, `sanitize_promoted(now)`, `bulk_reject(since, reason)`. |
| `memory/write_gateway/review.py` | `ReviewWorker.run_due_pending(now)`, `compute_stage2b_at(created_at, night_hour, tz)`. Использует `proposals.py` + `intake.SECRET_PATTERNS`. **Auto-approve в этом плане не реализуется** (см. раздел 3). |
| `memory/write_gateway/promote.py` | `promote_proposal(proposal_id)`: **идемпотентно с compensating recovery**. Алгоритм: (1) `SELECT promoted_memory_id, status FROM memory_proposals WHERE id=?`. (2a) Если `promoted_memory_id IS NOT NULL` → вернуть существующий memory_id, store не вызывать. (2b) Если `status='approved'` но `promoted_memory_id IS NULL` (партиальный сбой прошлого approve) → сначала попытка восстановления: `SELECT id FROM memory WHERE metadata.proposal_id=?`; если нашли → `UPDATE memory_proposals SET promoted_memory_id=?` и вернуть. (2c) Иначе нормальный путь: `memory.storage.store(text, project, agent='chatgpt', kind, metadata={'source':'chatgpt','proposal_id':id,'tags':[...]})` → `UPDATE memory_proposals SET status='approved', promoted_memory_id=?, promoted_at=?`. Если store прошёл, а UPDATE упал — следующий вызов идёт по ветке (2b). UNIQUE-индекс защищает от race. **Важно**: `memory.storage.store()` использует своё соединение, поэтому атомарность всей операции не гарантируется — compensating path обязателен. |
| `memory/write_gateway/audit.py` | Append-only writer для `data/logs/write-gateway-proposals.jsonl`. Содержимое: timestamp, event (proposal_received/intake_rejected/quota_exceeded/stage2a/stage2b/approved/rejected/sanitized/expired), proposal_id, sha256(text), text_length, kind, project, matched_patterns, client_fingerprint, oauth_scope, decision_actor (chatgpt|stage2|cli|retention). |
| `tests/test_write_gateway_intake.py` | По одному positive+negative тесту на каждый паттерн в SECRET_PATTERNS. Тест strict schema (extra-fields → 400). Тест server-side override (клиент прислал `agent="claude"` → запись с `proposed_by="chatgpt"`). Тест нормализации tags. |
| `tests/test_write_gateway_review.py` | `compute_stage2b_at` — 5 граничных кейсов (см. п. 8). In-memory SQLite: создать proposal, прокрутить время, проверить state transitions Stage 2a → Stage 2b. |
| `tests/test_write_gateway_proposals.py` | Repository unit-тесты: insert / list by status / state-machine: `proposed → reviewed_clean → approved` и `proposed → reviewed_suspicious → rejected`. Тест что invalid transition (например `rejected → approved`) отвергается. |
| `tests/test_write_gateway_promote.py` | После approve в `memory` появляется запись с `agent='chatgpt'`, `metadata.source='chatgpt'`, `metadata.proposal_id=N`. `search_memory(query)` её находит. **Idempotency**: повторный `promote_proposal(id)` не создаёт дубликат, возвращает тот же `memory.id`. |
| `tests/test_write_gateway_retention.py` | Retention: rejected/suspicious старше 30 дней → DELETE. Approved старше 7 дней → text=NULL, sanitized_at, sanitized_reason. Traceability: `metadata.proposal_id` всё ещё указывает на существующую строку. |
| `tests/test_write_gateway_isolation.py` | Тест что public daemon 8767 (`search_memory`, `fetch`) **никогда** не возвращает proposals из `memory_proposals`, даже если попытаться SQL-инъекцией указать таблицу. |
| `tests/test_write_gateway_quota.py` | Daily quota: 100-й proposal от одного `client_fingerprint` за UTC-день — 200. 101-й — 429 `quota_exceeded`. На следующий UTC-день — 200 снова. |
| `tests/test_write_gateway_readiness.py` | `/healthz` всегда 200. `/readyz` = 503 при пустом allowlist, 503 при DB locked, 200 при норме. |
| `tests/test_write_gateway_app.py` | httpx.AsyncClient smoke: 200 healthz/readyz, 401 без токена, 401 с read-scope токеном (cross-scope rejected), 400 schema_validation, 400 на запрещённый kind, 400 на secret, 400 text<8 / text>4000, 200 happy path → proposal_id; `search_memory(query)` через 8767 ничего не находит. |
| `scripts/install_write_gateway_service.ps1` | Account `ai-memory-write-gateway`, project-local runtime, scheduled task для gateway daemon. По образцу `install_public_daemon_service.ps1`. |
| `scripts/install_write_review_scheduled_task.ps1` | Hourly task `AI-memory-write-review`, runs `uv run python -m memory.cli write-review-run-now --stage all`. |
| `scripts/install_write_retention_scheduled_task.ps1` | Daily 04:00 task `AI-memory-write-retention`, runs `uv run python -m memory.cli write-retention-run-now`. |
| `docs/CHATGPT-WRITE-GATEWAY.md` | Полная документация: endpoints (`/healthz`, `/readyz`, `/mcp`, `/authorize`, `/token`, `/.well-known/*`), OAuth registration со scope `ai-memory.propose`, Tailscale Funnel sub-path, hardening gate, threat model, troubleshooting, ChatGPT Developer Mode setup, раздел «When to enable auto-approve» с явным указанием что в MVP auto-approve выключен. |

## 7. SECRET_PATTERNS (regex набор)

В `memory/write_gateway/intake.py`. Имена паттернов попадают в `intake_patterns_json` / `stage2*_patterns_json` / audit.

`aws_access_key`, `aws_secret_key`, `github_token`, `openai_key`, `slack_token`, `google_api_key`, `jwt_token`, `bearer_in_text`, `private_key_pem`, `password_assignment`, `iban` (+Luhn-like checksum для IBAN), `credit_card` (Luhn), `ru_passport`, `ru_snils`, `email_password_pair`, `gcp_service_account_email`, `azure_storage_key_b64`.

`memory/validation.py:213` уже содержит `validate_no_secrets()` и `FORMAT_LEAK_PATTERNS` — используется как первый barrier, SECRET_PATTERNS дополняет, не дублирует. ENV `AI_MEMORY_WRITE_GATEWAY_DISABLE_PATTERNS` (comma-separated) — точечно отключить отдельные паттерны при false positives.

## 8. Формула `compute_stage2b_at`

```python
from datetime import datetime, time, timedelta
from zoneinfo import ZoneInfo

def compute_stage2b_at(proposed_at_utc: datetime, night_hour: int, tz: ZoneInfo) -> datetime:
    threshold_local = proposed_at_utc.astimezone(tz) + timedelta(hours=25)
    candidate = datetime.combine(threshold_local.date(),
                                 time(hour=night_hour), tzinfo=tz)
    if candidate < threshold_local:
        candidate += timedelta(days=1)
    return candidate.astimezone(proposed_at_utc.tzinfo)
```

Юнит-тесты (Europe/Zurich, `night_hour=3`):
- `T=2026-05-14 14:00 UTC` → Stage 2b = `2026-05-16 01:00 UTC` (≡ 03:00 CEST)
- `T=2026-05-14 23:00 UTC` → Stage 2b = `2026-05-16 01:00 UTC`
- `T=2026-05-14 02:00 UTC` → Stage 2b = `2026-05-15 01:00 UTC`
- `T=2026-05-14 00:59 UTC` (≡ 02:59 CEST) → Stage 2b = `2026-05-15 01:00 UTC`
- `T=2026-05-14 01:00 UTC` (≡ 03:00 CEST) → Stage 2b = `2026-05-15 01:00 UTC`

## 9. Изменения в существующих файлах

| Файл | Что меняем |
|---|---|
| `memory/cli.py` | Новые команды: `write-gateway-daemon`, `write-gateway-healthcheck`, `write-gateway-readycheck`, `write-gateway-smoke-propose --dry-run` (валидирует schema/auth, не пишет), `write-oauth-client-register/list/revoke`, `write-token-rotate`, `write-token-show-fingerprint`, `proposals-list [--status]`, `proposals-show <id>`, `proposals-approve <id>`, `proposals-reject <id> [--reason]`, `proposals-reject-all --since <iso> [--reason]`, `archive-by-agent --agent chatgpt --since <iso> [--reason]`, `write-review-run-now [--stage 2a|2b|all]`, `write-review-stats [--days]`, `write-retention-run-now [--dry-run]`. (Команда `proposals-archive` **исключена** — `archived_at`/`archive_reason` существуют только в production `memory`, для proposals достаточно `reject` с осмысленным `reason`.) |
| `memory/config.py` | ENV-парсеры: `get_write_gateway_enabled()` (default False), `get_write_gateway_port()` (default 8770), `get_write_gateway_allowed_kinds()` (default `("fact","note")`), `get_write_gateway_project_allowlist()` (**обязателен**; пустой → fail-fast при старте daemon), `get_write_gateway_allowed_hosts()`, `get_write_gateway_rate_limit()` (default 20/мин), `get_write_gateway_daily_quota()` (default 100/день per `client_fingerprint`), `get_write_gateway_oauth_scope()` (default `"ai-memory.propose"`), `get_review_night_hour()` (default 3), `get_review_tz()` (default Europe/Zurich), `get_proposals_retention_rejected_days()` (default 30), `get_proposals_retention_promoted_sanitize_days()` (default 7), `get_audit_retention_days()` (default 90). **`AUTO_APPROVE_CLEAN`** в этом плане не вводится. |
| `memory/db.py` | Миграция schema v6→v7: создать таблицу `memory_proposals` и индексы (п. 5). Регистрация миграции — по существующему паттерну. |
| `memory/runtime_contract.py` | Новые константы и поля dataclass: `write_gateway_host`, `write_gateway_port=8770`, `write_gateway_path="/mcp"`, `write_gateway_health_path="/healthz"`, `write_gateway_ready_path="/readyz"`, `write_gateway_keyring_service`, `write_gateway_oauth_clients_keyring_service`, `write_gateway_jwt_secret_keyring_user`, `write_gateway_oauth_scope="ai-memory.propose"`, `write_gateway_tools=("propose_memory",)`. **Не изменять** `PUBLIC_PUBLIC_REGISTRABLE` и `PUBLIC_NATIVE_READ_TOOLS` — инвариант. |
| `AGENTS.md` | Новый раздел «ChatGPT write gateway» под существующим «ChatGPT public MCP» (английский, по L.3). Зафиксировать: gateway отдельный (порт 8770), регистрирует ровно один tool `propose_memory`, writes идут в staging-таблицу `memory_proposals`, production memory `memory` записывается ТОЛЬКО приватным `store_memory` после approve, retention rules для proposals и audit. Не трогать существующую формулировку про 8767. |
| `ARCHITECTURE.md` | Добавить диаграмму write-канала (раздел 2 этого плана). Описать состояния proposal state machine. Зафиксировать, что `metadata.proposal_id` в production memory entries указывает на исходный proposal (для audit traceability). |
| `docs/CHATGPT.md` | В разделе «Endpoints» ссылка: «Write channel documented separately in `docs/CHATGPT-WRITE-GATEWAY.md`». Threat model для 8767 не меняется. |
| `pyproject.toml` | Console script `ai-memory-write-gateway = "memory.write_gateway.daemon:main"` — если репозиторий уже использует console_scripts; иначе не трогать. |

## 10. Verification (end-to-end)

### 10.1 Линтер и тесты
```powershell
uv run ruff check .
uv run python -m unittest discover -s tests
```

### 10.2 Smoke: liveness и readiness
```powershell
uv run python -m memory.cli write-gateway-daemon          # отдельное окно
uv run python -m memory.cli write-gateway-healthcheck     # /healthz = 200 если процесс жив
uv run python -m memory.cli write-gateway-readycheck      # /readyz = 200 только если allowlist задан, DB OK, keyring доступен
```
Тест отказа readiness: временно unset `AI_MEMORY_WRITE_GATEWAY_PROJECT_ALLOWLIST`, рестарт daemon — либо fail-fast (preferred), либо `/readyz=503 allowlist_empty`. `/healthz` остаётся 200.

### 10.2a Smoke без записи (dry-run)
```powershell
uv run python -m memory.cli write-gateway-smoke-propose --dry-run --text "dry run test" --kind fact --project portfolio
```
Валидирует schema, OAuth, scope, daily quota. Не пишет в БД. Текст в примере ≥ 8 chars, чтобы пройти `minLength`. Полезно для CI и для проверки connector'а без загрязнения staging.

### 10.3 Negative: extra-field (strict schema)
```powershell
curl ... -d '{"name":"propose_memory","arguments":{"text":"x","kind":"fact","project":"portfolio","agent":"claude"}}'
```
Ожидаем `400 schema_validation` (доп. поле `agent` отвергается). Даже если бы прошло — server-side override обнулил бы `agent` в `proposed_by="chatgpt"`.

### 10.4 Negative: kind/project не в enum
```powershell
curl ... -d '{...,"kind":"decision"}'        # 400
curl ... -d '{...,"project":"unknown-proj"}'  # 400
```

### 10.5 Negative: secret в тексте
```powershell
curl ... -d '{...,"text":"OPENAI_API_KEY=sk-proj-abcdef1234567890abcdef1234567890",...}'
```
Ожидаем `400 secret_detected`. В `data/logs/write-gateway-proposals.jsonl` — sha256+pattern, **не** полный текст.

### 10.6 Negative: text > 4000
```powershell
$big = "a" * 5000
curl ... -d "{...,\"text\":\"$big\",...}"
```
Ожидаем `400 text_too_long`.

### 10.7 Happy path → proposal создан, в production не виден
```powershell
curl ... -d '{...,"text":"Предпочитаю SAP MM роли в Цюрихе","kind":"fact","project":"portfolio"}'
```
Ожидаем `200 {"proposal_id": N, "status":"proposed", "stage2b_at":"..."}`.

Проверка:
```powershell
uv run python -m memory.cli search --query "SAP MM"       # из локального CLI
uv run python -m memory.cli proposals-list --status proposed
```
- `search` (production memory) — **пусто** для этой строки.
- `proposals-list` показывает строку с id N, status=`proposed`.

### 10.8 Stage 2a вручную
```powershell
uv run python -m memory.cli write-review-run-now --stage 2a
uv run python -m memory.cli proposals-show <id>
```
Запись с `stage2a_at < now` получает `stage2a_decision='clean'`. Подозрительная (тест: insert через repository helper минуя intake с фейк-CC, прошедшим Luhn) — `status='reviewed_suspicious'`, `review_reason="stage2a:credit_card"`.

### 10.9 Stage 2b → `reviewed_clean`
```powershell
uv run python -m memory.cli write-review-run-now --stage 2b
uv run python -m memory.cli proposals-show <id>
```
Чистая запись со `stage2a_decision='clean'` и `stage2b_at < now` → `status='reviewed_clean'`. **Auto-approve в этом плане отключён**; запись всё ещё не в production memory.

### 10.10 Ручной approve (единственный mode в MVP)
```powershell
uv run python -m memory.cli proposals-approve <id>
uv run python -m memory.cli search --query "SAP MM"
```
Запись появляется в production memory **только** после явного approve. Поля: `agent='chatgpt'`, `metadata.source='chatgpt'`, `metadata.proposal_id=N`.

### 10.10a Идемпотентность approve
```powershell
uv run python -m memory.cli proposals-approve <id>     # повторно
```
Ожидаем: возврат того же `memory_id`, без создания дубликата. В `memory` ровно одна строка с `metadata.proposal_id=N`.

### 10.11 Rollback / bulk
```powershell
uv run python -m memory.cli proposals-reject-all --since 2026-05-14T00:00:00Z --reason "compromised connector"
uv run python -m memory.cli archive-by-agent --agent chatgpt --since 2026-05-14T00:00:00Z --reason "compromised connector"
```
Первая команда: все proposals от ChatGPT после T становятся `rejected`. Вторая: уже promoted записи в production memory с `agent='chatgpt'` softly архивируются (`archived_at`, `archive_reason`).

### 10.11a Daily quota
```powershell
# Сгенерировать 101 proposal от одного OAuth client (тот же fingerprint)
for ($i=1; $i -le 101; $i++) { curl ... -d "{...,\"text\":\"q test $i\",...}" }
```
Ожидаем: первые 100 — `200`, 101-й — `429 quota_exceeded`. На следующий UTC-день — снова можно.

### 10.11b Tags нормализация
```powershell
curl ... -d '{...,"tags":["  Zurich ","ZURICH","zurich/job"]}'
```
Ожидаем `200`. В БД `tags_json = ["zurich"]` (нижний регистр, trim, dedupe, `/` отвергнут → 400 либо очищен — решить при имплементации; предлагаю отвергать как 400 чтобы клиент знал).

### 10.12 Retention
```powershell
uv run python -m memory.cli write-retention-run-now --dry-run
uv run python -m memory.cli write-retention-run-now
```
DRY-run показывает что будет сделано:
- `rejected`/`reviewed_suspicious` старше 30 дней → DELETE.
- `approved` старше 7 дней → `UPDATE memory_proposals SET text=NULL, sanitized_at=now, sanitized_reason='retention:promoted'`.
- `promoted_memory_id` и `text_sha256` сохраняются → `metadata.proposal_id` в production memory остаётся валидным указателем.
Audit JSONL имеет отдельный rotate-механизм (90 дней, daily rotation).

### 10.12a Изоляция: 8767 не видит proposals
```powershell
curl http://127.0.0.1:8767/mcp -H "Authorization: Bearer $readtok" `
     -d '{"name":"search","arguments":{"query":"SAP MM"}}'
```
До approve — `results: []`. После approve — найдено в production memory с `agent='chatgpt'`. На любой попытке cross-scope (Bearer от write daemon на 8767, и наоборот) ожидаем `401 invalid_scope`.

### 10.13 ChatGPT end-to-end (после rollout)
- В ChatGPT: «Запомни что я предпочитаю SAP MM роли в Цюрихе».
- ChatGPT в Developer Mode вызывает `propose_memory` — UI confirmation проходит.
- В локальном `proposals-list --status proposed` появляется запись.
- Через час Stage 2a отрабатывает, на следующую ночь Stage 2b.
- Если auto-approve выключен — ждём ручного `proposals-approve N`. После approve `search_memory` из Claude/Codex находит запись.

## 11. Rollout (phased)

| Фаза | Действие | Validation |
|---|---|---|
| 0 | Перевести **#141** из «Бэклог» в «К выполнению» через `approve_task`. Если необходимо — создать новый work item для подзадач (8770 hardening, retention) | `get_work_item #141` показывает статус «К выполнению» |
| 1 | Schema миграция v6→v7 в `memory/db.py` + unit-тесты миграции. Запустить миграцию на тестовой копии `memory.db` (резервная копия снимается автоматически по существующему механизму) | `proposals-list` запускается без ошибок |
| 2 | Реализация модулей + unit-тесты, всё за `AI_MEMORY_WRITE_GATEWAY_ENABLED=false` | ruff + tests зелёные |
| 3 | Обновить AGENTS.md, ARCHITECTURE.md, создать `docs/CHATGPT-WRITE-GATEWAY.md` | Документация ревьюится |
| 4 | Локальный smoke под текущим user account, loopback only, проверка 10.2–10.12 | Все проверки 10.2–10.12 проходят |
| 5 | Hardening: создать Windows account `ai-memory-write-gateway`, project-local runtime, ACL: read+write только `memory.db`/`data/logs/write/`/`data/ai-memory-write-gateway.lock`. Scheduled tasks gateway-daemon, write-review и write-retention **выключены** | `install_write_gateway_service.ps1 -CreateUser` отрабатывает чисто, account существует |
| 6 | Регистрация OAuth client: `write-oauth-client-register --redirect-uri "https://chatgpt.com/connector/oauth/<new-connector-id>"`. **Только** loopback — Funnel ещё не открыт | Client id/secret в keyring (`write-oauth-client-list`) |
| 7 | Включить scheduled tasks (gateway-daemon, write-review, write-retention) на loopback. `AI_MEMORY_WRITE_GATEWAY_ENABLED=false` — `tools/list` пуст | Hourly review log пишется (на холостом — «0 proposals due»). Retention dry-run прошёл. `tools/list` через loopback возвращает пустой список |
| 8 | Tailscale Funnel: добавить sub-path или отдельное Funnel-приложение для write FQDN. **Флаг всё ещё false**. ChatGPT connector НЕ подключаем сейчас (иначе он закеширует пустой `tools/list` и потом не увидит `propose_memory` без manual refresh) | Funnel resolved, `/healthz` снаружи=200, `/readyz` снаружи=503 `service_disabled`, `tools/list` пуст |
| 9 | Включить `AI_MEMORY_WRITE_GATEWAY_ENABLED=true`, рестарт gateway scheduled task. Verify локально что `tools/list` теперь содержит `propose_memory` | `/readyz`=200, `tools/list`={propose_memory} |
| 10 | **Только теперь** в ChatGPT Custom MCP (Developer Mode): добавить второй MCP connector, указать write Funnel URL. OAuth client id/secret из шага 6 | ChatGPT успешно проходит OAuth, видит `propose_memory` в `tools/list` |
| 11 | Первая боевая запись из ChatGPT: «Запомни что …». Проверить proposal в `proposals-list`. Проверить audit JSONL | Запись в staging, не в production memory |
| 12 | Через час: Stage 2a отрабатывает. На следующую ночь: Stage 2b → `reviewed_clean`. Approve вручную через `proposals-approve <id>`. Проверить что запись появилась в production memory | `search_memory` из Codex/Claude находит запись |
| 13 | Включить scheduled task `AI-memory-write-retention` | Daily 04:00, dry-run в первый день — проверить корректность правил |

**Auto-approve вне MVP.** Включение `AUTO_APPROVE_CLEAN` — отдельная задача в PM-MCP с предусловиями: (a) минимум неделя audit-наблюдений без false negative; (b) отдельный security review; (c) явное согласие пользователя; (d) обновление `docs/CHATGPT-WRITE-GATEWAY.md` с обоснованием.

Аварийный откат на любой фазе: `AI_MEMORY_WRITE_GATEWAY_ENABLED=false` + рестарт. Чтение через 8767 не страдает. Полный wipe staging — `proposals-reject-all --since 1970-01-01T00:00:00Z --reason "incident"`. Откат уже promoted записей — `archive-by-agent --agent chatgpt --since 1970-01-01T00:00:00Z --reason "incident"`.

## 12. Threat model

| Угроза | Митигация |
|---|---|
| Компрометация write OAuth-токена | Отдельный JWT secret и keyring scope; read токен не страдает. Rate limit 20/мин. Tool ровно один (`propose_memory`), пишет в staging — НЕ в production memory. До approve данные не видны другим агентам. `proposals-reject-all --since` мгновенно гасит все вкинутые предложения. |
| Prompt injection заставляет ChatGPT положить мусор | Synchronous intake regex (15+ паттернов) ловит структурные секреты. Strict jsonschema отвергает extra-fields. Stage 2a через час и Stage 2b на следующую ночь — независимые барьеры. **Самое главное**: пока человек не нажал approve (или AUTO_APPROVE_CLEAN не сработал), запись недоступна другим агентам. |
| Tailscale Funnel взломан | Hardened account `ai-memory-write-gateway` (no admin, project-local runtime). ACL только на нужные файлы. Audit JSONL append-only, retain 90 дней — атрибуция полная. |
| Regex classifier пропустил уникальный секрет | До approve запись в staging. `proposals-show <id>` показывает текст для глаз человека. Ручной reject доступен в любой момент. После promotion `archive-by-agent --agent chatgpt --since` — мгновенный откат. |
| Worker crash / scheduler не сработал | Pending proposals остаются в staging. Следующий запуск обработает. `write-review-stats` показывает «lag». |
| Левый OAuth client | Регистрация требует доступа к keyring под аккаунтом пользователя. `write-oauth-client-list` показывает всех. |
| WAL contention read↔write | Та же `memory.db` (WAL). `memory.storage.store()` берёт write-handle только при promotion (редко). Read-only handles 8767 не блокируются. |
| Semantic-секреты («у меня диабет 2 типа») | Regex не ловит. Митигация: документировать в `docs/CHATGPT-WRITE-GATEWAY.md` warning «Не рассказывай ChatGPT медицинскую/финансовую информацию, если не хочешь её сохранить». Human-review в default mode (auto-approve off) — последняя гарантия. |
| Утечка через audit log | В audit пишется только sha256+length+pattern, никогда не сам текст. Полный текст в `memory_proposals` под ACL аккаунта. |
| Compromise приводит к удалению чужих proposals | CLI `proposals-reject/archive` работает только локально под пользователем; gateway не имеет таких tools. Через сеть нельзя удалить чужой proposal. |

## 13. Open questions (для обсуждения в имплементации)

1. **Tailscale Funnel: один FQDN с sub-path vs два FQDN?** Один FQDN с путями `/read/mcp` и `/write/mcp` проще; уточнить возможности `tailscale serve`/`funnel` на текущей подписке. Альтернатива — два разных Funnel-приложения с разными FQDN.
2. **`memory_proposals` — read-only handle 8767 должен иметь к ней доступ?** Нет. `tests/test_write_gateway_isolation.py` фиксирует инвариант: даже SQL-injection через `search` не должна возвращать строки из `memory_proposals`. SQLite не даёт column/table-level ACL — обеспечивается на уровне кода (search SQL никогда не упоминает `memory_proposals`).
3. **Patterns ru_passport/ru_snils — включать по умолчанию?** Включить. `AI_MEMORY_WRITE_GATEWAY_DISABLE_PATTERNS` для точечного отключения при false positives.
4. **Стоит ли позволять ChatGPT смотреть статус своих proposals (read `proposals-mine`)?** **НЕТ в этом плане.** Ответ на `propose_memory` уже содержит `proposal_id` и `stage2b_at`. Чтение статуса через connector — лишняя поверхность. ChatGPT не должен ретраить.
5. **Tags normalization при недопустимых символах: 400 или silent strip?** Предлагаю **400** с осмысленным сообщением — клиент должен знать что его теги отвергнуты. Иначе ChatGPT будет недоумевать почему его теги пропали.
6. **client_fingerprint формула?** sha256(`client_id` || `oauth_sub` || первые два октета IP). Не использовать User-Agent. IP — только grouping/audit/daily-quota, не для авторизации.
7. **`promote_proposal` — синхронный CLI или асинхронный?** Синхронный с try/except. На ошибку `storage.store()` — лог в audit + статус proposal остаётся `reviewed_clean`. CLI retry-safe благодаря idempotency.
8. **Tailscale serve vs funnel — нужно ли требовать аутентификацию tailnet'а как доп. слой?** Funnel открыт в интернет. Если выбрать `tailscale serve` (только для tailnet'а), то ChatGPT не доступится. Funnel + OAuth + scope — единственный путь.
9. **Audit log rotation механизм** — встроенный (size-based + daily) или внешний (Windows scheduled cleanup)? Предлагаю встроенный в `audit.py` (daily file: `write-gateway-proposals-YYYY-MM-DD.jsonl`, retention 90 дней).
10. **Когда удалять `rejected` через 30 дней — стоит ли сохранять в `archive/` JSONL до окончательного DELETE?** Это дополнительная strана. Минимально: при DELETE писать в audit JSONL запись `event=proposal_deleted` с метаданными — этого достаточно.

## 13a. PM-MCP task breakdown

В соответствии с AGENTS.md K.2 и K.7, до старта кода в `D:\GitHub\AI-memory` создаётся набор work items в PM-MCP. Связь — все дочерние ссылаются на **#141** как родителя через `link_task_dependency`. Существующий #141 переводится в «К выполнению» и служит зонтиком.

| Задача | Скоуп | Зависит от |
|---|---|---|
| `#141a` Schema & repository | Миграция v6→v7 в `memory/db.py`, таблица `memory_proposals` с CHECK constraints + UNIQUE-индексами, `memory/write_gateway/proposals.py` + unit-тесты | #141 |
| `#141b` Gateway daemon & OAuth | `memory/write_gateway/{runtime,app,daemon,intake,audit}.py`, OAuth со scope `ai-memory.propose`, middleware chain, `/healthz` + `/readyz`, fail-fast на пустом allowlist | #141a |
| `#141c` Review worker & retention | `memory/write_gateway/{review,promote}.py` (idempotent + compensating recovery), retention scheduled task, daily quota, audit rotation | #141a |
| `#141d` CLI tooling | `memory/cli.py` команды: `write-gateway-*`, `proposals-*`, `write-review-*`, `write-retention-*`, `write-oauth-client-*`, `write-token-*`, `write-gateway-smoke-propose --dry-run`, `archive-by-agent` | #141a, #141b, #141c |
| `#141e` Tests | `tests/test_write_gateway_*.py` — intake/review/proposals/promote/retention/isolation/quota/readiness/app | #141a..d |
| `#141f` Docs | AGENTS.md новый раздел «ChatGPT write gateway», ARCHITECTURE.md диаграмма, `docs/CHATGPT-WRITE-GATEWAY.md` полный, ссылка из `docs/CHATGPT.md` | #141b |
| `#141g` Hardening & deployment | `scripts/install_write_gateway_service.ps1`, `install_write_review_scheduled_task.ps1`, `install_write_retention_scheduled_task.ps1`. Создание Windows account `ai-memory-write-gateway`, project-local runtime, ACL | #141b, #141c |
| `#141h` Production rollout | OAuth client registration, Tailscale Funnel sub-path, ChatGPT connector setup (phase 8-12 раздела 11), мониторинг первой недели | #141d, #141e, #141f, #141g |

Через `link_task_dependency` все 8 подзадач связываются с #141 как родителем; внутри — последовательность зависимостей по таблице. Это даёт исполнителю (Codex) явный порядок и видимость в Assistant-UI.

## 14. Pre-implementation checklist (AGENTS.md compliance)

- [ ] Прочитан AGENTS.md, ARCHITECTURE.md, README.md проекта AI-memory.
- [ ] Прочитана свежая память: `get_recent_memory(project="AI-memory", limit=10)` + `search_memory(query="ChatGPT write proposal", project="AI-memory")`.
- [ ] PM-MCP work item **#141** переведён в статус «К выполнению» через `approve_task`.
- [ ] Зафиксировано в task description: «реализация по плану `chatgpt-woolly-acorn.md`, MVP без auto-approve, порт 8770».
- [ ] Существующий `public-healthcheck` на 8767 возвращает 200 (baseline).
- [ ] `uv sync --dev` свежий.
- [ ] Подтверждено: `AI_MEMORY_WRITE_GATEWAY_PROJECT_ALLOWLIST` будет задан (например `"portfolio,AI-memory,PM-MCP-server,PM-Agent"`). Пустой allowlist = fail-fast.
- [ ] Решена развилка `client_fingerprint` (см. open question 6) и формула формализована в `intake.py`.

## 15. Файлы-якоря (для имплементатора)

Существующие точки, которые **не меняем**, используем как образец:

- `memory/public_app.py:460` — паттерн регистрации tools с wrappers (для `write_gateway/app.py`).
- `memory/mcp_app.py:87` — существующий `register_write_tools(mcp)` (его контракт — образец, но gateway регистрирует `propose_memory`, а не `store_memory`).
- `memory/public_security.py` — middleware (импортируем как есть).
- `memory/public_filters.py` — output wrappers как образец (но для write нужны **input** wrappers — пишем новые).
- `memory/public_daemon.py` — single-instance lock, uvicorn boot.
- `memory/storage.py` — `store()`, вызывается из `promote.py`.
- `memory/validation.py:213` — `validate_no_secrets()`, первичный barrier.
- `memory/db.py` — миграции (добавить v7).
- `scripts/install_public_daemon_service.ps1` — образец hardening-скрипта.
- `docs/CHATGPT.md` — образец документа интеграции.
