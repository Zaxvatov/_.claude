# Impeccable Audit — Assistant UI (product register)

Контекст загружен из [docs/PRODUCT.md](docs/PRODUCT.md) и [docs/DESIGN.md](docs/DESIGN.md); shared Design-system обнаружен в `../Design-system/` и принят как авторитативный (MD3 tokens, `@material/web`, `theme.assistant.css`). Audit покрывает 18 шаблонов, [Design-system/assets/assistant-ui.css](../../Design-system/assets/assistant-ui.css) и app shell.

## Audit Health Score

| # | Dimension | Score | Key Finding |
|---|-----------|-------|-------------|
| 1 | Accessibility | 2/4 | 4 detail-страницы без sidebar; нет focus trap в кастомных диалогах; `<md-outlined-text-field>` как hidden input |
| 2 | Performance | 3/4 | Полный `location.reload()` после каждой мутации в budget/settings/ideas; крупные inline-скрипты |
| 3 | Responsive Design | 3/4 | Бюджетный журнал 10px font (фиксированный); icon-button 40×40 < WCAG 44×44 |
| 4 | Theming | 3/4 | Хардкоженные хексы в `assistant-ui.css` (`--assistant-choice-*`, month tints) — нарушение DESIGN_SYSTEM.md MUST NOT |
| 5 | Anti-Patterns | 3/4 | Кастомные `<div class="...-dialog">` обходят `<md-dialog>`; ~10 идентичных panel-карточек подряд на dashboard |
| **Total** | | **14/20** | **Good (address weak dimensions)** |

## Anti-Patterns Verdict — Pass с оговорками

Это **не AI slop**. Нет gradient text, glass, hero-metric SaaS-шаблонов, identical card grids с icon+heading+text, generic OKLCH-from-scratch. Палитра приходит из source `#0F766E` через MD3 Color Utilities, шрифт Roboto Flex. Дашборд намеренно «плотный, предсказуемый» по [docs/PRODUCT.md](docs/PRODUCT.md) — это соответствует cockpit-нарративу.

Что портит картину: **кастомные `<div>`-диалоги в [budget.html:394](app/templates/budget.html:394) и [settings.html:319](app/templates/settings.html:319)** вместо `<md-dialog>` (который уже подключен и используется в [_assistant_head.html:30](app/templates/_assistant_head.html:30)). Дублирование affordance внутри одного приложения — главный product-side anti-pattern здесь.

## Executive Summary

- Score **14/20** — Good. Слабые места — A11y и hardcode-цвета.
- Найдено: **1 P0**, **7 P1**, **9 P2**, **5 P3**.
- Top-3 критичных:
  1. **P0** Сломанный SSE-парсер в [console.html:64](app/templates/console.html:64) (`split("\\n\\n")` вместо `"\n\n"`) — AI-консоль не парсит стрим вообще.
  2. **P1** Кастомные диалоги в budget/settings обходят `<md-dialog>`, нарушают DESIGN_SYSTEM.md §5.4 + ломают focus trap/restore.
  3. **P1** 4 detail-страницы (work_item, goal, project, drift_watch) без `app-shell`/sidebar — навигация недоступна с этих страниц, mobile-bar отсутствует.

## Detailed Findings by Severity

### P0 — Blocking

**[P0] Сломанный SSE-парсер в AI-консоли**
- **Location**: [app/templates/console.html:64](app/templates/console.html:64)
- **Category**: Performance / Correctness
- **Impact**: `buffer.split("\\n\\n")` в Jinja-шаблоне рендерится как литеральная строка из 4 символов `\n\n` в JS, а SSE-разделитель — два настоящих перевода строки. В результате `parseSse` никогда не находит блок → токены и tool-call события не отображаются в `/console`. Сравнить с корректным [dashboard.html:1010](app/templates/dashboard.html:1010), где `split("\n\n")`.
- **Recommendation**: заменить `"\\n\\n"` на `"\n\n"` в обеих `.split()` ([console.html:64](app/templates/console.html:64), [console.html:67-68](app/templates/console.html:67)).
- **Suggested command**: `/impeccable harden`

### P1 — Major

**[P1] Detail-страницы без shell-навигации**
- **Location**: [work_item.html:5](app/templates/work_item.html:5), [goal.html:5](app/templates/goal.html:5), [project.html:5](app/templates/project.html:5), [drift_watch.html:5](app/templates/drift_watch.html:5)
- **Category**: Accessibility / Consistency
- **Impact**: эти страницы используют `<div class="content detail">` без `.app-shell`/`_sidebar.html`/`mobile-bar`. На мобильном вообще нет sidebar toggle — единственный путь назад это текстовая ссылка «← Панель управления». Sidebar-collapsed состояние теряется. Это inconsistent component vocabulary, по [reference/product.md](.claude/skills/impeccable/reference/product.md) — anti-pattern.
- **Recommendation**: обернуть в стандартный `app-shell + mobile-bar + _sidebar.html + content`, как в [dashboard.html](app/templates/dashboard.html) / [overview.html](app/templates/overview.html).
- **Suggested command**: `/impeccable adapt`

**[P1] Кастомные диалоги вместо `<md-dialog>`**
- **Location**: [budget.html:394](app/templates/budget.html:394) (`budget-transaction-dialog`), [settings.html:319](app/templates/settings.html:319) (`settings-edit-dialog`)
- **Category**: Anti-Pattern / Accessibility
- **Impact**: оба строят `<div role="dialog" aria-modal="true">` вручную с backdrop через `position: fixed; inset: 0`. `<md-dialog>` доступен (используется в [_assistant_head.html:30](app/templates/_assistant_head.html:30)) и обязателен по DESIGN_SYSTEM.md §5.4. Отсутствует focus trap, focus не возвращается на opener после `dialog.remove()`, нет inert на остальном контенте, Escape реализован вручную и навешивается на `document` без cleanup при reload диалога.
- **Recommendation**: мигрировать оба на `<md-dialog>` с slots `headline / content / actions`, удалить `.budget-transaction-*` и `.settings-edit-*` классы из `assistant-ui.css` (в Design-system PR).
- **Suggested command**: `/impeccable harden`

**[P1] Хардкоженные хексы в shared CSS**
- **Location**: [Design-system/assets/assistant-ui.css:18-35](../../Design-system/assets/assistant-ui.css:18), `--assistant-budget-month-01..12`, `--assistant-choice-red/blue/strong-red/on-*`
- **Category**: Theming
- **Impact**: прямо нарушает MUST NOT из DESIGN_SYSTEM.md TL;DR («hardcode hex anywhere»). Month-tints оправданы как «пользовательская палитра месяцев» (§8.2 явно разрешает), но `--assistant-choice-strong-red: #ff0000` и пара `#ffc7ce / #b4c6e7` — Excel-импорт без MD3-роли, в тёмной теме контраст не пересчитывается.
- **Recommendation**: `choice-*` перевести на `--md-sys-color-error-container / -primary-container / -secondary-container`. Если палитра должна сохранять Excel-эстетику — задокументировать исключение в §8.2 DESIGN_SYSTEM.md и сделать парные dark-варианты.
- **Suggested command**: `/impeccable colorize`

**[P1] Login: `<md-outlined-text-field>` как hidden input**
- **Location**: [login.html:10](app/templates/login.html:10)
- **Category**: Accessibility / Anti-Pattern
- **Impact**: `class="sr-only" tabindex="-1" aria-hidden="true"` на интерактивном web-component — invented affordance, нагружает MD-runtime ради скрытого значения. Может появиться в табе на некоторых браузерах, теряет value при медленной гидратации `<md-*>`.
- **Recommendation**: `<input type="hidden" name="next" value="{{ next }}">`.
- **Suggested command**: `/impeccable distill`

**[P1] Журнал бюджета: фиксированный 10px шрифт**
- **Location**: [assistant-ui.css:322](../../Design-system/assets/assistant-ui.css:322) (`.budget-journal-table { font-size: 10px }`)
- **Category**: Accessibility / Responsive
- **Impact**: ниже WCAG-рекомендованного floor для table body. Не растёт от browser zoom text-only, не уважает `--md-sys-typescale-body-*`. На retina выглядит как dot pattern, а на мобильном (где это и так overflow horizontal) нечитаемо.
- **Recommendation**: перевести на `var(--md-sys-typescale-label-small-size)` или минимум 12px; колонки сжать `text-overflow: ellipsis` + tooltip с полным значением.
- **Suggested command**: `/impeccable typeset`

**[P1] Нет `aria-current="page"` в навигации**
- **Location**: [_sidebar.html:4-13](app/templates/_sidebar.html:4)
- **Category**: Accessibility
- **Impact**: active ссылка различима только визуально (`class="active"` → background). Screen reader не знает, что юзер на этой странице.
- **Recommendation**: добавить `{% if active == 'overview' %}aria-current="page"{% endif %}` к каждому `<a>`.
- **Suggested command**: `/impeccable harden`

**[P1] CSRF-cookie с `samesite="strict"` без `secure`**
- **Location**: [app/main.py:307](app/main.py:307)
- **Category**: Security-adjacent (не A11y, но критично для local-cockpit)
- **Impact**: `httponly=False` — нормально (JS должен читать), но без `secure=True` при HTTPS-окружении токен утечёт через man-in-the-middle. Single-user локальный режим это терпит, но при exposure через tunnel-туннель CSRF rotation сломается.
- **Recommendation**: добавить `secure=request.url.scheme == "https"` и зафиксировать в [docs/RUNTIME.md](docs/RUNTIME.md), что cockpit бьётся только по localhost. (Если уже есть — игнорировать.)
- **Suggested command**: `/impeccable harden`

### P2 — Minor

- **[P2] Icon-button 40×40 < WCAG 2.5.5 (44×44)** — [assistant-ui.css:174](../../Design-system/assets/assistant-ui.css:174). Поднять `--md-icon-button-state-layer-size` до 44px на mobile breakpoint.
- **[P2] Toast без `aria-live`** — [dashboard.html:291](app/templates/dashboard.html:291). `role="status"` подразумевает `polite`, но Safari не всегда применяет — добавить явно.
- **[P2] `location.reload()` после каждой мутации** — [ideas.html:120](app/templates/ideas.html:120), [settings.html:289](app/templates/settings.html:289), [budget.html:715](app/templates/budget.html:715). Журнал бюджета на ~1000 строк перегружается целиком, теряется scroll position. Заменить на точечный update + SSE-broadcast.
- **[P2] Фильтр-поля audit без accessible label** — [audit.html:25-28](app/templates/audit.html:25): только `placeholder`. Заменить на `label="since"`.
- **[P2] Dashboard EventSource не закрывается** — [dashboard.html:988](app/templates/dashboard.html:988), [overview.html:129](app/templates/overview.html:129): нет `beforeunload` cleanup, нет reconnect strategy при temporary 502.
- **[P2] 5-колоночная `.metrics`** — [assistant-ui.css:1037](../../Design-system/assets/assistant-ui.css:1037): `repeat(5, minmax(78px, 1fr))` = 390px floor, overflows на узком dashboard column. Использовать `auto-fit, minmax(78px, 1fr)`.
- **[P2] Inline-скрипты по 800+ строк** — [dashboard.html](app/templates/dashboard.html), [budget.html](app/templates/budget.html). Не cacheable, парсится при каждой загрузке, не дружит со strict-CSP. Вынести в `Design-system/assets/assistant-dashboard.js` (отдельным PR).
- **[P2] Дублирование filter-pattern** — фильтры на dashboard через `<details>+md-checkbox` ([dashboard.html:95-105](app/templates/dashboard.html:95)), а на budget через `<md-menu>+md-menu-item` ([budget.html:57-67](app/templates/budget.html:57)). Inconsistent vocabulary — выбрать один.
- **[P2] Mobile-bar дублирует `<a href="/dashboard">панель</a>`** в overview/audit/weekly_report/ideas/console, но CSS `mobile-bar a { display: none }` на 620px скрывает их. Мёртвый код — убрать.

### P3 — Polish

- **[P3]** `.budget-row-future { opacity: 0.7 }` и `.budget-row-locked { opacity: 0.78 }` могут опустить контраст ниже 4.5:1 на pastel month-tints. Заменить opacity на `color: var(--md-sys-color-on-surface-variant)`.
- **[P3]** Кастомный tooltip `.budget-date::after` с `transition-delay: 1s` ([assistant-ui.css:508](../../Design-system/assets/assistant-ui.css:508)) — реализован вручную через CSS, отсутствует при keyboard focus. Использовать `title` или native `<md-tooltip>` если появится.
- **[P3]** `confirmationText()` дублируется в [dashboard.html:1020](app/templates/dashboard.html:1020) и [console.html:75](app/templates/console.html:75). Вынести в `_assistant_head.html`.
- **[P3]** Status filter `<details>` в [project.html:41](app/templates/project.html:41) и [dashboard.html:95](app/templates/dashboard.html:95) использует `summary::-webkit-details-marker { display: none }` без эквивалента для FF/Safari (`details > summary { list-style: none }` уже есть, но не везде).
- **[P3]** `<pre>{{ ... | tojson(indent=2) }}</pre>` в `<details>Сырая диагностика</details>` — норм (это collapsed diagnostic), но без `lang="en"` или syntax highlighting на больших payload утомляет.

## Patterns & Systemic Issues

1. **Diverging dialog vocabulary** — Assistant-UI имеет три формата модалок: `<md-dialog>` (через `assistantConfirm` в head), `.budget-transaction-dialog` (бюджет), `.settings-edit-dialog` (настройки). При product-register консистентность это affordance.
2. **Detail-страницы выпадают из shell** — work_item / goal / project / drift_watch созданы без app-shell, как будто они под другой layout. Похоже на legacy перед миграцией на Design-system (#648, #651, #654).
3. **Хардкод-цвета в Design-system asset** — Excel-палитра пробралась за границу системы. Если она нужна, это надо легитимировать как extra MD3-роль `--md-sys-color-budget-month-N`.
4. **Inline JS как первичный паттерн** — 800-строчные `<script>` в каждом шаблоне делают CSP nonce-only невозможным без переделки.

## Positive Findings

- **Полное соответствие MD3-bundle**: все интерактивные элементы — `<md-*>`, шрифт и иконки — local fonts через Design-system, никакого Tailwind/Bootstrap.
- **Хорошее использование SSE**: dashboard и overview подписываются на `task.status_changed`, `pm.files_changed` без polling.
- **Грамотная dark-mode логика** через `[data-theme]` + `prefers-color-scheme` без FOUC (благодаря [ds/assets/fouc.js](../../Design-system/assets/fouc.js)).
- **CSRF на месте**, `X-CSRF-Token` единый паттерн через `cookieValue("assistant_ui_csrf")`.
- **Keyboard shortcuts**: `/` фокусирует чат, `n` — следующая рекомендация, `c` — завершить задачу, `gg`/`gc` — навигация. Cockpit-friendly.
- **`<pre>` raw payload спрятан в `<details>`** — соблюдён frontend invariant из [AGENTS.md:93](AGENTS.md:93).
- **`role="tabpanel"` / `aria-controls` / `aria-labelledby`** корректно расставлены на `<md-tabs>` в бюджете и настройках.

## Recommended Actions

1. **[P0] `/impeccable harden`** — починить SSE-парсер в [console.html:64](app/templates/console.html:64) (`\\n\\n` → `\n\n`).
2. **[P1] `/impeccable harden`** — мигрировать `budget-transaction-dialog` и `settings-edit-dialog` на `<md-dialog>` с focus trap/restore; удалить кастомные классы из `Design-system/assets/assistant-ui.css` отдельным PR в Design-system.
3. **[P1] `/impeccable adapt`** — обернуть `work_item.html`, `goal.html`, `project.html`, `drift_watch.html` в стандартный `app-shell + _sidebar.html + mobile-bar`.
4. **[P1] `/impeccable colorize`** — перевести `--assistant-choice-*` на MD3-токены `*-container` или легитимировать в DESIGN_SYSTEM.md §8.2 (Design-system PR).
5. **[P1] `/impeccable typeset`** — поднять `.budget-journal-table` font до `label-small`, прижать колонки.
6. **[P1] `/impeccable harden`** — `aria-current="page"` в `_sidebar.html`; заменить `<md-outlined-text-field class="sr-only">` на `<input type="hidden">` в [login.html:10](app/templates/login.html:10).
7. **[P2] `/impeccable distill`** — убить дублирование filter-vocabulary (выбрать `<md-menu>` единым); убрать мёртвые mobile-bar ссылки; вынести `confirmationText` и большие inline-скрипты в shared assets.
8. **[P2] `/impeccable optimize`** — заменить `location.reload()` на точечный re-render + SSE-broadcast; добавить `beforeunload`-close для EventSource.
9. **[P3] `/impeccable polish`** — финальный проход после фиксов P0–P2.

> Запускать можно по одному, все сразу или в любом порядке. После фиксов перезапусти `/impeccable audit`, чтобы увидеть рост оценки.