# Audit — Verua_automation frontend

**Контекст:** FastAPI/Jinja2 медицинский планировщик, использует shared MD3 Design-system (`../Design-system/`). Аудит проведён против правил `DESIGN_SYSTEM.md` + `AGENTS.md F/G` + универсальных design laws.

## Audit Health Score

| # | Dimension | Score | Key Finding |
|---|-----------|-------|-------------|
| 1 | Accessibility | 2/4 | Нативные `<input>`/`<button>` без `aria-label`, inline `onclick`, touch targets 24–38px |
| 2 | Performance | 2/4 | 6 `backdrop-filter: blur()`, 28 градиентов, `repeating-linear-gradient` на каждой ячейке месяца |
| 3 | Responsive Design | 2/4 | Брейкпойнты есть, но `.day=34×34px`, `.color-swatch=24×24px`, `.tool=30px` < WCAG 2.5.5 (44×44) |
| 4 | Theming | 1/4 | 174 hardcoded hex, 31 `!important`, 7 параллельных систем переменных поверх MD3 |
| 5 | Anti-Patterns | 1/4 | Glassmorphism как default, hardcoded `"Segoe UI"`, gradient soup, нативные элементы вместо `<md-*>` |
| **Total** | | **8/20** | **Poor (major overhaul)** |

## Anti-Patterns Verdict — FAIL

Интерфейс **визуально не выглядит "AI-сгенерированным"**, но **архитектурно** нарушает все MUST NOT из `DESIGN_SYSTEM.md`. Конкретные tells:

1. **Glassmorphism как default** — `backdrop-filter: blur(8-10px)` на `top-nav`, `top-nav-exit`, `sticky-shell`, `create-actions-sticky` (6 случаев). Прямой запрет shared laws.
2. **Generic font hardcoded** — `font-family: "Segoe UI", Tahoma, sans-serif` в `app-shell.css:38` и `planner-shared.css:5` вместо `var(--md-sys-typescale-body-medium-font)`. Design-system поставляет Roboto Flex локально — он не используется.
3. **Gradient soup** — `body` имеет двухслойный фон (radial + linear), `summary`/`message`/`btn.secondary`/`patient-section-*`/`sheet-weekday`/`calendar-summary` каждый со своим градиентом. Это не «richness», это шум.
4. **Hex chaos поверх MD3** — параллельная палитра `--verua-*` × 20+ переменных дублирует роли, уже определённые в `tokens/theme.verua.css`.

## Executive Summary

- **Health Score: 8/20 (Poor)** — фронтенд работает, но накопил систематический tech debt относительно дизайн-системы.
- **Issues by severity:** P0 = 2, P1 = 6, P2 = 4, P3 = 3.
- **Корневая причина:** AGENTS.md F.b явно говорит «`static/*.css` — tech debt, новые правила запрещены», но эти файлы продолжают расти. Никакой новой работы тут быть не должно — нужна миграция в Design-system.
- **Top issues:**
  1. Нативные `<button>/<input>/<select>/<textarea>` (≈143 случая) вместо `<md-*>` web-components.
  2. 174 hardcoded hex в `static/*.css` поверх готовых MD3 токенов.
  3. Glassmorphism (`backdrop-filter`) — прямой ban shared laws.
  4. Touch targets ниже WCAG (`day-button`, `color-swatch`, `tool`).
  5. Inline `<script>` + `onclick` в `top_nav.html` ломает strict-CSP план.

## Detailed Findings by Severity

### P0 Blocking

**[P0] Нативные HTML-контролы вместо `<md-*>` web-components**
- **Location:** `templates/index.html:15-20,47,62-74`, `templates/patient_create.html` (61 input/select/textarea), `templates/calendar.html`, `templates/system.html`, `partials/planner_*.html` — суммарно ≈110 нативных input/select/textarea + 33 native `<button>`/`class="btn"`.
- **Category:** Anti-Pattern (DS violation).
- **Impact:** Нарушает `DESIGN_SYSTEM.md` MUST §TL;DR. Внешний вид кастомизирован через CSS, который теперь дублирует токены, тёмную тему, focus-ring, hover — всё это `@material/web` уже даёт «из коробки». Каждый правлемый компонент = новый источник расхождения с тремя другими портфельными приложениями.
- **Standard:** Design-system §5 Component Map.
- **Recommendation:** Заменить пациент-`<select>` → `<md-outlined-select>`, тариф-`<input>` → `<md-outlined-text-field>`, `<button class="btn">` → `<md-filled-button>`, экспорт-`<button>` уже корректен в `index.html:100`. Удалить custom CSS, который их кастомизировал.
- **Suggested command:** `/impeccable migration-discipline` (тоже как процесс) или ручной план — это объёмная задача, разбить по шаблонам.

**[P0] Inline `<script>` и `onclick=` в `top_nav.html`**
- **Location:** `templates/partials/top_nav.html:10-18`.
- **Category:** Accessibility + Security (CSP).
- **Impact:** `<button onclick="exitApp(this)">` + inline `<script>` блокируют переход на strict-CSP (Design-system §1 `assets/fouc.js` явно «strict-CSP-compatible, без inline»). Также `<button>` без `aria-label` и без `type="button"` (subform submit risk если когда-то окажется внутри `<form>`).
- **Standard:** WCAG 4.1.2, CSP Level 3.
- **Recommendation:** Вынести в `static/top-nav.js`, заменить на `<md-text-button id="exit-app">Выход</md-text-button>` + `addEventListener`. По правилам F.b — этот JS-файл существует на стороне проекта, но самое правильное переместить в `../Design-system/assets/`.
- **Suggested command:** `/impeccable harden`.

### P1 Major

**[P1] Glassmorphism (`backdrop-filter: blur`) в 6 местах**
- **Location:** `static/app-shell.css:96,237`, `static/planner-shared.css:28`, `static/catalog-pages.css:777`.
- **Category:** Anti-Pattern (absolute ban) + Performance.
- **Impact:** Прямое нарушение «Glassmorphism as default» из shared design laws. На медицинском интерфейсе блюр под навигацией — пустая декорация. Также дорогой композитный слой на каждый scroll-tick (`sticky-shell` с блюром).
- **Recommendation:** Заменить полупрозрачные фоны с `blur(8-10px)` на solid `var(--md-sys-color-surface)` или `surface-container-low`.
- **Suggested command:** `/impeccable distill`.

**[P1] Hardcoded `font-family: "Segoe UI"` поверх Roboto Flex**
- **Location:** `static/app-shell.css:38`, `static/planner-shared.css:5`.
- **Category:** Theming + Anti-Pattern.
- **Impact:** Design-system загружает Roboto Flex (4 woff2, latin/cyrillic/ext) в `tokens/common.css` через `@font-face`. Жёстко-заданный Segoe UI отменяет это для **всего body**. Брендовая типографика не применяется ни в одной странице.
- **Standard:** `DESIGN_SYSTEM.md` §6 «использовать CSS-переменные `--md-sys-typescale-*`».
- **Recommendation:** Удалить `font-family` из `body` (Design-system base.html уже выставляет правильное значение через `common.css`). Если приложение хочет переопределить — через тoken-роли, не строкой.
- **Suggested command:** `/impeccable typeset`.

**[P1] `!important` спам, особенно в `app-shell.css`**
- **Location:** `static/app-shell.css` 19 случаев; `planner-form2.css` 8; `planner-shared.css` 1; `catalog-pages.css` 2; `calendar.css` 1.
- **Category:** Theming.
- **Impact:** В `app-shell.css:122-167` `!important` использован чтобы переопределить «что-то снизу» (вероятно legacy `material-verua.css`). Это явно противоречит правилу G.4 «`!important` в `app-shell.css` запрещён без inline-комментария `/* needs !important because ... */`», ни одного такого комментария в файле нет → `verify.ps1` должен это ловить (проверить).
- **Recommendation:** Каждый `!important` либо удалить, либо снабдить inline-комментарием с причиной. Лучшее решение — миграция блока в `../Design-system/assets/app-shell.css`.
- **Suggested command:** `/impeccable harden`.

**[P1] 174 hardcoded hex вне токенов**
- **Location:** все 6 `static/*.css`. Худшие места: `planner-form2.css:159-164` (палитра цветов редактора), `catalog-pages.css:793-820` (фоны patient-section-*), `calendar.css:174,194,225` (sheet bg, weekend color), `planner-shared.css:233` (calendar-card gradient).
- **Category:** Theming.
- **Impact:** `DESIGN_SYSTEM.md` TL;DR: «MUST NOT hardcode hex / rgb / hsl colors anywhere in app code». Тёмная тема не работает корректно: например, `.color-swatch[data-swatch="black"] { background: #000000 }` в dark mode = пятно на тёмном фоне.
- **Recommendation:** Перевести каждый hex на роль (`surface`, `outline-variant`, `primary-container`, `error`...). Палитру цветов редактора оставить hex, но через CSS custom property с пометкой «editor-only».
- **Suggested command:** `/impeccable colorize` или ручная миграция.

**[P1] Touch targets ниже WCAG 2.5.5 (44×44)**
- **Location:** `planner-shared.css:281` `.day {34×34}`, `planner-form2.css:153-155` `.color-swatch {24×24}`, `planner-shared.css:449-452` `.tool {min-width:30px; padding:4px 6px}`, `calendar.css:67-78` `.month-arrow {38×38}`, `app-shell.css:194` `.tariff-row input {38px}`.
- **Category:** Accessibility + Responsive.
- **Impact:** На планшете/touch-устройстве — типичный сценарий для медицинской практики — попасть пальцем в день календаря 34×34px тяжело. WCAG 2.5.5 (AA) рекомендует 44×44, 2.5.8 (AAA) — 24×24 минимум.
- **Recommendation:** Поднять до 44×44 либо обернуть `<md-icon-button>` (он сам соответствует). Календарь — отдельный кейс: использовать `<md-list-item>` с `density="-2"`, но не меньше 40px высота.
- **Suggested command:** `/impeccable adapt`.

**[P1] Слабое покрытие `aria-*`/labels**
- **Location:** Только 22 a11y-атрибута на 14 шаблонов и 110 input/select/textarea. Например, `templates/index.html:62-74` — тариф-инпуты только с `name=`, нет связанного `<label for=>`.
- **Category:** Accessibility.
- **Impact:** Screen reader проговорит «edit text» без контекста. Сейчас часть имён в `<label>` macro обёртках вроде `tariff-row span` — но `<span>` не label.
- **Standard:** WCAG 4.1.2, 3.3.2.
- **Recommendation:** В `tariff-row` использовать `<label for="...">` или, после P0-миграции, `<md-outlined-text-field label="a">` — Material компонент сам создаёт правильный label.
- **Suggested command:** `/impeccable harden`.

### P2 Minor

**[P2] Дублирующая система переменных (`--verua-*`) поверх MD3**
- **Location:** `static/planner-shared.css:1-32` (20+ переменных), `planner-form1.css:1-12`, `planner-form2.css:1-10`.
- **Category:** Theming.
- **Impact:** Параллельная палитра для тех же ролей: `--verua-disabled-fill` дублирует `--md-sys-color-surface-container-lowest`, `--verua-toolbar-text` ≈ `--md-sys-color-on-surface-variant`, `--accent-soft` ≈ `--md-sys-color-primary-container`. Сложность поддержки × 2.
- **Recommendation:** Удалить `--verua-*` слой, заменить usages на MD3-роли.

**[P2] Дорогой `repeating-linear-gradient` в `.sheet-day`**
- **Location:** `static/calendar.css:194-207`.
- **Category:** Performance.
- **Impact:** Линованная бумага рендерится на каждой из 31–42 ячеек месяца, повторно — на каждом repaint (scroll, hover, theme switch). Замерить надо в DevTools, но визуально с заметной нагрузкой при resize.
- **Recommendation:** Заменить на один SVG-pattern `data:` URL, либо `background-image` с маленькой png — браузер закеширует.

**[P2] Дублирование dark-mode блоков в 5 файлах**
- **Location:** каждый `static/*.css` имеет свой `@media (prefers-color-scheme: dark)`.
- **Category:** Theming.
- **Impact:** Design-system уже даёт dark тему через MD3 токены — нужны только переопределения для нестандартных hex (которых тоже не должно быть). Сейчас тёмная схема живёт в 5 разных местах = легко расходится.
- **Recommendation:** После P1-чистки hex → весь dark-блок исчезает сам собой.

**[P2] `cursor: pointer` на не-button элементах**
- **Location:** `planner-shared.css:124` `.slot summary`, `planner-shared.css:196` `.calendar-group summary` (для `<summary>` — нормально), но также `catalog-pages.css:312,320` `.toggle-all`, `.patient-row` (`<div>` без `role="button"`).
- **Category:** Accessibility.
- **Impact:** Кликабельный `<div>` без клавиатурной обработки.
- **Recommendation:** Превратить в `<a>` или `<md-list-item type="button">`.

### P3 Polish

**[P3] `box-sizing: border-box` объявлен в двух файлах**
- **Location:** `planner-shared.css:1`, `app-shell.css:34`.
- **Recommendation:** Оставить только в Design-system `common.css`.

**[P3] Тяжёлая тень в `body` и `.panel`**
- **Location:** `app-shell.css:53` `0 10px 28px rgba(0,0,0,.06)`.
- **Recommendation:** Использовать `var(--md-sys-elevation-level1..3)`, токен уже подгоняет тень под dark/light.

**[P3] `padding/border-radius` не через токены формы**
- **Location:** всё `border-radius: 8px` и `999px` хардкодом.
- **Recommendation:** `var(--md-sys-shape-corner-small)` (8px по умолчанию), `var(--md-sys-shape-corner-full)` (для «pill»).

## Patterns & Systemic Issues

- **«Параллельная вселенная» вместо Design-system.** В шаблонах `<md-filled-button>` соседствует с `<button class="btn">`; в стилях `--md-sys-*` соседствует с `--verua-*`. Это полу-миграция, застрявшая посередине: новый код (например, `patient_detail.html` — 12 `<md-*>`) уже идёт правильно, а старый (`planner-form1/2`, `top_nav`) — нет. AGENTS.md F.b явно говорит «новых правил в `static/*.css` не добавлять» — этот контракт работает только если параллельно идёт миграция, а её, судя по объёму, не было.

- **Тёмная тема как декорация, а не первый класс.** В каждом CSS есть `@media (prefers-color-scheme: dark)`, но переопределяются только hex-цвета. После полной миграции на MD3 токены весь dark-блок становится не нужен — Design-system уже это покрывает.

- **Touch-first аспект забыт.** Учитывая что Verua это медицинская практика (врач + ноутбук/планшет на приёме), мелкие touch-targets — это user-facing проблема, не косметика.

## Positive Findings

- **Архитектура подключения Design-system правильная** — `base_system.html → base.html`, `{{ ds_static }}` для assets, `?v={{ static_version }}` для cache-bust (G.1 соблюдается).
- **Новые страницы (`patients.html`, `patient_detail.html`, `doctor_detail.html`) уже на `<md-*>` компонентах** — пример как должно быть.
- **A4 print-стили в `calendar.css`** реально работают: правильный `@page`, тёмные оттенки убраны для печати, fixed grid. Это хорошая инженерия.
- **Тёмная тема не сломана** — несмотря на хардкоды, основные сценарии читаемы (благодаря `var(...)` fallback-ам).
- **Семантика HTML в основном корректна** — `<section>`, `<nav>`, `<details>/<summary>`, `<form>`.

## Recommended Actions

1. **[P0] Заменить нативные контролы на `<md-*>`** в `index.html`, `patient_create.html`, `partials/planner_editor.html`, `partials/planner_calendar_inner.html`. Большая работа, разбить по форме. Команда: ручная миграция по таблице `DESIGN_SYSTEM.md §5`.
2. **[P0] `/impeccable harden` для `top_nav.html`** — вынести inline JS, заменить `<button>` на `<md-text-button>`.
3. **[P1] `/impeccable distill`** — убрать 6 `backdrop-filter`, упростить градиенты в `body`/`message`/`summary`.
4. **[P1] `/impeccable typeset`** — снять hardcoded Segoe UI, дать дорогу Roboto Flex из Design-system.
5. **[P1] `/impeccable colorize`** — миграция 174 hex на `var(--md-sys-color-*)`.
6. **[P1] `/impeccable adapt`** — поднять touch-targets до 44×44 (день календаря, color-swatch, tool, month-arrow, tariff input).
7. **[P1] `/impeccable harden`** — пройти `aria-label`/`<label for>` по 110 нативным input (после P0 эта проблема исчезает сама — оставшиеся пара ручных).
8. **[P2] `/impeccable distill`** — удалить `--verua-*` параллельную палитру.
9. **[P2] `/impeccable optimize`** — заменить `repeating-linear-gradient` в `.sheet-day` на закешированный SVG-pattern.
10. **`/impeccable polish`** в конце — финальный проход.

> Эти задачи можешь запускать по одной, все сразу или в любом порядке.
>
> Перезапусти `/impeccable audit` после фиксов — score должен подняться.

**Замечание по процессу:** все CSS-правки **не должны добавляться в `static/*.css`** (правило F.b). Каждое изменение — это либо удаление правила, либо миграция в `../Design-system/assets/` отдельным PR в Design-system-репозитории. Это согласуется с памятью «всё в main без воркдеревьев» — но сам факт что нужны два коммита (Design-system + Verua_automation) надо учесть.