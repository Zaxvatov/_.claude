# AI-memory: устаревание, переоценка, синтез, провенанс

> Статус: **черновик для обсуждения с Codex**. Не реализация — меню доработок
> с приоритетами. Каждый поток (W0–W5) утверждается и реализуется отдельно.

PM-MCP: umbrella **#866**. Задачи (assignee codex): #867 W0, #868 W1, #869 W2, #870 W3,
#871 W5a, #872 W5b, #873 W4. Зависимости: #868→#867; #869→#868; #873→#868,#869;
#872→#871; #870→#862 (WP-6, assistant-ui — уже Готово). Startable сейчас: #867 (W0), #871 (W5a).

Связанные ADR (точные файлы): `docs/adrs/0004-idea-memory-contract.md`,
`0005-memory-lifecycle-retention.md`, `0006-weekly-digest-runtime-model.md`,
`0009-intent-tree-recursive-goals.md`, `0010-memory-graph-read-contract.md`.
⚠️ **ADR-0018 (owner-auth) и ADR-0019 (service identity) ЗАНЯТЫ** планом
`secure-agent-platform-hardening` (задачи #859/#865). Этот план берёт **ADR-0020**
(W1 staleness; в `docs/adrs` сейчас 0001–0017, 0020 свободен) и отдельный ADR под W4
(следующий свободный — сверить live backlog перед созданием файла/задачи).
Связанные планы: `goal-intent-tree.md`, `capture-inbox.md`, `secure-agent-platform-hardening.md`.

---

## Контекст

Триггер — обзор обновлений памяти ChatGPT (memory «dreaming», несколько
источников персонализации, объяснение источников ответа, авто-устаревание,
memory summary). Сверка идей обзора с фактическим состоянием AI-memory:
**4 из 5 предложений уже реализованы**. План фиксирует только реальные пробелы,
чтобы не строить заново то, что есть, и чтобы Codex обсуждал конкретику, а не
общие места из новости.

### Что уже есть — НЕ переделывать

| Идея из обзора | Реализация |
|---|---|
| Источник/провенанс записи | `metadata.source` (lowercase, массив), `agent`, `files`, `repo`, `commit` — `ai-memory/ARCHITECTURE.md` |
| Memory Summary | `store_summary` / `preview_summary`, `metadata.summary_of` / `digest_of` |
| Авто-устаревание (operational) | ADR-0005: `lifecycle` durable/project_state/ephemeral; `ephemeral` >60д → архив при закрытой задаче и недостижимости по supersedes |
| Граф Цель→Задача→Файл→Воспоминание (рёбра) | ADR-0009 intent-tree (`parent_goal_id`, `get_goal_tree/path/decomposition`), `set_work_item_goals`/`related_goals`, `metadata.work_item_id`/`files`, `search_by_workitem`, `traverse_memory` + `get_tag_graph` (ADR-0010) |
| Свежесть в ранжировании | recency-decay half-life 30д + supersedes-цепочка |
| Просмотр памяти / review | UI `/memory`, proposal-очередь + signal-gate shadow (ADR-0005 P1.3) |

### Non-goals — явные отказы (с причиной)

- **Числовой `confidence` на каждый факт.** Write-burden + неоткалиброванное
  число. Потребность «это ещё правда?» закрывают supersedes + recency + W1.
- **4-е значение `lifecycle`.** Запрещено ADR-0005; W1 — аддитивные metadata-поля,
  не новый фасет жизненного цикла.
- **GraphRAG / entity extraction / graph DB.** Отклонено ADR-0010 для small
  single-user корпуса.
- **Авто-apply синтеза без review.** Отклонено ADR-0006.

---

## Рабочие потоки

### W0 — Contract hygiene (перед W1, блокер)

Перед добавлением полей устранить дрейф «доки vs код», иначе W1 ляжет поверх
рассогласованного контракта:

- **lifecycle default для `note` — РАССЛЕДОВАТЬ, не считать дрейф доказанным.**
  ADR-0005 и `ai-memory/ARCHITECTURE.md` говорят `note → project_state`, но
  `memory/validation.py:49` (`DEFAULT_LIFECYCLE_BY_KIND`) даёт `note → ephemeral`.
  Сначала найти источник расхождения (git-blame / история задач): если `ephemeral`
  было осознанным поздним решением — **НЕ переписывать историю ADR-0005**, а оформить
  refinement-ADR; если это баг — чинить код на `project_state`. Зафиксировать решение
  и привести доки+код+тест к одному значению.
- **Точные ссылки на ADR** в этом плане (см. шапку): `0004-idea-memory-contract.md`,
  `0006-weekly-digest-runtime-model.md`, `0009-intent-tree-recursive-goals.md`.
- **Drift в доках verify-страниц** (см. Верификация): список сценариев в
  `assistant-ui/AGENTS.md` уже, чем фактический `verify_frontend.py`.

**Файлы:** `ai-memory/memory/validation.py`, `docs/adrs/0005-memory-lifecycle-retention.md`,
`ai-memory/ARCHITECTURE.md`, `ai-memory/tests/test_validation.py`, `assistant-ui/AGENTS.md`.
**Acceptance:** источник дрейфа `note` lifecycle установлен и решение зафиксировано
(refinement-ADR ИЛИ фикс кода — что именно подтверждено расследованием); один источник
истины во всех местах + тест; ADR-ссылки в плане точные; verify-доки совпадают со `SCENARIOS`.

### W1 — Устаревание личных / `durable` фактов  ⟵ анкер, начинать (после W0)

**Проблема.** Retention ADR-0005 трогает только `ephemeral`. Личные
`fact`/`durable` (карьерная цель, статус Permit S, поиск квартиры, курсы
немецкого, выплаты) **не истекают никогда** и со временем тихо протухают в выдаче.

**Дизайн** (по образцу ADR-0004 — аддитивные metadata-поля, **без миграции схемы**).
Два РАЗНЫХ механизма, не смешивать (уточнение Codex):

1. **Hard expiry — `metadata.valid_until`** (ISO-8601). Дата в прошлом → запись
   **не выдаётся как свежая**: исключается из обычного search/recent, возвращается
   только при `include_stale=true` и тогда несёт `stale_reason` (напр. `expired`).
   Для фактов с реальным сроком (Permit S до даты, курс до даты).
2. **Soft review — `metadata.review_after`** (ISO-8601). Дата в прошлом → запись
   **остаётся видимой**, но помечается `review_due=true`. Это per-fact «скорость
   устаревания»: разные факты переоцениваются по-разному, поэтому `review_after`
   обязателен — одного `last_confirmed` мало.
3. **`metadata.last_confirmed`** (ISO-8601) — провенанс: когда факт последний раз
   подтверждали. Обновляется при «подтвердить» (W2); подтверждение двигает
   `review_after` вперёд на интервал, заданный подтверждающим (без отдельного
   поля `staleness_days`).

**Временная семантика (зафиксировать до тестов).** Все три поля — **date-only
`YYYY-MM-DD`** (не datetime) в пользовательской / конфигурируемой timezone. Правило:
`valid_until < today` → expired; `review_after < today` → review_due (строгое `<`,
сравнение по локальной дате). Источник `today` — единый helper (env/config-driven tz)
в `memory/validation.py` / `search.py`, чтобы тесты не зависели от системных часов.
Убирает расхождение тестов и поведения поиска от таймзон/часов.

**Авто-архив выключен по умолчанию.** Истёкший личный факт → кандидат на переоценку
(W2), не авто-удаление. Держит fail-closed урок ADR-0005 (нельзя архивировать по
возрасту в одиночку); `valid_until` лишь меняет видимость/ранжирование, не архивирует и
**не ломает прямой lookup / lineage / `traverse_memory`** (запись остаётся active-row,
лишь прячется из обычного search/recent без `include_stale`). Авто-архив — отдельный
явный opt-in, не в MVP.

**Валидация** (`memory/validation.py`): строгий ISO-формат всех трёх полей; поля
допустимы на любом `kind`, но stale/review-логика применяется к `durable`.

**Tech-stack.** brick #3 — JSON1 partial indexes по `$.valid_until` и `$.review_after`
(`... WHERE archived_at IS NULL`); проверить `EXPLAIN QUERY PLAN` = `SEARCH ... USING
INDEX`. Никакого нового внешнего тех-выбора. Контракт → **ADR-0020** (staleness facet,
расширяет ADR-0005; 0018/0019 заняты security-планом) + строка в контракт-таблице
`ai-memory/ARCHITECTURE.md`.

**Файлы:** `ai-memory/memory/validation.py`, `memory/search.py`, `memory/db.py`
(2 индекса + тест миграции), `memory/storage.py`, `ai-memory/ARCHITECTURE.md`,
`docs/adrs/0020-*.md`, тесты `test_validation.py` / `test_search.py` / `test_storage.py`.

**Acceptance:** три поля нормализуются/валидируются (мусор отклоняется); `valid_until`
в прошлом скрыт без `include_stale` и несёт `stale_reason`; `review_after` в прошлом →
`review_due=true`, запись видна; JSON1 indexes в плане запроса; старая БД без полей =
прежнее поведение; описана semi-manual backfill стратегия (как P1.2 ADR-0005).

### W2 — Очередь переоценки («is this still true?»)

**Что.** Read-инструмент (MCP + CLI): вернуть `durable`, у которых `valid_until < today`
ИЛИ `review_after < today` → review-список с `review_due` / `stale_reason`.

**⚠️ Подтверждение НЕ проходит через текущий `store_memory`** (находка Codex,
подтверждена кодом). `storage.py:259-261`: при совпадении `text+project+agent+kind`
возвращается `skipped_duplicate` **до** вставки, а суперседы из
`metadata.supersedes_memory_id` применяются только на пути вставки (`:310+`). Значит
«подтвердить» (тот же текст, новый `last_confirmed`) тихо ничего не обновит. Механизм
**решён с Codex — вариант A**: узкий
**`confirm_memory_entry(id, last_confirmed?, review_after?)`** через **контролируемый
supersede или явный audit trail**, НЕ простое in-place по умолчанию (in-place ослабляет
lineage durable-фактов). Вариант B (обход duplicate short-circuit через
`metadata.supersedes_memory_id` в `store_memory`) **отклонён** — слишком широкий, легко
превращается в общий escape hatch. Любой write-механизм подчиняется write-gating из
W3 (не чат-toolset).

**Runtime.** Advisory-модель ADR-0006: периодический **dry-run report** (не авто-apply).

**Файлы:** `ai-memory/memory/search.py`/`admin.py` (запрос), новый write-tool в
`memory/storage.py` + `memory/mcp_app.py` + `memory/cli.py` + sync
`runtime_contract.EXPECTED_TOOLS` и `scripts/check_stdio_contract.py` (hazard #8),
опц. `assistant-ui` вкладка.

**Acceptance:** read-инструмент возвращает stale / `review_due` `durable`;
«подтвердить» реально обновляет `last_confirmed` / `review_after` (механизм A, с
тестом, что больше нет тихого `skipped_duplicate`); routine-прогон не пишет
self-audit строки в AI-memory (урок ADR-0006).

### W3 — Answer-time провенанс в UI («какие записи сформировали ответ»)

**Что.** В чате assistant-ui показать N записей, реально попавших в контекст ответа
(retrieval-провенанс), отдельно от хранимого `metadata.source`. Данные уже есть
(`search_memory` отдаёт `score`-компоненты и `id`).

**⚠️ Prerequisite — read-only gating чат-toolset (находка Codex, подтверждена).**
`assistant-ui/app/tool_router.py:8`: `MEMORY_TOOLS` включает `store_memory`, т.е.
модель в чате может писать в память напрямую, а confirmation включён только для
`batch_run` / `portfolio_run`. До provenance-фичи чат-toolset памяти должен стать
**read-only**, записи — только через явный UI/proposal/confirmation контур. Это уже
**WP-6 #862** плана `secure-agent-platform-hardening` (capability gating) — НЕ
дублировать, а сделать W3 зависимым от него.

**Зависимость (проверить):** как assistant-ui инжектит память в контекст ответа (путь
recall в чат-цепочке) — непрослеженный участок, короткий аудит `assistant-ui/app`.

**Tech-stack.** brick #6 / #16 / #10. UI-сторона, без изменения контракта AI-memory.

**Acceptance:** после ответа доступен список использованных memory `id` + score;
чат-toolset памяти read-only (writes через gated контур); инварианты `tool_router`
сохранены.

### W4 — Синтез / «dreaming» (advisory, review-gated)  ⟵ самый рискованный, последним

**Что.** Проход синтезирует сводки по скользящему окну записей по weekly-digest модели
ADR-0006: dry-run report → **reviewed apply** (manual id-list), итоговая строка с
`digest_of`/`summary_of` + supersede. **MVP — только manual/dry-run, БЕЗ scheduled
runtime.** Scheduled task (как `AI-memory-write-retention`, память id 1275) добавляется
**только после** safety-ADR + eval-гейта + периода наблюдения; никогда proposals daemon
и никогда авто-apply.

**⚠️ Отдельный safety contract (отдельный ADR — следующий свободный после 0020,
сверить backlog).** Синтез над `durable`/личными фактами опаснее обычного digest,
поэтому фиксируем явно: (1) никакого cloud-LLM без явного opt-in (по умолчанию —
локальная модель / детерминированный digest); (2) только dry-run, apply лишь по
reviewed id-list; (3) каждая синтез-строка несёт cited source IDs (`digest_of` /
`summary_of`); (4) eval-гейт ДО/ПОСЛЕ (P2.2 ADR-0006) — синтез не должен ухудшать
retrieval; (5) env kill-switch; (6) reports owner-only; (7) **никакого auto-supersede
и auto-archive**; (8) **отчёты в `data/exports/` могут содержать личные факты** →
owner-only доступ, gitignore, retention-политика, без raw full text без явного запроса.

**Детали реализации (уточнить при старте W4):** эвристика отбора окна и дедуп против
существующих `summary_of`; граница `digest_of` (детерминированный) vs synthesized
summary. (Eval-набор уже зафиксирован в «Решённые вопросы»: golden retrieval set +
source-id coverage + no-new-claims check + diff top-k.)

**Acceptance:** соблюдён весь safety contract выше; синтез никогда не пишет в
production напрямую; routine не засоряет память audit-строками.

### W5 — Единый обход Цель→Проект→Задача→Файл→Воспоминание (read-view)

**ВАЖНО:** это **не пробел в модели данных** — рёбра уже есть (см. таблицу «что уже
есть»). Пробел — **одно read-представление**, проходящее границу PM-MCP ↔ AI-memory.
Обзор ChatGPT назвал это «highest leverage», но 80% уже лежит.

**⚠️ Prerequisite в AI-memory — расширить matching `search_by_workitem` (находка
Codex, подтверждена).** `search.py:1004` матчит только `metadata.work_item_id`, а
реальные записи массово используют `metadata.task` / `metadata.tasks` (`#NNN` — почти
вся память в recall). Без этого read-view пропустит большинство записей. Надо матчить
`work_item_id | task | tasks` с канонизацией в `#NNN`. Поэтому W5 **не чисто
consumer-side**: сначала маленькая правка в ai-memory, потом агрегатор.

**Дизайн агрегатора (решено — assistant-ui первым).** Начинать как read-view для
`/tree` в `assistant-ui`; PM-MCP read-tool — позже и только если агентам реально нужен
тот же агрегат. ADR-0010 запрещает AI-memory резолвить граф целей/задач → агрегатор
живёт НЕ в ai-memory. Обход: `get_goal_tree` (PM-MCP) → work items узла → расширенный
`search_by_workitem` (AI-memory) → `metadata.files`. **Лимиты обязательны:** только
выбранная цель, max linked tasks/memories, timeout, partial-result markers.

**Acceptance:** `search_by_workitem` матчит `work_item_id | task | tasks` (тест на
`#NNN`); одна операция отдаёт срез goal→tasks→memories→files; cross-boundary,
read-only; AI-memory не знает о goal-графе (boundary ADR-0010).

---

## Матрица задач и зависимости

| Поток | Subsystem(ы) | Зависит от |
|---|---|---|
| W0 hygiene | ai-memory (validation) + docs / ADR-0005 | — |
| W1 staleness | ai-memory | W0 |
| W2 review queue + confirm | ai-memory (read-tool + write-tool) | W1 |
| W3 answer-time provenance | assistant-ui | WP-6 #862 (read-only gating) |
| W5 graph read-view | ai-memory (расширить `search_by_workitem`) → assistant-ui / PM-MCP | ai-memory prereq внутри W5 |
| W4 synthesis / dreaming | ai-memory (MVP manual/dry-run; scheduled — позже) | W1/W2 + safety-ADR; делать последним |

PM-MCP подключается, только если read-tool W5 нужен переиспользуемым (иначе агрегатор —
в assistant-ui). Per-subsystem work items заводятся после утверждения направления
(как в #853): отдельная задача на каждый затронутый subsystem, с `link_task_dependency`
по таблице выше. Ключевые корректировки против исходной редакции: W3 зависит от
security-gating (#862), а W5 имеет ai-memory-prerequisite (не чисто UI). Зависимость
W3→#862 оформить реальным `link_task_dependency` с корректным `dependency_project_path`
(cross-plan зависимость на security-задачу), не только текстом в плане.

## Риски

- **W1:** авто-архивация личных фактов по дате может скрыть нужное — отсюда
  «по умолчанию down-rank+review, авто-архив только opt-in» (fail-closed, урок
  ADR-0005). Backfill может ошибочно проставить `valid_until` → semi-manual review.
- **W4:** авто-синтез зашумляет память; обязательны review-apply + kill-switch +
  eval-гейт + период наблюдения (ADR-0006).
- **W3/W5:** cross-boundary провенанс/обход не должен дать AI-memory резолвить
  PM-граф (boundary ADR-0010).
- **Общий:** расширение контракта без миграции и без синхронного обновления
  `EXPECTED_TOOLS` / stdio bridge / `check_stdio_contract.py` (hazard #8 в ARCHITECTURE.md).

## Чек-лист контрактных изменений (hazard #8)

Любой новый MCP-tool (W2, возможно W5) синхронно: daemon-регистрация → stdio
bridge → `memory/runtime_contract.py` `EXPECTED_TOOLS` → `scripts/check_stdio_contract.py`
(brick #5). Новые metadata-поля (W1) → `validation.py` + контракт-таблица
ARCHITECTURE.md + JSON1 index (brick #3). Новый контракт → **ADR-0020** (W1 staleness)
+ отдельный ADR под W4 (следующий свободный после 0020). ADR-0018/0019 заняты #859/#865.

## Верификация

- ai-memory: `uv run ruff check .` + `uv run pytest` (валидация полей, search-фильтр,
  миграция старой БД, `EXPLAIN QUERY PLAN` на индексе).
- stdio-контракт (внутри ai-memory): `uv run python scripts/check_stdio_contract.py`
  (если добавлены tools).
- assistant-ui (W3/W5): `uv run python scripts/verify_frontend.py --page tree|memory`
  — **подтверждено**: `tree` и `memory` есть в `SCENARIOS` (`verify_frontend.py:380`,
  help-строка `:434`; brick #17 тоже); более узкий список в `assistant-ui/AGENTS.md` —
  стейл, чинится в W0. Изолированный ephemeral-instance, не живой сервис.
- ai-memory CLI smoke для W2/W4 dry-run report против временной БД (не live).

## Решённые вопросы (Codex round 2)

Зафиксировано в этой редакции: поля = `valid_until` + `review_after` + `last_confirmed`
(date-only `YYYY-MM-DD`, локальная tz, строгое `<`); авто-архив выключен; ADR = новый
**0020** (0018/0019 заняты); W4 — отдельный safety-ADR. Ответы на прежние развилки:

1. **W2 confirm-механизм:** вариант **A** (`confirm_memory_entry`), но через
   контролируемый supersede / audit trail, не in-place; B отклонён.
2. **W2 поверхность ревью:** сначала **CLI/dry-run**, UI — после закрепления
   confirm-контракта.
3. **W4 eval-гейт:** golden retrieval set + source-id coverage + no-new-claims check +
   diff top-k (до/после).
4. **W5 место агрегатора:** **assistant-ui** первым; PM-MCP read-tool — позже, если
   нужен агентам.
5. **W0 `note` lifecycle:** сначала **расследовать** источник дрейфа; refinement-ADR
   если решение было осознанным, фикс кода если баг — не считать drift доказанным заранее.

Открытых блокеров для утверждения как дорожной карты не осталось.

## После утверждения (по central-plan-workflow)

1. Создать PM-MCP задачу(и) per-subsystem; записать `PM-MCP: #<id>` в этот файл.
2. Ретроспектива по 4 осям перед закрытием: tech-stack (вероятный brick «staleness
   facet»), Design-system (W3/W5 UI), skills, hooks — изменения общих правил
   только с подтверждением пользователя.
3. Outcome → `ai-memory-capture`; архитектурную причину → ADR-0020 (W1) и safety-ADR (W4).
