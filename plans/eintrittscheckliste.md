# Eintrittscheckliste: ввод при создании пациента, версионируемый предпросмотр в карточке

## Контекст

Для каждого пациента нужен контрольный чек-лист, повторяющий структуру `D:\Загрузки\1.Eintrittscheckliste.docx` (распакован — 5 разделов, ~30 чек-боксов и текстовых строк, см. ниже). Цели:

- При создании пациента в форме [templates/patient_create.html](templates/patient_create.html) под секцией «Представитель пациента» появляются те же поля, что и в листе, в визуальной раскладке листа A4.
- После сабмита: пациент уходит в VeruA через существующий поток, **а копия чек-листа сохраняется локально** как версия №1 (по согласованию с пользователем — авто-сохранение).
- В карточке пациента [templates/patient_detail.html](templates/patient_detail.html) рядом с кнопкой «Новый представитель» появляется кнопка «Eintrittscheckliste», открывающая лайтбокс-предпросмотр поверх страницы (паттерн взят из `D:\GitHub\Stalking_offline\src\safe_offline\static\app.js:582–718`).
- Боковые стрелки переключают версии (новые → старые), клик по подложке/Esc закрывает и возвращает фокус на кнопку.
- В тулбаре предпросмотра: «Новая версия», «Удалить версию», «Печать». «Новая версия» — inline-правка тех же полей в раскладке листа (без рисования поверх); сохранение всегда создаёт новый снимок с датой.

Чек-лист хранится только локально (формулировка «форма сохраняется внутри сервиса»). В VeruA он не выгружается.

## Соглашения и проектные ограничения

- `AGENTS.md` K.2 — перед кодом создаём work-item в PM-MCP, переводим в `В работе`.
- `AGENTS.md` C.4 — изменение схемы только новой миграцией `_migrate_v4_*`, бамп `MIGRATIONS`/`CURRENT_SCHEMA_VERSION`, smoke-тест в [tests/test_db_migrations.py](tests/test_db_migrations.py).
- `AGENTS.md` C.7 — новый домен → отдельный роутер `web/routers/patient_checklists.py`, регистрация в [web/app.py](src/verua_automation/web/app.py).
- `AGENTS.md` C.8 — новая таблица → отдельный модуль `db/patient_checklists.py`, dataclass в [db/models.py](src/verua_automation/db/models.py), реэкспорт из [db/__init__.py](src/verua_automation/db/__init__.py).
- `AGENTS.md` C.6 — никакого SQL в роутерах, только в `db/*`.
- `AGENTS.md` F.b/G.2 — стили живут в `../Design-system/assets/`. В рамках задачи делаем отдельный коммит в репо `Design-system`, затем подтягиваем.
- `AGENTS.md` O.1 — поднять `data_schema_version` в [Portable/release-versions.json](Portable/release-versions.json).
- `AGENTS.md` E.1/E.2/E.10 — комментарии на русском, имена и пути на английском, `print` в `src/` запрещён.
- Память пользователя: «Весь код — только в main, без воркдеревьев и PR» — изменения коммитим прямо в `main` отдельными логически целостными коммитами. Push только по явному разрешению.

## Структура чек-листа (источник истины)

Из распакованного `word/document.xml` (5 разделов, порядок и тексты сохраняем как в оригинале):

1. **Mit Klienten gemeinsam erledigen (durch Spitex-Mitarbeiter)** — `cb_anmeldung_durch + tx_anmeldung_durch + tx_start_einsaetze`, `cb_spitexinformation`, `cb_aufnahmeformular`, `cb_krankenkassen_karte`; справочные текстовые строки `tx_arzt`, `tx_apotheke`, `tx_familienansprechsperson`, `tx_familie`.
2. **Pflegebedarf bei Anfrage** — `cb_behandlungspflege + tx_behandlungspflege_umfang`, `cb_grundpflege + tx_grundpflege_umfang`, `cb_betreuung + tx_betreuung_umfang`, `cb_hauswirtschaft + tx_hauswirtschaft_umfang`, `cb_bisherige_leistungen + tx_bisherige_leistungen`.
3. **Dokumente unterschreiben lassen** — `cb_agb`, `cb_pflegevereinbarung`, `cb_schweigepflicht`, `cb_notfallnummern`, `cb_patientenverfuegung + tx_patientenverfuegung + tx_rea`, `cb_sda`, `cb_rai_hc + tx_rai_hc_groesse + tx_rai_hc_gewicht + tx_rai_hc_vitalzeichen`, `cb_rai_hw`.
4. **Informationen über Klienten** — `cb_klienten_ma_vorstellen`, `cb_arzt_pflegebedarf`, `cb_diagnosen_vorhanden`, `cb_austrittsberichte`, `cb_vitalparameter + tx_vitalparameter`, `cb_medikamentenplan + tx_medikamentenplan_richtet`, `cb_einnahme_medikamente + tx_einnahme_medikamente`. **Строка Impfungen** — без левого чек-бокса, только внутренние: `cb_impfpass`, `cb_grippenimpfung`, `cb_pneumokokken`, `cb_herpes_zoster`. Затем `cb_therapie_pflege_personen + tx_therapie_pflege_personen`.
5. **Organisatorisches** — `cb_einsatzplan`. **Строка Ärztliche Verordnung unterschrieben für** — без левого чек-бокса: `cb_av_1monat`, `cb_av_3monate`, `cb_av_6monate`. Затем `cb_ppl`. Подпись внизу: `tx_datum`, `tx_visum`.

Все поля имеют префикс `checklist_` в `name=` атрибутах формы (`checklist_cb_anmeldung_durch` и т.д.), хранятся в JSON-словаре `fields`.

## Модель данных

Миграция v4 — новая таблица `patient_checklists` в [src/verua_automation/db_migrations.py](src/verua_automation/db_migrations.py):

```sql
CREATE TABLE patient_checklists (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    patient_external_id TEXT NOT NULL DEFAULT '',
    patient_local_id INTEGER NOT NULL DEFAULT 0,
    template_version INTEGER NOT NULL DEFAULT 1,  -- версия шаблона чек-листа (структура полей)
    data_json TEXT NOT NULL DEFAULT '{}',         -- {"schema_version": 1, "fields": {...}}
    source TEXT NOT NULL DEFAULT 'manual',        -- 'auto' для автосохранения версии №1
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
-- Карточка пациента открывается по локальному id, поэтому основной индекс — по нему
CREATE INDEX idx_patient_checklists_local
    ON patient_checklists(patient_local_id, created_at, id);
-- Запас на случай поиска по external_id (пациент создан через VeruA-sync)
CREATE INDEX idx_patient_checklists_external
    ON patient_checklists(patient_external_id, created_at);
```

Разделение двух понятий «версия» (по замечанию Codex):

- **`template_version`** — версия структуры формы (DOCX-шаблона). Сейчас `1`. Если позже добавим/удалим поля, поднимаем до `2`, а `checklist_schema.py` умеет рендерить старые сохранённые версии в режиме readonly без потери данных. Нужно для совместимости при изменении DOCX.
- **`id` + `created_at`** — версия конкретного заполнения чек-листа этим пациентом. Это и есть «версия N из M», по которой ходят стрелки в лайтбоксе.

`data_json` — `{"schema_version": 1, "fields": {<имя поля без префикса checklist_>: <значение>}}`. Чекбоксы — `bool`, текстовые поля — `str`. Поле `schema_version` внутри `data_json` дублирует `template_version` колонки и нужно как самодостаточная метка снимка.

Поле `label` в БД **не храним** — текст «Версия N из M от DD.MM.YYYY HH:MM» вычисляется в момент отдачи в UI на основе `created_at` и порядка по индексу.

## Источник истины формы

Новый модуль [src/verua_automation/checklist_schema.py](src/verua_automation/checklist_schema.py) — древовидное описание разделов и строк (dataclass `ChecklistField`, `ChecklistRow`, `ChecklistSection`). Используется:

- Jinja-партиалом для рендера (один обход — единый порядок).
- Хелпером для парсинга `multipart/form-data` в `dict[str, bool | str]`.
- Шаблоном печати.
- Тестами для валидации того, что в каждой версии присутствуют все известные ключи.

Модуль ≤200 строк, все строки `[manual]` E.1.

## Файлы

**Создать:**
- [src/verua_automation/checklist_schema.py](src/verua_automation/checklist_schema.py) — описание полей.
- [src/verua_automation/db/patient_checklists.py](src/verua_automation/db/patient_checklists.py) — CRUD, **все операции принимают `patient_local_id` явно**: `list_versions_for_patient`, `get_version_for_patient(patient_local_id, version_id)`, `create_version_for_patient(patient_local_id, ...)`, `delete_version_for_patient(patient_local_id, version_id)`. Чужую версию по голому `version_id` достать нельзя — это требование безопасности (по замечанию Codex).
- [src/verua_automation/web/routers/patient_checklists.py](src/verua_automation/web/routers/patient_checklists.py) — endpoints (везде ключ доступа = `patient_id` из URL + `version_id`; запись из чужого пациента возвращает 404):
  - `GET /directories/patients/{patient_id}/checklists` → JSON `[{id, created_at, template_version, source}]`, отсортировано **oldest → newest**
  - `GET /directories/patients/{patient_id}/checklists/{version_id}` → JSON `{id, created_at, template_version, fields, position, total}`
  - `GET /directories/patients/{patient_id}/checklists/{version_id}/fragment` → server-rendered HTML (тот же партиал в `editable=false`) — единственный источник layout-а для лайтбокса (см. ниже про single-source-of-truth)
  - `POST /directories/patients/{patient_id}/checklists` (form) → создаёт версию, 303 на карточку с `?message=...`
  - `DELETE /directories/patients/{patient_id}/checklists/{version_id}` → 204
  - `GET /directories/patients/{patient_id}/checklists/{version_id}/print` → HTML для печати, **расширяет DS-base** (а не `base_system.html`, чтобы не тащить навигацию приложения)
- [templates/partials/eintritts_checklist.html](templates/partials/eintritts_checklist.html) — Jinja-партиал; параметры `data`, `editable: bool`, `name_prefix` (для формы создания пациента поля идут с `checklist_`, для формы новой версии — с `checklist_`, та же раскладка).
- [templates/checklist_print.html](templates/checklist_print.html) — печатная страница, расширяет минимальный print-only base; вызывает `window.print()` после загрузки.
- [static/checklist-viewer.js](static/checklist-viewer.js) — лайтбокс: open/close, prev/next, режим правки, save/delete/print. Структура взята из `app.js:582–718` `Stalking_offline` (`activeIndex`, `openViewer`, `moveViewer`, `closeViewer`, обработчики Esc/←/→ на `document`, `data-*-close` на подложке).
- Тесты:
  - [tests/test_checklist_schema.py](tests/test_checklist_schema.py) — обход схемы, обязательные ключи.
  - [tests/test_db_patient_checklists.py](tests/test_db_patient_checklists.py) — CRUD, удаление, сортировка по дате.
  - [tests/test_checklist_router.py](tests/test_checklist_router.py) — JSON-эндпойнты, авто-сохранение версии №1, 404 для чужих пациентов.
  - расширение [tests/test_db_migrations.py](tests/test_db_migrations.py) — v3→v4 на ненулевой БД, бэкап создан.

**Изменить:**
- [src/verua_automation/db_migrations.py](src/verua_automation/db_migrations.py): функция `_migrate_v4_patient_checklists`, добавить кортеж в `MIGRATIONS`, поднять `CURRENT_SCHEMA_VERSION`.
- [src/verua_automation/db/models.py](src/verua_automation/db/models.py): dataclass `PatientChecklistEntry(id, patient_external_id, patient_local_id, label, data_json, source, created_at, updated_at)`.
- [src/verua_automation/db/__init__.py](src/verua_automation/db/__init__.py): реэкспорт `patient_checklists` и `PatientChecklistEntry`.
- [src/verua_automation/web/app.py](src/verua_automation/web/app.py): подключить новый роутер.
- [src/verua_automation/web/helpers.py](src/verua_automation/web/helpers.py): новая функция `build_patient_checklist_payload(form) -> dict` (использует `checklist_schema` для перебора имён). Никакого SQL — только маппинг.
- [src/verua_automation/web/routers/directories.py](src/verua_automation/web/routers/directories.py): после успешного `create_patient_bundle_in_verua()` в обработчике экспорта вызвать `db.patient_checklists.create_version(...)` с `label="Версия от <локальная дата/время>"` и `source="auto"`. Ошибка сохранения чек-листа не отменяет экспорт пациента — логируется через `logging` (E.10) и пробрасывается во flash. В `patient_detail()` добавить параметры шаблона `checklist_versions: list[PatientChecklistEntry]` и `checklist_initial: dict | None` (поля последней версии).
- [templates/patient_create.html](templates/patient_create.html): после строки 406 (внутри секции `patient-section-contact`) — `{% include "partials/eintritts_checklist.html" with context %}` в режиме `editable=true`, `name_prefix="checklist_"`.
- [templates/patient_detail.html](templates/patient_detail.html): добавить `<md-outlined-button>` «Eintrittscheckliste» рядом со строкой 16 (используя `data-checklist-open`). После закрытия `</section>` — оверлей `<div class="checklist-viewer" data-checklist-viewer-root hidden>` (структура воспроизводит `base.html:128–151` из `Stalking_offline`: backdrop, shell, prev/next, content, caption, toolbar). JSON-блок `<script type="application/json" data-checklist-bootstrap>` с массивом версий и URL-ами.
- [static/patient-create.js](static/patient-create.js): расширить `verua-patient-create-draft-v1`-сериализацию, добавив сбор/восстановление полей `checklist_*` (checkbox через `.checked`, остальное — через `.value`).
- [Portable/release-versions.json](Portable/release-versions.json): инкремент `data_schema_version`.

**В отдельном репозитории `../Design-system/`:**
- `assets/eintritts-checklist.css` — A4-раскладка (`max-width: 210mm`, поля 16mm, типографика согласно DESIGN_SYSTEM, подчёркнутые input-ы вместо outlined, точечная заливка пустого пространства). Print media query без декораций — чистый лист.
- `assets/checklist-viewer.css` — backdrop, shell, side-nav, scroll lock; повторяет селекторы Stalking_offline (`.media-viewer*`), переименованные в `.checklist-viewer*` и завязанные на MD3-токены (`var(--md-sys-color-scrim)`, `--md-sys-color-surface`).
- Обновить [`../Design-system/DESIGN_SYSTEM.md`](../Design-system/DESIGN_SYSTEM.md) — раздел про лайтбокс и чек-лист.

## Поведение лайтбокса

Однофайловый стейт-машинный JS с теми же инвариантами, что в Stalking_offline, плюс уточнения по замечаниям Codex:

- **Layout — единственный источник Jinja-партиал**, JS не строит A4-форму сам. Контент версии загружается через `GET .../checklists/{version_id}/fragment` (server-rendered HTML того же партиала в `editable=false`). При входе в режим «Новая версия» — копия текущего fragment-а с `editable=true` (или fetch той же страницы с `?edit=1`). JS отвечает только за переключения экранов, навигацию и AJAX.
- **Порядок версий** — oldest → newest (бэкенд так и отдаёт). По умолчанию открывается последняя (новейшая). Стрелка влево = старее, стрелка вправо = новее. Подпись в caption: `Версия N из M · 07.05.2026 14:32`.
- **Если версий 0** (старые пациенты, для которых чек-лист ещё не заполняли) — клик по кнопке «Eintrittscheckliste» сразу открывает лайтбокс в режиме «Новая версия» с пустой формой. Стрелки скрыты, кнопка «Удалить версию» не показывается.
- При закрытии возвращаем фокус на кнопку-триггер (фикс «no focus restoration» из эталона — нужно для доступности).
- `Esc`, `←`, `→` обрабатываются на `document` только когда `activeIndex >= 0` (страж).
- Кнопки prev/next — `disabled` на границе диапазона.
- Тулбар: «Новая версия» / «Удалить версию» / «Печать».
  - **Новая версия**: переключаем в форму (`disabled` снимается). Бар «Сохранить» / «Отменить». «Сохранить» → `POST` (FormData) → клиент перезагружает список версий через `GET .../checklists`, перепрыгивает на новую (последнюю). «Отменить» возвращает к снимку до начала правки.
  - **Удалить версию**: `<md-dialog>` с подтверждением → `DELETE` → если версий больше нет, закрываем оверлей (или возвращаемся к пустой «Новой версии»).
  - **Печать**: `window.open(printUrl)`; печатный шаблон сам вызывает `window.print()` после `DOMContentLoaded`.

## Авто-сохранение версии №1

В `directories.patient_create_export()` после возврата `create_patient_bundle_in_verua()` и записи пациента в SQLite: `db.patient_checklists.create_version_for_patient(patient_local_id=result.patient_local_id, patient_external_id=result.patient_external_id, template_version=1, data_json=json.dumps(payload), source="auto")`.

Атомарность не нужна — экспорт пациента уже произошёл в VeruA, поэтому при ошибке локального сохранения версии:

- логируем `log.exception(...)` (стандартный `logging`, не `print` — E.10),
- редирект на карточку пациента сохраняем, но в URL добавляем `?message=checklist_save_failed` (в проекте нет flash-механизма — сообщения проходят через query-param, как уточнил Codex; шаблон карточки рендерит баннер по этому ключу).

## Раскладка чек-листа на «лист A4»

Партиал [partials/eintritts_checklist.html](templates/partials/eintritts_checklist.html) — единственный источник раскладки. Чтобы режимы «редактирование при создании пациента» / «inline-правка в предпросмотре» / «печать» дали одинаковую вёрстку (требование пользователя — это «черновик конечного документа»):

- Корень — `<article class="eintritts-checklist eintritts-checklist--<editable|readonly>">`.
- Заголовок «Eintrittscheckliste für neue Klienten» + строка с `Klientenname`.
- Каждый раздел — `<section class="eintritts-checklist__section">` с `<h2>` названием раздела и подзаголовком в скобках («durch Spitex-Mitarbeiter» и т.п.).
- Каждая строка — `<div class="eintritts-checklist__row">`:
  - левая ячейка — чек-бокс (или пусто для Impfungen / Ärztliche Verordnung — там просто текст);
  - центральная ячейка — текст-метка строки;
  - правая ячейка — текстовый ввод/несколько вводов в исходном порядке (`<input class="eintritts-checklist__line">`).
- Внутренние подчек-боксы (Impfpass, Grippenimpfung и т.п., 1 Monat / 3 Monate / 6 Monate) — на той же строке справа от метки.
- Datum / Visum — в финальной строке, две колонки.
- В режиме `readonly` все `<input>` идут с `disabled`, чекбоксы тоже.

CSS из Design-system добивается «листа»: `max-width: 210mm`, верт. ритм по 8px, шрифт `--md-sys-typescale-body-medium`, заголовки разделов жирные, подзаголовки лёгкие. Подчёркивания вводов — `border-bottom`, без обводки. `@media print` убирает всё лишнее (тулбары, навигацию, `box-shadow`, `backdrop`).

## Изменение поведения формы создания пациента

`build_patient_bundle_payload` остаётся только для пациента/врача/представителя. Чек-лист собирается в [helpers.py](src/verua_automation/web/helpers.py:354) рядом, в новой `build_patient_checklist_payload`, и передаётся в обработчик отдельным шагом. Это соответствует существующему стилю: одна функция — одна сущность.

## Тесты

- `test_checklist_schema.py`: схема обходит все 5 разделов, имена ключей уникальны, типы совпадают (`bool`/`str`).
- `test_db_patient_checklists.py`: создание двух версий, чтение списка отсортированно, удаление одной, последняя версия определена корректно.
- `test_checklist_router.py`: создание пациента + авто-версия, GET списка, GET одной, POST новой версии, DELETE, печать (статус 200, заголовок Content-Type `text/html`).
- `test_db_migrations.py`: v3→v4 миграция применяется на копии БД без потерь, индекс создан, `CURRENT_SCHEMA_VERSION == 4`.
- Smoke-проверка `verify.ps1`: ruff, pytest, отсутствие raw SQL в роутерах, отсутствие абсолютных путей.

## Декомпозиция работ для Codex (work-items в Assistant-UI / PM-MCP)

Каждый пункт — отдельный work-item, который Codex берёт по очереди. Кросс-репозиторная задача `DS-1` создаётся в проекте `D:\GitHub\Design-system` (правило K.4).

### Project: `D:\GitHub\Verua_automation`

**T1 — Миграция v4 + CRUD `patient_checklists`**
- Файлы: `db_migrations.py` (`_migrate_v4_patient_checklists`, бамп `MIGRATIONS`/`CURRENT_SCHEMA_VERSION`), `db/patient_checklists.py`, `db/models.py` (`PatientChecklistEntry`), `db/__init__.py`, `Portable/release-versions.json` (бамп `data_schema_version`).
- Индексы — оба: `(patient_local_id, created_at, id)` и `(patient_external_id, created_at)`.
- CRUD — все операции принимают `patient_local_id` (защита от чужой версии): `list_versions_for_patient`, `get_version_for_patient`, `create_version_for_patient`, `delete_version_for_patient`.
- Тесты: `tests/test_db_migrations.py` (v3→v4 на ненулевой БД), `tests/test_db_patient_checklists.py`.
- Acceptance: миграция применяется идемпотентно, бэкап создан, оба индекса присутствуют, CRUD-тесты зелёные.
- Зависимости: нет.

**T2 — Схема полей `checklist_schema.py`**
- Файл: `src/verua_automation/checklist_schema.py` — древовидное описание 5 разделов, dataclass-узлы (`ChecklistField`, `ChecklistRow`, `ChecklistSection`), константа `TEMPLATE_VERSION = 1`. Источник истины: распакованный `D:\Загрузки\1.Eintrittscheckliste.docx` (см. раздел плана «Структура чек-листа»). Особо: строки `Impfungen` и `Ärztliche Verordnung unterschrieben für` — без левого чекбокса, только внутренние.
- Тесты: `tests/test_checklist_schema.py` — обход всех полей, уникальность ключей, корректные типы (`bool` для cb_*, `str` для tx_*).
- Acceptance: `verify.ps1` зелёный, тест зелёный, имена полей match договоренности (`cb_*`/`tx_*`).
- Зависимости: нет.

**T3 — Jinja-партиал `partials/eintritts_checklist.html` (single source of layout)**
- Файл: `templates/partials/eintritts_checklist.html` — параметры `data: dict`, `editable: bool`, `name_prefix: str`. Раскладка A4: заголовок, 5 секций с `<h2>` и подзаголовком в скобках, строки в исходном порядке из docx, итоговая строка Datum/Visum.
- Партиал импортирует список секций через макрос, который вызывает `checklist_schema.iter_sections()`.
- Acceptance: рендер партиала из тестового FastAPI-вью (или smoke-теста) даёт HTML, в котором присутствуют все ключи из схемы.
- Зависимости: T2.

**T4 — Endpoints `web/routers/patient_checklists.py`**
- Файл: `src/verua_automation/web/routers/patient_checklists.py`. Регистрация в `web/app.py`. Эндпойнты:
  - `GET /directories/patients/{patient_id}/checklists` (JSON list, `oldest → newest`).
  - `GET /directories/patients/{patient_id}/checklists/{version_id}` (JSON c `position`/`total`).
  - `GET /directories/patients/{patient_id}/checklists/{version_id}/fragment` (HTML с партиалом, `editable=false`).
  - `POST /directories/patients/{patient_id}/checklists` (form → 303 на карточку с `?message=checklist_saved`).
  - `DELETE /directories/patients/{patient_id}/checklists/{version_id}` (204).
  - `GET /directories/patients/{patient_id}/checklists/{version_id}/print` (HTML, расширяет DS print-base).
- Все эндпойнты используют CRUD с `patient_local_id`; чужой `version_id` → 404.
- Helper: `web/helpers.build_patient_checklist_payload(form) -> dict` (используется и в форме создания пациента, и в POST-эндпойнте).
- Тесты: `tests/test_checklist_router.py` — happy-path, 404 на чужого пациента, fragment рендерит ключевые поля, JSON-список отсортирован oldest→newest.
- Acceptance: ruff/pytest зелёные, в роутере нет raw SQL (C.6), нет hardcoded путей.
- Зависимости: T1, T2, T3.

**T5 — Интеграция в форму создания пациента + автосохранение v1**
- Файлы: `templates/patient_create.html` (вставка `{% include "partials/eintritts_checklist.html" %}` после строки 406, `editable=true`, `name_prefix="checklist_"`), `static/patient-create.js` (расширить `verua-patient-create-draft-v1` на поля `checklist_*`), `web/routers/directories.py` (после успешного `create_patient_bundle_in_verua` — `db.patient_checklists.create_version_for_patient(..., source="auto")`; ошибка → лог + `?message=checklist_save_failed`), `templates/patient_detail.html` (баннер по `?message=`).
- Тесты: smoke-тест `tests/test_checklist_router.py::test_auto_save_version_one`.
- Acceptance: новый пациент → в БД появляется ровно одна запись с `source='auto'` и заполненным `data_json`.
- Зависимости: T3, T4.

**T6 — Лайтбокс в карточке пациента**
- Файлы: `templates/patient_detail.html` (`<md-outlined-button data-checklist-open>` рядом с «Новый представитель», оверлей `<div class="checklist-viewer" data-checklist-viewer-root hidden>`, bootstrap-JSON `<script type="application/json" data-checklist-bootstrap>` со списком версий и URL-ами), `static/checklist-viewer.js` (стейт-машина из `Stalking_offline app.js:582–718`, адаптированная: контент берётся через `GET .../fragment`, focus restoration на триггер, страж `activeIndex >= 0`).
- Поведение версий: `oldest → newest`, открывается последняя; стрелки prev/next с `disabled` на границах; caption `Версия N из M · DD.MM.YYYY HH:MM`.
- Если версий 0 — открывается сразу режим «Новая версия» с пустой формой, без стрелок и без «Удалить».
- Тулбар: «Новая версия» (POST → reload list → land на новую), «Удалить версию» (`<md-dialog>` confirm → DELETE), «Печать» (`window.open(printUrl)`).
- `web/routers/directories.py::patient_detail` — передаёт в шаблон `checklist_versions: list[PatientChecklistEntry]` и URL-ы эндпойнтов.
- Acceptance: вручную: создать пациента → две версии → стрелки работают → удаление работает → печать открывает новое окно с диалогом печати.
- Зависимости: T4, T5, DS-1.

**T7 — Печатный шаблон `checklist_print.html`**
- Файл: `templates/checklist_print.html` — расширяет DS print-base (НЕ `base_system.html`), включает партиал `editable=false`, в `{% block scripts %}` — `window.addEventListener('DOMContentLoaded', () => window.print())`.
- Acceptance: открытие URL вручную даёт диалог печати с раскладкой A4 без навигации/футера.
- Зависимости: T3, DS-1.

**T8 — Документация**
- Файлы: `docs/ARCHITECTURE.md` (раздел про таблицу `patient_checklists`, поток автосохранения и лайтбокса; обновить таблицу схемы БД), `README.md` (короткая пользовательская заметка про чек-лист, если у README есть user-facing раздел).
- `AGENTS.md` правок не требует (новых rule-классов нет).
- Acceptance: `verify.ps1` (L.2 — все ссылки в README/ARCHITECTURE существуют).
- Зависимости: T1, T4, T5, T6.

### Project: `D:\GitHub\Design-system` (cross-repo, правило K.4)

**DS-1 — Стили чек-листа и лайтбокса**
- Перед началом: проверить чужие незакоммиченные изменения в `D:\GitHub\Design-system`, не трогать их. Если конфликт — приостановить и согласовать с пользователем.
- Файлы: `assets/eintritts-checklist.css` (A4 `max-width: 210mm`, типографика по DS-токенам, подчёркнутые `<input>` без обводки, `@media print` без декораций), `assets/checklist-viewer.css` (backdrop, shell, side-nav, scroll lock — структура из `Stalking_offline style.css:1277–1565`, переименовано в `.checklist-viewer*`, цвета из `--md-sys-color-*`), правка `DESIGN_SYSTEM.md` — раздел про лайтбокс и чек-лист.
- Coupling: верстку партиала и DOM лайтбокса согласуем с T3 и T6 — стили подключаем по тем же селекторам.
- Acceptance: коммит в `Design-system/main`, обновлены имена ассетов в нашем `Verua_automation/templates/base_system.html` (если требуется).
- Зависимости: нет (но сборка `T6`/`T7` без него не финишируется визуально).

### Граф зависимостей

```
T1 ─┐
    ├─► T4 ─┐
T2 ─┼──► T3 ─┼─► T5 ─► T6 ─┐
    │       │              ├─► T8
    │       └─► T7 ────────┘
DS-1 ─► T6, T7
```

### Глобальные правила выполнения (для Codex)

- Перед стартом каждой задачи Codex: `start_task` в Assistant-UI, при завершении: `complete_task` (К.1, К.3).
- Коммиты делаются **только по явному запросу пользователя** — план показывает логические границы, но не разрешает автоматический `git commit`.
- `TASKS.md` не обновляем — single source of truth по задачам = PM-MCP/Assistant-UI.
- Перед каждой задачей: `get_recent_memory(project="Verua_automation", limit=10)`.
- После каждой задачи: `verify.ps1` зелёный, `uv run pytest -q` зелёный, ruff зелёный.

## Верификация (end-to-end)

1. `uv sync && uv run python -m verua_automation.web_main` — старт UI на 8010.
2. Открыть `/directories/patients/new`, заполнить пациента + чек-лист, сабмит → пациент в VeruA, локально создана версия №1.
3. Перейти `/directories/patients/{id}` — кнопка «Eintrittscheckliste» рядом с «Новый представитель».
4. Клик → лайтбокс с раскладкой A4, поля заполнены значениями версии №1, навигация неактивна (одна версия).
5. «Новая версия» → поля редактируемы → меняем значения → «Сохранить» → две версии, стрелки активны, попадаем на новую.
6. ←/→ — переключение версий; Esc/клик по подложке — закрытие, фокус на кнопке-триггере.
7. «Печать» текущей → новое окно с печатной раскладкой → автоматически открывается диалог печати.
8. «Удалить версию» → `<md-dialog>` подтверждение → версия удалена; если последняя — оверлей закрывается.
9. `uv run pytest -q` — все тесты зелёные.
10. `uv run ruff check src/ tests/` и `uv run ruff format --check src/ tests/`.
11. `.\verify.ps1` — зелёный (нет SQL в роутерах, нет абсолютных путей, нет hardcoded `?v=`).

## Вне зоны задачи

- Загрузка чек-листа в VeruA (требование явно: «форма сохраняется внутри сервиса»).
- Ручной рисованный слой поверх формы (отвергнуто на этапе уточнения).
- Редактирование старой версии «на месте» — каждый сейв всегда создаёт новую запись (иммутабельная история).
- Локализация интерфейса чек-листа: контент остаётся по-немецки (как в исходнике), хром UI — по-русски.
- Миграция уже существующих в проде пациентов (новых пока нет; миграция v4 ничего не пересчитывает).
