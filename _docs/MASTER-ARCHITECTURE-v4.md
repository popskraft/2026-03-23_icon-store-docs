# MASTER-ARCHITECTURE-v4.md

> **Версия:** 4.2 — все спецификации приведены к каноническому формату (стандартный заголовок + раздел "Архитектурный контекст v4.2"), пройдена редакторская минимизация: удалены туториалы и объяснения для новичков, сохранены все операционные схемы, контракты, таблицы статусов. Корпус документации стабилен.
> **Автор:** Claude Sonnet 4.6
> **Дата:** 2026-03-24
> **Статус:** Стабильный канон. Заменяет v4.1.
> **Предыдущие версии:** v1 (Codex), v2 (merged), v3 (ролевая модель + потоки), v4.1 (спецификации skeleton) — архивированы, не удалять

---

## 1. Что это за система

Это не просто интернет-магазин. Это **операционная система каталога икон**, в центре которой находится единая карточка каждого товара, а BigCommerce и Amazon — выходные каналы публикации.

**Amazon — технологический стандарт.** Master record проектируется в Amazon-compatible формате. BigCommerce получает производное представление из того же master record.

Главная инженерная задача — не сайт, а:
- надёжный приём данных от владельца и обработка изображений
- двухслойное наполнение карточек: базовые факты от владельца + AI-обогащение системой
- автогенерация текстов под каждый канал (Amazon и сайт — независимо)
- пакетная публикация и обновления без ручной работы

---

### 1.1 Карта пайплайна — общий вид

```
Поставщик → Google Drive (фото) ──────────────────────────────┐
                                                               │
Владелец → Supabase Table Editor (базовые факты иконы)        │
               │                                              │
               ▼                                              ▼
         PostgreSQL                              Обработчик изображений
     (Icon Master Record)                    (Lightroom → approved master)
               │                                              │
               │◄─────────────────── S3/R2 ──────────────────┘
               │                  (мастера + деривативы)
               │
         n8n → Claude API
         (обогащение: title_en, description, bullets,
          category, faceted filters, Amazon payload)
               │
         Оператор review
         (CSV review pack для батчей; Table Editor для
          точечных правок и exception handling)
               │
      ┌────────┴────────┐
      ▼                 ▼
 BigCommerce        Amazon SP-API
 (n8n → BC API)    (n8n → Listings API)
 независимо        независимо
```

---

### 1.2 Двухслойная модель данных

Каждая карточка иконы наполняется в два этапа одного PostgreSQL-рекорда:

| Слой | Кто | Что заполняется | Инструмент |
|------|-----|-----------------|-----------|
| **Слой 1 — Базовые факты** | Владелец | saint_or_subject, material, technique, размеры, base_cost, notes | Supabase Table Editor |
| **Слой 2 — AI-обогащение** | Система (n8n → Claude API) | title_en, description_en, amazon_title, 5 bullets, keywords, category, faceted filters, amazon_payload | Автоматически |

Оба слоя живут в одной PostgreSQL-записи. После обогащения владелец и оператор видят полную карточку в том же Table Editor. Никакого переключения между инструментами.

---

### 1.3 Карта документов

| Файл | Роль |
|------|------|
| **`MASTER-ARCHITECTURE-v4.md`** | **Этот файл** — executive layer, роли, принципы, стек, решения |
| `SPEC-1-CATALOG-DATA-MODEL.md` | Модель данных, поля PostgreSQL, BigCommerce mapping |
| `SPEC-2-IMAGE-PIPELINE.md` | Обработка изображений, Python worker, S3/R2 структура |
| `SPEC-3-CONTENT-GENERATION.md` | Claude API, промпты, глоссарий, review workflow |
| `SPEC-4-AMAZON-LAUNCH.md` | Операционный запуск Amazon: GTIN, Brand Registry, SP-API |
| `SPEC-5-PIPELINES.md` | Все 7 пайплайнов (A–G) — детальные n8n workflows |
| `SPEC-6-STOREFRONT.md` | Storefront IA & SEO (Phase 2 skeleton) |
| `SPEC-7-OPERATOR-PLAYBOOK.md` | Процедуры оператора, статусная матрица, SQL quick ref |
| `AGENTS.md` | Роли агентов, архитектурные решения (краткий канон) |
| `ROADMAP.md` | Фазы, прогресс, decision log |
| `OPEN-QUESTIONS.md` | Открытые и закрытые решения |

---

### 1.4 Порядок запуска (Phase 2)

1. **Supabase** → создать проект, применить DDL из Spec-1, настроить RLS для owner-доступа
2. **Google Drive** → папки `incoming/` и `approved/` для фото
3. **VPS** → развернуть n8n CE (`docker compose up -d`), домен + SSL
4. **n8n credentials** → Supabase, S3/R2, Claude API, SP-API, BigCommerce API
5. **Владелец вводит первые 10 SKU** → Supabase Table Editor (Слой 1)
6. **n8n запускает AI-обогащение** → Claude API заполняет Слой 2
7. **Оператор review** → CSV review pack (`APPROVE / REJECT / REGEN`), точечные правки через Table Editor
8. **Amazon** → GTIN Exemption → тестовый листинг 1 SKU (SPEC-4)
9. **BigCommerce** → первая публикация карточки

---

### 1.5 Инфраструктура и стоимость

| Сервис                   | Роль                                              | Старт                        |
| ------------------------ | ------------------------------------------------- | ---------------------------- |
| **Supabase**             | PostgreSQL + Studio UI (owner intake + AI review) | $0 → $25/мес                 |
| **n8n CE**               | Оркестрация всех потоков                          | Self-hosted VPS ~$5–10/мес   |
| **Cloudflare R2**        | Мастера и деривативы изображений (launch default) | ~$0.015/GB/мес               |
| **Claude API**           | AI-обогащение карточек                            | ~$0.01–0.05 за SKU           |
| **Remove.bg**            | Удаление фона                                     | $0.2/фото или rembg локально |
| **BigCommerce Standard** | Сайт + CDN                                        | ~$29/мес                     |
| **Amazon FBM**           | Маркетплейс + SP-API                              | 15% referral fee             |
| **Google Drive**         | Inbox для входящих фото                           | Бесплатно                    |

---

## 2. Роли в системе

Все роли — разные люди. Владелец ≠ Оператор ≠ Обработчик.

---

### Роль 1 — Автор / поставщик

| | |
|---|---|
| **Кто** | Иконописец, мастерская или оптовый поставщик |
| **Что делает** | Присылает исходные фото икон и первичные данные о товаре |
| **Входящее** | Запрос от владельца или инициатива поставщика |
| **Исходящее** | Фотографии (исходники) + данные о товаре в согласованном формате |
| **Инструменты** | Google Drive (папка incoming), email |
| **Ограничения** | Не имеет доступа к системе. Только передаёт материалы. |

---

### Роль 2 — Владелец проекта

| | |
|---|---|
| **Кто** | Владелец бизнеса, принимает стратегические и товарные решения |
| **Что делает** | Принимает материалы от поставщика, делает первичный approve, вводит базовые факты иконы в систему, принимает все решения по ассортименту и ценам |
| **Входящее** | Материалы от поставщика (фото + данные о товаре) |
| **Исходящее** | Заполненные строки в Supabase (Слой 1); фото в Google Drive папке `incoming/[supplier-ref]/`; подтверждение оператору что товар идёт в продажу |
| **Инструменты** | **Supabase Table Editor** (ввод данных), Google Drive (фото) |

**Что владелец вводит — Слой 1 (8–10 полей):**

| Поле | Пример | Обязательно |
|------|--------|-------------|
| `saint_or_subject` | Mother of God | ✅ |
| `material` | Wood | ✅ |
| `technique` | Hand-painted | ✅ |
| `construction_type` | flat-board | ✅ |
| `size_w_mm` | 130 | ✅ |
| `size_h_mm` | 180 | ✅ |
| `base_cost` | 12.50 | ✅ |
| `supplier_sku` | SUP-00123 | если есть |
| `title_raw` | Казанская Богоматерь | если есть |
| `notes` | позолоченный оклад | если есть |

**Что система заполняет автоматически — Слой 2 (AI-обогащение):**
`site_title_en`, `site_description_html_en`, `amazon_title`, `amazon_bullet_1..5`, `amazon_backend_search_terms`, `category`, `faceted filter values`, `amazon_payload_draft`

> Владелец вводит факты. Система генерирует контент. Оператор проверяет и публикует.

**Правило именования фото-папок до получения SKU:**
Фото складываются в Drive как `incoming/[supplier_sku]/` или `incoming/[поставщик]-[порядковый номер]/`. После импорта оператор переименовывает в `incoming/IC-000001/`. Владелец не занимается переименованием.

**Первичный approve — зона ответственности владельца:**
- Фото от поставщика получены и переложены в Drive
- 8 обязательных полей заполнены в Supabase Table Editor
- Товар подтверждён к выводу в продажу

> Approve публикации (AI-контент + листинги) — зона ответственности **оператора**, не владельца.

---

### Роль 3 — Обработчик иллюстраций

| | |
|---|---|
| **Кто** | Штатный сотрудник, фрилансер с биржи или автоматический Python-скрипт — в зависимости от сценария запуска |
| **Что делает** | Получает исходные фото, нормализует геометрию, исправляет цвет и перспективу, удаляет фон, возвращает один обработанный мастер-файл |
| **Входящее** | Исходники в папке Google Drive: `incoming/{IC-000001}/` |
| **Исходящее** | Один approved master-файл в папке Google Drive: `approved/{IC-000001}/master.jpg` |
| **Инструменты** | Photoshop / Lightroom / аналоги; доступ только к своей папке в Drive |
| **Требования к результату** | Мин. 2000px по длинной стороне; белый или нейтральный фон; без обрезки объекта; RGB; JPG или PNG; предмет занимает ≥ 80% кадра |

**Три сценария (выбирается при запуске):**

| Сценарий | Когда применять | Доступ к Drive |
|----------|----------------|----------------|
| Python image worker (авто) | Стандартные фото с чистым фоном | Системный, через n8n |
| Штатный обработчик | Постоянный поток, контроль качества важен | Полный доступ к папке |
| Внешний аутсорс (биржа) | Разовые задачи, сложные случаи | Ограниченный: только папка конкретного SKU |

> Независимо от сценария — результат всегда в одном месте (`approved/{SKU}/master.jpg`) и с одинаковыми требованиями. Система не знает, кто обрабатывал.

---

### Роль 4 — Оператор магазина

| | |
|---|---|
| **Кто** | Операционный менеджер магазина, также отвечает за техническую инфраструктуру |
| **Что делает** | Курирует полный цикл от получения материалов до живых листингов; администрирует систему; вручную исправляет ошибки |
| **Инструменты** | Google Drive, n8n, PostgreSQL (SQL), BigCommerce Admin, Amazon Seller Central, VPS/сервер |

**Операционные обязанности оператора:**

| Зона | Конкретные действия |
|------|-------------------|
| Изображения | Забирает фото у обработчика; проверяет соответствие требованиям; загружает в Drive incoming/; при несоответствии возвращает на доработку |
| Каталог | Получает заполненную мастер-базу от владельца; запускает source-import в n8n; проверяет результат загрузки в PostgreSQL |
| Контент | Запускает генерацию (n8n); открывает review CSV; ставит APPROVE / REJECT / REGEN; загружает обратно |
| Публикация BC | Контролирует автоматическую прогрузку; вручную правит то, что встало некорректно в BigCommerce Admin |
| Публикация Amazon | Финальный approve перед публикацией листингов; проверяет листинги в SC после публикации; устраняет suppressed |
| Заказы | Принимает уведомления о FBM-заказах; отгружает; вводит трекинг |
| Инвентарь | Обновляет остатки в системе при пополнении склада |
| Мониторинг | Ежедневная проверка n8n Dashboard, алертов, ошибок батча |
| Система | Разворачивает и поддерживает VPS, PostgreSQL, n8n; управляет credentials; делает бэкапы |

**QA-чекпоинты оператора — контроль качества на каждом переходе:**

| # | Момент | Что проверяется |
|---|--------|----------------|
| ① | После получения от владельца | Все фото переданы, мастер-база заполнена полностью, данные соответствуют фото |
| ② | При передаче обработчику | ТЗ сформулировано чётко: формат, размер, требования к фону и кадрированию |
| ③ | После приёмки от обработчика | Соответствие ТЗ: размер ≥ 2000px, белый фон, предмет ≥ 80% кадра, без артефактов |
| ④ | Перед загрузкой в систему | Полнота мастер-базы: все поля заполнены, кол-во фото совпадает с кол-вом SKU |
| ⑤ | После публикации на сайте | Визуальная проверка карточки в BigCommerce: фото, описание, цена, категория |
| ⑥ | После публикации на Amazon | Визуальная проверка листинга в SC: главное фото, bullets, категория, статус active/suppressed |

> Оператор — единственная роль с доступом к PostgreSQL, n8n и серверной инфраструктуре.
> Оператор несёт ответственность за качество данных на каждом переходе — не только за нажатие кнопок.

---

### Роль 5 — Система (n8n + автоматика)

| | |
|---|---|
| **Кто** | Автоматические workflows: n8n, Python workers, Claude API, BC API, SP-API |
| **Что делает** | Выполняет все автоматизированные шаги между человеческими точками принятия решений |
| **Инструменты** | n8n, PostgreSQL, Claude API, Amazon SP-API, BigCommerce API v3, S3/R2, Remove.bg API |

**Что делает система (без участия человека):**
- Детектирует новые файлы в Drive → запускает image processing
- Обрабатывает изображения: удаление фона, нормализация, нарезка производных
- Генерирует контент через Claude API (ночные батчи)
- Публикует товары в BigCommerce после approve оператора
- Публикует листинги на Amazon после финального approve
- Синхронизирует остатки между каналами
- Опрашивает Amazon Orders API, создаёт заказы в PostgreSQL
- Отправляет алерты оператору при ошибках и низком стоке

> Система никогда не принимает решений о качестве контента и не публикует листинги без явного подтверждения человека.

---

### Роль 6 — Покупатель

| | |
|---|---|
| **Кто** | Конечный покупатель в США, ищет православные иконы |
| **Каналы взаимодействия** | BigCommerce-сторфронт (прямая покупка) или Amazon (листинг) |
| **Что важно для него** | Подлинность и происхождение иконы, материал и техника, провенанс и благословение (если есть), качество фото, понятная цена и доставка |
| **Ожидания от системы** | Точные описания, белый фон на фото, доставка FBM в срок, корректный трекинг |

> Покупатель не взаимодействует с системой напрямую. Его потребности определяют требования к контенту (Spec-3), изображениям (Spec-2) и навигации сторфронта (Spec-6).

---

**Правила разграничения ролей:**
- Обработчик не имеет доступа к системе и не знает о публикации
- Владелец не выполняет операционные задачи и не работает с n8n / PostgreSQL
- Система никогда не публикует без явного human gate
- Оператор не занимается стратегическими решениями по ассортименту и ценообразованию — это зона владельца

---

## 3. Архитектурные принципы

1. **PostgreSQL на Supabase — канонический реестр** — именно там живёт главная карточка иконы. Supabase Table Editor — единый интерфейс для владельца и оператора.
2. **BigCommerce — канал публикации** — принимает готовые данные, не является источником истины.
3. **Двухслойная модель данных** — Слой 1 (owner: базовые факты) + Слой 2 (система: AI-обогащение) живут в одной PostgreSQL-записи. Единый источник правды.
4. **Оригиналы неприкосновенны** — ни один входящий файл не перезаписывается автоматикой.
5. **Три стадии изображения** — Incoming → Approved Master → Published Derivatives. Смешивать нельзя.
6. **Каждый батч перезапускаем** — сбой не портит состояние, процесс возобновляется с последнего успешного шага.
7. **Человек подтверждает перед Amazon** — ни один AI-текст не уходит в листинг без явного approve оператора.
8. **Каналы изолированы** — текст для Amazon и текст для сайта генерируются независимо из одних исходных фактов.
9. **Цена живёт в мастер-карточке** — не в BigCommerce напрямую. Изменение правила → пакетное обновление всех каналов.
10. **Минимум технологий** — каждый компонент должен быть заменим без переписывания всего.
11. **Рабочая документация на русском** — финальные артефакты для публикации на английском.

---

## 4. Стек

| Слой | Выбор | Статус |
|------|-------|--------|
| Commerce core | BigCommerce Standard | ✅ Подтверждён |
| Marketplace | Amazon Seller Central + FBM | ✅ Подтверждён |
| Канонический реестр + Owner UI | **Supabase** (PostgreSQL 15 + Table Editor + RLS) | ✅ Подтверждён 2026-03-24 |
| Оркестрация | n8n CE (self-hosted VPS) | ✅ Подтверждён |
| Хранилище изображений | Cloudflare R2 (S3-compatible; launch default) | ✅ Подтверждён |
| Обработка и нарезка фото | Python (Pillow / libvips) | ✅ Подтверждён |
| Удаление фона | Remove.bg API → fallback: rembg (локальный) | ⚠️ Выбрать на пилоте |
| AI-обогащение контента | Claude API (Messages + Batches) | ✅ Подтверждён |
| Входящие фото от поставщика | Google Drive (inbox only) | ✅ Подтверждён |
| Оповещения | Email / Slack через n8n | ✅ Достаточно для MVP |

---

## 5. Центральный объект: Icon Master Record

Каноническая карточка иконы — главный объект всей системы. Из неё рождаются тексты, изображения, цены и публикации. Живёт в PostgreSQL. Полная DDL — в **Spec-1 (раздел 15)**.

### 5.1 Идентичность

```
icon_id           — внутренний постоянный UUID
merchant_sku      — главный идентификатор (IC-000001, IC-000001-S/M/L)
supplier_sku      — артикул поставщика (внешний reference, не PK)
product_family_id — если есть вариации одной иконы
asin              — после публикации на Amazon
bc_product_id     — после публикации в BigCommerce
```

**SKU-правило:** `IC-000001` — простой последовательный номер. Семантику в SKU не кодируем — она живёт в полях карточки. Суффикс `-S/M/L` только для размерных вариантов.

### 5.2 Исходные факты (от поставщика)

```
title_raw           — исходное название от автора
saint_or_subject    — кто изображён
feast               — праздник (если применимо)
purpose             — назначение (домашняя молитва, путевая, подарок...)
style               — стиль
iconographic_type   — иконографический тип
material            — материал (дерево, металл...)
technique           — техника (печать, ручная работа...)
construction_type   — конструкция (плоская доска, триптих, складень...)
origin_school       — школа / происхождение
size_w_mm, size_h_mm, size_d_mm
weight_g
manufacturer        — автор / мастерская
notes               — свободные заметки
```

### 5.3 Изображения

```
incoming_source_uri     — путь к исходнику (Google Drive path или URL)
approved_master_uri     — путь к approved master в S3/R2
derivatives             — JSONB: {amazon-main, site-pdp, ...} → URL
image_qa_status         — incoming | processing | approved | rejected
image_qa_notes
last_processed_at
```

### 5.4 Контент

```
content_version
content_status          — not_started | generating | review | approved | published
approved_by
approved_at
review_notes

site_title_en
site_description_html_en
meta_title_en
meta_description_en

amazon_title            [★ sync]
amazon_bullet_1..5      [★ sync]
amazon_description_html
amazon_backend_search_terms [★ sync]
amazon_browse_node      [★ sync]
amazon_item_type_keyword [★ sync]
amazon_target_marketplace [★ sync]
```

`[★ sync]` — поля, которые должны совпадать в карточке, BigCommerce metafields и Amazon payload.

### 5.5 Цены

```
base_cost           — закупочная цена
pricing_rule_id     — ссылка на правило расчёта маржи
site_price          — рассчитывается: base_cost × markup
amazon_price        — рассчитывается: site_price × amazon_factor
map_price           — минимальная допустимая цена
price_last_updated
```

### 5.6 Публикация

```
bc_status           — draft | published | archived
amazon_status       — not_listed | pending | live | suppressed | live_manual
publication_notes
```

### 5.7 Inventory

```
site_stock          — остаток для сайта (отдельный пул)
amazon_reserved     — остаток зарезервирован под Amazon FBM (отдельный пул)
reorder_threshold
supplier_lead_days
```

### 5.8 Sync / Audit

```
last_hash           — SHA-256 от значимых полей карточки
last_synced_at
batch_id
sync_error_code
source_of_last_update
```

---

## 6. Хранилище данных

| Что | Где | Почему |
|-----|-----|--------|
| Канонические карточки икон | PostgreSQL | Главное ядро системы |
| Исходники фото (incoming) | Google Drive | Обмен между людьми |
| Approved master фото | S3 / Cloudflare R2 | Дёшево, публичный URL, API |
| Производные версии фото | S3 / R2 | Рядом с master, стабильные URL |
| Опубликованные продукты | BigCommerce | Витрина и checkout |
| Amazon listing state | Amazon Seller Central | FBM и статус листинга |
| Review-артефакты | JSONL + CSV | Repo или managed workspace |
| Рабочие заметки | Obsidian managed workspace | Долгосрочная нечувствительная память |

**BigCommerce metafield namespaces:**

| Namespace | Назначение |
|-----------|-----------|
| `ops.review` | `image_qa_status`, `content_status`, `approved_by` |
| `ops.sync` | `last_batch_id`, `last_synced_at`, `amazon_status`, `sync_error_code` |
| `media.storage` | `master_uri`, `derivatives_json` |
| `content.amazon` | `amazon_title`, `amazon_bullets`, `backend_search_terms`, `browse_node` |
| `content.site` | дополнительные SEO-поля если нужны |
| `catalog.source` | `icon_id`, `supplier_sku`, `product_family_id` |

---


## 7. Пайплайны

Все 7 операционных пайплайнов (A–G) — детальные workflow, n8n-шаги, триггеры, обработка ошибок, валидация.

> **Детали:** `SPEC-5-PIPELINES.md`

## 8. Управление ценами

Цена не живёт "просто в BigCommerce". Она рассчитывается:

```
base_cost (в master card)
    × markup_multiplier (в таблице pricing_rules)
    + handling_fee
    = site_price

amazon_price = site_price × amazon_fee_factor
    (amazon_fee_factor: 1.0 = паритет, 1.15 = покрытие referral fee 15%)
```

**Пример правила "Standard Markup":**
```
markup_multiplier : 2.5
handling_fee      : $0
amazon_fee_factor : 1.0   (паритет с сайтом)
min_site_price    : $15   (floor — ниже не публиковать)
```

При изменении правила → batch `price-update` → обновляет все каналы пакетно.
Ручная правка цены в BigCommerce — только исключение, фиксируется в `audit_log`.

---
## 9. Сторфронт

BigCommerce Standard — сайт и канал прямых продаж. Информационная архитектура, навигация, фильтры, SEO-структура и trust signals.

> **Детали:** `SPEC-6-STOREFRONT.md` (Phase 2 — skeleton)

## 10. Безопасность и инфраструктура

### Fixed minimum (с первого дня)

| Сервис                     | ~Месяц   |
| -------------------------- | -------- |
| BigCommerce Standard       | $39      |
| Amazon Professional Seller | $39.99   |
| Домен + email              | ~$3      |
| **Итого fixed**            | **~$82** |

### Масштабируемые расходы

| Сервис | ~Месяц |
|--------|--------|
| VPS для n8n + PostgreSQL | $12–20 |
| S3 / Cloudflare R2 | < $5 |
| Claude API (батчи) | $10–30 |
| Remove.bg API | $5–15 |
| **Итого scaling** | **~$27–70** |

**Итого MVP: ~$109–152/month** [DEFAULT — подтвердить потолок с owner]

### n8n + PostgreSQL (illustrative — не production-ready)

```yaml
# ILLUSTRATIVE — не деплоить как есть
services:
  n8n:
    image: n8nio/n8n
    environment:
      - DB_TYPE=postgresdb
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
```

Минимум для прода: TLS через nginx/Caddy, PostgreSQL не открыт наружу, `N8N_ENCRYPTION_KEY` в секрете, бэкап PostgreSQL ежедневно, retention ≥ 30 дней, тест восстановления ежемесячно.

### Права API (минимальные)

| Ключ | Разрешено |
|------|-----------|
| BigCommerce | Catalog r/w, Order r, Webhook w |
| S3 / R2 | Upload, read, list — не delete |
| Claude API | Messages + Batches |
| Amazon SP-API | Catalog Items r; Listings w только после пилота; Orders r (FBM) |
| Remove.bg | Image processing только |

---

## 11. Открытые вопросы к owner

### Блокирующие

- ~~**[#1]** Какой именно Amazon connector доступен на BigCommerce Standard?~~ **Закрыто 2026-03-23** — коннектор не используется, путь: n8n → SP-API напрямую.
- Какой формат исходного каталога от поставщика? (CSV, Excel, другой)
- Есть ли готовые фото на все SKU? Где хранятся сейчас?
- Юридическое лицо и аккаунт Amazon Seller Central для US?
- Есть ли зарегистрированный бренд для Brand Registry + A+ Content?

### Near-term

- Потолок бюджета на инфраструктуру до выхода в доход (~$100 или ~$200/мес)?
- Вариантная стратегия: размеры/рамы как BigCommerce variants или отдельные SKU?
- Физический склад: сколько SKU реально готово к отгрузке сейчас?
- Один поставщик или несколько?
- Страны за пределами US в горизонте запуска?
- Как оператор будет вводить трекинг-номера FBM-заказов: через n8n-форму или напрямую в Seller Central?

---

## 12. Что закрыто и что осталось

| Тема | Статус | Где закрыто |
|------|--------|-------------|
| UX ревью-процесса (daily approve/reject) | ✅ Закрыто | Spec-3: CSV-based review для MVP |
| Ценовые правила и формула маржи | ✅ Закрыто | Раздел 8 + Spec-1 (pricing_rules DDL) |
| PostgreSQL DDL полная | ✅ Закрыто | Spec-1 (раздел 15) |
| BigCommerce field map | ✅ Закрыто | Spec-1 (раздел 15) |
| Prompt-шаблон для генерации | ✅ Закрыто | Spec-3 (раздел 17) |
| Иконографический глоссарий | ✅ Закрыто | Spec-3 (раздел 17) |
| S3/R2 структура файлов | ✅ Закрыто | Spec-2 (раздел 16) |
| Путь синдикации Amazon | ✅ Закрыто | n8n → SP-API напрямую (2026-03-23) |
| Модель фулфилмента | ✅ Закрыто | FBM, не FBA (2026-03-23) |
| Модель инвентаря | ✅ Закрыто | Единый склад, два логических резерва (раздел 7 Pipeline F) |
| Процедура оператора end-to-end | ✅ Закрыто | Spec-5 (раздел 19) |
| Amazon browse node mapping | ⚠️ Частично | Spec-4 (раздел 18): подход определён; node IDs — после пилота |
| GTIN Exemption | ⚠️ Действие | Нужно подать в SC до первой публикации |
| SP-API credentials & scopes | ⚠️ Действие | Настроить в n8n перед пилотом |
| Runbook восстановления PostgreSQL | ❌ Открыто | Нужен отдельный операционный документ |
| Кто занимается exception triage | ❌ Открыто | Ждёт owner |
| Порог миграции с BigCommerce Standard | ⚠️ Default | ~$50K GMV / год |
| n8n workflow groups (структура) | ✅ Закрыто | Spec-3 (раздел 17) + Spec-5 (раздел 19) |

---

## 13. Реестр решений

| Статус | Решение |
|--------|---------|
| ✅ | Platform-led modular архитектура без custom backend |
| ✅ | BigCommerce Standard как publication channel |
| ✅ | PostgreSQL как канонический реестр икон |
| ✅ | Amazon Seller Central + **FBM** как основной канал (FBA не используется) |
| ✅ | Amazon syndication: **n8n → SP-API напрямую** (BC-коннектор не используется) |
| ✅ | Инвентарь: единый физический склад, два логических резерва (site / amazon_reserved) |
| ✅ | S3 / R2 как approved master storage (Cloudinary убран) |
| ✅ | Python worker для обработки и нарезки фото |
| ✅ | Human review hard gate перед Amazon |
| ✅ | Google Drive — только inbox/обмен, не production storage |
| ✅ | Три стадии изображения: Incoming / Approved Master / Derivatives |
| ✅ | Рабочая документация на русском, финальные артефакты на английском |
| ✅ | site_ru — не launch output |
| ✅ | merchant_sku (IC-000001) — главный ключ; supplier_sku — внешний reference |
| ✅ | Цена живёт в master card + pricing rule, не только в BigCommerce |
| ✅ | Batch lifecycle + SQL schema (batch_runs / batch_items) — Adopted |
| ✅ | Icon Master Record как центральный объект (8 блоков) — Adopted |
| ✅ | BigCommerce metafield namespaces (6 namespaces) — Adopted |
| ✅ | Content output JSON contract — Adopted |
| ✅ | Матрица ракурсов по construction type — Adopted |
| ✅ | CSV-based review workflow для MVP — Adopted |
| ✅ | pricing_rules таблица с markup_multiplier + amazon_fee_factor — Adopted |
| ✅ | content_versions таблица для истории апрувов — Adopted |
| ✅ | audit_log таблица для изменений цен и аномалий — Adopted |
| ⚠️ | Remove.bg vs rembg — тестировать оба, выбрать на пилоте |
| ⚠️ | Retry thresholds (3 попытки) — валидировать на пилоте |
| ⚠️ | Порог батча (> 10 SKU → Batches API) — валидировать |
| ⚠️ | Alert thresholds (сайт < 3, amazon_reserved < 3) — настроить под реальную экономику |
| ⚠️ | n8n hosting: self-hosted vs Cloud — ждёт owner |
| ⚠️ | Budget ceiling ($109–152/month) — ждёт owner |
| ⚠️ | Amazon browse node: research approach определён, финальный node ID — после пилота |

---

## 14. Следующие шаги

1. **[ДЕЙСТВИЕ]** Получить GTIN Exemption в Amazon Seller Central (handmade religious items) — разблокирует первый листинг
2. **[ДЕЙСТВИЕ]** Настроить SP-API credentials в n8n (Listing w + Orders r/w) — разблокирует Пайплайн E
3. **[Параллельно]** Запустить 10-SKU пилот по Spec-1 + Spec-2 + Spec-3: PostgreSQL → image worker → content generate → review
4. **[После пилота]** Spec-4: Amazon Operations — browse node validation, первые листинги через SP-API
5. **[После пилота]** Spec-5 уже написан; проверить на первом реальном батче
6. **[После пилота]** Spec-6: Storefront IA & SEO — навигация, фильтры, trust signals

---

## Спецификации (внешние файлы)

Детальные спецификации вынесены в отдельные файлы:

| Файл | Содержание |
|------|-----------|
| `SPEC-1-CATALOG-DATA-MODEL.md` | PostgreSQL DDL, BigCommerce field map, идентификаторы |
| `SPEC-2-IMAGE-PIPELINE.md` | Python worker, S3/R2 структура, QA-статусы изображений |
| `SPEC-3-CONTENT-GENERATION.md` | Claude API промпты, глоссарий, CSV review workflow |
| `SPEC-4-AMAZON-LAUNCH.md` | GTIN Exemption, Shipping Template, Brand Registry, чеклист |
| `SPEC-5-PIPELINES.md` | Все 7 пайплайнов (A–G), n8n workflow детали |
| `SPEC-6-STOREFRONT.md` | Storefront IA, навигация, SEO (Phase 2 skeleton) |
| `SPEC-7-OPERATOR-PLAYBOOK.md` | Процедуры оператора, статусная матрица, SQL quick ref |
