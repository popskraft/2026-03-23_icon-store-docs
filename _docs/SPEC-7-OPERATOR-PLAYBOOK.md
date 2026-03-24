# SPEC-7: Плейбук оператора

Версия: 2.0
Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.2
Область: end-to-end процедуры оператора, статусная матрица, ежедневная рутина, SQL quick reference, обработка ошибок

---

## Архитектурный контекст (v4.2)

- Оператор управляет системой через n8n UI, Supabase Table Editor и SQL.
- Два обязательных human gate: review контента (Пайплайн C) и подтверждение Amazon-публикации (Пайплайн E).
- Все пайплайны идемпотентны: повторный запуск безопасен.
- PostgreSQL — источник истины; BC и Amazon — production channels.

---

## 1. Матрица статусов

```
[НОВЫЙ SKU]
    ↓  Пайплайн A
image_qa_status = 'incoming'
content_status  = 'not_started'
bc_status       = 'draft'
amazon_status   = 'not_listed'

    ↓  Пайплайн B (авто + ручной QA)
image_qa_status = 'approved'          ← разблокирует генерацию контента

    ↓  Пайплайн C (авто + оператор → APPROVE)
content_status  = 'approved'          ← разблокирует публикацию в оба канала

    ↓  Пайплайн D (авто)
bc_status       = 'published'         ← товар виден на сайте

    ↓  Пайплайн E (авто после финального approve)
amazon_status   = 'live'              ← листинг активен на Amazon
```

**Правило блокировки:**

| Нужно для BigCommerce | Нужно для Amazon |
|-----------------------|-----------------|
| `image_qa_status = approved` | `image_qa_status = approved` |
| `content_status = approved` | `content_status = approved` |
| — | `bc_status = published` |
| — | GTIN Exemption одобрен |

---

## 2. Разовая настройка (перед первым SKU)

```
□ Развернуть VPS: Docker Compose с n8n + PostgreSQL
□ Применить DDL из SPEC-1-CATALOG-DATA-MODEL.md
□ Создать S3/R2 бакет: incoming/ · approved/ · derived/
□ Настроить credentials в n8n:
    - BigCommerce API key (Catalog r/w, Webhook w)
    - Amazon SP-API (Listings w, Orders r/w)
    - S3/R2 (upload, read, list)
    - Claude API (Messages + Batches)
    - Remove.bg API key
□ Подключить Google Drive → n8n webhook на incoming/
□ Получить GTIN Exemption в Amazon Seller Central
□ Прогнать тест 1 SKU end-to-end (без публикации)
□ Настроить email/Slack алерты в n8n
```

---

## 3. Шаг 1 — Загрузка каталога (Пайплайн A)

**Триггер:** получен CSV/Excel от поставщика.

1. Проверить: обязательные колонки — `title_raw`, `saint_or_subject`, `material`, `size_w_mm`, `size_h_mm`.
2. n8n → запустить `source-import`, указать файл.
3. Сводка: `N created / M skipped / K errors`. При ошибках — открыть `batch_items` (статус ERROR), исправить, перезапустить.

```sql
-- Проверка: все новые SKU должны иметь image_qa_status='incoming'
SELECT merchant_sku, image_qa_status, content_status
FROM icons
WHERE created_at > NOW() - INTERVAL '1 hour'
ORDER BY merchant_sku;
```

---

## 4. Шаг 2 — Загрузка изображений (Пайплайн B)

1. Создать папку Google Drive: `incoming/{IC-000001}/`.
2. Загрузить исходные фото (raw-001.jpg, raw-002.jpg …).
3. n8n-webhook запускает `image_worker.stage_a_to_b()` автоматически.
4. При успехе: `image_qa_status = approved`, деривативы генерируются.
5. При ошибке: алерт с кодом.

| Код ошибки | Действие |
|------------|---------|
| `IMG_TOO_SMALL` | Запросить пересъёмку ≥ 2000px |
| `IMG_BG_REMOVAL_FAILED` | Отправить ретушёру, вернуть в approved/ |
| `IMG_NO_SUBJECT` | Переснять |
| `IMG_COLOR_CAST` | Approve/reject вручную через SQL |

```sql
-- Ручной approve
UPDATE icons
SET image_qa_status = 'approved',
    image_qa_notes  = 'Manual approve: minor color cast, acceptable'
WHERE merchant_sku = 'IC-000001';
```

---

## 5. Шаг 3 — Генерация контента (Пайплайн C, часть 1)

**Триггер:** `image_qa_status = 'approved'` AND `content_status = 'not_started'`. Авто по расписанию или ручной запуск в n8n.

n8n: формирует JSON фактов → Claude API генерирует → валидирует lengths/compliance → отправляет `review_{date}.csv` оператору.

---

## 6. Шаг 4 — Review контента (Пайплайн C, human gate)

CSV-колонки: `sku | amazon_title | bullet_1..3 | site_title | compliance_flags | review_action | review_notes`

1. Приоритет — строки с непустыми `compliance_flags`.
2. Проверить `amazon_title`: ключевое слово, ≤ 200 символов, Title Case.
3. Bullets: польза в начале, < 300 символов каждый.
4. Заполнить `review_action`:

| Действие | Когда |
|----------|-------|
| `APPROVE` | Всё корректно |
| `REJECT` | Серьёзная ошибка, нужна ручная правка |
| `REGEN` | Слабый контент — регенерировать |

5. Сохранить CSV → загрузить в `review-inbox/` в Drive (или через n8n-форму).

После обработки:
- `APPROVE` → `content_status = 'approved'`
- `REJECT` → `content_status = 'not_started'`
- `REGEN` → `content_status = 'not_started'`, `content_version` +1

---

## 7. Шаг 5 — Публикация в BigCommerce (Пайплайн D)

Полностью автоматически после `content_status → approved`. n8n workflow `bc-publish`: PostgreSQL → BC API → `bc_status = 'published'`.

```sql
SELECT merchant_sku, bc_product_id, bc_status, last_synced_at
FROM icons
WHERE bc_status = 'published'
ORDER BY last_synced_at DESC
LIMIT 20;
```

---

## 8. Шаг 6 — Публикация на Amazon (Пайплайн E, human gate)

**Условие:** `content_status = 'approved'` AND `bc_status = 'published'`.

1. n8n отправляет сводку готовых SKU (email / n8n UI).
2. Оператор проверяет `amazon_browse_node` и `amazon_price`.
3. Подтверждает → `amazon-syndicate` запускает `PUT /listings/2021-08-01/items/{sellerId}/{sku}`.
4. ASIN записывается в PostgreSQL + BC metafield, `amazon_status = 'live'`.
5. Визуальная проверка в Amazon Seller Central (фото, цена, категория).

**Suppressed листинг:**
1. SC → Inventory → Fix stranded listings.
2. Проверить: compliance error, price alert, image issue.
3. Исправить в PostgreSQL → перезапустить `amazon-syndicate` для SKU.
4. Категориальная ошибка → обновить `amazon_browse_node` → regenerate.

---

## 9. Ежедневная рутина

**Утром (5–10 мин):**
```
□ n8n Dashboard → Executions → failed workflows
□ batch_runs WHERE status IN ('FAILED_WITH_EXCEPTIONS', 'PARTIALLY_COMPLETE')
□ Алерты инвентаря: site_stock < 3 OR amazon_reserved < 3
□ Amazon SC → Manage Inventory → suppressed листинги
```

**Новые материалы от поставщика:**
```
□ Фото → Google Drive incoming/{SKU}/
□ CSV → n8n source-import
□ Дождаться обработки изображений
□ Review CSV → approve контент
□ Подтвердить Amazon-публикацию
```

**Новый FBM-заказ:**
```
□ Собрать и отправить со склада
□ Ввести трекинг в n8n-форму или SC напрямую
□ Убедиться, что заказ помечен отправленным в SC
```

**Пополнение инвентаря:**
```
□ n8n → форма обновления остатков: merchant_sku, количество, канал (site/amazon/both)
```

---

## 10. Обработка ошибок батча

```sql
SELECT bi.sku, bi.last_step, bi.error_code, bi.error_detail, bi.retry_count
FROM batch_items bi
JOIN batch_runs br ON bi.batch_id = br.batch_id
WHERE bi.status = 'FAILED'
  AND br.created_at > NOW() - INTERVAL '24 hours'
ORDER BY br.created_at DESC;
```

| error_code | Действие |
|------------|----------|
| `BC_RATE_LIMIT` | Ничего — n8n retry с backoff |
| `BC_DUPLICATE_SKU` | Проверить bc_product_id, переключить на PATCH |
| `AMZN_INVALID_BROWSE_NODE` | Обновить `amazon_browse_node` → retry |
| `AMZN_PRICE_ALERT` | Повысить `amazon_price` → retry |
| `AMZN_IMAGE_REJECTED` | Перепроверить изображение, re-derive → retry |
| `AMZN_LISTING_INACTIVE` | SC → проверить требования категории |
| `CONTENT_VALIDATION_ERROR` | Пересмотреть промпт, regenerate SKU |

Исправить данные в PostgreSQL → n8n: batch_id → Retry failed items (только FAILED строки).

---

## 11. n8n Workflow Groups

```
catalog_ops:
  source-import         — CSV от поставщика → PostgreSQL
  price-update          — пересчёт цен → BC + Amazon

image_ops:
  image-ingest          — Drive webhook → stage_a_to_b → stage_b_to_c
  media-refresh         — перегенерация деривативов

content_ops:
  content-generate      — Claude API → review CSV
  content-review-process — обработка review CSV
  content-republish     — patch BC metafields
  seo-refresh           — обновление SEO без полной регенерации

publication_ops:
  bc-publish            — PostgreSQL → BC REST API v3
  amazon-syndicate      — PostgreSQL → SP-API Listings API

inventory_ops:
  inventory-sync        — BC ↔ PostgreSQL ↔ Amazon SC
  amazon-orders-poll    — Orders API → FBM-заказы + уведомления

monitoring_ops:
  alert-low-stock       — пороги инвентаря
  batch-health-check    — ежечасная проверка FAILED batch_items
```

---

## 12. SQL Quick Reference

**Статус каталога:**
```sql
SELECT image_qa_status, content_status, bc_status, amazon_status, COUNT(*) AS cnt
FROM icons
GROUP BY 1, 2, 3, 4
ORDER BY 1, 2, 3, 4;
```

**Готово к генерации контента:**
```sql
SELECT merchant_sku, saint_or_subject, material
FROM icons
WHERE image_qa_status = 'approved'
  AND content_status  = 'not_started';
```

**Готово к Amazon:**
```sql
SELECT merchant_sku, amazon_title, amazon_price, amazon_browse_node
FROM icons
WHERE content_status = 'approved'
  AND bc_status      = 'published'
  AND amazon_status  = 'not_listed';
```

**Низкий инвентарь:**
```sql
SELECT merchant_sku, saint_or_subject, site_stock, amazon_reserved,
       (site_stock + amazon_reserved) AS total
FROM icons
WHERE bc_status = 'published'
  AND (site_stock < 3 OR amazon_reserved < 3)
ORDER BY total ASC;
```

**История батчей (7 дней):**
```sql
SELECT batch_type, status, item_count, success_count, failure_count,
       created_at, completed_at
FROM batch_runs
WHERE created_at > NOW() - INTERVAL '7 days'
ORDER BY created_at DESC;
```
