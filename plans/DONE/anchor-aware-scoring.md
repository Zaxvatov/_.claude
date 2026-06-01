# Слоистая оценка фото: anchor-aware scoring → AI-судья в серии

## Контекст

В текущем коде две накладывающиеся проблемы. По убыванию серьёзности:

1. **Глобальный face-mode переключатель.** `Scripts/12_build_best.py:180–184`
   (`load_subject_data`) возвращает `subject_mode = "face_metrics"` для
   ВСЕГО batch'а сразу же, как только в `face_metrics.csv` есть хоть
   одна валидная строка. Дальше `Scripts/12_build_best.py:1187–1225`
   ветвит итоговую формулу по этому глобальному флагу, и документ,
   пейзаж или фото предмета получают face-heavy формулу с весами на
   `gaze_camera_score`, `eyes_score`, `smile_n`, `face_quality_score` —
   при нулевых face-сигналах. То есть не-лицевые фото системно
   проигрывают любой группе, в которой хоть где-то нашлись лица.
   Это первое, что надо чинить, и для этого никаких новых моделей
   не нужно.

2. **Сюжет привязан к одному лицу и трём hard-категориям.**
   - `Scripts/07_compute_composition.py:111–123` делает свою Haar
     детекцию и берёт max-area лицо для `subject_placement`,
     `face_coverage`, `edge_penalty`.
   - `Scripts/08_compute_subject.py:40–73` — ещё одна независимая
     Haar детекция, max-score одного лица.
   - `Scripts/08b_compute_face_metrics.py:619–625` агрегирует
     face-сигналы только от primary face.
   - `Scripts/12_build_best.py:710–745` присваивает hard
     `content_type ∈ {people, document, picture}`. Hard-категория
     плохо описывает смешанные кадры и используется в основном
     для тегов/viewer/self-learning, а не для выбора основной
     формулы качества (выбирает её именно перекос из пункта 1).

Цель:
- pipeline понимает, что изображено на фото — открыто, через набор
  значимых областей (anchors) и семантические признаки;
- считает «красоту» с учётом смеси якорей на конкретном фото, а не
  по глобальному переключателю режима;
- сравнивает кадры внутри серии (группировка от `05_group_similar_images`);
- задействует AI-судью только там, где алгоритмическое сравнение
  ненадёжно.

## Целевая модель

### Слой A. Понимание сцены (per-photo)
Открытый набор **anchors** и семантические признаки кадра — без
hard-категоризации.

Источники в финальном виде:
- **Face anchors** — единственная face-детекция, оставшаяся в `08b`.
- **Region anchors** — заметные области от лёгких средств:
  saliency на основе spectral residual / edge density, простой
  поиск прямоугольных контуров (document/screen) через
  `cv2.findContours + approxPolyDP`, OCR-блок для текста как сигнал.
  Тяжёлые open-vocabulary детекторы (OWLv2 / Grounding DINO) —
  только в фазе 8, если этого окажется мало.
- **CLIP image embedding + open-vocab tags** — расширенный
  `09_compute_scene_semantics.py` (бывший `09_compute_person_presence.py`).
  Богатый prompt-набор без сведения к трём классам. Embedding хранится
  в отдельном компактном артефакте (`.npy` + manifest), не в CSV.
- **`anchor_kinds_mix`** — per-photo непрерывные доли якорей по
  типам, не one-hot.

Выходы:
- `photo_anchors.csv` — per-anchor строки со стабильной схемой
  (см. ниже).
- `photo_scene.csv` — per-photo `scene_tags`, `anchor_kinds_mix`,
  ссылка в embedding-store, версия модели.
- Embedding-store: `.npy` файлы + `manifest.csv` (или JSON), не
  parquet (в `pyproject.toml` нет `pyarrow`).

### Слой B. Per-anchor метрики
Face — текущие метрики `08b`. Object/region — sharpness внутри bbox,
contrast vs окружение, completeness, centeredness. Document/text —
`skew_score`, `readability_score`, `occlusion_score`.

### Слой C. Photo-level сигналы (всегда)
- Технические первичные: `dark_clip_ratio`, `bright_clip_ratio`,
  `luma_std`, `saturation_mean`, `noise_sigma` — отдельно от
  свёрнутых оценок (отмечено в `Метрики обработки фото.xlsx`
  как «нужно добавить как первичную метрику»).
- Композиция (anchor-aware, без своей face-детекции):
  плавный `tilt_score`, `anchors_thirds_score`,
  `anchors_edge_penalty` (непрерывный), `anchors_balance`.
- `aesthetic_score` — нейроэстетика, как сейчас в
  `11_compute_aesthetic.py`.

### Слой D. Photo score с независимыми блоками и нормализованными весами
**Не** `generic = 1 - face` — это слишком грубо: фото с одним слабым
лицом теряет предметную/эстетическую оценку. Правильнее независимые
блоки и пер-row нормализация по **эффективным**, а не базовым весам.
Иначе слабый face-сигнал продолжит штрафовать фото, на котором лиц
почти нет.

```
raw_face    = w_face_base    * face_quality_block * face_confidence
raw_object  = w_object_base  * object_block       * object_confidence
raw_text    = w_text_base    * text_block         * text_confidence
raw_tech    = w_tech_base    * technical_quality                       (всегда)
raw_compose = w_compose_base * composition_quality                     (всегда)
raw_aes     = w_aes_base     * aesthetic_score                         (всегда)

denominator =
    w_tech_base + w_compose_base + w_aes_base +          (полные базовые
                                                          для всегда активных)
    w_face_base   * face_confidence   +                  (эффективные веса
    w_object_base * object_confidence +                   для условных блоков)
    w_text_base   * text_confidence

photo_score = (raw_face + raw_object + raw_text +
               raw_tech + raw_compose + raw_aes) / denominator
```

`face_confidence` берётся per-row из `face_validity_score` и
`prominent_faces`. Тех же правил придерживаются `object_confidence`
и `text_confidence`. Это сохраняет сравнимость между разными типами
фото и не штрафует одно за отсутствие другого.

### Слой E. Series ranking + AI-судья
Поверх `photo_score`:
1. Сортировка внутри `group_id` (серия из `05_group_similar_images`).
2. Confidence gate: `score_gap_top1_top2`, согласованность
   доминирующих anchor kinds, confidence learned
   `photo_best_model.pkl`.
3. AI-судья только на парах top-K в close-call случаях.
4. **Отдельный артефакт** `series_ai_preferences.csv` и
   **отдельный** `selection_score_ai_adjusted`. Никакого молчаливого
   переписывания `selection_score` в `review_groups.csv` —
   подключается только после проверки и через явный
   viewer-тоггл «текущий алгоритм / алгоритм + AI-судья».
5. `judge_reason_short` сохраняется и показывается во viewer.

## Стабильная схема `photo_anchors.csv`

Сразу зафиксирована, чтобы потом не ломать downstream:
```
file_path, asset_id, anchor_id, anchor_kind, anchor_source,
bbox_x_px, bbox_y_px, bbox_w_px, bbox_h_px,
bbox_x_norm, bbox_y_norm, bbox_w_norm, bbox_h_norm,
center_x_norm, center_y_norm,
prominence_score, confidence_score,
anchor_label, schema_version
```
- `anchor_kind` ∈ {face, salient_region, document_region, text_region}
  (расширяется в фазе 8 на object).
- `anchor_source` — какой детектор/эвристика породила якорь
  (`haar`, `spectral_residual`, `contour_quad`, `ocr`, и т.д.).
- `schema_version` — целое, инкрементируется при изменении схемы.

## Поэтапный план миграции

Принцип Q.1: сначала новый путь, миграция потребителей, потом
удаление старого. Каждая фаза — отдельная задача в Assistant-UI
(правило K.2).

### Фаза 1 (маленькая, изолированная). Починить ранжирование
**Цель**: убрать batch-level `face_metrics` mode, перевести `12` на
per-row score с независимыми блоками. Никаких новых артефактов,
никаких переименований, никаких новых моделей.

Изменения:
- `Scripts/12_build_best.py:180–193` — `load_subject_data`
  перестаёт возвращать batch-level режим, просто отдаёт merged
  DataFrame с face-полями (когда есть) и без них (когда нет).
- `Scripts/12_build_best.py:1187–1240` — заменяется один блок
  if-else на единую формулу с независимыми блоками (см. слой D).
  Per-row `face_confidence` = `face_validity_score *
  saturating(prominent_faces / k)` (с обработкой NaN/missing).
- Веса блоков подбираются так, чтобы воспроизвести текущее
  ранжирование на фото с лицами и не задавить documents/landscapes.

Тесты:
- Новый regression-кейс: смешанная серия (групповое фото +
  документ + пейзаж + фото предмета) — документ и пейзаж не
  получают face-весов, ранжирование внутри каждого подтипа
  сравнимо.
- `tests/test_face_metrics_prominence.py` — face-агрегация не
  сломана.
- `tests/test_all_critical_regressions.py`.

Документация: обновить `12_build_best.py` блок в
`docs/pipeline.md` (новая формула, без режимов).

### Фазы 2–4 (связанный этап). Первичные сигналы → anchors → композиция
Раскладывается на три задачи в Assistant-UI, но идёт одним
архитектурным куском, потому что 07 после переписывания читает
вывод 08b.

#### Фаза 2. Первичные сигналы в 06
- `Scripts/06_compute_sharpness.py` сохраняет в CSV первичные:
  `luma_mean`, `luma_std`, `dark_clip_ratio`, `bright_clip_ratio`,
  `saturation_mean`, `noise_sigma`, `aspect_ratio`. Свёрнутые
  оценки (`exposure_score` и т.д.) строятся поверх первичных,
  не вместо них.
- Имя файла **не переименовываем**: переименование numbered
  script затронет orchestrator, docs, tests и resume markers
  без видимой пользы.
- Schema-bump CSV артефакта 06; downstream обновляется.

#### Фаза 3. photo_anchors.csv с финальной схемой
- Расширить `Scripts/08b_compute_face_metrics.py` (имя оставить
  до фазы 7) — помимо `face_detections.csv` писать
  `photo_anchors.csv` со схемой из раздела выше.
- Путь к `photo_anchors.csv` регистрируется в
  `Scripts/config_paths.py` **сразу** на этой фазе (правило
  B.1), даже если пока его читают только `07/12`.
- На этой фазе в `photo_anchors.csv` пишутся:
  - face-anchors (`anchor_source = haar`);
  - saliency-anchors через spectral residual / edge density
    (`anchor_source = spectral_residual`) — когда лиц нет или
    кроме лиц есть выраженный saliency-регион;
  - document/screen-anchors через простую геометрию контуров
    (`anchor_source = contour_quad`) — **без** зависимости от
    CLIP-тегов. Триггер «сохранять или нет document/screen
    anchor» — порог по площади четырёхугольника и
    rectangleness; CLIP в фазе 5 будет его уточнять, но не
    блокирует появление якоря здесь.
- text-anchors через OCR — **отложены**. В `pyproject.toml`
  уже есть `easyocr`, но это тяжёлый слой, и для первого
  document-region сигнала геометрии контуров достаточно.
  OCR-anchors возвращаются как опция в фазе 5 или позднее.
- `face_metrics.csv` в этой фазе **не расширяется**
  weighted-aggregate полями. Все aggregate-вычисления уезжают
  в `12_build_best.py`, которое читает `photo_anchors.csv` и
  считает per-row блоки сразу из per-anchor строк. Иначе
  получится два места под один смысл.

#### Фаза 4. Anchor-aware composition
- `Scripts/07_compute_composition.py`:
  - убрать `get_cascade` и Haar-детекцию (`32–39`, `111–115`);
  - читать `photo_anchors.csv`;
  - `subject_placement`, `face_coverage`, `edge_penalty`,
    `tilt_score` становятся функциями всех якорей с весами по
    prominence;
  - `tilt_score` плавный (заменить логику `66–78`);
  - `composition_score` — anchor-aware свёртка.
- `Scripts/08_compute_subject.py` НЕ удаляется. После фазы 1
  его выход уже не определяет формулу. Удаление — фаза 7.

### Фаза 5. scene_semantics (расширенный 09)
- Переработать `Scripts/09_compute_person_presence.py` →
  `09_compute_scene_semantics.py`:
  - расширить `Scripts/clip_presence_base.py`, чтобы возвращать
    image embedding отдельно от presence scores;
  - embedding пишется в `.npy` файлы + единый `manifest.csv`
    (или JSON) с `asset_id`, `mtime`, `model_version`,
    `prompt_set_hash`, `npy_path`;
  - `photo_scene.csv` с `scene_tags`, `anchor_kinds_mix`,
    `embedding_ref`;
  - кэш по `(file_path, mtime, model_version)`;
  - `person_presence_score` сохраняется как одно из полей для
    обратной совместимости текущих потребителей до фазы 7.
- `Scripts/10_compute_document_presence.py` пока **не удаляется**.

### Фаза 6. AI-судья на близких случаях
Делается **двумя** последовательными задачами.

**6a. Только артефакт preferences, без viewer-интеграции.**
- Новый `Scripts/12b_series_ai_judge.py`:
  - читает `best_combined.csv` и `review_groups.csv`;
  - считает confidence gate per group;
  - на close-call парах вызывает VLM;
  - пишет **только** `series_ai_preferences.csv`
    (`group_id`, `asset_id_a`, `asset_id_b`, `pref_a_over_b`,
    `judge_model`, `judge_version`, `judge_reason_short`,
    `requested_at`);
  - **не трогает** `review_groups.csv` и viewer ни в одном
    поле.
- `Scripts/vlm_judge.py` — wrapper, конкретная модель и runtime
  выбираются на этапе реализации; обязательное кэширование
  preferences по `(asset_id_a, asset_id_b, judge_model,
  judge_version)`.
- Verification: прогон на `golden_cases/photo_best`, ручная
  выборочная проверка `judge_reason_short`.

**6b. Подключение к viewer через явный тоггл.**
- Только после того, как `series_ai_preferences.csv` прошёл
  выборочную проверку и regression на golden cases:
  - добавляется колонка `selection_score_ai_adjusted` в
    `review_groups.csv` (или в parallel review-артефакт);
  - viewer получает явный тоггл с двумя режимами:
    «текущий алгоритм» / «алгоритм + AI-судья».
- Никакого молчаливого переписывания `selection_score`.

### Фаза 7. Очистка после стабилизации
Когда новый pipeline проходит regression на golden_cases/photo_best
и viewer-тесты, и `series_ai_preferences.csv` стабилен:
- удалить `Scripts/08_compute_subject.py` и `subject.csv`;
- удалить `Scripts/10_compute_document_presence.py` и
  `document_presence.csv`;
- удалить fallback ветку `subject_scores` в `load_subject_data`
  (она к этому моменту уже не вызывается, поскольку 12 читает
  напрямую `photo_anchors.csv`);
- **`content_type_*` НЕ удаляется из viewer**, пока во viewer
  не появятся замены фильтров «люди / документы / пейзажи» на
  основе `scene_tags` и `anchor_kinds_mix`. Это отдельная
  под-задача viewer'а; до её закрытия hard `content_type`
  живёт во viewer как UX-фильтр, даже если в scoring не
  участвует;
- путь `SUBJECT`, `DOCUMENT_PRESENCE` из `Scripts/config_paths.py`
  удаляются;
- self-learning (`27/28/29` для photo_content): либо снимается
  с pipeline'а, либо переводится на open-vocab tags — решается
  отдельной задачей после готовности viewer-фильтров.

### Фаза 8 (опционально). Тяжёлые модели
Если фаз 3–6 окажется недостаточно по качеству на реальных
сериях:
- open-vocab object detector (OWLv2 / Grounding DINO) в
  `Scripts/anchor_detector.py`;
- нейросетевая saliency-модель (TRACER / U2Net / BASNet) в
  `Scripts/saliency_base.py`;
- замена локальной VLM на более крупную или внешний API.

## Артефакты

| Артефакт | Изменение по фазам |
|----------|---------------------|
| `photo_index.csv` | без изменений |
| 06-output (sharpness/basics) | фаза 2: + первичные поля |
| `face_detections.csv` | без структурных изменений |
| `face_metrics.csv` | без структурных изменений; aggregate-вычисления в `12`, а не здесь |
| `subject.csv` | без изменений до фазы 7, потом удаляется |
| `composition.csv` | контент меняется в фазе 4, схема стабильна |
| `person_presence.csv` | без изменений до фазы 5, поле сохраняется в `photo_scene.csv` |
| `document_presence.csv` | без изменений до фазы 7, потом удаляется |
| `aesthetic.csv` | без изменений |
| `photo_anchors.csv` | **новый, фаза 3**, схема выше |
| `photo_scene.csv` | **новый, фаза 5** |
| embedding store (`.npy` + manifest) | **новый, фаза 5** |
| `series_ai_preferences.csv` | **новый, фаза 6** |
| `best_combined.csv`, `review_groups.csv` | схема стабильна; `selection_score_ai_adjusted` добавляется **после** проверки фазы 6, отдельной правкой |

## Документация

Правила L.1, L.2, Q.2:
- `docs/pipeline.md` — описание шагов на каждой фазе;
- `ARCHITECTURE.md` — Photo pipeline блок, Stable extension
  points, change hazards (особенно фазы 5 и 6);
- `AGENTS.md` — секции C/D, если меняются инварианты;
- `README.md` — обновления порядка шагов.

## Верификация

На каждой фазе:
1. `uv run python -m unittest discover -s tests`.
2. `uv run python -m unittest tests/test_face_metrics_prominence.py`.
3. `uv run python -m unittest tests/test_all_critical_regressions.py`.
4. `uv run python -m unittest tests/test_review_web_smoke.py
   tests/test_viewer_regression.py`.

Дополнительно по фазам:

- **Фаза 1**: тестовая папка с смешанной серией — документ/пейзаж
  не давятся face-весами.
- **Фазы 3–4**: разумный `photo_anchors.csv` и `composition_score`
  на серии групповых фото с разным числом людей.
- **Фаза 5**: cache hit-rate embedding-store на повторном прогоне
  100% при неизменных входах.
- **Фаза 6**: VLM вызывается только в close-call случаях;
  `series_ai_preferences.csv` не пересчитывается на hot-prune.
- **Фаза 7**: full regression на `golden_cases/photo_best` —
  preference-метрика не деградирует относительно бейзлайна
  фазы 1; viewer-фильтры по типу не сломаны.

## Открытые решения, финализируются по ходу

1. **Конкретная VLM-модель и runtime** (локальный Qwen2-VL /
   LLaVA-OneVision / Llama-3.2-Vision; либо внешний API). Решается
   по VRAM, приватности и стоимости.
2. **Конкретная saliency-эвристика** в фазе 3 (spectral residual
   vs edge density vs их комбинация). Бенчмарк на golden cases.
3. **Confidence gate thresholds** — после фазы 1.
4. **Cловарь open-vocab tags для CLIP** — расширяемый список,
   стартовый набор и процедура его расширения.
5. **Замена viewer-фильтров «люди / документы / пейзажи»** на
   open-vocab — отдельная UX-задача в фазе 7.

## Что НЕ входит в этот план

- Реализация — план только; работы заводятся отдельными задачами
  через Assistant-UI per K.2, по одной фазе на task.
- Identity-aware person clustering между фото (отдельная цель в
  `docs/pipeline.md`).
- Изменения в video-ветке (`13–20`).
- Эстетическая модель сама по себе — `11_compute_aesthetic.py`
  остаётся как есть до фазы 8.
