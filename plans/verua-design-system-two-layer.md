# Verua_automation → two-layer: вынос eintritts-checklist.css / checklist-viewer.css из Design-system

PM-MCP: #787 (Verua_automation). Статус: **ЧЕРНОВИК — не согласован.** Согласовывать с Codex отдельно перед стартом.

## Context

Прецедент — план `iridescent-twirling-panda.md` (#781-#784): app-specific ассеты Assistant-UI вынесены из
Design-system в app-local `/static`. Принцип закреплён org-wide: tech-stack **brick #10** (обновлён) + ADR
`AI-Assistant/docs/adrs/0008-two-layer-design-system-boundary.md` + memory 1552/1559.

Codex обработал ТОЛЬКО Assistant-UI. У Verua та же drift: **2 app-specific CSS живут в общем Design-system**
и используются только этим приложением:

| Ассет в DS | Ссылки в Verua |
|---|---|
| `assets/eintritts-checklist.css` | `templates/base_system.html:10`, `templates/checklist_print.html:6` |
| `assets/checklist-viewer.css` | `templates/base_system.html:11` |

Тест принадлежности (ADR 0008): «Стал бы Best_photo_ai/Stalking использовать ровно этот CSS?» Нет → код приложения.

**Уже частично two-layer:** собственный JS Verua лежит локально (`static/checklist-viewer.js`, ссылка
`/static/checklist-viewer.js?v={{ static_version }}`). Мигрировать нужно только CSS — поставить рядом с JS.

## Архитектурные факты (важно для миграции)

- **FastAPI-фабрика** `src/verua_automation/web/app.py`: mount `/static` → `STATIC_DIR`
  (`src/verua_automation/web/constants.py`), mount `/ds` → DS; `ChoiceLoader` (app-шаблоны побеждают DS);
  глобал `static_version` для cache-bust. Целевой слой `/static` УЖЕ есть.
- **Design-system подключён git-submodule'ом** (`.gitmodules`), не sibling. Удаление из DS = коммит в каноническом
  Design-system repo, затем **bump submodule-ref** в Verua.
- **Packaging:** `Portable/build-client-zip.ps1` бандлит `Design-system/{tokens,assets,vendor,fonts,templates}`.
  После переноса CSS в `static/` нужно убедиться, что сборка включает `static/` (там уже лежит `checklist-viewer.js`
  → вероятно включает; подтвердить при согласовании).
- Локальное правило `AGENTS.md:192` (F.b): «add to `../Design-system/assets/`» — старая модель, переписать на
  two-layer ПЕРВЫМ.

## Этапы (migration-discipline: build → migrate → remove → docs; правило-первым)

1. **Rule-first (Verua):** переписать `AGENTS.md` F.b (стр. ~187-195) на two-layer — app-specific фиче-CSS живёт
   в `static/`, shared-примитивы в DS. Ссылка на ADR 0008 / brick #10.
2. **Build app-local:** скопировать `eintritts-checklist.css` + `checklist-viewer.css` из Design-system/assets
   в `STATIC_DIR`. (Копировать рабочее содержимое DS-submodule.)
3. **Migrate refs:** в `base_system.html` (стр. 10-11) и `checklist_print.html` (стр. 6) заменить
   `{{ ds_static }}/assets/*.css` → `/static/*.css?v={{ static_version }}`. Прочие `{{ ds_static }}/...`
   (tokens, theme, fonts, bundle, fouc) — не трогать.
4. **Remove old (Design-system):** удалить 2 CSS; обновить дерево ассетов в `DESIGN_SYSTEM.md`; запись в
   `CHANGELOG.md`; version bump (см. «Координация DS-версии»). Затем **bump submodule-ref** в Verua.
5. **Packaging + docs:** проверить/обновить `Portable/build-client-zip.ps1` (CSS теперь в `static/`, не в
   `Design-system/assets/`); обновить `README.md`/`docs/ARCHITECTURE.md`, если упоминают расположение этих CSS.
6. **Verify:** поднять Verua UI, открыть карточку пациента (checklist-viewer lightbox) и `checklist_print`,
   визуально сверить (desktop+mobile), 404 на CSS отсутствуют. Прогнать тесты репозитория, если есть.
   Доп.: собрать Portable-zip и проверить, что CSS попал в сборку.

## Координация DS-версии

Breaking removal путей `/ds/assets/eintritts-checklist.css` и `/ds/assets/checklist-viewer.css`. Решить при
согласовании: отдельный DS-bump или **батч с Best_photo_ai** (#786) одним DS-релизом. Удалять из DS только
ПОСЛЕ миграции Verua и обновления submodule-ref (иначе served/packaged 404).

## Критерии приёмки

- [ ] 2 CSS лежат в `STATIC_DIR`, удалены из `Design-system/assets/`.
- [ ] Нет `{{ ds_static }}/assets/*checklist*.css` в шаблонах; `/ds` остаётся для tokens/theme/fonts/bundle/fouc.
- [ ] Карточка пациента + checklist print рендерятся без 404, визуально идентичны.
- [ ] Portable-zip включает перенесённый CSS.
- [ ] `AGENTS.md` F.b описывает two-layer; `DESIGN_SYSTEM.md`/`CHANGELOG.md`/version обновлены; submodule-ref bumped.

## При согласовании

- Разложить на per-repo work items (K.4): rule+docs (Verua) → move+packaging (Verua) → removal (Design-system).
  Текущая #787 — umbrella-трекер.
