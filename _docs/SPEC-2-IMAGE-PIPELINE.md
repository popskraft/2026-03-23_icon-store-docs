# SPEC-2: Пайплайн обработки изображений

Версия: 3.0
Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.3
Область: приём изображений, handoff, обработка, хранение approved masters, генерация derivatives, QA

---

## Часть A. Главное

### 0.1 Краткое резюме

- Google Drive is inbox only.
- One approved master per SKU or variant.
- Production storage is R2/S3, not Drive.
- Image identity is held by `icon_id` and immutable source file IDs, not by folder renaming.
- Operator owns image QA; processor edits pixels but does not approve publication.

### 0.2 Каноническая последовательность handoff

1. Owner deposits source files.
2. Operator validates intake.
3. Operator links files to `icon_id` and assigns `merchant_sku`.
4. Processor or Python worker edits against that record.
5. Operator accepts or rejects the result.
6. System ingests the approved master and generates derivatives.

---

## Часть B. Детальный контракт

## 1. Назначение

Один approved master на SKU. Всё остальное генерируется из него. Drive нужен только для входящего handoff, production assets живут в R2/S3.

---

## 2. Принципы

1. Не менять реальный внешний вид иконы.
2. Не использовать generative enhancement.
3. Approved master один на SKU или variant.
4. Derivatives генерируются детерминированно.
5. Batch-safe и restartable behaviour обязателен.
6. Cloudinary не используется.

---

## 3. Роли

| Роль | Ответственность |
|---|---|
| Supplier | присылает исходники |
| Owner | передаёт материалы и подтверждает товар к запуску |
| Processor | нормализует и ретуширует изображения |
| Operator | intake validation, QA, accept/reject, publication readiness |
| System | ingestion, derivative generation, storage writes |

---

## 4. Image identity and storage

### 4.1 Identity rule

- canonical link: `icon_id`
- file references: `source_file_id`, `incoming_hash`, `approved_master_hash`
- SKU path and folder name are operational aliases

### 4.2 Storage layout

```text
incoming/{source_system}/{batch_id}/{source_file_id}/{source_filename}
approved/{merchant_sku}/master.jpg
derived/{merchant_sku}/amazon-main.jpg
derived/{merchant_sku}/amazon-gallery-01.jpg
derived/{merchant_sku}/site-pdp.webp
derived/{merchant_sku}/site-card.webp
derived/{merchant_sku}/site-zoom.jpg
qa/{batch_id}/{merchant_sku}/contact-sheet.jpg
archive/{batch_id}/manifest.json
```

Rules:
- `incoming` is immutable
- database is the source of truth, not the path
- derivatives are generated from approved master only

---

## 5. Canonical statuses

| Статус | Значение |
|---|---|
| `incoming` | файл получен, intake не завершён |
| `retouch_required` | нужен ручной processor pass |
| `processing` | worker запущен |
| `media_blocked` | нет обязательного front image |
| `media_partial` | сайт возможен, Amazon blocked |
| `approved` | approved master принят оператором |
| `rejected` | нужна пересъёмка или полный redo |

Internal milestones `approved_master` and `derivatives_ready` допустимы в workflow logs, но не заменяют канонический record status.

---

## 6. Processing stages

### 6.1 Stage A: Intake

- owner deposits source files
- operator validates completeness and file-to-record match
- operator assigns or confirms `merchant_sku`
- missing or unusable front image -> `media_blocked`

### 6.2 Stage B: Approved master

Worker or processor does:
- orientation normalization
- sRGB normalization
- exposure / contrast cleanup
- white or neutral background cleanup
- padding and containment
- minimum long side: 2000 px

Operator then decides:
- accept -> `image_qa_status = approved`
- reject for retouch -> `retouch_required`
- reject for reshoot -> `rejected`

### 6.3 Stage C: Derivatives

Generated automatically from the approved master:
- `amazon-main` 2000x2000 JPG
- `amazon-gallery` 1500x1500 JPG
- `site-pdp` 800x800 WebP
- `site-card` 400x400 WebP
- `site-zoom` 1600x1600 JPG
- `thumbnail` 150x150 WebP

---

## 7. Angle matrix

| Construction type | Minimum for Amazon | If incomplete |
|---|---|---|
| flat-board | front + detail-frame | `media_partial` |
| metal oklad | front + detail-riza | `media_partial` |
| triptych | front-open + side-closed | `media_partial` |
| travel icon | front + back + side | `media_partial` |

If front image is missing, set `media_blocked`.

---

## 8. QA rules

Allowed:
- EXIF orientation
- color normalization to sRGB
- mild exposure and contrast correction
- resize, sharpen, padding, format conversion

Forbidden:
- recoloring the icon
- generative fill
- synthetic decorative backgrounds
- reconstruction of missing visual content

Amazon main image must have:
- white background
- no logo or watermark
- object fully visible

---

## 9. Failure handling

| Failure | Action |
|---|---|
| minor background issue | re-run or manual QA decision |
| edge artifacts | `retouch_required` |
| geometry or glare problem | `retouch_required` |
| image too small or unusable | `rejected` |
| storage upload failure | retry with same approved source |

Rule: never silently publish an image that failed QA.

---

## 10. Idempotency

- each batch has `batch_id`
- input file gets `incoming_hash`
- approved master gets `approved_master_hash`
- unchanged source hash -> skip
- existing master -> regenerate only missing or outdated derivatives

---

## 11. Открытые вопросы

1. Нужно ли 100% QA для всех derivatives или только для Amazon main и approved master?
2. Какой процент сложных кейсов потребует ручного processor flow на пилоте?
