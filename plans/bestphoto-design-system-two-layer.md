# Best_photo_ai → two-layer: вынос bestphoto-*.css из Design-system

PM-MCP: #786 (Best_photo_ai). Статус: **ЧЕРНОВИК — не согласован.** Согласовывать с Codex отдельно перед стартом.

## Context

Прецедент — план `iridescent-twirling-panda.md` (#781-#784): для Assistant-UI app-specific фиче-ассеты
вынесены из Design-system в app-local `/static`. Принцип закреплён org-wide: tech-stack **brick #10**
(уже обновлён) + ADR `AI-Assistant/docs/adrs/0008-two-layer-design-system-boundary.md` + memory 1552/1559.

Codex обработал ТОЛЬКО Assistant-UI. У Best_photo_ai та же drift: **3 app-specific CSS живут в общем
Design-system** и используются только этим приложением:

| Ассет в DS | Ссылка в Best_photo_ai |
|---|---|
| `assets/bestphoto-review.css` | `viewer_static/index.html:10` (`/ds/assets/bestphoto-review.css?v=…`) |
| `assets/bestphoto-evaluate.css` | `viewer_static/evaluate.html:10` |
| `assets/bestphoto-print.css` | `viewer_static/print.html:8` |

Тест принадлежности (ADR 0008): «Стал бы Verua/Stalking использовать ровно этот CSS?» Нет → код приложения.

**Уже частично two-layer:** собственный JS Best_photo_ai лежит локально (`viewer_static/app*.js`,
ссылки `/static/app*.js?v=…`). Мигрировать нужно только CSS — поставить рядом с JS.

## Архитектурные факты (важно для миграции)

- **FastAPI** в `Scripts/review_web_app.py`: `STATIC_DIR = …/viewer_static`, mount `/static` → `viewer_static`
  (review_web_app.py:1802), mount `/ds` → DS (review_web_app.py:1805). Целевой слой `/static` УЖЕ есть.
- **Design-system подключён git-submodule'ом** (`.gitmodules` → `path = Design-system`), а не sibling-checkout
  как в Assistant-UI. Значит удаление из DS = коммит в каноническом Design-system repo, затем **bump submodule-ref**
  в Best_photo_ai. Это дополнительный шаг против прецедента.
- `viewer_static/*.html` — статические HTML с ручным cache-bust `?v=<hash>` (не Jinja). Ссылки правятся вручную.
- Локальное правило `AGENTS.md:181` (F.b): «Do not add new CSS rules to project-level CSS files (`viewer_static/*.css`)»
  — запрещает app-local CSS. Его надо переписать на two-layer ПЕРВЫМ, иначе агенты вернут CSS в DS.
- `ARCHITECTURE.md:170`: «UI-стили подключаются только из `Design-system/assets/` через `/ds/assets/bestphoto-*.css`» — обновить.

## Этапы (migration-discipline: build → migrate → remove → docs; правило-первым)

1. **Rule-first (Best_photo_ai):** переписать `AGENTS.md` F.b (стр. 181) на two-layer — app-specific фиче-CSS
   живёт в `viewer_static/` (слой /static), shared-примитивы остаются в DS. Ссылка на ADR 0008 / brick #10.
2. **Build app-local:** скопировать `bestphoto-review.css`, `bestphoto-evaluate.css`, `bestphoto-print.css`
   из Design-system/assets в `viewer_static/`. (Копировать рабочее содержимое DS-submodule, проверить чистоту.)
3. **Migrate refs:** в `index.html` / `evaluate.html` / `print.html` заменить `/ds/assets/bestphoto-*.css` →
   `/static/bestphoto-*.css` (обновить `?v=`). Прочие `/ds/...` (fouc.js, theme-toggle.js, tokens) — не трогать.
4. **Remove old (Design-system):** удалить 3 `bestphoto-*.css`; обновить дерево ассетов в `DESIGN_SYSTEM.md`;
   запись в `CHANGELOG.md`; version bump (см. «Координация DS-версии»). Затем **bump submodule-ref** в Best_photo_ai.
5. **Docs:** обновить `ARCHITECTURE.md:170`.
6. **Verify:** поднять review web app, открыть `index`/`evaluate`/`print`, визуально сверить (desktop+mobile),
   убедиться в отсутствии 404 на CSS. Прогнать тесты репозитория, если есть.

## Координация DS-версии

Удаление — breaking removal публичных путей `/ds/assets/bestphoto-*`. Решить при согласовании: отдельный
DS-bump (0.2.0 → 0.3.0) или **батч с Verua** (#787) одним DS-релизом, чтобы не плодить версии. Удалять из DS
только ПОСЛЕ того, как Best_photo_ai мигрирован и его submodule-ref обновлён (иначе served/packaged 404).

## Критерии приёмки

- [ ] 3 `bestphoto-*.css` лежат в `viewer_static/`, удалены из `Design-system/assets/`.
- [ ] Нет `/ds/assets/bestphoto-*` в HTML; `/ds` остаётся для tokens/theme/fonts/bundle/fouc/theme-toggle.
- [ ] `index`/`evaluate`/`print` рендерятся без 404, визуально идентичны.
- [ ] `AGENTS.md` F.b и `ARCHITECTURE.md` описывают two-layer; `DESIGN_SYSTEM.md`/`CHANGELOG.md`/version обновлены; submodule-ref bumped.

## При согласовании

- Разложить на per-repo work items (K.4) как в прецеденте: rule+docs (Best_photo_ai) → move (Best_photo_ai)
  → removal (Design-system). Текущая #786 — umbrella-трекер.
- Подтвердить: пакуется ли Best_photo_ai (build/dist), и попадает ли `viewer_static/*.css` в сборку.
