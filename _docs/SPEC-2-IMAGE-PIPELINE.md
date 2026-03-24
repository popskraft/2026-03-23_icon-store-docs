# SPEC-2: Пайплайн обработки изображений

Версия: 2.1
Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.2
Область: приём изображений, обработка, хранение утверждённых мастеров, генерация деривативов, QA, handoff для публикации

---

## 1. Назначение

Один утверждённый мастер на SKU — всё остальное генерируется из него. Google Drive — только входящий handoff. Production storage — S3/R2.

---

## 2. Принципы

- Сохранять фактический внешний вид иконы, не применять AI-enhancement.
- Один канонический мастер на SKU / вариант.
- Деривативы генерируются детерминированно из утверждённого мастера.
- Пайплайн batch-safe, возобновляемый, аудитируемый.
- Cloudinary — не используется.

---

## 3. Стек

| Уровень | Дефолт | Альтернатива |
|---|---|---|
| Входящий inbox | Google Drive | — |
| Хранилище мастеров | Cloudflare R2 (S3-compatible) | AWS S3 |
| Processing worker | Python (Pillow / pyvips) | Node.js / Sharp |
| Удаление фона | Локально в Python worker (алгоритмы Pillow/pyvips) | Ручная ретушь через operator flow |
| QA-записи | PostgreSQL + batch review артефакты | — |
| Publication layer | BigCommerce Catalog API + n8n → SP-API | — |

Код не должен хардкодить конкретного S3-провайдера.

---

## 4. Роли

| Роль | Ответственность |
|---|---|
| Автор / поставщик | Присылает исходники |
| Владелец | Принимает батчи, первичная сортировка |
| Обработчик изображений | Нормализует / ретуширует, возвращает approved master |
| Оператор | QA-чекпоинты, публикация assets в каналы |

Обработчик и оператор — разные роли. Тот, кто правит пиксели, не принимает решений о публикации.

---

## 5. Модель состояний

| Статус | Значение |
|---|---|
| `incoming` | Файл получен, не разобран |
| `retouch_required` | Нужна ручная правка |
| `processing` | Авто-нормализация запущена |
| `approved_master` | Утверждённый мастер зафиксирован |
| `derivatives_ready` | Деривативы созданы |
| `published` | Asset привязан к BC и/или Amazon |
| `rejected` | Не годится, нужна пересъёмка |

---

## 6. Структура хранилища

```
incoming/{source_system}/{batch_id}/{merchant_sku}/{source_filename}
approved/{merchant_sku}/master.jpg
derived/{merchant_sku}/amazon-main.jpg
derived/{merchant_sku}/amazon-gallery-01.jpg
derived/{merchant_sku}/site-pdp.webp
derived/{merchant_sku}/site-card.webp
derived/{merchant_sku}/site-zoom.jpg
qa/{batch_id}/{merchant_sku}/contact-sheet.jpg
archive/{batch_id}/manifest.json
```

Правила: `incoming` — иммутабельный; деривативы генерируются из мастера, не из деривативов; база данных — источник истины, не файловый путь.

---

## 7. Три стадии обработки

```
Stage A: Incoming
Владелец сортирует фото → папки incoming/{SKU}/
    ↓
Оператор QA (чекпоинт ③):
    ├── OK → Drive webhook запускает Stage B
    └── Проблемные → ТЗ обработчику → approved/{SKU}/ → Stage B

Stage B: Approved Master (Python worker → S3/R2)
    - удаление/очистка фона: локально в worker (без внешних background-removal API)
    - нормализация: ориентация, sRGB, экспозиция, контраст, padding на белый фон
    - проверка: ≥ 2000px по длинной стороне
    - PostgreSQL: approved_master_uri, image_qa_status = approved

Stage C: Published Derivatives (Python worker → S3/R2, авто из Stage B)
    amazon-main:    2000×2000, белый фон, JPG q95
    amazon-gallery: 1500×1500, белый фон, JPG q90
    site-pdp:       800×800,   белый фон, WebP q85
    site-card:      400×400,   белый фон, WebP q80
    site-zoom:      1600×1600, белый фон, JPG q90
    thumbnail:      150×150,   fill crop, WebP q70
    PostgreSQL: derivatives{} заполнен URL-адресами
```

---

## 8. Матрица ракурсов

| Тип конструкции | Минимум Amazon | Статус при нехватке |
|---|---|---|
| Плоская доска | front + detail-frame | `media_partial` если нет detail |
| Металлическая риза | front + detail-riza | `media_partial` |
| Триптих / складень | front (открыт) + side (закрыт) | `media_partial` |
| Путевая икона | front + back + side | `media_partial` |

`media_partial` — сайт OK, Amazon заблокирован до добавления фото.
`media_blocked` — front отсутствует, все шаги заблокированы.

---

## 9. Правила обработки

**Разрешено:** ориентация по EXIF, нормализация в sRGB, мягкая экспозиция и контраст, sharpening, resize, padding/containment, конвертация формата.

**Запрещено:** перекраска иконы, generative fill, синтетические фоны, реконструкция контента, любые изменения реального внешнего вида.

**Background removal:** применять только если фон не чистый. Не применять, если края рамки тонкие — риск артефактов.

---

## 10. QA-чеклист перед публикацией

- изображение чёткое, икона полностью в кадре
- нет отражения фотографа, рук, посторонних предметов
- фон чистый, белый или нейтральный
- деривативы соответствуют утверждённому мастеру
- Amazon main: белый фон (#FFFFFF), без логотипов/вотермарков

---

## 11. Идемпотентность

- Каждый batch имеет уникальный `batch_id`
- Входящий файл получает `source_hash`, утверждённый выход — `target_hash`
- Если source hash не изменился — пропустить
- Если мастер существует — регенерировать только отсутствующие деривативы
- Если изменился preset деривата — регенерировать только деривативы

---

## 12. Обработка сбоев

| Сбой | Действие |
|---|---|
| Незначительная проблема фона | `processing` → re-qa |
| Геометрия, блики | `retouch_required` |
| Непригодный файл | `rejected` |
| Amazon main не соответствует | Регенерировать → re-qa |
| BC upload failure | Retry с тем же утверждённым asset |

Не публиковать молча изображение, провалившее QA.

---

## 13. Открытые решения

| Решение | Текущая рекомендация | Почему открыто |
|---|---|---|
| Object storage провайдер | Cloudflare R2 | AWS S3 остаётся совместимым fallback, но launch default уже выбран |
| Background removal default | Локально в Python worker | Порог качества валидируется на пилоте |
| Sampling rate QA | 100% для Amazon main | Нужны данные пилота |
