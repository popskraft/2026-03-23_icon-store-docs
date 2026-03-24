# SPEC-1: Модель данных каталога

Статус: рабочий черновик (implementation-facing)
Базис: `MASTER-ARCHITECTURE-v4.md` v4.2
Область: канонический реестр каталога, политика идентификаторов, владение полями, маппинг каналов, контракты импорта и обновлений

---

## 1. Назначение

Канонические записи живут в PostgreSQL (Supabase). BigCommerce — слой публикации. Amazon SC — второй выходной канал. Все нижестоящие процессы (импорт, image processing, генерация контента, публикация, синдикация, batch-обновления) обязаны следовать этому контракту.

---

## 2. Принципы проектирования

1. Одна каноническая запись на каждый SKU.
2. Единая политика идентификаторов во всех системах.
3. Идентификаторы поставщика хранятся как внешняя ссылка, не становятся системным ключом.
4. BigCommerce хранит только поля для commerce-публикации.
5. Мастер-изображения и деривативы хранятся за пределами BigCommerce.
6. Весь публичный текст генерируется из канонической записи и проходит human review.
7. Пакетные операции идемпотентны и воспроизводимы.
8. Двухслойная модель: Слой 1 — владелец вводит базовые факты; Слой 2 — n8n → Claude API обогащает контентом.
9. PostgreSQL — единственный источник истины.

---

## 3. Ограничения платформы

### BigCommerce

| Ограничение | Следствие |
|---|---|
| Продукты через V3 Catalog API | V3 — основной integration surface |
| Продукт в нескольких категориях | Хранить `primary_category_id`, BC использует множественное назначение |
| Variants — SKU-bearing с inventory | Variants для размерных вариантов |
| Modifiers не меняют fulfillment SKU | Не использовать modifiers для инвентарных выборов |
| Custom fields ≤ 250 символов | Только display-данные |
| Metafields — скрытые | Для sync state, source refs, channel payloads |
| Inventory API writes асинхронны | Сериализовать обновления инвентаря |
| Rate limit: 150 req / 30 сек | Batch writers обязаны делать back off |
| Bulk price-list upserts сериализованы | Не запускать параллельные bulk price операции |

### Amazon SP-API

| Ограничение | Следствие |
|---|---|
| Listings Items API — основной путь | n8n → SP-API напрямую, без BC-коннектора |
| GTIN Exemption для handmade religious | Получить до первого листинга (SPEC-4) |
| FBM: продавец отгружает сам | `amazon_reserved` — логический резерв в PostgreSQL |
| Атрибуты через Product Type Definitions API | Валидация схемы перед листингом |

---

## 4. Архитектура

### 4.1 Роли систем

| Система | Роль |
|---|---|
| **PostgreSQL (Supabase)** | Канонический реестр. Источник истины. |
| **Supabase Table Editor** | UI для владельца (Слой 1) и оператора (review Слоя 2) |
| Google Drive | Inbox входящих фото. Только handoff. |
| S3/R2 | Утверждённые мастера и деривативы |
| BigCommerce | Сторфронт (publication layer) |
| Amazon Seller Central | Листинги + FBM |
| n8n | Оркестрация batch |
| Claude API | Генерация контента (Слой 2) |

### 4.2 Поток данных

```
Владелец (Table Editor Слой 1) → PostgreSQL → n8n → Claude API (Слой 2) →
→ Оператор review (Table Editor) → BigCommerce (n8n → BC API) + Amazon (n8n → SP-API)
```

---

## 5. Политика идентификаторов

Каждая запись иконы содержит:
- `icon_id` — внутренний неизменяемый UUID
- `merchant_sku` — первичный бизнес-ключ (`IC-000001`, с суффиксом `-S/M/L` для размерных вариантов)
- `supplier_sku` — внешняя ссылка на поставщика, не PK
- `product_family_id` — группировка вариантов одного дизайна
- `bigcommerce_product_id`, `bigcommerce_variant_id` — назначаются после публикации
- `asin` — назначается после создания Amazon листинга

**Правила merchant_sku:** не кодировать семантику в SKU; не повторно использовать SKU удалённых товаров.

---

## 6. Icon Master Record — структура полей

### 6.1 Идентификаторы
`icon_id` · `merchant_sku` · `supplier_sku` · `product_family_id` · `asin` · `bigcommerce_product_id` · `bigcommerce_variant_id`

### 6.2 Базовые факты (Слой 1 — владелец)

| Поле | Описание |
|---|---|
| `title_raw` | Рабочее название от поставщика |
| `saint_or_subject` | Кто или что изображено |
| `feast` | Праздник или память |
| `purpose` | Назначение иконы |
| `style` | Стилистическая метка |
| `iconographic_type` | Mother of God / Christ / Saint / Archangel и т.д. |
| `material` | Основной материал |
| `technique` | Техника исполнения |
| `construction_type` | flat-board / triptych / travel и т.д. |
| `origin_school` | Школа или традиция |
| `size_w_mm`, `size_h_mm`, `size_d_mm` | Размеры в мм |
| `weight_g` | Вес в граммах |
| `manufacturer` | Автор или мастерская |
| `base_cost` | Закупочная стоимость |
| `notes` | Произвольные операционные заметки |

### 6.3 Медиа

| Поле | Описание |
|---|---|
| `source_system` | `google_drive` / `email` / `s3_upload` / `manual` |
| `source_folder_id`, `source_file_id`, `source_filename` | Ссылка на источник |
| `received_at`, `incoming_hash` | Время приёма и хэш |
| `approved_master_uri`, `approved_master_hash` | Утверждённый мастер в S3/R2 |
| `derivatives` | JSON: имя деривата → URI |
| `image_qa_status` | `incoming` / `retouch_required` / `processing` / `media_blocked` / `media_partial` / `media_ready` / `approved` / `rejected` |
| `image_qa_notes`, `last_processed_at` | QA заметки и время обработки |

### 6.4 Контент (Слой 2 — AI-обогащение)

| Поле | Описание |
|---|---|
| `content_version`, `content_status` | `not_started` / `generating` / `review` / `approved` / `published` |
| `approved_by`, `approved_at`, `review_notes` | Трассировка approval |
| `site_title_en`, `site_description_html_en` | Контент сторфронта |
| `meta_title_en`, `meta_description_en` | SEO поля |
| `amazon_title`, `amazon_bullet_1..5` | Amazon title и буллеты |
| `amazon_description_html`, `amazon_backend_search_terms` | Amazon описание и ключевые слова |
| `amazon_browse_node`, `amazon_item_type_keyword`, `amazon_target_marketplace` | Категоризация Amazon |
| `amazon_payload` | Полный JSON-payload для SP-API |
| `category_primary`, `faceted_filters` | Внутренняя категория и фасеты |
| `compliance_flags` | Флаги review для рискованных утверждений |

### 6.5 Ценообразование

`pricing_rule_id` · `site_price` · `amazon_price` · `map_price` · `price_last_updated`

### 6.6 Инвентарь и публикация

| Поле | Описание |
|---|---|
| `site_stock`, `amazon_reserved` | Логические резервы (единый физический склад) |
| `reorder_point`, `supplier_lead_time_days` | Пополнение |
| `bc_status` | `draft` / `published` / `archived` |
| `amazon_status` | `not_listed` / `pending` / `live` / `suppressed` / `live_manual` |
| `last_hash`, `last_synced_at`, `last_batch_id` | Sync audit |
| `sync_error_code`, `sync_error_message` | Текущая ошибка |

---

## 7. Владение полями

| Группа | PostgreSQL | BigCommerce | Amazon |
|---|---|---|---|
| Идентификаторы | ✅ источник | SKU поля | Listing SKU / ASIN |
| Базовые факты (Слой 1) | ✅ источник | Покупательское подмножество | Подмножество payload |
| Мастера URI | ✅ + S3/R2 | Не источник | Не источник |
| Деривативы URI | ✅ + S3/R2 | Published copies | Image payload |
| Контент (Слой 2) | ✅ источник | name, description, SEO, metafields | title, bullets, keywords |
| Инвентарь | ✅ источник | `site_stock` | `amazon_reserved` |
| Цены | ✅ источник | Product price | Listing price |
| Sync state | ✅ источник | `ops.sync` metafields | Listing status refs |

---

## 8. Маппинг BigCommerce

### Product object

| Поле BC | Источник |
|---|---|
| `name` | `site_title_en` |
| `sku` | `merchant_sku` |
| `price` | `site_price` |
| `weight` | `weight_g` → конвертировать в единицы BC |
| `page_title` | `meta_title_en` |
| `meta_description` | `meta_description_en` |
| `custom_fields` | Компактные видимые покупателю факты (≤ 250 символов) |
| `metafields` | Операционные данные по неймспейсам |

**Variants vs Modifiers:** Variant — если выбор меняет SKU, инвентарь, цену или fulfillment. Modifier — только если выбор видим покупателю, но не меняет инвентарь.

**Metafield namespaces:** `ops.review` · `ops.sync` · `media.storage` · `content.amazon` · `content.site` · `catalog.source`

**Изображения:** BigCommerce хранит только published copies. Master URI и хэши — в PostgreSQL.

**Инвентарь:** только `site_stock`. `amazon_reserved` — только в PostgreSQL и Amazon state. Сериализовать inventory writes.

---

## 9. Batch: типы, контракт, порядок

### Типы batch

| Тип | Назначение |
|---|---|
| `source-import` | Нормализовать данные поставщика в master record |
| `image-ingest` | Принять и утвердить image assets |
| `content-generate` | Генерировать контент (Слой 2) |
| `content-republish` | Пересобрать утверждённые версии |
| `bc-publish` | Создать или обновить продукты в BigCommerce |
| `amazon-syndicate` | Подготовить или обновить Amazon listing payloads |
| `price-update` | Применить rule-driven изменения цен |
| `inventory-sync` | Согласовать остатки |
| `media-refresh` | Перегенерировать деривативы из утверждённых мастеров |

### Порядок выполнения

`source-import` → `image-ingest` → `content-generate` → `bc-publish` → `amazon-syndicate` → `price-update` → `inventory-sync`

### Правила идемпотентности

- Повторный запуск с тем же input hash — безопасен.
- Неизменённые строки — no-op.
- Строка с ошибкой валидации отклоняется, не останавливает batch.
- Back off на 429; сериализовать bulk inventory и price-list upserts.

---

## 10. Ворота валидации

**Для создания записи:** `merchant_sku` · `title_raw` · `saint_or_subject` · `material` или `technique` · `size_w_mm`, `size_h_mm`

**Для публикации в сторфронт:** approved English title + description · publishable image · storefront category · valid `site_price` · `bc_status = published`

**Для Amazon:** approved Amazon title + bullets · Amazon-совместимое main image (белый фон) · browse node · item type keyword · compliance review завершён

**Жёсткие ограничения:** custom fields ≤ 250 символов; цена decimal precision 4 знака; не использовать modifiers для инвентарных выборов.

---

## 11. Таблицы PostgreSQL

`icons` · `pricing_rules` · `pricing_snapshots` · `inventory_snapshots` · `batch_runs` · `batch_items` · `content_versions` · `audit_log` · `alert_events`

---

## 12. Открытые решения

1. Один SKU на размер или family с size variants?
2. Price Lists на запуске или отложить?
3. Варианты рамки/оклада — variants или отдельные SKU?
