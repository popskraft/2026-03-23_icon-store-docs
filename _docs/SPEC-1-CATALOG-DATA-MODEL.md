# SPEC-1: Модель данных каталога

Статус: рабочий черновик (implementation-facing)
Базис: `MASTER-ARCHITECTURE-v4.md` v4.3
Область: канонический реестр каталога, политика идентификаторов, владение полями, маппинг каналов, контракты импорта и обновлений

---

## Часть A. Главное

### 0.1 Краткое резюме

- PostgreSQL on Supabase is the only source of truth.
- Owner writes Layer 1 facts directly in Supabase Table Editor.
- CSV / Excel are allowed only as supplier bulk-import inputs, not as owner source-of-truth tools.
- Batch content review is done in CSV; Table Editor is for single-SKU fixes and exception handling.
- Publish requires explicit gates, not just a generic `approved` status.
- Amazon launch requires confirmed GTIN exemption and validated browse node before the first working batch publish.

### 0.2 Ключевые контракты

| Contract | Rule |
|---|---|
| Record identity | `icon_id` is immutable; `merchant_sku` is the main business key |
| File identity | `icon_id` + immutable source file IDs win over folder names |
| Batch review | `approval_source = csv_batch` for normal batch review; `table_editor_exception` only for exception handling |
| Channel publish | `images_approved`, `content_approved`, `price_ready`, `inventory_ready`, and channel-specific approval flags are required |
| Amazon gate | GTIN exemption confirmed, browse node validated, payload hash approved |
| Sync safety | external API side effects run through durable sync states and reconciliation |

---

## Часть B. Детальный контракт

## 1. Назначение

Канонические записи живут в PostgreSQL (Supabase). BigCommerce и Amazon получают только производные публикационные представления. Все нижестоящие процессы обязаны следовать этому контракту.

---

## 2. Принципы проектирования

1. Одна каноническая запись на каждый SKU.
2. PostgreSQL остаётся единственным источником истины.
3. Идентификаторы поставщика хранятся как внешний reference, а не системный ключ.
4. Batch review контента идёт через CSV; Table Editor используется только для exception handling.
5. Publish разрешается только при полном наборе explicit gate flags.
6. Любой внешний publish должен быть idempotent и reconciliation-safe.
7. Изменение базовых фактов, изображений, цены или stock должно инвалидировать downstream readiness там, где это нужно.

---

## 3. Ограничения платформы

### BigCommerce

| Ограничение | Следствие |
|---|---|
| V3 Catalog API | основной integration surface |
| Variants — SKU-bearing с inventory | variants только для реальных inventory-различий |
| Modifiers не меняют fulfillment SKU | не использовать modifiers для инвентарных выборов |
| Custom fields ≤ 250 символов | только compact display data |
| Inventory writes асинхронны | обновления инвентаря сериализуются |
| Rate limit: 150 req / 30 сек | batch writers делают backoff |

### Amazon SP-API

| Ограничение | Следствие |
|---|---|
| Listings Items API — основной путь | `n8n -> SP-API` напрямую |
| GTIN Exemption для handmade religious | hard prerequisite до первого publish |
| Product Type Definitions API | browse node и required attributes проверяются до submit |
| FBM | inventory и publish идут из PostgreSQL, не из Seller Central вручную |

---

## 4. Архитектура

### 4.1 Роли систем

| Система | Роль |
|---|---|
| PostgreSQL (Supabase) | канонический реестр и sync-control plane |
| Supabase Table Editor | owner input, single-record correction, exception handling |
| Google Drive | inbox входящих фото, только handoff |
| S3/R2 | approved masters и derivatives |
| BigCommerce | storefront publication layer |
| Amazon Seller Central | marketplace presence + FBM operations |
| n8n | orchestration, retries, reconciliation |
| Claude API | Layer 2 content generation |

### 4.2 Поток данных

```text
Owner direct entry in Supabase Table Editor or operator-led supplier import
  -> PostgreSQL
  -> n8n
  -> Claude API (Layer 2)
  -> operator review (CSV batch review; Table Editor only for exceptions)
  -> BigCommerce (n8n -> BC API)
  -> Amazon (n8n -> SP-API)
```

---

## 5. Политика идентификаторов

Каждая запись иконы содержит:
- `icon_id` — внутренний неизменяемый UUID
- `merchant_sku` — первичный бизнес-ключ (`IC-000001`, при необходимости с variant suffix)
- `supplier_sku` — внешний reference на поставщика
- `product_family_id` — группировка связанных вариантов
- `bc_product_id`, `bc_variant_id` — появляются после публикации
- `asin` — появляется после Amazon listing

Правила:
- семантика не кодируется в `merchant_sku`
- удалённый SKU не переиспользуется
- папка и имя файла не являются источником истины

---

## 6. Icon Master Record — структура полей

### 6.1 Идентификаторы

`icon_id` · `merchant_sku` · `supplier_sku` · `product_family_id` · `asin` · `bc_product_id` · `bc_variant_id`

### 6.2 Базовые факты (Layer 1 — owner)

| Поле | Описание |
|---|---|
| `title_raw` | рабочее название от поставщика |
| `saint_or_subject` | кто или что изображено |
| `feast` | праздник или память |
| `purpose` | назначение иконы |
| `style` | стилистическая метка |
| `iconographic_type` | базовый иконографический тип |
| `material` | основной материал |
| `technique` | техника исполнения |
| `construction_type` | flat-board / triptych / travel и т.д. |
| `origin_school` | школа или традиция |
| `size_w_mm`, `size_h_mm`, `size_d_mm` | размеры в мм |
| `weight_g` | вес в граммах |
| `manufacturer` | автор или мастерская |
| `base_cost` | закупочная стоимость |
| `notes` | операционные заметки |

### 6.3 Медиа

| Поле | Описание |
|---|---|
| `source_system` | `google_drive` / `email` / `s3_upload` / `manual` |
| `source_folder_id`, `source_file_id`, `source_filename` | immutable source references |
| `received_at`, `incoming_hash` | время приёма и хэш |
| `approved_master_uri`, `approved_master_hash` | approved master в S3/R2 |
| `derivatives` | JSON: derivative name -> URI |
| `image_qa_status` | `incoming` / `retouch_required` / `processing` / `media_blocked` / `media_partial` / `approved` / `rejected` |
| `image_qa_notes`, `last_processed_at` | QA notes и время обработки |

### 6.4 Контент (Layer 2 — AI enrichment)

| Поле | Описание |
|---|---|
| `content_version`, `content_status` | `not_started` / `generating` / `needs_review` / `approved` / `published` |
| `approved_by`, `approved_at`, `approval_source`, `review_notes` | трассировка approval |
| `site_title_en`, `site_description_html_en` | storefront content |
| `meta_title_en`, `meta_description_en` | SEO |
| `amazon_title`, `amazon_bullet_1..5` | Amazon title and bullets |
| `amazon_description_html`, `amazon_backend_search_terms` | Amazon description and keywords |
| `amazon_browse_node`, `amazon_item_type_keyword`, `amazon_target_marketplace` | Amazon categorization |
| `amazon_payload` | JSON payload for SP-API |
| `category_primary`, `faceted_filters` | internal category and filters |
| `compliance_flags` | review flags |

### 6.5 Ценообразование

`pricing_rule_id` · `site_price` · `amazon_price` · `map_price` · `price_last_updated`

### 6.6 Инвентарь и публикация

| Поле | Описание |
|---|---|
| `physical_on_hand`, `site_stock`, `site_reserved`, `amazon_reserved`, `committed`, `released` | lifecycle физического stock, site allocation и логических резервов |
| `available_to_allocate`, `inventory_version` | расчёт доступности и optimistic locking |
| `reorder_point`, `supplier_lead_time_days` | пополнение |
| `images_approved`, `content_approved`, `price_ready`, `inventory_ready` | readiness flags |
| `bc_publish_approved`, `amazon_publish_approved` | channel approvals |
| `approved_payload_hash`, `published_payload_hash` | контроль publish drift |
| `bc_status` | `draft` / `published` / `archived` |
| `amazon_status` | `not_listed` / `pending` / `live` / `suppressed` / `live_manual` |
| `sync_state`, `workflow_run_id`, `step_name`, `step_attempt`, `idempotency_key` | durable sync contract |
| `bc_sync_version`, `amazon_sync_version`, `last_hash`, `last_synced_at`, `last_batch_id` | sync audit and versioning |
| `sync_error_code`, `sync_error_message` | текущая ошибка |

---

## 7. Владение полями

| Группа | Кто владеет | Что можно делать | Что блокирует / инвалидирует |
|---|---|---|---|
| Layer 1 базовые факты | Owner | прямой ввод и правка до content approval | изменение после content approval сбрасывает derived content и publish-ready |
| AI content fields | System | генерирует и обновляет только через workflow | не публикуется без human approval |
| Approval и override fields | Operator | approve, reject, exception edits, channel gates | существенное изменение данных сбрасывает gates |
| Sync fields | System | ведёт sync states и версии | ручная правка только для incident repair |
| Inventory fields | System + operator under SOP | авто sync и ручная корректировка по runbook | конфликт версий требует reconcile before publish |

Правило инвалидации:
- изменение owner input после `content_approved = true` сбрасывает `content_approved`, `bc_publish_approved`, `amazon_publish_approved`
- изменение approved image сбрасывает image-derived readiness
- изменение цены или stock сбрасывает `approved_payload_hash` для зависимых каналов

---

## 8. Маппинг BigCommerce

### 8.1 Product object

| Поле BC | Источник |
|---|---|
| `name` | `site_title_en` |
| `sku` | `merchant_sku` |
| `price` | `site_price` |
| `weight` | `weight_g` converted to BC units |
| `page_title` | `meta_title_en` |
| `meta_description` | `meta_description_en` |
| `custom_fields` | compact customer-visible facts |
| `metafields` | operational data by namespace |

Rules:
- variant only if selection changes SKU, price, inventory, or fulfillment
- modifier only if selection is display-only
- BigCommerce stores published copies, not canonical masters

---

## 9. Batch contract and idempotency

### 9.1 Типы batch

`source-import` · `image-ingest` · `content-generate` · `content-republish` · `bc-publish` · `amazon-syndicate` · `price-update` · `inventory-sync` · `media-refresh`

### 9.2 Порядок выполнения

`source-import` -> `image-ingest` -> `content-generate` -> `bc-publish` -> `amazon-syndicate` -> `price-update` -> `inventory-sync`

### 9.3 Правила идемпотентности

- повторный запуск с тем же input hash безопасен
- неизменённые строки дают no-op
- строка с validation error не роняет весь batch
- `idempotency_key` обязателен на шаге публикации и channel update
- `sync_state` использует: `pending_sync` / `sync_in_progress` / `sync_succeeded` / `sync_failed_transient` / `sync_failed_terminal`
- внешний publish считается завершённым только после reconciliation step

---

## 10. Ворота валидации

### 10.1 Для создания записи

`merchant_sku` · `title_raw` · `saint_or_subject` · `material` или `technique` · `size_w_mm` · `size_h_mm`

### 10.2 Для публикации в BigCommerce

`images_approved = true` · `content_approved = true` · `price_ready = true` · `inventory_ready = true` · `bc_publish_approved = true`

### 10.3 Для публикации в Amazon

storefront gate + `amazon_publish_approved = true` · approved Amazon title + bullets · Amazon-compatible main image · validated browse node · item type keyword · GTIN exemption confirmed · approved payload hash

---

## 11. Constraints And Auditability

### 11.1 Required constraints

- `UNIQUE (merchant_sku)`
- `UNIQUE (asin) WHERE asin IS NOT NULL`
- `UNIQUE (bc_product_id) WHERE bc_product_id IS NOT NULL`
- non-negative checks for stock, reserves, prices, and base cost
- schema validation or equivalent contract for `derivatives` and `amazon_payload`

### 11.2 Таблицы PostgreSQL

`icons` · `pricing_rules` · `pricing_snapshots` · `inventory_snapshots` · `batch_runs` · `batch_items` · `content_versions` · `audit_log` · `icon_status_audit` · `alert_events`

`icon_status_audit` logs:
- actor
- action
- old_value
- new_value
- reason
- timestamp
- workflow_run_id

---

## 12. Открытые решения

1. Один SKU на размер или family с size variants?
2. Price Lists на запуске или отложить?
3. Варианты рамки или оклада — variants или отдельные SKU?
