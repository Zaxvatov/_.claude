# Two-layer Design-system: вынос app-specific ассетов Assistant-UI

PM-MCP: A #781 (_engineering_rules) → B1 #782 → B2 #783 (assistant-ui) → C #784 (Design-system). Зависимости A→B1→B2→C проставлены (K.7). Assignee: codex.

## Context

`Design-system` (`D:\GitHub\Design-system`) задумывался как общий MD3-слой для 4 consumer-проектов
(Verua_automation, Stalking_offline, Best_photo_ai, Assistant-UI). Но он начал владеть не только
примитивами (base.html, токены, темы, шрифты, `@material/web` bundle, app-shell, dialog-хелперы),
а ещё и **фиче-логикой экранов Assistant-UI**:

| Ассет в DS | Используется | Чей экран |
|---|---|---|
| `assets/assistant-ui.css` | только Assistant-UI | shell + budget/settings/analytics таблицы + Excel-форматирование |
| `assets/assistant-budget.js` | только Assistant-UI | `/budget` |
| `assets/assistant-dashboard.js` | только Assistant-UI | `/dashboard` |
| `assets/assistant-memory.js` | только Assistant-UI | `/memory` |

Другие consumers эти файлы не подключают. Подключение из шаблонов Assistant-UI (4 точки):
- `app/templates/_assistant_head.html:1` → `assistant-ui.css`
- `app/templates/budget.html:271` → `assistant-budget.js`
- `app/templates/dashboard.html:332` → `assistant-dashboard.js`
- `app/templates/memory.html:103` → `assistant-memory.js`

**Проблема.** Правка бюджетного UI Assistant-UI становится коммитом в общий DS (память #771/#764/#1540:
«No assistant-ui repository files changed»). Blast radius раздут до всех consumers, ownership размыт
(«где лежит логика страницы?»), а версионный контракт DS (F.b) размывается приватными правками.

**Это закреплено правилами, поэтому переезд ассетов без правки правил бессмысленен** — агенты честно
вернут фичу в DS:
- tech-stack brick **#10**: «нет `static/*.css`», всё в `../Design-system/assets/`.
- `assistant-ui/AGENTS.md:85-89`: «Do not create new project-level CSS files … add to `../Design-system/assets/`».
- (брик **#16** при этом уже разрешает `<app>/static/<page>.js` — частичная рассинхронизация, которую заодно чиним.)

**Outcome.** Двухслойная модель: общий примитивный слой остаётся в DS; собственная фиче-логика
Assistant-UI переезжает в `assistant-ui/app/static/` рядом со своими шаблонами. Цель — простота:
фиче-правка = один коммит в одном репозитории.

## Цель (целевое состояние)

- **Слой 1 — Design-system (общий, версионируемый):** `base.html`, MD3-токены, темы (`theme.assistant.css`),
  шрифты, `@material/web` bundle, `app-shell.css`, общие компоненты и dialog-хелперы. Только то, что
  реально нужно ≥2 приложениям.
- **Слой 2 — assistant-ui (app-local):** шаблоны страниц **и** собственная фиче-логика
  (`app.css` + `budget.js` / `dashboard.js` / `memory.js`) в `assistant-ui/app/static/`.

`extends "ds/base.html"` и `{{ ds_static }}/tokens/theme.assistant.css` **остаются**. Переезжают только
4 app-specific ассета. Тест принадлежности: *«Стал бы Best_photo_ai / Stalking_offline использовать
ровно это правило?»* Нет → код приложения.

## Ограничения и принципы

- **migration-discipline:** строим новый путь → переносим → удаляем старый → чистим доки. Ни одной
  ссылки `/ds/assets/assistant-*` в итоге не остаётся.
- **Правило — первым.** Этап 1 (brick #10 + AGENTS.md + ADR) до/вместе с переносом, иначе дрейф вернётся.
- **Не ломать consumers** (DS AGENTS.md F.c, A.4): перед удалением — grep по Verua_automation /
  Stalking_offline / Best_photo_ai (ожидаемо пусто).
- **K.4:** отдельный work item на каждый репозиторий.
- **Generic-слой не трогаем:** `app-shell.css`, токены, темы, `base.html`, fonts, bundle, dialog-хелперы
  остаются в DS. `theme.assistant.css` генерируется `scripts/generate_theme.py` — часть DS-контракта, НЕ переносим.
- `assistant-ui.css` при переносе ревизуется: если найдётся действительно generic-правило (нужное другим
  apps) — оставить в DS `app-shell.css`; ожидаемо таких нет, файл целиком Assistant-UI-specific.

## Этапы

### Этап 1 — Разрешить app-local слой (enabler, doc-only)
- **_engineering_rules** `tech-stack-choices.md` brick **#10**: переформулировать на двухслойную модель
  (shared в DS; app-specific feature CSS/JS в `<app>/static/`); согласовать с #16; явно зафиксировать
  причину отклонения от прежнего «нет `static/*.css`» (org-level deviation note).
- **assistant-ui/AGENTS.md** раздел «UI and Design System» (стр. 79-89): переписать — DS-first для
  токенов/компонентов/shell; app-specific фиче-стили/скрипты — в `app/static/` рядом со своими шаблонами.
- **AI-Assistant** `docs/adrs/`: новый ADR «Two-layer Design-system boundary» (что shared, что app-local,
  тест принадлежности) — авторитетная запись решения; brick #10 и AGENTS.md ссылаются на него.

### Этап 2 — Построить новый путь в assistant-ui
- `app/main.py`: добавить второй mount `app.mount("/static", StaticFiles(directory=str(APP_STATIC)), name="static")`
  (`APP_STATIC = Path(__file__).resolve().parent / "static"`) и Jinja global `app_static="/static"` рядом
  с существующим `ds_static` ([main.py:97-113](D:/GitHub/AI-Assistant/assistant-ui/app/main.py:97), [main.py:300](D:/GitHub/AI-Assistant/assistant-ui/app/main.py:300)).
- Создать `app/static/` с контентом из DS-ассетов: `app.css` (из `assistant-ui.css`), `budget.js`,
  `dashboard.js`, `memory.js` (префикс `assistant-` снимаем — мы уже внутри приложения).
  **Внимание:** `Design-system` сейчас dirty (uncommitted `assistant-*.{css,js}`, `assistant-memory.js`) —
  копировать текущее рабочее содержимое файлов, НЕ состояние из `HEAD`.
- Переключить 4 ссылки в шаблонах с `{{ ds_static }}/assets/assistant-*` на `{{ app_static }}/*`
  (`_assistant_head.html`, `budget.html`, `dashboard.html`, `memory.html`). Прочие `{{ ds_static }}`
  (theme, app-shell, fonts, bundle) — не трогать.
- **Обновить тесты** (иначе `pytest` упадёт — они проверяют старые пути): 3 ассерта в
  `tests/test_api_endpoints.py` → `/static/...`: [:627](D:/GitHub/AI-Assistant/assistant-ui/tests/test_api_endpoints.py:627)
  (`assistant-ui.css` → `app.css`), [:757](D:/GitHub/AI-Assistant/assistant-ui/tests/test_api_endpoints.py:757)
  и [:772](D:/GitHub/AI-Assistant/assistant-ui/tests/test_api_endpoints.py:772) (`assistant-dashboard.js` → `dashboard.js`).
- **Добавить сценарий `memory`** в `scripts/verify_frontend.py`: `_scenario_memory(browser, base_url, out_dir)`
  по образцу `_scenario_budget`/`_scenario_dashboard` ([verify_frontend.py:222](D:/GitHub/AI-Assistant/assistant-ui/scripts/verify_frontend.py:222)),
  зарегистрировать в `SCENARIOS`. Ассертить рендер каркаса страницы (tabs «Записи»/«Кандидаты»), а не
  наличие строк (память может быть пуста, AI-memory fail-open).
- Проверка: `uv run python scripts/verify_frontend.py --page all` (budget+dashboard+memory) desktop+mobile,
  `uv run ruff check .`, `uv run pytest`.
- **Preconditions проверок:** `budget` — нужен `budget.db` (brick #14, без MCP); `dashboard` — нужен живой
  PM-MCP `8766`, иначе сценарий честно падает на известной #780, а не на миграции ассетов; `memory` —
  AI-memory stdio (fail-open, каркас рендерится и при пустой памяти).

### Этап 3 — Удалить старый путь в Design-system
- Grep consumers на `assistant-(ui|budget|dashboard|memory)` (Verua/Stalking/BestPhoto) — подтвердить, что ссылок нет.
- Удалить `assets/assistant-ui.css`, `assistant-budget.js`, `assistant-dashboard.js`, `assistant-memory.js`.
- `DESIGN_SYSTEM.md`: убрать из дерева ассетов (стр. 53-56) и §8.2 «Assistant-UI budget and settings»
  (стр. 403-425); заменить короткой заметкой «фиче-ассеты Assistant-UI живут в `assistant-ui/app/static/`».
- **Changelog / version (конкретно, т.к. файла нет):** создать `Design-system/CHANGELOG.md` (Keep a Changelog),
  записать removal публичных путей `/ds/assets/assistant-*`; bump `pyproject.toml` `version` `0.1.0 → 0.2.0`
  (pre-1.0 breaking → minor); обновить `DESIGN_SYSTEM.md` §12 «Версия документа». (F.b: version bump + changelog note.)
- DS `AGENTS.md` — обновить, если ссылается на эти файлы.

### Этап 4 — Финальные доки + память
- `assistant-ui/ARCHITECTURE.md`: описать `/static` mount рядом с `/ds` (Key modules / Directory layout).
- AI-memory (`ai-memory-capture`): по одному `change`-outcome на assistant-ui и Design-system + один
  `decision` о двухслойной модели (`project="portfolio"`, ссылка на ADR и этот план).
- Перенос плана в `plans/DONE/` по критериям central-plan-workflow.

## Риски и митигировка

| Риск | Митигировка |
|---|---|
| Регресс верстки/поведения (особенно `/budget`, Material Web select-sync) | `verify_frontend` desktop+mobile (+dark), `pytest`, визуальное сравнение до/после |
| Пропущенная ссылка на старый ассет → 404 | grep `assistant-(ui|budget|dashboard|memory)` после переноса; smoke всех 3 страниц + `/ds/...` 200 |
| Неожиданная ссылка из другого consumer | grep Verua/Stalking/BestPhoto перед удалением (Этап 3, до `rm`) |
| Агенты по привычке снова кладут фичу в DS | Этап 1 (правило + ADR) обязательно первым |
| Случайно перенести `theme.assistant.css` | Явно исключено — генерится в DS, остаётся частью контракта |

## Критерии приёмки

- [ ] 4 app-specific ассета физически в `assistant-ui/app/static/`, удалены из `Design-system/assets/`.
- [ ] Ни одного `/ds/assets/assistant-*` в шаблонах; `/ds` остаётся только для base/app-shell/tokens/theme/fonts/bundle.
- [ ] `verify_frontend.py` имеет сценарий `memory`; `--page all` (budget+dashboard+memory) desktop+mobile,
      `ruff`, `pytest` — зелёные; `/budget`, `/dashboard`, `/memory` без 404 и визуально идентичны
      (dashboard — при живом PM-MCP 8766, см. preconditions).
- [ ] brick #10 и `assistant-ui/AGENTS.md` описывают двухслойную модель; ADR зафиксирован.
- [ ] `DESIGN_SYSTEM.md` не содержит Assistant-UI фиче-контракта; создан `CHANGELOG.md`,
      `pyproject.toml` `0.1.0 → 0.2.0`, §12 «Версия документа» обновлён.
- [ ] AI-memory: outcome (×2 repo) + decision записаны.

## Cross-repo задачи PM-MCP (K.4, с зависимостями K.7)

Правило-первым отражено в раскладке: enabler-доки (A + B1) идут до кодового переноса (B2).

- **A (#781). _engineering_rules** — brick #10 (+ #16 reconcile). *enabler (Этап 1).*
- **B1 (#782). AI-Assistant / assistant-ui** — `assistant-ui/AGENTS.md` (§UI переписать) + ADR `docs/adrs/` «Two-layer
  Design-system boundary». *enabler-доки (Этап 1), repo AI-Assistant, отдельный work item от B2.* `depends_on #781`.
- **B2 (#783). AI-Assistant / assistant-ui** — `/static` mount + перенос 4 ассетов + 4 ссылки в шаблонах +
  3 ассерта в `test_api_endpoints.py` + сценарий `memory` в verify + `ARCHITECTURE.md`.
  *код (Этап 2 + 4), repo AI-Assistant.* `depends_on #782`.
- **C (#784). Design-system** — удаление 4 ассетов + `DESIGN_SYSTEM.md` + `CHANGELOG.md` + version bump. *(Этап 3).* `depends_on #783`.

## Кандидаты на дополнения (только с подтверждением пользователя, после утверждения)

- brick #10 update — это и есть фиксируемое отклонение от org-level выбора (входит в Этап 1).
- Hook-напоминание «app-specific фиче-CSS/JS → `app/static`, не в DS» через `hook-authoring` —
  **отклонено пользователем 2026-06-02**: достаточно brick #10 + ADR (меньше движущихся частей).
  Вернуться к идее, если дрейф повторится.
