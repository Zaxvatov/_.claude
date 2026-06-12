# AI-memory: поиск личного контекста (retrieval-фикс, MVP-gate)

**Проект:** `D:\GitHub\AI-Assistant\ai-memory`
**Scope (рекомендуемый, до approval):** Phase 1 как самостоятельный **MVP-gate** (последовательность MVP-gate
выбрана пользователем). Классификация (memory_scope / topics / ranking-boost) вынесена в отдельный Phase 2 plan
и пишется только если Phase 1 не закрывает симптом.
**PM-MCP work items** (project `ai-memory`, все `К выполнению`, assignee `codex`): **#765** baseline+golden eval → **#766** multilingual model+reindex → **#767** semantic scan; **#768** search_by_metadata membership; **#769** Language policy persist + hygiene; **#770** GATE (зависит от #766/#767/#768).
**Ревью Codex:** 9/10. Учтены 10 пунктов первого ревью + 5 второго + 4 финальных (заголовок, `uv run`, persist
Language policy, расширение формулировки).

---

## Context

Англоязычный диагностический запрос `"Stepan career goals Switzerland SAP consultant German B2 Zurich"` в
`search_memory` вернул только инженерные записи. Запрос пришёл из ChatGPT — это **тест на устойчивость retrieval**:
агент может сам сформулировать tool-query по-английски, хотя личная память хранится **по-русски** (см. Language policy).
Это НЕ значит, что память надо хранить на английском. Гипотеза пользователя (нет разделения personal/project) —
реальная, но **третья** по важности. Доминируют две retrieval-механические причины, воспроизведённые на живой базе.

| Проверка | Результат |
|---|---|
| Русский запрос `Швейцария Цюрих работа…` | switzerland-записи всплывают, `score_semantic` **0.56–0.63** |
| Англоязычный диагностический запрос (ChatGPT/агент) | те же записи `score_semantic` **~0.25**, в топ-15 их нет |
| `search_by_metadata(field="tags", value="user-profile")` | `0 results`, хотя записи 1108/1123 имеют тег |

### Root causes (verified в коде)

- **RC1 — англоязычная embedding-модель (доминирует).** `memory/config.py:9` → `all-MiniLM-L6-v2`. Семантика весит
  **0.78** итогового балла (`memory/search.py:118`); на EN↔RU вектора слабые → ранжирование рушится.
- **RC2 — кандидаты выбираются лексически + по свежести.** `fetch_candidate_rows` (`memory/search.py:245-299`):
  FTS5/LIKE по токенам + **всегда дополняет «200 новейших»** (`config.py:11` `DEFAULT_SEARCH_CANDIDATE_LIMIT=200`).
  Англоязычный запрос не матчит русский текст по FTS → пул схлопывается до 200 новейших (id ~1330+); карьерные записи
  (id 1107–1167) старее окна → **вообще не скорятся**. _Смены модели одной мало — выбор кандидатов лексический._
- **RC3 — нет фасета personal/project.** Личные факты под `project="portfolio"` вперемешку с инженерными. _(Адресуется
  в Phase 2, не сейчас.)_
- **Баг — `search_by_metadata` не матчит член массива.** `memory/search.py:592-593,627-632` сравнивает скаляр с полем
  и запрещает не-скаляры → `tags`-membership невозможен.
- **Шум — операционные «Задача #NNN переведена…»** (`kind=change`, `agent=pm-mcp-server`, `event=task_status_changed`)
  активны и засоряют выдачу.

---

## Language policy (контракт)

- **`memory.text` user-facing / personal / profile / career / preferences / context записей хранится по-русски**
  (включая семейные, языковые, бытовые и документальные факты). Исключения: имена технологий, файлов, путей,
  вакансий, официальные термины, оригинальные цитаты/названия.
- **Машинные метки `metadata.tags` / `kind` / прочие `metadata`-поля могут быть английскими** — это допустимые
  машинные ярлыки.
- **Retrieval обязан понимать EN и RU запросы без перевода сохранённых записей на английский.** Именно это даёт
  multilingual-модель (Step 1): русский `text` остаётся русским, а англоязычный agent-query находит его семантически.
  Превращать личную память в англоязычный слой «ради удобства/дешевизны агентов» — **запрещено** (экономия токенов
  сомнительна, а потеря качества и нарушение ожидаемого языка реальны).
- _Persist:_ правило фиксируется в `ARCHITECTURE.md` («Контракт записи») и `docs/CONTRACT.md`, не только в плане.

---

## Phase 1 — Recommended MVP (scope на реализацию после approval)

Цель: исправить retrieval минимальным, низкорисковым набором изменений, затем **остановиться у gate** и измерить.
Все команды — через subsystem env (`uv run`, brick #7).

**Step 0 — Backup + baseline eval.**
Бэкап БД (`uv run python -m memory.cli backup`). Снять baseline через существующий eval-харнесс
(`uv run python -m memory.cli eval-retrieval --golden-path … --report …`; модуль `tests/eval/test_retrieval_quality.py`)
на наборе golden-запросов, включая точный англоязычный диагностический запрос.

**Step 1 — Multilingual embedding model + reindex.**
- `memory/config.py:9` — `DEFAULT_EMBEDDING_MODEL` → `paraphrase-multilingual-MiniLM-L12-v2` (384d, как сейчас;
  50+ языков, сильный EN↔RU; ~470MB). Env-override `AI_MEMORY_EMBEDDING_MODEL` сохраняется (`config.py:95`).
- _Зачем именно так:_ модель реализует **Language policy** — записи остаются русскими, а EN/RU запросы находят их
  семантически без перевода контента. Это и есть правильная альтернатива «хранить на английском».
- Реиндекс через готовый `memory/reindex.py` (`reindex_embeddings`, CLI `uv run python -m memory.cli reindex`,
  тест `tests/test_reindex.py`) — пере-эмбеддит все строки и проставит новый `embedding_model` (`migration_004`).
- Обновить warmup-контракт под новую модель (`server.py` MCP startup; cold-start вырастет — заметка в RUNTIME.md).
- ⚠️ **Tech-stack deviation** — см. отдельную секцию ниже (это замена org-level brick #2).

**Step 2 — Семантический выбор кандидатов вместо recency-окна.**
- `memory/search.py:fetch_candidate_rows` — когда `query_embedding` доступен, финальный скан берёт **все активные
  строки** до cap `AI_MEMORY_SEMANTIC_SCAN_LIMIT` (default 5000; новый ключ в `config.py`). Для текущих ~1.5k строк —
  исчерпывающе.
- **Degraded-mode (Codex #7):** если активных строк > cap — **не резать молча** старые. Выбрать осознанно
  (напр. FTS-кандидаты + новейшие до cap) и вернуть/залогировать флаг `degraded_candidate_scan=true`. Follow-up
  на `sqlite-vec`/ANN — отдельной задачей, в этот план не входит.
- Векторизовать cosine через `numpy` (`memory/search.py:cosine_similarity`), чтобы скан всех строк был <50мс.

**Step 3 — `search_by_metadata`: membership для list-полей.**
- `memory/search.py:578-646` — для list-полей (`tags`, `files`) матчить `value ∈ field`, сохранив скалярное
  равенство для остальных. `value` остаётся скаляром (сигнатура tool в `memory/mcp_app.py:240-255` не меняется).
- **Контракт (Codex #9):** обновить `ai-memory/docs/CONTRACT.md` — явная семантика «scalar `value` ⇒ равенство ИЛИ
  membership в list-поле»; добавить тесты.

**Adjacent hygiene (низкий риск, вне gate-критериев).**
Архивировать операционный шум **специализированным** `uv run python -m memory.cli archive-status-transitions`
(`memory/admin.py:archive_status_transitions`, `cli.py:435`). **НЕ** `archive-by-agent` — он откатывает promoted
proposal-записи, а не чистит PM-MCP шум (Codex #10).

### 🚦 GATE (после Phase 1)
Повторить точный англоязычный диагностический запрос + golden eval. **Acceptance (Codex #5):** в топ-K всплывают
**существующие** career/switzerland profile-факты — проверять по **ожидаемым тегам/фрагментам text**, а не по
конкретным id (они дрейфуют): `career`, `switzerland`, `user-profile`, `cover-letter`, `german-learning`; для EN и RU
запросов; `search_by_metadata(field="tags", value="user-profile")` ≠ 0.
- **Если качество достаточно** → задача закрыта, классификация не нужна.
- **Если нет** → писать отдельный Phase 2 plan (ниже).
_Захват недостающих карьерных фактов (SAP MM/SD/EWM, B2, Zurich, зарплата) — НЕ в scope; их в памяти как
дискретных фактов нет (Codex #5). Это отдельная approved-задача/proposal (и её `text` — по-русски, см. Language policy)._

---

## Tech-stack deviation (обязательно, Codex #2 + hook)

`tech-stack-choices.md` **brick #2** фиксирует `all-MiniLM-L6-v2` и прямо перечисляет ограничение
«Не подходит для multi-language reasoning». Переход на `paraphrase-multilingual-MiniLM-L12-v2` — это **изменение
org-level brick**, а не только `ai-memory` config: мы попали ровно в задокументированное ограничение.
- В плане декларация отклонения зафиксирована (этот раздел).
- **После approval** предложить обновление brick #2 в `D:\GitHub\_engineering_rules\tech-stack-choices.md`
  (новый выбор + «было» в Trade-off), **только с подтверждением пользователя**. JSON1-индексы (если дойдём до
  Phase 2) уже покрыты brick #3 — отдельного отклонения не требуют.

---

## Files to modify (Phase 1)

| Файл | Изменение |
|---|---|
| `memory/config.py` | мультиязычная модель по умолчанию; `AI_MEMORY_SEMANTIC_SCAN_LIMIT` |
| `memory/search.py` | семантический скан кандидатов + degraded-mode; numpy-cosine; membership в `search_by_metadata` |
| `memory/reindex.py` / `cli.py` | реиндекс под новую модель (механизм есть; проверить на новой модели) |
| `server.py` / `docs/RUNTIME.md` | warmup под новую модель; заметка про cold-start |
| `docs/CONTRACT.md` | семантика `search_by_metadata` для list-полей **+ Language policy** |
| `ARCHITECTURE.md` | зафиксировать **Language policy** в секции «Контракт записи» |
| `tests/eval/test_retrieval_quality.py` | golden-запрос (англоязычный диагностический) как регрессия (EN + RU) |
| `tests/test_search.py`, `tests/test_reindex.py` | старая релевантная строка попадает в скоринг; membership; реиндекс |

_MCP tool-схемы живут в `memory/mcp_app.py` (Codex #6); в Phase 1 их сигнатуры не меняются — правки внутренние._

---

## Verification (Phase 1)

1. **Eval-харнесс:** baseline (Step 0) vs после — golden-запрос (англоязычный) поднимает career/switzerland записи в
   топ-K для EN и RU; отчёт через `write_eval_report` (`uv run python -m memory.cli eval-retrieval --report`).
2. **Unit:** `test_search.py` (RC2 — старая по id строка скорится при доступном эмбеддинге; membership tags);
   `test_reindex.py` (новая модель); degraded-mode-флаг при превышении cap.
3. **Live MCP smoke (после reindex):** точный англоязычный диагностический запрос;
   `search_by_metadata(field="tags", value="user-profile")` ≠ 0.
4. **Final validation (AGENTS.md):** `uv run ruff check .` + `uv run pytest`.

---

## Deferred — Phase 2 (ОТДЕЛЬНЫЙ план, писать только если gate провален)

Гибрид: стабильный грубый фасет для ранжирования + эмерджентные темы для discovery (best-practice ответ на вопрос
про авто-облако тегов). Не строить преждевременно (Codex #1, #8).
- **`metadata.memory_scope = personal | project | operational`** (Codex #4 — точнее generic `category`; коллизий в
  `admin.py` нет). Контролируемый enum в `memory/validation.py`; JSON1 partial index — `migration_010`, schema v10
  (`memory/db.py`, паттерн `migration_009` + brick #3).
- **Авто-присвоение:** zero-shot по «якорям» (эмбеддинги описаний классов, nearest-centroid) + детерминированные
  правила (`agent=pm-mcp-server`+`event=task_status_changed` → operational). **sklearn-кластеризация/HDBSCAN —
  out** до отдельного решения; если войдёт — `uv add scikit-learn` явно (сейчас только transitive в `uv.lock`,
  Codex #3).
- **Backfill:** один **one-off** CLI `classify` + report; **без scheduler** (постоянная фоновая классификация —
  отдельное арх-решение: владелец lifecycle, частота, audit, rollback, confidence — Codex #8).
- **Ranking:** фильтр `memory_scope` на `search_memory`/`get_recent_memory` (правки в `memory/mcp_app.py` +
  `runtime_contract.py` EXPECTED_TOOLS schema + tests + `docs/CONTRACT.md`) и модест category-boost в `final_score`.
- **Topics-облако / LLM-обогащение** — ещё позже, поверх Phase 2.

---

## Open items

- Подтвердить размер/латентность `paraphrase-multilingual-MiniLM-L12-v2` приемлемы (cold-start вырастет; альтернатива
  меньше — обсудить, если важен footprint).
- После approval: предложить обновление tech-stack brick #2 (с подтверждением).
- PM-MCP задачи созданы: **#765–#770** (`ai-memory`); зависимости #765→#766→#767 и GATE #770 ← {#766, #767, #768}.
