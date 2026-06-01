# AI-memory hygiene v2 — lifecycle, write gate, ephemeral archive, weekly digest, retrieval eval

> **Статус**: draft v2. Создан Claude 2026-05-27, обновлён 2026-05-27 после ревью Codex.
> Approved 2026-05-31. PM-MCP implementation tasks: #727-#735.
> Связанный draft: [pm-mcp-bilingual-enums.md](pm-mcp-bilingual-enums.md) (независим, можно делать параллельно).

## Changelog
- **P1.2 applied (2026-05-31)** — production `assign-default-lifecycle`
  semi-manual batch applied to 150 reviewed records. Distribution: 28 durable
  (18.67%), 91 project_state (60.67%), 31 ephemeral (20.67%). Backup:
  `D:\GitHub\AI-Assistant\ai-memory\data\backups\pre-lifecycle-backfill-20260531-154615.db`.
- **P1.2 final backfill + P1.4 dry-run (2026-05-31)** — remaining 402 active
  rows received lifecycle; active distribution is 143 durable, 350 project_state,
  59 ephemeral. `compact-ephemeral --dry-run` with 60-day threshold found 0
  candidates (no old enough ephemeral rows). Control run with `--min-age-days 0`
  found 44 candidates (74.58% of ephemeral), so no production archive was
  applied because it exceeds the safety threshold.
- **P2.1 applied (2026-05-31)** — deterministic `weekly-digest` CLI added.
  Production scoped apply compressed `ai-memory / 2026-W20` from 11 active
  `change` rows into memory `#1495`; source rows were soft-archived via
  `supersedes_memory_id`, digest metadata contains `digest_of`.
- **P2.2 baseline (2026-05-31)** — retrieval eval suite added with a
  privacy-safe golden set. Production advisory report
  `data\exports\retrieval-eval-20260531-160307.json` returned recall@5=1.0,
  MRR@5=1.0, durable_top5_fraction=1.0.
- **P3 ADRs (2026-05-31)** — ADR-0006 fixed weekly digest runtime model as
  manual/advisory until observation proves scheduled dry-run is useful. ADR-0007
  fixed ephemeral retention runtime model as dry-run only until a real 60-day
  candidate pool exists; apply remains reviewed/manual.
- **approval (2026-05-31)** — план утверждён, файл переименован в
  `727-ai-memory-hygiene-v2.md`, созданы PM-MCP задачи #727-#735.
- **v3 (2026-05-31)** — применены 4 правки Codex: P1.1 index path,
  P1.3 `task`/`tasks`, P3.2/P3.3 placeholder, §11 preflight.
- **v2 (2026-05-27)** — учтены 8 замечаний Codex:
  1. Runtime термины: `write-gateway` → `gateway/` + `ai-memory-proposals`; убран `AI-memory-write-retention` scheduled task (review/retention теперь background thread в `AI-Assistant-AI-memory-proposals` daemon).
  2. Lifecycle зафиксирован на 3 значениях (`durable`/`project_state`/`ephemeral`); `incident` выражается тегом `incident-YYYY-MM-DD`.
  3. P1.1 признан как schema v9 migration (JSON1 partial index требует init_db + migration test + EXPLAIN QUERY PLAN); строка про «обойдёмся без миграции» удалена.
  4. P1 перестроен в строгий последовательный pipeline (ADR/docs → schema+validation+filter → backfill → shadow gate → compact-ephemeral).
  5. Pre-write gate: 7 дней shadow + <5% FP (убрано противоречие «сутки»); error contract — `{"status": "error", "error": {...}}` (как текущий storage), не `{"ok": False}`.
  6. `compact-ephemeral` пишет JSON export + PM-MCP audit; в AI-memory — только один итоговый `change` после значимого apply, не routine self-audit rows.
  7. Weekly digest: отдельный deterministic digest mechanism с explicit `item_ids` (НЕ через `store_summary` — он по контракту ARCHITECTURE.md:137 сохраняет только `kind=note` с `metadata.summary_of`); `decision`/`fact` только при реальном наличии в исходниках.
  8. Retrieval eval разделён: pytest на фиксированной БД-fixture + production `eval-retrieval --report` advisory baseline. Golden set строго вручную.
- Ответы Codex на §8 вопросы зафиксированы в §8.
- ADR-0005 (lifecycle/retention) вынесен в P1.0 как обязательный first step.

## 1. Контекст

ChatGPT провёл оценку текущего состояния AI-memory. Часть критики устарела (compaction и audit уже сделаны задачами #641-#646), часть — в точку (operational noise, смешение durable knowledge с changelog/incident, отсутствие meaningful retention policy для основной памяти, а не только для proposal queue).

PM-MCP бэклог проекта `D:\GitHub\AI-Assistant\ai-memory` пуст — все задачи `Готово` или `Не актуально` (preflight check см. §11).

### 1.1 Состояние памяти сегодня (факты, проверено 2026-05-27)

- 5 kinds: `fact`, `decision`, `task_context`, `change`, `note` (ADR-0001 D-4).
- **Schema v8** (`memory/db.py:18` `CURRENT_SCHEMA_VERSION = 8`), FTS5 как primary candidate source, hybrid ranking (semantic + recency + kind_weight + exact_match).
- Active rows: длинных entries `0`, legacy drift `0`, cleanup candidates `0` (audit_snapshot после #644-#646).
- Compaction через superseding summaries работает; есть `normalize-legacy-active`, `compact-long-entries`.
- Soft archive (`archived_at`, `archive_reason`) есть и используется search/recent.
- **Proposal queue runtime** (актуальный контракт, AGENTS.md:77-89, ADR-0001 D-3/D-8):
  - External clients идут через `gateway/`; ai-memory не экспонирует прямые public read/write daemons.
  - `propose_memory` в production memory НЕ загрязняет до явного `proposals-approve`.
  - Stage 2a/2b review и retention выполняются как **background thread внутри `ai-memory-proposals` daemon** (см. `memory/proposals/scheduler.py`).
  - NSSM service `AI-Assistant-AI-memory-proposals` — единственная liveness anchor; **отдельных Windows scheduled tasks для review/retention НЕТ**.
  - CLI `proposals-review-run-now` и `proposals-retention-run-now` — для manual ad-hoc запусков.
- `store_summary` контракт (ARCHITECTURE.md:137): «собрать компактный brief и явно сохранить его как `note` с `metadata.summary_of`». Сохранять через `store_summary` другие kinds — невозможно.
- Multi-agent чтение: codex, claude, chatgpt (через gateway → propose), stepa (manual).
- ADR-0001 (target architecture + kinds), ADR-0002 (workflow composite + audit), ADR-0004 (idea memory contract). ADR-0005 будет введён этим планом (lifecycle/retention).

### 1.2 Подтверждённые проблемы

1. **Operational exhaust доминирует.** В последних 20 записях: 17 × `change` (rollout/migration/verification details), 1 × `note` (correction), 1 × `fact` (compacted incident), 0 × `decision/principle`. Retrieval по запросу «принцип/решение» вытаскивает много audit-шума.
2. **Memory смешивает durable knowledge с changelog.** Архитектурные решения, incident logs, weekly rollout-сводки, smoke-test verification соседствуют в одном пространстве kinds. Невозможно отфильтровать «покажи только долговечное» без хрупких tag-фильтров.
3. **Нет meaningful retention для основной памяти.** `archived_at` существует, но триггеров автоматической архивации (по возрасту + lifecycle + reachability) нет. Старые operational entries живут вечно.
4. **Нет measurable retrieval quality.** Сейчас «лучше или хуже» оценивается интуитивно. Нет golden set'а запросов и recall@K метрик.

### 1.3 Что **не** проблема (вопреки оценке ChatGPT)

- Metadata discipline — это сильная сторона, не слабость. Не «упрощать».
- 5 kinds — достаточная таксономия для FTS-based retrieval; ввод 6+ новых kinds (PROFILE/ARCHITECTURE/INCIDENT/WORKFLOW/TEMP_AUDIT) ломает ADR-0001 D-4 и требует миграции 1471+ записей ради того, что решается одним JSON1-полем.
- Длинные entries уже сжаты до 0 через `compact-long-entries`.
- Архитектура (hybrid ranking, FTS5, audit_snapshot, soft archive) не требует переделки.

## 2. Цель

Сместить баланс памяти с **operational completeness → cognitive signal**. Сделать так, чтобы запросы «какое решение приняли по X», «какой принцип в Y», «что мы знаем про user/profile» возвращали durable knowledge в top-5, а operational details сохранялись только пока актуальны и не доминировали в выдаче.

## 3. Ограничения и non-goals

### Ограничения
- Multi-agent чтение (codex/claude/chatgpt) — нельзя удалять «что было сделано в проекте N месяцев назад», это рабочий запрос.
- Любое решение через `metadata.*` предпочтительнее новой колонки или нового kind.
- Schema migration на v9 (JSON1 partial index для `lifecycle`) — **in scope этого плана**, реализуется как стандартная миграция с `EXPLAIN QUERY PLAN` тестом (tech-stack-choices #3).
- Tech stack remains: SQLite+FTS5+WAL, sentence-transformers, FastMCP+loopback (tech-stack-choices #2, #3, #5). Отклонений нет.
- ChatGPT-side proposal queue (gateway → ai-memory-proposals) и его retention thread не трогаем — отдельная подсистема.

### Non-goals
- **Не** вводить новые kinds (PROFILE/INCIDENT/WORKFLOW и т.п.). Решается через `metadata.lifecycle` + теги.
- **Не** вводить 4-ю lifecycle категорию (`incident`). Incident выражается тегом `incident-YYYY-MM-DD` + lifecycle выбирается по смыслу записи (обычно `durable` если это lessons-learned).
- **Не** вводить TTL по возрасту для kind=change. История изменений нужна.
- **Не** удалять записи. Только soft archive (`archived_at`), reversible.
- **Не** переписывать ranking. Текущий hybrid работает.
- **Не** менять процесс ChatGPT proposal review (stage 2a/2b живёт в `ai-memory-proposals` thread).

## 4. Принципы решения

1. **`lifecycle` как фасеточный фильтр.** Дополнительное измерение поверх kind. **Ровно три значения**, валидируемые в `validation.py`:
   - `durable` — знание с долгим сроком жизни: решения, принципы, ADR-резюме, user profile, hardening паттерны.
   - `project_state` — текущее состояние проекта/фичи: «budget MVP реализован», «daemon перенесён на 8770». Архивируется когда проект меняется (через supersedes_memory_id) или фаза завершается.
   - `ephemeral` — операционный exhaust: verification, smoke-test logs, rollout details, single-commit changelog. Кандидат на auto-archive по возрасту + reachability.

2. **Incidents — через теги, не lifecycle.** Тег `incident-YYYY-MM-DD` + `lifecycle=durable` (если lessons-learned) или `lifecycle=ephemeral` (если просто rollout log). Никаких новых kinds или lifecycle-категорий для incidents.

3. **ADR-0005 как first step.** Lifecycle/retention контракт фиксируется в новом ADR со ссылкой на ADR-0001 D-4 (5 kinds таксономия неизменна). Без ADR — изменения в `validation.py` не вносим.

4. **Quality gate на записи, а не только cleanup после.** Лучше не записывать noise, чем чистить его потом. Pre-write validation для kind=change должна требовать минимум один из: `tasks`/`commit`/`why-в-тексте`/`decision-в-тексте`. **Shadow-mode 7 дней до enforcement, false-positive rate <5%.**

5. **Auto-archive по reachability**, не только по возрасту. Если ephemeral-запись старше 60 дней, не упоминается в active supersedes-цепочке, и связанная задача закрыта — архивируем. Сохраняем восстанавливаемость через `archive_reason`.

6. **Weekly digest как принудительный compaction.** Группы из N однотипных `change` записей одного проекта за ISO-week сжимаются в одну сводку через **отдельный deterministic digest mechanism** (не `store_summary` — он по контракту только `kind=note`). Kind результирующей записи: `change` по умолчанию, `decision`/`fact` только если в исходниках реально есть decision/fact-substance.

7. **Retrieval quality measurable, но без production leakage.** pytest на фиксированной БД-fixture (regression check, blocks CI), production `eval-retrieval --report` — advisory baseline, не блокирующий.

## 5. Этапы

### P1 — Pipeline (строго последовательно)

**P1.0 — ADR-0005 + docs contract** *(блокирует всё P1)*
- Новый `D:\GitHub\AI-Assistant\docs\adrs\0005-memory-lifecycle-retention.md`.
- Контекст: ADR-0001 D-4 (5 kinds неизменны), эта ADR добавляет фасет `metadata.lifecycle`.
- Содержит: enum значений (`durable`/`project_state`/`ephemeral`), default-маппинг от kind, retention эвристика (60d + reachability), incident pattern (тег + lifecycle by meaning), explicit non-extension (никаких 4-х значений или новых kinds).
- Обновить `ai-memory/docs/CONTRACT.md`, `ai-memory/ARCHITECTURE.md`, `ai-memory/AGENTS.md` (lifecycle field в `propose_memory`/`store_memory`).
- Acceptance: ADR-0005 merged, docs ссылаются на него, тесты не меняются (это только контракт).

**P1.1 — Schema v9 + validation + search filter** *(зависит от P1.0)*
- Schema migration `v8 → v9` в `memory/db.py`:
  - `CURRENT_SCHEMA_VERSION = 9`.
  - JSON1 partial index `CREATE INDEX IF NOT EXISTS idx_memory_lifecycle ON memory(json_extract(metadata,'$.lifecycle')) WHERE archived_at IS NULL` (tech-stack-choices #3 паттерн).
  - Idempotent: запуск дважды — no-op.
- `memory/validation.py`: добавить enum `{"durable", "project_state", "ephemeral"}` для `metadata.lifecycle`; default по kind применять при отсутствии поля (`decision`/`fact` → `durable`; `task_context` → `project_state`; `change`/`note` → `project_state`).
- `memory/search.py` + `memory/mcp_app.py`: expose `lifecycle` как фильтр в `search_memory`, `get_recent_memory`, `search_by_metadata` (уже есть top-level metadata поиск).
  - Index-backed path: `search_memory` и `get_recent_memory` добавляют SQL-предикат `json_extract(metadata,'$.lifecycle') = ?` в основной candidate-query / recent-query fast path вместе с `archived_at IS NULL`; именно этот путь должен использовать `idx_memory_lifecycle`.
  - `search_by_metadata(field="lifecycle", ...)` либо получает отдельный SQL special-case с тем же предикатом и `EXPLAIN QUERY PLAN` проверкой, либо остаётся generic metadata scan без обещания index coverage. В плане и acceptance явно не утверждать, что generic `search_by_metadata` всегда использует JSON1 index.
- Тесты:
  - migration v8→v9 на fixture (схема, индекс, idempotency).
  - `EXPLAIN QUERY PLAN SELECT ... WHERE json_extract(metadata,'$.lifecycle') = 'durable' AND archived_at IS NULL` → `SEARCH ... USING INDEX idx_memory_lifecycle`.
  - validation отвергает unknown lifecycle.
  - default-маппинг при отсутствии поля.
- Acceptance: миграция применяется на production memory.db, EXPLAIN использует индекс, ruff+pytest green.

**P1.2 — Backfill semi-manual** *(зависит от P1.1)*
- Admin CLI `assign-default-lifecycle` (по аналогии с `normalize-legacy-active`):
  - `--dry-run --report-path data/exports/lifecycle-backfill-<ts>.json` — генерирует JSON со списком записей, предложенным lifecycle, причиной (heuristic match).
  - **Эвристики**: tags содержат `verification`/`smoke`/`rollout` → предложить `ephemeral`; ADR/`principles`/`hardening`/`milestone`/incident lessons → `durable`; иначе → `project_state`.
  - `--apply --report-path data/exports/lifecycle-backfill-<ts>-apply.json` — применяет только записи, явно одобренные через `--id-list-path` (semi-manual review).
- Backup БД перед apply (паттерн `pre-*-<date>.db`).
- Acceptance: dry-run JSON ревьюится вручную, частичное apply работает, distribution % durable / % project_state / % ephemeral задокументирован после первого batch.

**P1.3 — Pre-write gate в shadow-mode** *(зависит от P1.1, не зависит от P1.2)*
- `memory/storage.py`: новая функция `evaluate_memory_signal(kind, text, metadata) -> SignalDecision`.
- Правила (для `kind=change`):
  - Длина text < 150 символов И нет `metadata.task`/`metadata.tasks`/`metadata.commit` → flag.
  - Нет ни `metadata.task`, ни `metadata.tasks`, ни `metadata.commit`, ни `why:`/`решение:`/`принцип:` в тексте → flag.
  - Перед оценкой нормализовать task references из `metadata.task` ИЛИ `metadata.tasks` в единый список; оба поля считаются валидным task-сигналом.
- **Shadow-mode**: первые 7 дней — только логировать в `data/logs/signal-shadow.jsonl` что было бы reject, реально не reject.
- Override через явный `metadata.lifecycle=ephemeral` + `metadata.bypass_signal_check=true` (для редких легитимных случаев, аудит логирует).
- Error layer: validation-level ошибки продолжают делать `raise`, а `memory/storage.py` возвращает результат записи с существующим success/duplicate контрактом (`status=ok` / `status=skipped_duplicate`). После enforcement недостаточный сигнал возвращается как структурированная ошибка `{"status": "error", "error": {"code": "memory_signal_insufficient", "message": "...", "details": {...}}}` без исключения наружу из storage-layer.
- Для chatgpt proposals — проверять не при `propose_memory` (там intake свой), а при `promote_proposal` / `proposals-approve` (это окончательная запись в production memory).
- Acceptance после 7 дней shadow: <5% legitimate writes flagged. Если ≥5% — ослабить эвристики, продлить shadow ещё на неделю. Покрытие acceptance: direct MCP `store_memory`, CLI `memory store`, `store_batch`, `promote_proposal` (`memory/proposals/promote.py:69`) проходят через один и тот же signal gate перед production write.

**P1.4 — `compact-ephemeral` admin CLI** *(зависит от P1.2)*
- Admin CLI по аналогии с `compact-long-entries`.
- Эвристика: `lifecycle=ephemeral` И `created_at < now - 60 days` И связанная задача (`metadata.tasks`/`metadata.task`) закрыта в PM-MCP (через PM-MCP MCP call) → candidate for archive.
- Reachability check: не архивировать запись, на которую ссылается active (не archived) `supersedes_memory_id`.
- Двухпроходный: `--dry-run` → JSON export → review → `--apply` → JSON export → backup before.
- **Self-noise contract**: routine audit rows в самой AI-memory НЕ создаём при каждом запуске. JSON export + PM-MCP audit_log — единственный источник истины об операции. В AI-memory сохраняется только **один итоговый `change` с `lifecycle=durable, tags=[maintenance, audit, milestone]` после значимого apply** (например, ≥100 записей архивировано или раз в месяц), не после каждого `--dry-run`.
- Fail-closed: при недоступности PM-MCP MCP — пропустить запись (не архивировать), залогировать в JSON.
- Acceptance: dry-run candidate set sane (≥ 50, ≤ 30% от всех ephemeral); apply не ломает active supersedes-цепочки.

### P2 — Структурные (после P1 стабилизации)

**P2.1 — Weekly digest workflow**
- Новый admin CLI `weekly-digest --project <X> --week <YYYY-Www>` (или `--manual-ids <id-list-path>`).
- Группирует `lifecycle=project_state` + `kind=change` за ISO-week по проекту.
- **Threshold**: триггер сжатия если ≥10 записей в группе ИЛИ передан `--manual-ids`.
- Сжимает через **отдельный deterministic digest mechanism** (новая функция `memory/digest.py:build_weekly_digest(item_ids, project, week) -> DigestPayload`), не через `store_summary`:
  - Шаблон саммари: «<Project> <YYYY-Www>: <bullet themes from change titles>. Files touched: <unique files>. Tasks: <unique task refs>. Source items: [<id1, id2, ...>].»
  - Kind результирующей записи: `change` по умолчанию (это и есть change-aggregate); `decision`/`fact` только если в исходниках есть substantive decision/fact (явный `--promote-kind decision` flag после ручного ревью).
  - `lifecycle=durable` для саммари; metadata: `digest_of: [item_ids]`, `digest_period: <YYYY-Www>`, `digest_project: <X>`.
- Оригиналы получают `supersedes_memory_id` указывающий на саммари + `archived_at` (используется существующий механизм soft archive).
- **Запуск**: вручную пользователем (или CLI-driven Codex/Claude). Автоматизация — НЕ в P2; host/process model обсуждается в отдельном P3.2 ADR/плане после validation качества.
- Acceptance: один проект (например, budget или ai-memory) прошёл weekly-digest, ≥10 → 1 запись, retrieval по этим темам не упал по P2.2 metric.

**P2.2 — Retrieval eval suite (разделённый)**
- **pytest на fixture (regression)**: `tests/eval/test_retrieval_quality.py` + `tests/eval/fixtures/memory_fixture.db`.
  - Golden set: ~25-30 пар `(query, expected_top5_memory_ids)` в `tests/eval/retrieval_golden.yaml`.
  - **Строго вручную составленный** (privacy-safe), не из production telemetry.
  - Покрытие: durable principles, project state, hardening patterns, user profile/feedback (если есть), historical decisions, ChatGPT-imported items, ideas (ADR-0004).
  - Метрики: recall@5, MRR@5, фракция top-5 из `lifecycle=durable` для durable-запросов.
  - Threshold: regression если recall@5 упал на ≥10% от baseline — fail тест.
  - Blocks CI (после baseline зафиксирован).
- **Production advisory CLI**: `eval-retrieval --report` для smoke-check на реальной БД.
  - Использует **тот же** golden set (но запускает на production memory.db).
  - Возвращает report JSON, не падает.
  - **Не блокирует**, используется для tracking тренда (можно вызвать вручную или из периодической проверки).
- Acceptance: pytest проходит на fixture; CLI report выдаёт sane numbers (>0 recall@5).

### P3 — Стратегические (отложенные, после P1+P2 валидации)

**P3.1 — Identity/Profile mirror** — **вынесено в отдельное privacy-решение, не часть этого плана**.
- Сейчас user profile/feedback живут в `C:\Users\Zaxva\.claude\projects\D--GitHub-AI-Assistant\memory\` (CLAUDE-local memory), не видны ChatGPT/Codex.
- Опция: durable `kind=fact`, `lifecycle=durable`, `agent=stepa`, `tags=[profile, user, feedback]` мирроринг в AI-memory.
- **Требует**: privacy contract (что mirror'ится, что нет), explicit user approval, audit trail.
- Создаётся как отдельный план/ADR, не зависит от этого. Out of scope здесь.

**P3.2 — Scheduled weekly digest**
- См. отдельный ADR/план (TBD) — обоснование: proposals daemon vs новая `AI-Assistant-AI-memory` service для main-memory maintenance.

**P3.3 — Ephemeral retention background worker**
- См. отдельный ADR/план (TBD) — обоснование: proposals daemon vs новая `AI-Assistant-AI-memory` service для main-memory maintenance.

## 6. Риски

| Риск | Влияние | Mitigation |
|------|---------|------------|
| Schema v9 migration ломает active БД | **Высокое**: production memory.db corrupt | Idempotent migration; backup перед apply; тест миграции на fixture; `EXPLAIN QUERY PLAN` проверка индекса; первое apply на копии БД |
| `lifecycle` backfill неправильно классифицирует существующие записи | Среднее: durable-данные могут попасть под архивацию | Semi-manual через `--id-list-path` review, JSON export; не запускать `compact-ephemeral` до подтверждения classification quality на ≥100 ревьюнутых записях |
| Pre-write gate отвергает легитимные записи агентов | Высокое: ломает Codex/Claude автоматику | Shadow-mode минимум **7 дней** до enforcement; threshold <5% legitimate flagged; структурированная ошибка `{"status":"error","error":{...}}` (не raise); override через `bypass_signal_check` |
| Weekly digest теряет важный сигнал при сжатии | Среднее: durable inference остаётся в исходниках, но retrieval ухудшается | Не удалять — только archive с supersedes; саммари содержит `digest_of: [item_ids]`; eval suite (P2.2) ловит regression ≥10% |
| Eval suite фиксирует случайный baseline вместо «правильного» | Низкое-среднее | Golden set строго вручную, ревьюится перед commit; privacy: никаких production telemetry источников |
| Auto-archive ломает работающие ссылки `supersedes_memory_id` | Низкое | Не архивировать запись, на которую ссылается active (не archived) supersedes; reachability check в `compact-ephemeral` |
| Дрейф между PM-MCP closed-status и `compact-ephemeral` reachability | Среднее: если PM-MCP MCP недоступен, эвристика deg | Fail-closed: при недоступности PM-MCP MCP — пропустить запись (не архивировать), залогировать |
| `compact-ephemeral` сам создаёт self-noise в памяти | Низкое (но обидное в свете цели плана) | JSON export + PM-MCP audit — единственный routine источник; в AI-memory — только один `change` после ≥100 архивов или раз в месяц |
| `store_summary` использован вместо нового digest mechanism | Низкое: контракт `store_summary` сохраняет только `kind=note` (ARCHITECTURE.md:137) | Не использовать `store_summary` для P2.1; новый `memory/digest.py` — отдельный код с тестами |

## 7. Критерии приёмки

### После P1.0 (ADR)
- ADR-0005 merged, в `docs/adrs/`.
- CONTRACT.md, ARCHITECTURE.md, AGENTS.md ai-memory ссылаются на ADR-0005.
- Никаких изменений кода — только контракт.

### После P1.1 (schema + validation + filter)
- Production memory.db мигрирован v8→v9.
- `EXPLAIN QUERY PLAN` для lifecycle-фильтра показывает `SEARCH ... USING INDEX idx_memory_lifecycle`.
- `metadata.lifecycle` валидируется и виден в search/search_by_metadata.
- ruff + pytest green.

### После P1.2 (backfill)
- Backfill распределение задокументировано (% durable / % project_state / % ephemeral).
- ≥100 записей semi-manually ревьюнуто и применено.
- Backup сохранён в `data/backups/pre-lifecycle-backfill-<ts>.db`.

### После P1.3 (shadow gate)
- Shadow-mode минимум 7 дней.
- false-positive rate <5% legitimate writes flagged (измерено по `signal-shadow.jsonl`).
- Если ≥5% — ослабить эвристики, продлить shadow ещё на 7 дней (не enforce).

### После P1.4 (compact-ephemeral)
- `compact-ephemeral --dry-run` показывает sane candidate set (≥ 50, ≤ 30% от всех ephemeral).
- Apply не ломает active supersedes-цепочки (validation check).
- В AI-memory создан только один итоговый `change` после apply, не series of audit rows.

### После P2
- Один проект прошёл weekly-digest для последних 2-4 ISO-weeks, сжато ≥ 10 записей → 1-3 саммари, retrieval по этим темам не упал по P2.2 metric.
- pytest eval suite запускается, baseline зафиксирован.
- `eval-retrieval --report` CLI выдаёт sane numbers.

### После P3
- Отдельный ADR/план определяет host/process model для weekly-digest и ephemeral-retention; до этого P3 не создаёт runtime commitments.
- Privacy contract для profile mirror либо merged, либо явно отклонён.

## 8. Ответы Codex на §8 (зафиксировано 2026-05-27)

| # | Вопрос | Ответ Codex |
|---|---|---|
| 1 | Lifecycle taxonomy: 3 или 4 значения? | **3** (`durable`/`project_state`/`ephemeral`). `incident` — через тег `incident-YYYY-MM-DD`, lifecycle по смыслу. |
| 2 | Backfill heuristics: tag-based или semi-manual? | **Semi-manual через review export** (`--dry-run` JSON → manual review → `--apply --id-list-path`). |
| 3 | Pre-write gate strictness: shadow на всех или сразу enforce на proposals? | **Shadow для всех direct writes**; proposals проверять на `promote`/`approve` (не на `propose`). |
| 4 | Weekly digest scope и threshold? | **Per-project per-ISO-week**, threshold **10+ записей** ИЛИ explicit `--manual-ids`. |
| 5 | Eval suite: вручную или из telemetry? | **Вручную**. Privacy — никакой production telemetry без явного contract. |
| 6 | Profile mirror (P3.1): делать? | **Вынести в отдельное privacy-решение**. Не часть этого плана. |
| 7 | ADR-0001 update или ADR-0005? | **Отдельный ADR-0005** со ссылкой на ADR-0001 (ADR-0001 D-4 остаётся неизменным). |
| 8 | (extra) Кодекс отметил: PM-MCP preflight | **Добавить в §11**: перед approval — preflight «PM-MCP доступен, открытые ai-memory задачи проверены». |

## 9. Out of scope (явно)

- ChatGPT-side proposal queue review process (`stage2a`/`stage2b`) — отдельная подсистема в `ai-memory-proposals` daemon thread.
- Перевод memory с SQLite на Qdrant/Chroma/Postgres — tech-stack-choices #3 фиксирует SQLite+FTS5.
- UI для memory triage — admin CLI достаточно для текущего workflow.
- Изменение hybrid ranking weights.
- Runtime-автоматизация weekly-digest / ephemeral-retention до отдельного P3 ADR/плана.
- Profile/identity mirror (вынесено в отдельное privacy-решение, P3.1).
- 4-я lifecycle категория или новые kinds (зафиксировано §4 принцип 2 + ответ Codex #1).

## 10. Связанные документы

- ADR-0001 (target architecture, kinds taxonomy): `D:\GitHub\AI-Assistant\docs\adrs\0001-target-architecture.md` (особенно D-3, D-4, D-8).
- ADR-0004 (idea memory contract): `D:\GitHub\AI-Assistant\docs\adrs\0004-idea-memory-contract.md`.
- ADR-0005 (lifecycle/retention) — будет создан в P1.0.
- Текущая maintenance документация: `D:\GitHub\AI-Assistant\ai-memory\docs\MAINTENANCE.md`.
- AGENTS.md ai-memory (proposal queue runtime): `D:\GitHub\AI-Assistant\ai-memory\AGENTS.md:77-89`.
- ARCHITECTURE.md ai-memory (`store_summary` контракт): `D:\GitHub\AI-Assistant\ai-memory\ARCHITECTURE.md:137`.
- `memory/db.py:18` (current schema v8).
- Tech stack: `D:\GitHub\_engineering_rules\tech-stack-choices.md` пункты 2, 3, 5, 12.
- Реализованный quality-maintenance: AI-memory PM-MCP #641-#646, #170-#175.

## 11. После approval

**Preflight (до создания задач):**
- PM-MCP MCP доступен через `list_work_items(project_path="D:\GitHub\AI-Assistant\ai-memory", status=<status>, limit=20)` для каждого открытого статуса: `Бэклог`, `На согласовании`, `К выполнению`, `В работе`.
- Открытые задачи проекта `D:\GitHub\AI-Assistant\ai-memory` повторно проверены. Факт Codex 2026-05-31: открытых задач 0 по четырём статусам (`Бэклог`, `На согласовании`, `К выполнению`, `В работе`) перед созданием P1-задач.
- Подтвердить с пользователем что bilingual-enums план (`pm-mcp-bilingual-enums.md`) не блокирует — оба плана независимы, но желательно скоординировать порядок.

**Поэтапно:**
1. Codex и Claude итеративно ревьюят план, фиксируют ответы на новые открытые вопросы в этом же файле (без переименования пока).
2. Создать PM-MCP задачи под P1 пайплайн — **5 задач последовательно** (P1.0 → P1.1 → P1.2 → P1.3 → P1.4), не параллельно. P1.3 (shadow gate) может идти параллельно с P1.4 (compact-ephemeral) после P1.2. Фактические ID: #727 P1.0, #728 P1.1, #729 P1.2, #730 P1.3, #731 P1.4.
3. После назначения первого ID — переименовать файл в `<id>-ai-memory-hygiene-v2.md`, создать harness в `C:\Users\Zaxva\.claude\plans\ai-memory-hygiene-v2.md` со ссылкой. Файл переименован в `727-ai-memory-hygiene-v2.md`; harness не создан, потому что активная работа ведётся прямо из central plan.
4. P2 задачи — после завершения P1 (нужно знать реальные пропорции lifecycle после backfill). Фактические ID: #732 P2.1, #733 P2.2.
5. P3 — после 1-2 месяцев наблюдения за P1+P2 в production. P3.1 — отдельный privacy ADR/план. Фактические ID: #734 P3.2, #735 P3.3.
