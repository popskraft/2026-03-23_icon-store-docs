# SPEC-5: Пайплайны — Операционные Workflows A–G

Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.2
Область: все 7 операционных пайплайнов системы (A–G), детальные шаги n8n, триггеры, обработка ошибок

---

## Обзор пайплайнов

```
A — Приём каталога:       CSV/Excel поставщика → PostgreSQL (icons, Слой 1)
B — Обработка изображений: Drive → Python worker → S3/R2 → image_qa_status=approved
C — Генерация контента:   Claude API → review CSV → content_status=approved
D — Публикация BC:        PostgreSQL → BigCommerce Catalog API → bc_status=published
E — Синдикация Amazon:    PostgreSQL → SP-API Listings API → amazon_status=live
F — Инвентаризация:       BC webhook + Orders API poll → site_stock / amazon_reserved
G — Batch Engine:         Сквозная инфраструктура идемпотентности и retry
```

---

## Пайплайн A — Приём каталога

### Входной формат (Google Sheet / CSV)

**Обязательные поля:**

| Колонка | Пример |
|---|---|
| `saint_or_subject` | Mother of God |
| `material` | Wood |
| `technique` | Print |
| `construction_type` | flat-board |
| `size_w_mm` | 130 |
| `size_h_mm` | 180 |
| `base_cost` | 12.50 |

**Необязательные:** `supplier_sku` · `title_raw` · `iconographic_type` · `feast` · `purpose` · `style` · `origin_school` · `size_d_mm` · `weight_g` · `manufacturer` · `notes`

### Нормализация значений

**Материал:** `дерево / wood / wooden board` → `Wood`; `металл / metal / brass / риза / oklad` → `Metal`; остальное → `Other` + флаг `REVIEW_MATERIAL`

**Техника:** `печать / print / UV print / литография` → `Print`; `ручная работа / hand-painted / роспись` → `Hand-painted`; остальное → оставить + флаг `REVIEW_TECHNIQUE`

**Конструкция:** `плоская доска / flat board` → `flat-board`; `складень / триптих / triptych` → `triptych`; `путевая / travel` → `travel`

### Генерация merchant_sku

```sql
CREATE SEQUENCE icons_sku_seq START 1;
CREATE OR REPLACE FUNCTION gen_merchant_sku()
RETURNS TEXT AS $$
  SELECT 'IC-' || LPAD(nextval('icons_sku_seq')::TEXT, 6, '0');
$$ LANGUAGE SQL;
```

Для обновлений — workflow `source-patch`. Загружает только непустые поля. При изменении данных влияющих на контент — запустить `content-generate` заново.

### Сценарии ошибок валидации

| Ошибка | Действие |
|---|---|
| Пустое обязательное поле | Строка пропускается, лог |
| Дубль SKU в файле | Первая принята, дубль пропущен |
| SKU уже есть в PostgreSQL | Skip (защита от перезаписи) |
| Нечисловое значение размера | Строка пропускается |
| Неизвестный материал | Загружается + флаг `REVIEW_MATERIAL` |
| base_cost = 0 или отрицательная | Строка пропускается |

Отчёт n8n: `source-import завершён: N загружено / M пропущено / K ошибок`

### После завершения

Pipeline B не запускается автоматически — оператор проверяет данные, затем запускает обработку фото.

---

## Пайплайн B — Обработка изображений

### Условие запуска

`image_qa_status = 'incoming'`

Credentials Google Drive (Service Account) из Pipeline A используются повторно — отдельная настройка не нужна.

### Три стадии

```
Stage A: Incoming
Владелец → Drive: incoming/{IC-XXXXXX}/raw-*.jpg
    ↓
Оператор QA (чекпоинт ③):
    ├── OK → Drive Trigger запускает Stage B
    └── Проблемы → ТЗ обработчику → approved/{SKU}/ → Stage B

Stage B: Approved Master (Python worker → S3/R2)
    POST http://localhost:8001/process
    Body: { "sku": "IC-000001", "source_uri": "...", "output_bucket": "r2://icons/" }
    - удаление/очистка фона локально в Python worker (при артефактах: флаг REVIEW_BG_FAILED)
    - нормализация: ориентация, sRGB, экспозиция, контраст, padding
    - проверка: ≥ 2000px по длинной стороне
    → approved/{SKU}/master.jpg в S3/R2
    → PostgreSQL: approved_master_uri, image_qa_status = 'approved'

Stage C: Published Derivatives (авто из Stage B)
    amazon-main:    2000×2000, белый фон, JPG q95
    amazon-gallery: 1500×1500, белый фон, JPG q90
    site-pdp:       800×800,   белый фон, WebP q85
    site-card:      400×400,   белый фон, WebP q80
    site-zoom:      1600×1600, белый фон, JPG q90
    thumbnail:      150×150,   fill crop, WebP q70
    → PostgreSQL: derivatives{} ← JSON с URL всех версий
```

### Матрица ракурсов

| Тип конструкции | Минимум Amazon |
|---|---|
| Плоская доска (печать/роспись) | front + detail-frame |
| Металлическая риза | front + detail-riza |
| Триптих / складень | front (открыт) + side (закрыт) |
| Путевая икона | front + back + side |

`media_partial` = front есть, остальное нет → сайт OK, Amazon заблокирован.
`media_blocked` = front отсутствует → все шаги заблокированы.

### Сбои обработки фона

| Ситуация | Статус | Действие оператора |
|---|---|---|
| Локальный background pass дал артефакты | Флаг `REVIEW_BG_FAILED` | Смотреть оригинал, выбрать путь |
| Ошибка worker при очистке фона | `image_qa_status = 'processing'`, `sync_error_code = BG_REMOVAL_FAILED` | Уведомление |

**Пути при `BG_REMOVAL_FAILED`:**
- Фон уже белый → `SET image_qa_status = 'approved'` → перезапустить `image-process`
- Фон сложный → ТЗ обработчику
- Фото низкого качества → `image_qa_status = 'rejected'`, запросить пересъёмку

### Сбой загрузки S3/R2

При `STORAGE_UPLOAD_FAILED`: проверить API Token R2, обновить credentials в n8n, перезапустить — worker выполнит полную обработку заново (операция идемпотентна).

---

## Пайплайн C — Генерация контента

### Условие запуска

```sql
image_qa_status = 'approved' AND content_status = 'not_started'
```

Поле `source_ready` не используется.

### Размер батча и API

- ≤ 10 SKU → Claude Messages API (синхронно)
- > 10 SKU → Claude Message Batches API (15–60 мин, стоимость ÷2)

Рекомендация для MVP — ручной запуск. По расписанию — когда поток товаров регулярный.

### Полный поток

```
n8n отбирает карточки с conditions выше
    ↓
Для каждого SKU: prompt input JSON (факты + глоссарий + правила каналов)
    ↓
Claude API → output JSON (контракт из SPEC-3)
    ↓
n8n валидирует:
    - все обязательные поля заполнены
    - amazon_title ≤ 200 символов
    - backend_search_terms ≤ 249 байт
    - нет prohibited phrases
    ↓
review CSV + JSONL → Drive: reviews/batch_{дата}/
    ↓
Уведомление оператору (email / n8n notification) со ссылкой на папку
    ↓
Оператор: APPROVE / REJECT / REGEN в колонке L CSV
    ↓
APPROVE → content_status = 'approved', запись в PostgreSQL + BC metafields
REJECT  → content_status = 'not_started', поля очищаются
REGEN   → content_status = 'not_started', content_version +1, review_notes сохраняются
```

### CSV review pack — структура

| Колонка | Содержание |
|---|---|
| A — `merchant_sku` | Артикул |
| B — `amazon_title` | Заголовок Amazon |
| C–G — `bullet_1..5` | Буллеты |
| H — `backend_search_terms` | Backend ключевые слова |
| I — `browse_node` | ID категории Amazon |
| J — `site_title` | Заголовок сайта |
| K — `site_description` | Описание сайта |
| **L — `decision`** | **APPROVE / REJECT / REGEN** |
| **M — `review_notes`** | **Причина или пожелания для REGEN** |

При REGEN заметки передаются Claude при следующей генерации:
```
REVISION NOTES FROM OPERATOR: "..."
Please revise the content taking into account the notes above.
```

### Сбои Claude API

| Ситуация | Действие n8n |
|---|---|
| Временный сбой | Retry 3 раза по 5 мин |
| Сбой > 15 мин | `FAILED_WITH_EXCEPTIONS`, уведомление |
| Rate limit | Автоожидание |
| Частичный результат | Успешные записаны, FAILED → retry |

---

## Пайплайн D — Публикация в BigCommerce

### Условие запуска

`content_status = 'approved'`

**Автоматический** (рекомендуется): n8n следит за изменением статуса. **Ручной**: оператор запускает `bc-publish` вручную для группы товаров.

### Последовательность API-вызовов

```
1. Проверить bc_product_id в PostgreSQL:
   НЕТ → POST /v3/catalog/products
   ДА  → PUT /v3/catalog/products/{id}

2. POST /v3/catalog/products
   name, sku, description, price, weight (г → фунты ÷ 453.592),
   categories[], is_visible=false

3. POST /v3/catalog/products/{id}/images
   URL из S3/R2 (BC скачивает сам)
   Порядок: amazon-main первым → главное фото

4. PUT /v3/catalog/products/{id}/variants  (только для размерных SKU)

5. PUT /v3/catalog/products/{id}/metafields  (6 namespaces)

6. PostgreSQL: bc_product_id, bc_status='published', last_synced_at
```

### Маппинг полей PostgreSQL → BigCommerce

| PostgreSQL | BigCommerce | Тип |
|---|---|---|
| `site_title_en` | `name` | Product field |
| `merchant_sku` | `sku` | Product field |
| `site_description_html_en` | `description` | Product field |
| `site_price` | `price` | Product field |
| `weight_g` | `weight` | Product field (÷ 453.592, фунты) |
| `bc_status` | `is_visible` | draft=false, published=true |
| `site_stock` | `inventory_level` | Product field |
| `size_w_mm`, `size_h_mm` | Custom Field «Size» | «130 × 180 mm» |
| `material` / `technique` / `construction_type` | Custom Fields | Display |
| `saint_or_subject` / `origin_school` / `feast` | Custom Fields | Display |
| `meta_title_en` | `page_title` | SEO field |
| `meta_description_en` | `meta_description` | SEO field |
| `amazon_title`, `amazon_bullet_1..5` | Metafield `content.amazon/` | Скрытые |
| `amazon_backend_search_terms`, `amazon_browse_node` | Metafield `content.amazon/` | Скрытые |
| `icon_id`, `asin` | Metafield `catalog.source/` | Скрытые |
| `amazon_status` | Metafield `ops.sync/` | Скрытые |

### Маппинг категорий

Таблица соответствий `saint_or_subject` → BC category ID хранится в n8n workflow `bc-publish` как JSON-объект. Заполняется один раз при настройке. ID категорий — из URL BC Admin.

```json
{
  "saint_map": {
    "Mother of God": [10, 11],
    "Jesus Christ":  [10, 12],
    "Saint Nicholas":[10, 13]
  },
  "purpose_map": {
    "home prayer": [20, 21],
    "travel":      [20, 22]
  },
  "default_category": [10]
}
```

### Draft → Visible

После публикации товар в `is_visible = false`. Перевод после чекпоинта ⑤:
- Ручной: BC Admin → Products → Visible: On
- Bulk: Products → выбрать → Bulk Actions → Set as Visible
- Автоматический: последний шаг `bc-publish` с `is_visible = true` (включить если QA стабильное)

### Вес по умолчанию

Если `weight_g` пустое — n8n использует 300 г и добавляет флаг `REVIEW_WEIGHT_DEFAULT`.

Ориентиры:
| Тип | Примерный вес |
|---|---|
| Плоская доска, малая | 200–350 г |
| Плоская доска, средняя | 350–550 г |
| Плоская доска, большая | 550–950 г |
| Металлическая риза | 400–750 г |
| Путевая / складень | 100–250 г |

### Ошибки BC API

| Ошибка | Действие n8n |
|---|---|
| 429 Rate Limit | Ждёт `Retry-After`, повторяет |
| 404 при PATCH | Переключается на POST |
| 422 Validation | Логирует, пропускает SKU |
| 500 | Retry 3 раза, затем FAILED |
| Ошибка фото | Retry фото, товар создан без него |

---

## Пайплайн E — Синдикация Amazon

> Решение зафиксировано 2026-03-23: PostgreSQL → n8n → SP-API напрямую.

### Условие

`content_status = 'approved'` AND `bc_status = 'published'`

**Запуск всегда ручной** — явное подтверждение оператора (принцип №7 архитектуры).

### SP-API Credentials (настройка один раз)

Получить в SC: Seller ID (`A2XXXXXXXXXXXXX`), Marketplace ID (`ATVPDKIKX0DER`).
Создать LWA App: SC → Apps & Services → Develop Apps → App name: `icon-store-n8n` → сохранить Client ID + Client Secret.
Получить Refresh Token через SC → Manage Apps → Authorize.

В n8n Credentials:
```
Seller ID, Marketplace ID, Client ID, Client Secret, Refresh Token (Atzr|...), AWS Region: us-east-1
```

При ошибке `403 Forbidden` — перевыпустить Refresh Token.

### Shipping Template в n8n

```
shipping_template_id: "icon-store-standard"
```

Передаётся через `merchant_shipping_group_name` в теле SP-API запроса.

### Полный поток

```
Оператор подтверждает список SKU
    ↓
n8n читает из PostgreSQL:
    amazon_title, bullets 1–5, description, backend_search_terms,
    browse_node, item_type_keyword, amazon_price, derivatives{} URLs
    ↓
SP-API: PUT /listings/2021-08-01/items/{sellerId}/{merchant_sku}
Body: { productType, attributes: { title, bullet_point[], brand, ... },
        fulfillment_availability: [{ quantity: amazon_reserved }],
        offers: [{ price: amazon_price }] }
    ↓
Amazon: PENDING (15 мин — 4 ч)
    ↓
n8n поллит каждые 30 мин:
    BUYABLE  → ASIN записан, amazon_status = 'live' ✅
    SUPPRESSED → alert оператору ⚠️
    PENDING > 24ч → alert ⚠️
```

### productType

Все иконы — один hardcoded `productType` в конфиге workflow. Кандидаты: `HOME_DECOR` (основной) или `WALL_ART`. Финальное значение — после первого тестового листинга в SC.

### Browse node

1. Pipeline C (Claude) предлагает `browse_node` → сохраняется в PostgreSQL.
2. Оператор проверяет в колонке I review CSV и при необходимости исправляет.
3. Финальный browse_node тестировать на первых 10 SKU.

### ASIN — когда записывается

```sql
UPDATE icons
SET asin = 'B0XXXXXXXXX', amazon_status = 'live', last_synced_at = NOW()
WHERE merchant_sku = 'IC-000001';
```

n8n также обновляет BC metafield `catalog.source/asin`.

### Ссылка на Amazon с BigCommerce

n8n формирует URL `https://www.amazon.com/dp/{ASIN}` → записывает в BC metafield `catalog.source/amazon_url`. Script Manager отображает кнопку «View on Amazon» на PDP.

### Ошибки SP-API

| Код | Значение | Действие |
|---|---|---|
| `400 Bad Request` | Неверный формат | Проверить title, browse_node, price > 0 |
| `403 Forbidden` | Авторизация | Перевыпустить Refresh Token |
| `429 Throttling` | Rate limit | n8n retry через 60 сек |
| `InvalidInput` | Поле не в схеме productType | Проверить обязательные атрибуты в SC |

**Suppressed после публикации:**

| Причина | Действие |
|---|---|
| Фото отклонено | Перепроверить: белый фон, нет текста/рамок |
| Неверный browse node | Обновить `amazon_browse_node` → retry |
| Цена ниже порога | Повысить `amazon_price` → retry |
| Отсутствует обязательный атрибут | Добавить поле → заполнить → retry |

### FBM: управление заказами

```
n8n poll каждые 30 мин → Orders API
    ↓
Новые заказы → PostgreSQL + уведомление оператору
    ↓
Оператор отгружает → вводит трекинг в n8n-форму (или SC напрямую)
    ↓
n8n → Orders API: подтвердить отгрузку с tracking
    ↓
PostgreSQL: статус заказа, amazon_reserved - 1
```

---

## Пайплайн F — Инвентаризация

> FBM, единый физический склад, два логических резерва.

### Источники событий

- Продажа на сайте: BC webhook `store/order/statusUpdated` → n8n → `site_stock - 1`
- Продажа Amazon FBM: Orders API poll → `amazon_reserved - 1` + уведомление
- Пополнение: n8n-форма → оператор вводит количество + распределение → обновление обоих каналов

### Обновление amazon_reserved через SP-API

```
SP-API: PATCH /listings/2021-08-01/items/{sellerId}/{sku}
Body: { "fulfillment_availability": [{ "fulfillmentChannelCode": "DEFAULT", "quantity": N }] }
```

При `amazon_reserved = 0` листинг показывает «Currently unavailable».

### Алерты

- `site_stock < 3` → предупреждение
- `amazon_reserved < 3` → предупреждение
- `site_stock + amazon_reserved < 2` при обоих активных → срочный алерт
- `site_stock = 0` при активном BC → автоскрытие товара

### BC Webhook — регистрация (один раз)

```
POST https://api.bigcommerce.com/stores/{STORE_HASH}/v2/hooks
Headers: X-Auth-Token: {BC_API_TOKEN}
Body: { "scope": "store/order/statusUpdated", "destination": "https://n8n.example.com/webhook/bc-orders", "is_active": true }
```

Аналогично для `store/product/inventory/updated`.

BC повторяет webhook до 24 часов при недоступности n8n. После > 24 ч — запустить `inventory-sync` вручную.

### Расхождение BC ↔ PostgreSQL

Ежедневная автосверка n8n: BC inventory ≠ PostgreSQL → уведомление.

Шаги:
1. Физически пересчитать склад
2. Выяснить причину (ручное изменение в BC, пропущенный webhook, прямое добавление в BC)
3. Привести к правильному значению через n8n-форму пополнения
4. Зафиксировать в `audit_log`

### Возвраты и отмены

- Отмена Amazon (до отгрузки): `amazon_reserved + 1` автоматически
- Возврат Amazon (после доставки): `amazon_reserved + 1` → оператор решает физически
- Отмена BC: `site_stock + 1` через webhook
- Возврат BC: ручной ввод через форму пополнения (+1 с пометкой «return»)

---

## Пайплайн G — Batch Engine

### Жизненный цикл

```
CREATED → VALIDATING → VALIDATED → RUNNING →
    ├── COMPLETE
    ├── PARTIALLY_COMPLETE
    ├── FAILED_WITH_EXCEPTIONS
    └── ROLLED_BACK
```

### Идемпотентность

Каждый item хранит `source_hash` и `target_hash`. Совпадают → пропустить. Безопасно перезапускать batch повторно.

### Retry

| Попытка | Пауза |
|---|---|
| 2-я | 5 мин |
| 3-я | 15 мин |
| 4-я | 30 мин |
| Далее | FAILED, ждёт оператора |

n8n обрабатывает items последовательно. Claude Batches API — параллельно на стороне Anthropic.

### При PARTIALLY_COMPLETE

1. Получить уведомление: «N success, M failed»
2. SQL:
   ```sql
   SELECT sku, error_code, error_detail
   FROM batch_items
   WHERE batch_id = '<id>' AND status = 'FAILED';
   ```
3. Исправить данные → n8n: «Retry failed items»

### Откат (ROLLED_BACK)

| Тип | Откат |
|---|---|
| Контент | Восстановить из `content_versions` |
| Изображения | Переключить `derivatives_json` на предыдущий набор |
| Цены | Переприменить предыдущий `pricing_rule_id` |
| Публикация BC | `bc_status = 'draft'` |
| Публикация Amazon | `quantity = 0` через PATCH (мягкий) или DELETE (жёсткий); ASIN не очищать |

### Workflows, использующие Batch Engine

| Workflow | Batch Engine |
|---|---|
| `source-import`, `source-patch` | ✅ |
| `content-generate`, `content-review-apply` | ✅ |
| `bc-publish`, `amazon-syndicate` | ✅ |
| `price-update`, `media-refresh` | ✅ |
| `image-process` | ⚠️ Частично |
| `inventory-replenishment` | ❌ Разовая операция |

---

## n8n Workflow Groups

```
catalog_ops:      source-import, price-update
image_ops:        image-ingest, media-refresh
content_ops:      content-generate, content-review-apply, content-republish, seo-refresh
publication_ops:  bc-publish, amazon-syndicate
inventory_ops:    inventory-sync, amazon-orders-poll
monitoring_ops:   alert-low-stock, batch-health-check
```
