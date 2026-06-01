# Impeccable Audit — Assistant-UI (AI-Assistant monorepo)

> **Статус плана:** ревизия после фидбека Codex. Прошлая версия рекомендовала `app/static/css/base.css` и сторонние SVG-наборы — это нарушает локальные правила. Текущая версия выровнена с [Design-system/DESIGN_SYSTEM.md](D:/GitHub/Design-system/DESIGN_SYSTEM.md), [assistant-ui/AGENTS.md](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md) и [impeccable product reference](D:/GitHub/_engineering_rules/skills/impeccable/reference/product.md).

## Context

Аудируем фронтенд подсистемы [assistant-ui](D:/GitHub/AI-Assistant/assistant-ui/) — FastAPI-сервис на 12 Jinja2-шаблонов: портфельный PM-дашборд, AI-консоль, обзоры, аудит-логи. Это **product**-register (внутренний инструмент для одного владельца портфеля).

Ключевое архитектурное обстоятельство, которое определяет всю стратегию: уже существует общий [Design-system](D:/GitHub/Design-system/) на MD3 + `@material/web` 2.3.0, разделяемый c Verua_automation, Stalking_offline, Best_photo_ai. По [assistant-ui/AGENTS.md:76-86](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:76) UI стили, MD3 tokens, typography и компоненты Assistant-UI обязаны идти через `../Design-system/`, а project-level `static/*.css` запрещены. По факту Assistant-UI этого **не делает** — весь визуал живёт inline в [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html) и [console.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html) с хардкодом hex-цветов, Arial-шрифта, emoji-иконок и `confirm()`/`prompt()`.

Поэтому аудит выявляет два пересекающихся слоя долга:

1. **Универсальные a11y / UX блокеры** (фокус, нативные `confirm`, контраст) — фиксятся независимо от Design-system.
2. **Несоответствие Design-system** (свой `:root`, свои тосты, эмодзи, custom CSS вместо `<md-*>`) — фиксится только через миграцию.

Решение, насколько глубоко идти, оставлено за пользователем (см. [Alignment Decision](#alignment-decision)). Базовый сценарий — **«причесать»** с привязкой к Design-system, не полный визуальный редизайн.

---

## Audit Health Score

| # | Dimension | Score | Key finding |
|---|-----------|-------|-------------|
| 1 | Accessibility | **1/4** | 0 правил `:focus-visible`; `confirm()`/`prompt()` ×5 как UX; `.muted` (#667085) на `--soft` ≈ 2.8:1 — WCAG AA fail |
| 2 | Performance | **3/4** | SSE используется правильно; layout-thrashing не найден; inline CSS дублируется в `console.html` |
| 3 | Theming | **1/4** | Игнорируется обязательная Design-system: свой `:root`, хардкод hex, нет MD3 токенов, тосты `#1f2937` мимо темы |
| 4 | Responsive | **2/4** | `min-width: 1120px` у `.dashboard-task-table` ломает mobile; кнопки `padding: 8px 10px` ≈ 28px (< 44×44 touch target) |
| 5 | Anti-patterns | **1/4** | Side-stripe `border-left` ×3 в [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html) (banned); Arial; emoji вместо Material Symbols; browser-native modals |
| **Total** | | **8/20** | **Poor — нужна структурная переработка через Design-system** |

---

## Anti-Patterns Verdict

**Выглядит ли это AI-generated?** Да. Главный системный tell — **полное игнорирование уже существующей Design-system** при наличии монорепозитория с готовыми токенами и компонентами. Это даёт второстепенные tells:

1. **Side-stripe borders — absolute ban нарушен трижды** в [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html): [:292](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:292), [:369](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:369), [:398](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:398).
2. **Generic font-stack** — `Arial, sans-serif` в [base.html:73](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:73) и [console.html:36](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html:36), хотя Design-system отдаёт Roboto Flex через `fonts/`.
3. **Browser-native UX** — `confirm()`/`prompt()` ×5 (см. P0 ниже). Из них **только один** ([dashboard.html:853](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:853)) — это явная process state change под локальным invariant [assistant-ui/AGENTS.md:73-74](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:73). Остальные — task completion, tool/workflow confirmation, native `prompt`. Все четыре всё равно подлежат замене на доступный UI, но по общей UX/a11y-причине, не по архитектурному invariant.
4. **Emoji как UI-иконки** — `☰ ⌕ ▶ ✓ ⏭ ▣` вместо обязательных Material Symbols Outlined ([DESIGN_SYSTEM.md:19](D:/GitHub/Design-system/DESIGN_SYSTEM.md:19)).
5. **Hero-metric template** — `.metric strong { font-size: 18px; display: block }` повторяется в 4+ местах однотипно. Лёгкий tell, не блокер.

**Чего НЕТ (это плюс):** нет gradient-text, нет glassmorphism, нет bounce/elastic easing.

**Identical card grids — НЕ проблема** для product register: [product.md:29](D:/GitHub/_engineering_rules/skills/impeccable/reference/product.md:29) явно требует "Predictable grids. Consistency IS an affordance". Прошлая версия плана ошибочно подсветила это как недостаток — теперь убрано.

---

## Executive Summary

- **Health: 8/20 (Poor)** — нужна миграция на Design-system, а не косметика.
- **Severity distribution:** P0 ×3, P1 ×7, P2 ×6, P3 ×4.
- **Top critical issues:**
  1. **[P0] Отсутствует `:focus-visible`** — WCAG 2.4.7 fail на всех страницах.
  2. **[P0] Browser-native `confirm/prompt`** — 5 мест. Один из них ([dashboard.html:853](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:853)) — это process state change и прямо нарушает [assistant-ui/AGENTS.md:73-74](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:73). Остальные четыре — task completion, tool/workflow confirmation, native `prompt` — общая UX/a11y-проблема, замена обязательна для всех.
  3. **[P0] `console.html` живёт мимо общей темы** — собственный `:root`, не extends `base.html`, два независимых набора токенов.
  4. **[P1] Игнорирование Design-system** — hex-цвета, Arial, emoji, custom CSS вместо `<md-*>` — все нарушают `MUST/MUST NOT` из [DESIGN_SYSTEM.md TL;DR](D:/GitHub/Design-system/DESIGN_SYSTEM.md:11).
  5. **[P1] Side-stripe borders ×3** — absolute ban impeccable, относительно безопасный визуальный долг, но требует замены.

---

## Detailed Findings by Severity

### P0 — Blocking

**[P0] Отсутствует `:focus-visible` нигде в проекте**
- Location: весь [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html), [console.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html) (grep `:focus-visible|:focus\b` → 0 matches)
- Category: Accessibility (WCAG 2.4.7 Level AA)
- Impact: Keyboard-юзер не видит фокус. Блокирует accessibility certification и базовую usability через клавиатуру.
- Recommendation: Глобальное правило в Design-system layer `:focus-visible { outline: 2px solid var(--md-sys-color-primary); outline-offset: 2px }`. Если миграция откладывается — добавить временно в [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html) перед таблицей правил (single rule, не нарушает запрет на новые CSS-файлы).
- Acceptance: keyboard Tab по dashboard/overview/console/login делает фокус видимым на каждом интерактивном элементе.

**[P0] Browser-native `confirm/prompt` в продуктовых флоу**
- Location:
  - [dashboard.html:768](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:768) `confirm("Завершить задачу?")` — task completion (общая UX-проблема)
  - [dashboard.html:853](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:853) `confirm("Перевести процесс в состояние ${state}?")` — **единственный явный process state change, нарушает [AGENTS.md:73-74](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:73)**
  - [dashboard.html:1109](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:1109) `confirm(confirmationText(data))` — tool/workflow confirmation в общем `handleConfirmation`
  - [console.html:272](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html:272) `confirm(confirmationText(data))` — то же `handleConfirmation` со стороны console
  - [ideas.html:128](D:/GitHub/AI-Assistant/assistant-ui/app/templates/ideas.html:128) `prompt("project_path для задачи")` — text input
- Category: Anti-pattern + Accessibility (+ один пункт — Architecture invariant)
- Impact: Блокирующие native dialogs не стилизуются, плохо озвучиваются screen-reader'ами, выпадают из темы. Конкретно [dashboard.html:853](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:853) дополнительно нарушает архитектурный invariant про process state changes.
- Recommendation:
  - **Все четыре `confirm()`** → `<md-dialog>` из Design-system. По [product.md:35](D:/GitHub/_engineering_rules/skills/impeccable/reference/product.md:35) компонент должен иметь все состояния (default/hover/focus/active/disabled/loading/error). Для [dashboard.html:853](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:853) — особое внимание к описанию перехода from/to.
  - **`prompt()` в ideas.html:128** → inline form в той же странице (`<md-outlined-text-field>` в существующей форме). Native prompt здесь не нужен.
- Acceptance: `grep -E "confirm\(|prompt\(|alert\(" assistant-ui/app/templates/` → 0 matches.

**[P0] `console.html` имеет параллельный `:root` и не extends `base.html`**
- Location: [console.html:1-31](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html:1) (собственный doctype, собственные переменные)
- Category: Theming
- Impact: Две независимые цветовые системы. `--tool: #f2f6ff` существует только тут. Изменение токенов в `base.html` не доходит до консоли. На длительной перспективе — постоянный source багов.
- Recommendation: Переписать как `{% extends "base.html" %}` (этап «причесать»). При миграции на Design-system — extends `../Design-system/templates/base.html` с подключённым через `StaticFiles` `/ds/` ([DESIGN_SYSTEM.md:96-129](D:/GitHub/Design-system/DESIGN_SYSTEM.md:96)).
- Acceptance: `console.html` начинается с `{% extends %}`, своего `<html>`/`<head>`/`:root` не содержит.

### P1 — Major

**[P1] Несоответствие Design-system по обязательным MUST/MUST NOT**
- Location: весь [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html), [console.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html), все `*.html`
- Category: Theming / Architecture
- Нарушения относительно [DESIGN_SYSTEM.md TL;DR](D:/GitHub/Design-system/DESIGN_SYSTEM.md:11):
  - Hardcode hex/rgb во всех `:root` и в `.toast`, `.project-count`, `button` — `MUST NOT hardcode`.
  - Custom CSS для button/input/select — `MUST NOT write custom CSS, use <md-*>`.
  - Не подключён `@material/web` bundle.
  - Не подключены MD3 tokens, нет `--md-sys-color-*`/`--md-sys-typescale-*`/`--md-sys-shape-corner-*`.
  - Шрифт Arial вместо Roboto Flex.
- Impact: Долгосрочно не масштабируется, нельзя автоматически сменить тему, нельзя обновить визуал централизованно вместе с тремя другими приложениями.
- Recommendation: см. [Alignment Decision](#alignment-decision) — либо «минимальный a11y fix сейчас, миграция позже отдельной задачей», либо полная миграция как часть этой работы.
- Acceptance (для миграционной ветки): hex-литералов в `assistant-ui/app/templates/**/*.html` не остаётся (grep `#[0-9a-fA-F]{3,6}` ≈ 0 за исключением комментариев), `font-family: Arial` отсутствует, все интерактивные контролы — `<md-*>`.

**[P1] Side-stripe borders на карточках**
- Location: [base.html:292](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:292), [base.html:369](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:369), [base.html:398](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:398)
- Category: Anti-pattern (impeccable absolute ban)
- Impact: Визуальный долг, не блокер runtime. Понижено с P0 → P1 по фидбеку.
- Recommendation: Заменить на `<md-elevated-card>`/`<md-outlined-card>` с tonal background или leading-индикатором (иконка статуса). При минимальной правке — полный outline `border: 1px solid var(--line)` + tinted background по project_color.
- Acceptance: `border-left: [2-9]px` в шаблонах = 0.

**[P1] Контраст `.muted` на `--soft`**
- Location: [base.html:24](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:24) (light: #667085), используется в `.meta`, `.muted`, `.project-stats span`, `.pill`
- Category: Accessibility (WCAG 1.4.3 AA)
- Impact: Контраст ~2.8:1, метаданные нечитаемы.
- Recommendation: При миграции на MD3 — заменить на `var(--md-sys-color-on-surface-variant)` (даёт WCAG AA). Без миграции — поднять `--muted` до `#4b5563` в светлой теме.
- Acceptance: axe DevTools или Lighthouse → 0 color-contrast violations на ключевых страницах.

**[P1] Process state change без accessible confirmation**
- Location: [dashboard.html:853](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:853) (и связанные)
- Category: Architecture invariant (`assistant-ui/AGENTS.md:73-74`)
- Impact: Локальное правило прямо требует доступной browser UI для confirmation; `confirm()` его не удовлетворяет.
- Recommendation: `<md-dialog>` с описанием перехода (from/to состояние), primary `<md-filled-button>`, secondary `<md-text-button>`.
- Acceptance: переход состояния процесса в UI вызывает `<md-dialog>` с правильным `aria-labelledby`, доступен через клавиатуру.

**[P1] Touch targets ниже 44×44 px**
- Location: [base.html:82-100](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:82) — `button { padding: 8px 10px }` ≈ 28×28; `.pill { padding: 4px 7px }`; `.nav a { min-height: 36px }`
- Category: Responsive / Accessibility (WCAG 2.5.5 AAA, 2.5.8 AA)
- Impact: Промахи по тапу на mobile.
- Recommendation: `<md-*>` контролы уже отвечают MD3 минимуму. Для остатков (nav-ссылки, pill'ы) — `min-height: 40px` desktop, `44px` mobile.
- Acceptance: Lighthouse mobile audit «Tap targets» = pass.

**[P1] `<details>` использован как dropdown без правильного ARIA**
- Location: [base.html:546](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:546) `.status-filter`
- Category: Accessibility
- Impact: Screen-reader озвучивает как «details», пользователь видит menu.
- Recommendation: При миграции → `<md-menu>` + `<md-menu-item>`. Без миграции → `aria-label` на `<summary>`, `role="menu"` на `.status-filter-menu`, `role="menuitemcheckbox"` на пунктах.
- Acceptance: screen-reader walkthrough называет элемент «menu» / «filter».

**[P1] `bubble` line-height = 1**
- Location: [base.html:648](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:648), аналогично в [console.html:83](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html:83)
- Category: Typography / Readability
- Impact: Диакритика режется, чат-сообщения визуально перегружены.
- Recommendation: `line-height: 1.4` минимум, либо `var(--md-sys-typescale-body-medium-line-height)`.
- Acceptance: визуальная проверка чата.

### P2 — Minor

**[P2] Hero-metric template повторяется 4+ раз**
- Location: [base.html:237-246](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:237) + `overview/dashboard/drift_watch/goal`
- Category: Anti-pattern, но в product register слабый
- Recommendation: Не править ради разнообразия (предсказуемые grids — фича product). Заменить только там, где данные не «метрика»: progress-bar / sparkline / inline-цифра в фразе.

**[P2] Emoji как UI-иконки**
- Location: повсеместно — `☰`, `⌕`, `▶`, `✓`, `⏭`, `▣`, `▤`
- Category: Anti-pattern / a11y
- Recommendation: Material Symbols Outlined через `<span class="material-symbols-outlined">play_arrow</span>` ([DESIGN_SYSTEM.md:19, 276-302](D:/GitHub/Design-system/DESIGN_SYSTEM.md:19)). НЕ lucide/phosphor/heroicons — это нарушит standards.
- Acceptance: emoji-глифов в `<button>`/`<a>` шаблонов = 0.

**[P2] EventSource без явного cleanup**
- Location: [dashboard.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html), [overview.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/overview.html)
- Category: Performance
- Recommendation: Хранить ссылку, закрывать на `pagehide`. Сейчас MPA, full reload — утечки не критичны, но паттерн стоит исправить.

**[P2] Animations с default `ease`**
- Location: [base.html:747](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:747) `transition: transform .18s ease`
- Recommendation: При миграции — `var(--md-sys-motion-easing-emphasized)` + `var(--md-sys-motion-duration-medium2)`. Без миграции — оставить как есть, это не блокер.

**[P2] `border-radius: 0` де-факто на всём**
- Location: повсеместно (`.panel`, `.metric`, `.pill`, `.item`, `.kanban-column`, button)
- Recommendation: При миграции — `var(--md-sys-shape-corner-small/medium)`. Без миграции — оставить, единственным «выпадающим» останется `.bubble` (хватит для контекста чата).

**[P2] Inline CSS дублирован**
- Location: [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html) (~770 lines), [console.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html) (~150 lines)
- Category: Performance / Maintenance
- ВНИМАНИЕ: `MUST NOT create new project-level CSS files` ([DESIGN_SYSTEM.md:27-28](D:/GitHub/Design-system/DESIGN_SYSTEM.md:27)). **Нельзя выносить в `app/static/css/base.css`** — единственно правильный путь это миграция на Design-system base.html и общие assets. До миграции inline остаётся как есть, чтобы не плодить технический долг.

### P3 — Polish

- **[P3] `dashboard.html` 50KB / 1041 строка** — кандидат на split через Jinja `{% include %}` partials. JS — в `app/static/js/` допустим (правило про CSS, не JS), но согласовать с архитектурой.
- **[P3] `.data-table` имеет `min-width: 1120px`** ([base.html:494](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:494)) — на mobile горизонтальный скролл (есть `.table-wrap { overflow-x: auto }`, смягчает). Долгосрочно — `<md-list>` карточный режим для mobile breakpoint.
- **[P3] Sidebar `width: 224px` фиксирован** — `clamp(200px, 18vw, 260px)` ровнее на больших экранах. Не нарушает [AGENTS.md:96-98](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:96), т.к. это compact control, не main content.
- **[P3] Placeholder-цвета не настроены** — браузерный default обычно низкоконтрастный.

---

## Systemic Patterns

1. **Architectural drift от Design-system**: Assistant-UI единственный в портфеле, кто не использует общий MD3-stack. Корень почти всех остальных находок.
2. **Theming-расхождение внутри Assistant-UI**: `console.html` живёт отдельно от `base.html`.
3. **Browser-native UX**: 5 мест с `confirm/prompt` в трёх файлах. Системная привычка обходить custom UI.
4. **Side-stripe `border-left`**: повторён ×3 в трёх компонентах.
5. **Emoji-иконки**: ≥6 разных глифов как кнопки.
6. **`.muted`-контраст**: одна переменная пробивает WCAG AA повсюду.
7. **Отсутствие focus-state**: системное упущение.

---

## Positive Findings

- **Семантичный HTML**: используются `<button>`, `<table>`, `<details>`, `<form>`, `<aside>`, `<main>`.
- **SSE-обновления** заменяют polling — соответствует [assistant-ui/AGENTS.md:55-57](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:55).
- **`aria-live="polite"`** на чат-логах ([console.html:163](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html:163)).
- **`aria-expanded` синхронизирован** для sidebar toggle ([base.html:820](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html:820)).
- **`viewport`-meta + breakpoints 980/620px** на месте, mobile drawer работает.
- **Нет тяжёлых absolute-ban'ов**: ни gradient-text, ни glassmorphism, ни bounce-easing.
- **`overflow-wrap: anywhere`** в табличных ячейках — правильно для длинных значений.
- **Главное содержимое не ограничено `max-width`** — соответствует [AGENTS.md:96-98](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:96).

---

## Alignment Decision

Перед фазами кода зафиксировать развилку. Это обязательный шаг — без решения дальше можно случайно увеличить долг.

| Опция | Когда выбрать | Что включает |
|-------|---------------|--------------|
| **A. Минимальный a11y fix + DS bootstrap** | Если бюджет ограничен и полная миграция вне scope этого квартала | P0: `:focus-visible`, замена `confirm/prompt` через `<md-dialog>`, консолидация `console.html` с локальным `base.html`. Design-system подключается минимально (mount `/ds`, common.css, **`theme.assistant.css` — обязательно, см. Phase 2.5**, `material-web.js`), но остальные шаблоны не мигрируются. Side-stripe оставить. Полная миграция — отдельная задача в backlog. |
| **B. Полная миграция на Design-system** | Если есть бюджет и приоритет — выровнять Assistant-UI с тремя другими проектами | Полный план ниже. Включает A. |

Решение оформляется как PM-MCP work item в `assistant-ui`. Обе ветки требуют **парной задачи в проекте Design-system** для создания `theme.assistant.css` (Phase 2.5). Ветка B добавляет ещё одну DS-задачу, если меняем `assets/app-shell.css` (шаг 13).

---

## Recommended Actions

### Phase 0 — Alignment (обязательно)

1. **PM-MCP work item** в `assistant-ui` с заголовком «Impeccable audit follow-up». Привязать к этому плану. Описание = ссылка на этот файл + выбранная опция (A или B).
2. **PRODUCT.md** в `assistant-ui/docs/` (~1 страница): один владелец портфеля, single-user внутренний инструмент, anti-references (стандартные SaaS-дашборды teal-blue), tone деловой.
3. **DESIGN.md** в `assistant-ui/docs/` (≤1 страница): pointer на [Design-system/DESIGN_SYSTEM.md](D:/GitHub/Design-system/DESIGN_SYSTEM.md) как источник токенов и компонентов. **Не дублировать** правила, не канонизировать текущий inline CSS.
4. **PM-MCP задача в `Design-system`** для создания `theme.assistant.css` (Phase 2.5) — нужна для обеих веток. Для ветки B — дополнительная задача в Design-system для правок `assets/app-shell.css` (шаг 13) по [DESIGN_SYSTEM.md:24-28](D:/GitHub/Design-system/DESIGN_SYSTEM.md:24).

### Phase 1 — P0 a11y и UX (ветка A и B одинаково)

5. **`:focus-visible` глобально**: одно правило с `outline: 2px solid var(--md-sys-color-primary, var(--accent)); outline-offset: 2px` на `:where(button, a, input, select, textarea, summary, [role="button"], [tabindex])`. Файлы: [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html), [console.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html).
6. **Минимальное подключение Design-system (требуется даже для ветки A)**: даже если миграция откладывается, для рабочего `<md-dialog>` в теме обязательно нужны:
   - `app.mount("/ds", StaticFiles(directory=<computed_path>))` — путь вычисляется как `Path(__file__).resolve().parents[3] / "Design-system"` от `assistant-ui/app/main.py` (parents[0]=`app`, [1]=`assistant-ui`, [2]=`AI-Assistant`, [3]=`D:\GitHub` → `D:\GitHub\Design-system`). Env-override через `ASSISTANT_UI_DS_PATH`. Если путь не существует — fail fast при старте.
   - `<script src="/ds/assets/fouc.js"></script>` (synchronous).
   - `<link rel="stylesheet" href="/ds/tokens/common.css">` и `<link rel="stylesheet" href="/ds/tokens/theme.assistant.css">`. Тема `theme.assistant.css` — prerequisite даже для ветки A (см. Phase 2.5). Использовать чужую тему (`theme.verua.css` и т.п.) как fallback нельзя: это создаст cross-brand зависимость и визуально неверный продукт.
   - `<script type="module" src="/ds/vendor/material-web.js"></script>` (defer).
   Подключить в `<head>` локального `base.html` и `console.html` минимально, не убирая остальные стили. Без этого `<md-dialog>` рендерится без темы.
7. **Заменить `confirm()` на `<md-dialog>`**: [dashboard.html:768, 853, 1109](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:768), [console.html:272](D:/GitHub/AI-Assistant/assistant-ui/app/templates/console.html:272). На [dashboard.html:853](D:/GitHub/AI-Assistant/assistant-ui/app/templates/dashboard.html:853) — особое внимание (process state change, AGENTS invariant).
8. **Заменить `prompt()`**: [ideas.html:128](D:/GitHub/AI-Assistant/assistant-ui/app/templates/ideas.html:128) — inline `<md-outlined-text-field>` в существующей форме.
9. **`console.html` extends локальный `base.html`** (в ветке B — extends Design-system base позже, см. Phase 2). Убрать собственный `:root`.

### Phase 2 — Design-system миграция (только ветка B)

10. **Подключить Design-system** в `assistant-ui/app/main.py`:
    - Путь не hardcoded. Использовать `DS_PATH = Path(os.environ.get("ASSISTANT_UI_DS_PATH") or (Path(__file__).resolve().parents[3] / "Design-system"))`. Проверять существование, fail fast если каталога нет.
    - `app.mount("/ds", StaticFiles(directory=str(DS_PATH)), name="ds")`.
    - **Jinja loader через `PrefixLoader`, не `ChoiceLoader`**, чтобы избежать саморекурсии при extends "base.html" — у локального и DS-base одинаковое имя:
      ```python
      from jinja2 import PrefixLoader, FileSystemLoader, ChoiceLoader
      templates.env.loader = ChoiceLoader([
          FileSystemLoader("app/templates"),
          PrefixLoader({"ds": FileSystemLoader(str(DS_PATH / "templates"))}),
      ])
      ```
      В шаблонах: `{% extends "ds/base.html" %}`.
    - Глобальные контекст-переменные `ds_static="/ds"`, `ds_theme="theme.assistant.css"`, `app_title="Assistant UI"`, `lang="ru"` через FastAPI dependency или middleware.
11. **Создать тему `theme.assistant.css`** — отдельный PR в Design-system (см. шаг 9.5 ниже про спецификацию).
12. **Мигрировать блоки в шаблонах**: текущий локальный `base.html` определяет `{% block body %}`, а DS-base ожидает `{% block content %}` ([DESIGN_SYSTEM.md:196-205](D:/GitHub/Design-system/DESIGN_SYSTEM.md:196)). Все 11 страниц-наследников нужно переключить на `{% extends "ds/base.html" %}` + `{% block content %}`. Сидибар-логика и app-shell не входят в DS base — это поднимает шаг 13.
13. **Sidebar/app-shell**: текущий `.ds-app-main { max-width: 1200px }` в [Design-system/assets/app-shell.css](D:/GitHub/Design-system/assets/app-shell.css) нарушает [assistant-ui/AGENTS.md:96-98](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:96). Два варианта (выбрать в PR Design-system, согласовать с владельцами Verua/Stalking/BestPhoto):
    - **(a) CSS-переменная `--ds-app-main-max-width`** с default `1200px`, override `none` на уровне `assistant-ui` через локальную обёртку `<main class="ds-app-main ds-app-main--fluid">` или inline `style="--ds-app-main-max-width: none"`.
    - **(b) Отдельный portfolio-shell** в Design-system: `<main class="ds-app-portfolio-main">` без max-width + опциональный sidebar slot.
    Решение оформляется issue/PR в Design-system **до** миграции base.html в assistant-ui.
14. **Перевести интерактивные элементы на `<md-*>`** по [Component Map §5](D:/GitHub/Design-system/DESIGN_SYSTEM.md:209). По [product.md:35](D:/GitHub/_engineering_rules/skills/impeccable/reference/product.md:35) каждый компонент должен иметь полный набор состояний.
15. **Заменить emoji на Material Symbols Outlined** ([DESIGN_SYSTEM.md:276-302](D:/GitHub/Design-system/DESIGN_SYSTEM.md:276)). Маппинг: ☰ → `menu`, ⌕ → `search`, ▶ → `play_arrow`, ✓ → `check`, ⏭ → `skip_next`, ▣ → `dock_to_right`/`view_sidebar`.
16. **Тосты через MD3 snackbar** ([DESIGN_SYSTEM.md §7](D:/GitHub/Design-system/DESIGN_SYSTEM.md:342)) с inverse-surface токенами.

### Phase 2.5 — Спецификация `theme.assistant.css` (PR в Design-system, **prerequisite обеих веток**)

Отдельный PR/issue в проекте Design-system. И ветка A, и ветка B блокируются на этом — без темы `<md-dialog>` не будет в правильных цветах:

9.5.1. **Source color** — выбрать. Кандидаты:
   - текущий `--accent: #0f766e` (teal) → ближайший material source `#0F766E` или округлить до `#0E7C70`. Сохранит continuity текущего UI.
   - нейтральный (как Best_photo_ai) — если хочется отстройки от Verua и подчеркнуть «инструмент, не приложение».
   Решение фиксируется в PM-MCP задаче Design-system.

9.5.2. **Обновить `THEMES`** в [Design-system/scripts/generate_theme.py:45-49](D:/GitHub/Design-system/scripts/generate_theme.py:45): добавить запись формата существующих, **обязательно с полем `out`**:
```python
"assistant": {"name": "Assistant_UI", "source": "<HEX>", "out": "tokens/theme.assistant.css"},
```
Поле `mode` опционально (только если нужен спец-режим вроде `"gray"` у bestphoto).

9.5.3. **Сгенерировать** `tokens/theme.assistant.css` только через documented env Design-system:
```powershell
cd D:\GitHub\Design-system
uv sync
uv run python scripts/generate_theme.py
```
Зависимость `material-color-utilities-python` описана в `pyproject.toml`; команды выше являются единственным поддерживаемым способом генерации.

9.5.4. **Обновить документацию** [DESIGN_SYSTEM.md §2](D:/GitHub/Design-system/DESIGN_SYSTEM.md:67) — добавить строку в таблицу «Брендовые цвета», обновить header (если меняется бренд-набор).

9.5.5. **Commit + PR в Design-system**, не как часть assistant-ui PR ([DESIGN_SYSTEM.md:24](D:/GitHub/Design-system/DESIGN_SYSTEM.md:24): token changes — separate PR).

### Phase 3 — Дочистка (обе ветки, в конце)

17. **Убрать side-stripe borders ×3** в [base.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/base.html). При миграции — `<md-elevated-card>` с tonal-fill или leading-индикатор.
18. **Контраст `--muted`/`on-surface-variant`** до AA.
19. **Touch-targets** ≥ 40px desktop, 44px mobile для остаточных custom-элементов.
20. **`.bubble` line-height** 1 → 1.4.
21. **`prompt|confirm|alert` grep-gate** — добавить в CI или pre-commit hook (`grep -rE '\bconfirm\(|\bprompt\(|\balert\(' assistant-ui/app/templates && exit 1`).

### Phase 4 — Verification и audit-повтор

22. Прогон verification из секции ниже.
23. **`/impeccable audit`** повтор — целевой Health Score ≥ 16/20 (ветка B), ≥ 12/20 (ветка A).

---

## Verification

Обязательный набор (по [assistant-ui/AGENTS.md:100-110](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:100) + расширение):

1. **Backend регресс**:
   - `cd assistant-ui ; uv sync` (если менялись зависимости).
   - `cd assistant-ui ; uv run ruff check .` → 0 errors.
   - `cd assistant-ui ; uv run pytest` → all green.
2. **SSE live-updates**: `curl http://127.0.0.1:8000/api/dashboard/events?once=1` отвечает событием.
3. **Запуск UI**: `cd assistant-ui ; uv run assistant-ui serve --reload`, открыть `http://127.0.0.1:8000/`.
4. **Smoke walkthrough** ключевых страниц (через in-app Browser tool / Claude Preview):
   - `/overview`, `/dashboard`, `/console`, `/project/{any}`, `/work-item/{any}`, `/goal/{any}`, `/ideas`, `/audit`, `/reports/weekly`, `/login`.
   - Каждая страница: light + dark тема (через `prefers-color-scheme` и `data-theme`).
   - Desktop (1440px) и mobile (390px) viewport.
5. **Focus navigation**: Tab через каждую страницу, на каждом интерактиве виден focus ring.
6. **Confirmations**: process state change на dashboard → `<md-dialog>`, не браузерный alert. Tool execution в console → то же.
7. **Greps**:
   - `grep -rE '\bconfirm\(|\bprompt\(|\balert\(' assistant-ui/app/templates/` → 0 matches.
   - `grep -rE 'border-left:\s*[2-9]px' assistant-ui/app/templates/` → 0 matches.
   - Для ветки B: `grep -rE '#[0-9a-fA-F]{3,6}' assistant-ui/app/templates/` → только в комментариях или 0.
8. **A11y**:
   - axe DevTools / Lighthouse Accessibility → 0 contrast violations, 0 missing focus indicators.
   - Lighthouse mobile «Tap targets» → pass.
9. **Design-system smoke (только ветка B)**:
   - Проверить, что Verua_automation / Stalking_offline / Best_photo_ai не сломались после изменений в `Design-system/assets/app-shell.css` (если внесли правки по шагу 13). Открыть по типовой странице каждого, убедиться в отсутствии регрессии layout/themes.
   - Проверить, что `theme.assistant.css` действительно подключается и `<md-filled-button>` рендерится в правильном цвете на dashboard.
10. **`/impeccable audit` повтор** для финальной оценки.

Если какая-либо проверка не выполнима в текущем окружении (например, нет axe) — явно отметить в отчёте, по правилу [AGENTS.md:110](D:/GitHub/AI-Assistant/assistant-ui/AGENTS.md:110).
