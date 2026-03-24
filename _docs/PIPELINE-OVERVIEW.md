# Icon Store — Pipeline Overview
> Версия: 1.1 · 2026-03-23 · Claude Sonnet 4.6 + Codex review

---

## Архитектурный принцип

**Amazon — технологический стандарт.** Master record проектируется в Amazon-compatible формате.
BigCommerce получает производное представление из того же master record.

```
Автор → Google Form → n8n → PostgreSQL (master)
                                  │
                      ┌───────────┴───────────┐
                      ↓                       ↓
              BigCommerce API          Amazon SP-API
              (независимо)             (независимо)
```

Два канала публикации — два отдельных n8n workflow. Ошибка одного не блокирует другой.

---

## Участники системы

| Роль | Что делает | Инструмент |
|------|-----------|-----------|
| **Автор / поставщик** | Передаёт икону: факты + фото | Google Form + Google Drive |
| **Обработчик изображений** | Ретушь, нормализация, загрузка мастера | Lightroom → Google Drive → S3/R2 |
| **Оператор магазина** | Проверяет карточки, одобряет публикацию | Neon (SQL) / n8n dashboard |
| **Владелец** | Ценообразование, финансы, решения | BigCommerce Admin + Neon |

---

## Полный пайплайн

### Этап 1 — Сбор данных от автора

Форма для фактов и Drive-папка для фото — разделены намеренно: файлы тяжёлые, логика разная.

```
Автор
  │
  ├── Заполняет Google Form (факты)
  │     Поля: имя автора, supplier_sku, название иконы,
  │           святой / сюжет, материал, техника,
  │           размер (мм), год, цена поставщика, примечания
  │
  └── Загружает фото в Google Drive (персональная shared папка)
        Формат: любой — JPEG/TIFF/RAW, оператор разберётся
```

**Хранение:** Google Sheets (ответы формы, автоматически) · Google Drive (входящие фото)

---

### Этап 2 — Создание Master Record (n8n)

```
Google Sheets (новая строка)
  │
  └── n8n trigger: "новая запись"
        │
        ├── Присваивает merchant_sku (IC-000001, IC-000002 ...)
        ├── Создаёт Icon Master Record в PostgreSQL
        │     статус: incoming
        │     все source facts записаны
        │     amazon_status: not_submitted
        │
        └── Уведомление оператору: "новая икона в очереди"
```

**Инструменты:** n8n · PostgreSQL на Neon (managed, не временная база)

> **Решение зафиксировано:** сразу поднимаем production PostgreSQL на Neon.
> Временная "псевдобаза" не создаётся — это двойная миграция.

---

### Этап 3 — Обработка изображений

```
Обработчик изображений
  │
  ├── Скачивает фото из Google Drive
  ├── Ретушь в Lightroom / Photoshop
  └── Загружает master-файл (TIFF/PNG, ≥3000px) → S3/R2
        │
        └── n8n trigger: "мастер появился в S3"
              │
              ├── Sharp/libvips — нарезка производных:
              │     • storefront_2000px.jpg  (BigCommerce)
              │     • thumb_400px.jpg        (оба канала)
              │     • amazon_main_2000px.jpg (белый фон RGB 255,255,255)
              │
              ├── Remove.bg API (опционально, если нужен белый фон)
              │
              └── Обновляет статус в PostgreSQL: incoming → ready
                    image_refs[] записаны в master record
```

**Требования Amazon к изображениям (обязательно):**
- Главное фото: чисто белый фон RGB(255,255,255)
- Товар занимает ≥85% кадра
- Минимум 1000px по длинной стороне (рекомендуется 2000px+)
- Никаких логотипов, текста, водяных знаков на главном фото
- До 9 фотографий на листинг

**Статусы изображения:**
`incoming` → `retouch_required` → `processing` → `partial` / `ready` → `approved` → `rejected`

**Хранение:** S3/Cloudflare R2 — мастера и все производные

---

### Этап 4 — Нормализация под Amazon-схему

Amazon требует атрибуты по **product type JSON schema** — не универсальные константы.
Лимиты полей (title, description, bullet points) читаются из схемы, не хардкодируются.

```
PostgreSQL запись (статус: ready)
  │
  └── n8n: запрос схемы через Product Type Definitions API
        │
        ├── Определяет productType (HOME_DECOR / WALL_ART)
        ├── Получает актуальную JSON schema для marketplace US
        └── Определяет required / conditionally_required поля
              │
              └── n8n: вызов Claude API (Batches)
                    │
                    ├── Вход: факты RU + schema requirements
                    └── Выход (English, schema-compliant):
                          • item_name (title, лимит из схемы)
                          • bullet_point × 5
                          • product_description
                          • generic_keywords (backend search)
                          • structured attributes (material,
                            technique, subject, dimensions...)
                    │
                    └── Сохраняет content + amazon_payload_draft
                          статус: content_ready
```

**Инструменты:** Claude API (Messages + Batches) · Amazon Product Type Definitions API · n8n

---

### Этап 5 — Проверка оператором

```
Оператор (Neon Table Editor или простой admin UI)
  │
  ├── Видит записи со статусом content_ready
  ├── Проверяет: текст EN, цена, изображение, amazon payload
  ├── Вносит правки если нужно
  └── Меняет статус: content_ready → approved
```

**Это единственный обязательный ручной шаг перед публикацией.**

---

### Этап 6A — Публикация на сайт (BigCommerce)

```
PostgreSQL: статус = approved
  │
  └── n8n workflow "publish-to-bc" (независимый)
        │
        ├── Маппит master record → BigCommerce product fields
        ├── POST /v3/catalog/products  → создаёт карточку
        ├── POST /v3/catalog/products/{id}/images → загружает фото
        ├── Устанавливает цену, наличие (site_stock), категорию
        │
        └── Сохраняет bc_product_id в PostgreSQL
              → статус: published_bc
```

**API:** BigCommerce REST API v3
**Инструменты:** n8n (BigCommerce node или HTTP Request)

> ⚠️ **Open risk:** прямой connector BigCommerce Standard → Amazon не подтверждён
> как надёжный базовый путь. Автоматизация строится через SP-API напрямую из PostgreSQL.

---

### Этап 6B — Публикация на Amazon (независимо от 6A)

```
PostgreSQL: статус = approved
  │
  └── n8n workflow "publish-to-amazon" (независимый)
        │
        ├── Собирает Amazon-compliant payload из master record
        │     (атрибуты по schema, content, image URLs)
        │
        ├── Pilot path (малый объём):
        │     PUT /listings/2021-08-01/items/{sellerId}/{sku}
        │     → Listings Items API, по одному товару
        │
        ├── Bulk path (большой объём):
        │     JSON_LISTINGS_FEED
        │     → до 25 000 товаров в одном feed
        │
        ├── Amazon сам скачивает изображения по URL из S3
        │
        └── Результат записывается в PostgreSQL:
              • Успех → amazon_asin, статус: published_amazon
              • Ошибка → amazon_status: rejected
                         amazon_rejection_reason: "..."
                         статус: amazon_review_required
```

**API:** Amazon SP-API · Listings Items API · JSON_LISTINGS_FEED
**Инструменты:** n8n (HTTP Request + Code node для AWS Signature V4)

> **Важно по inventory:** quantity на Amazon обновляется через Listings Items API
> или JSON_LISTINGS_FEED. FBA Inbound API — только для отгрузки товаров на склад Amazon,
> не для обновления остатков листинга.

---

### Этап 7 — Feedback loop (Amazon → master DB)

```
Amazon Processing
  │
  ├── Листинг принят → amazon_asin записан в PostgreSQL
  │
  ├── Листинг отклонён → причина записана в PostgreSQL
  │     amazon_status: rejected
  │     amazon_rejection_reason: "Missing required attribute: color"
  │     → оператор видит, исправляет в master record, re-submit
  │
  └── Листинг suppressed → suppression_reason записан в PostgreSQL
        → оператор видит причину без входа в Seller Central
```

**Принцип:** все Amazon-статусы живут в master DB. Seller Central — только для ручных операций.

---

## Карта хранилищ

| Что | Где | Зачем |
|-----|-----|-------|
| Ответы форм авторов | Google Sheets | Автоматически из Google Forms |
| Входящие фото | Google Drive | Обмен с авторами и ретушёром |
| Мастер-изображения | S3 / Cloudflare R2 | Постоянное хранение оригиналов |
| Производные (JPG) | S3 / Cloudflare R2 | Готовые к публикации варианты |
| Все данные товаров | **PostgreSQL на Neon** | Единый источник правды |
| Публичный каталог | BigCommerce CDN | Хостит изображения для сайта |
| Маркетплейс | Amazon (FBA) | Отдельный инвентарь, отдельный склад |

---

## Инфраструктура

| Сервис | Роль | Стоимость старт |
|--------|------|----------------|
| **Neon** | Managed PostgreSQL | Бесплатно → $15/мес |
| **n8n** | Оркестрация всех потоков | Docker локально = $0 |
| **Cloudflare R2 / S3** | Хранение изображений | ~$0.015/GB/мес |
| **Claude API** | Нормализация + генерация контента | ~$0.01–0.05 за карточку |
| **Remove.bg** | Удаление фона | $0.2/фото или local AI |
| **BigCommerce Standard** | Сайт + CDN | ~$29/мес |
| **Amazon** | FBA + SP-API | % с продаж + FBA fees |
| **Google Forms / Drive** | Intake от авторов | Бесплатно |

---

## Статусная модель Icon Master Record

```
incoming
    │
    ├── [фото плохое] ──→ retouch_required ──→ incoming
    │
    └── [фото ок] ──→ processing ──→ partial / ready
                                          │
                                   [контент создан]
                                          │
                                    content_ready
                                          │
                                   [оператор одобрил]
                                          │
                                       approved
                                          │
                          ┌──────────────┴──────────────┐
                          ↓                             ↓
                    published_bc                 published_amazon
                                                       │
                                          ┌────────────┴────────────┐
                                          ↓                         ↓
                                      rejected              amazon_review_required
                                   (причина в DB)           (suppression в DB)
```

---

## Amazon GTIN / Brand — подготовительные шаги

До начала автоматизации Amazon нужно:

1. **Brand Approval** в Seller Central — без этого нет GTIN Exemption
2. **GTIN Exemption** для категории HOME_DECOR — иконы handmade, штрихкодов нет
3. **SP-API access** — регистрация developer app, получение credentials
4. **Product Type schema** — запросить через Product Type Definitions API для HOME_DECOR/WALL_ART US marketplace, зафиксировать required fields

Всё это — разовые бизнес-действия перед первым листингом.

---

## Порядок запуска

1. **Neon** → создать проект, базу `icon_store`, app user, сохранить connection string
2. **Google Form** → форма для авторов (факты отдельно от фото)
3. **n8n локально** → `docker run -p 5678:5678 n8nio/n8n`
4. **Pipeline Form → Sheets → n8n → Neon** → первый рабочий intake
5. **S3/R2 bucket** → структура папок для мастеров и производных
6. **BigCommerce API** → первая публикация карточки
7. **Amazon** → Brand Approval → GTIN Exemption → SP-API credentials → первый листинг
8. **Claude API** → автогенерация Amazon-compliant контента

---

> Canonical архитектура: `MASTER-ARCHITECTURE-v4.md`
> Открытые вопросы: `OPEN-QUESTIONS.md` · Прикладное руководство Amazon: `SPEC-4-AMAZON-LAUNCH.md`
