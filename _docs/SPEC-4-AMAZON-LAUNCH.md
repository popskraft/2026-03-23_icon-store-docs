# SPEC-4: Операционный запуск Amazon

Версия: 3.0
Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.2
Область: запуск SC-аккаунта, GTIN Exemption, листинги, SP-API интеграция, операционная рутина

---

## Архитектурный контекст (v4.2)

- Amazon Seller Central + FBM — подтверждённый канал (FBA не используется).
- Синдикация: n8n → SP-API напрямую (BC-коннектор не используется).
- Launch account/entity for Seller Central is still pending owner confirmation.
- Все листинги в одном SC-кабинете; разделение финансов — на уровне бухгалтерии.
- `merchant_sku` формата `IC-XXXXXX` — идентификатор иконных листингов.
- GTIN Exemption необходим до первой публикации (handmade religious items без UPC/EAN).
- Human review обязателен перед любой публикацией (Пайплайн E, SPEC-5).
- Browse node: подход определён (Home Décor как старт), финальный node ID — после пилота.

---

## 1. Контекст

| Параметр | Магазин 1 | Магазин 2 (иконы) |
|---|---|---|
| SC-аккаунт | Existing seller account (если owner подтвердит) | Pending owner confirmation |
| Бренд | Бренд AK | Отдельный бренд икон |
| Банковский счёт | Один на оба | Разделение через бухгалтерию |
| Seller ID / SP-API credentials | — | Те же, не меняются |
| Склад | Общий | Общий |

---

## 2. Стратегия запуска

| | Вариант A: Прямые листинги | Вариант B: Brand Registry |
|--|---|---|
| Trademark нужен | ❌ | ✅ (USPTO) |
| Срок | 1–2 недели | 3–15 месяцев |
| Brand Store / A+ Content | ❌ | ✅ |
| **Рекомендация** | ✅ Запускать сразу | ✅ Добавить позже |

---

## 3. Вариант A — Прямые листинги

### A.0 — Решение об аккаунте (⚠ ДО ЛЮБЫХ ДРУГИХ ШАГОВ)

| | Существующий аккаунт | Новый аккаунт |
|---|---|---|
| Верификация | Не нужна | Полная (паспорт, банк, 1–3 нед) |
| Стоимость | Professional plan уже оплачен | +$39.99/мес отдельно |
| Финансы | Один disbursement, разделять через бухгалтерию | Отдельный банковский счёт |
| SP-API credentials | Те же Seller ID и Marketplace ID | Новые credentials |
| Рекомендация | Быстрый старт | Финансовая независимость |

Записать решение в `AGENTS.md` → сообщить оператору.

### A.1 — Название бренда (⚠ ПЕРЕД GTIN Exemption)

Название нельзя легко изменить после создания листингов.
- Отдельное от юридического имени и от второго магазина
- Короткое (1–3 слова): `SacredIcons`, `PalekStudio`, `OrthodoxArtworks`
- Проверить на занятость: [USPTO.gov → TESS](https://tmsearch.uspto.gov)
- Зафиксировать в `AGENTS.md` и в n8n → workflow `amazon-syndicate` → Set node → параметр `brand`

### A.2 — GTIN Exemption (⚠ Hard prerequisite)
```
SC → Catalog → Add Products → «I'm adding a product not sold on Amazon»
→ категория: Home Décor
→ Apply for GTIN Exemption
→ Brand name: [бренд икон]
→ Comments: «Handmade religious Orthodox icons, each unique, no barcode by nature.»
→ Прикрепить 2–3 фото + описание бизнеса
→ Срок: 3–5 рабочих дней
```

### A.3 — Ungating

SC → Add Products → вбить ASIN из Home Décor:
- Кнопка **«Sell this product»** → категория открыта ✓
- Кнопка **«Apply to sell»** → закрыта (нужны инвойсы $3000+, сайт, 5–14 дней)

При закрытой — заявка с инвойсами или выбрать Wall Art / Religious Items.

### A.4 — Shipping Template (⚠ ОБЯЗАТЕЛЬНО до первого листинга)
```
SC → Settings → Shipping Settings → New Template → «icon-store-standard»
```

| Зона | Скорость | Тариф |
|---|---|---|
| Continental US | Standard (5–8 дней) | $0 или $4.99 |
| Continental US | Expedited (2–3 дня) | $9.99 |
| Alaska / Hawaii | Standard (7–12 дней) | $7.99 |

Проверить в n8n: `amazon-syndicate` → параметр `merchant_shipping_group_name = "icon-store-standard"`

### A.5 — Return Policy

SC → Settings → Return Settings:
- Добавить Return Address (US-адрес)
- Политика: Accept returns within 30 days
- Contact email для вопросов по возврату

---

## 4. Ценообразование

Владелец вводит `site_price` напрямую. Amazon-цена рассчитывается системой автоматически.

| Поле | Кто | Описание |
|---|---|---|
| `site_price` | **Владелец** (каждый товар) | Розничная цена на сайте, USD. Пример: 120.00 |
| `amazon_price_factor` | **Оператор** (один раз глобально) | Коэффициент. 1.0 = одинаково; 1.05–1.15 = чуть выше для покрытия комиссии |
| `amazon_price` | **Авто** | `site_price × amazon_price_factor`. Владелец не вводит |
| `margin_floor` | **Оператор** (один раз) | Минимальная маржа-страховка. Если не покрывается → флаг `MARGIN_WARNING` |

```sql
-- Оператор настраивает один раз:
UPDATE pricing_rules SET amazon_price_factor = 1.0, margin_floor = 0.30
WHERE rule_name = 'icons-default';

-- Пример: site_price = $120, factor = 1.0
-- amazon_price = $120.00
-- Amazon referral fee 15%: net = $120 × 0.85 = $102.00
```

> **⚠ Не ставить `amazon_price` ниже ~$10–15** — Amazon блокирует листинг (`AMZN_PRICE_ALERT`). Рекомендуемый диапазон для икон: $50–200.

**Комиссия Amazon (FBM):**

| | |
|---|---|
| Referral fee (Home Décor / Fine Art) | **15%** от `amazon_price` — вычитается при disbursement |
| Professional fee $39.99/мес | Уже оплачен, не удваивается |
| FBA | ❌ не используется |

---

## 4.5. Загрузка товара в систему

### Инструмент и роли

| Кто | Инструмент | Что делает |
|---|---|---|
| **Владелец** | [app.supabase.com](https://app.supabase.com) → Table Editor | Вводит данные Слоя 1 (факты иконы) в таблицу `icons` |
| **Владелец** | Google Drive → `icon-store-assets/incoming/` | Загружает фотографии |
| **Оператор** | n8n UI | Запускает пайплайны обработки, генерации, публикации |

> **Принцип единого источника:** все данные хранятся в Supabase (PostgreSQL). Amazon и сайт — только выходные каналы. Никогда не редактировать листинги напрямую в Amazon Seller Central.

---

### Шаг 1. Ввод данных в Supabase (Слой 1) — выполняет ВЛАДЕЛЕЦ

```
app.supabase.com → войти → проект icon-store
→ Table Editor → таблица icons → Insert row
```

Заполнить поля Слоя 1 и нажать **Save** (Ctrl+Enter). Серые колонки (`content_status`, `bc_status`, `amazon_status`, `image_qa_status`) — не трогать, их заполняет система.

| Поле | Что вписать | Пример | Обяз. |
|---|---|---|---|
| `saint_or_subject` | Кто изображён (по-английски) | `Mother of God` | ✅ |
| `material` | Материал основы | `Linden wood` | ✅ |
| `technique` | Техника исполнения | `Hand-painted tempera` | |
| `size_w_mm` | Ширина в мм | `120` | ✅ |
| `size_h_mm` | Высота в мм | `160` | ✅ |
| `weight_g` | Вес в граммах | `280` | |
| `manufacturer` | Иконописец или мастерская | `Palekh Studio` | |
| `iconographic_type` | Тип иконографии | `Mother of God` | |
| `construction_type` | Конструкция | `flat-board` / `triptych` / `travel` | |
| `style` | Иконописная школа | `Byzantine` / `Russian` / `Greek` | |
| `purpose` | Назначение | `Home` / `Travel` / `Gift` | |
| `base_cost` | Закупочная цена USD | `45.00` | |
| `site_price` | Розничная цена на сайте USD | `120.00` | ✅ |
| `country_of_origin` | Страна производства (ISO) | `RU` / `GR` / `UA` / `RS` | ✅ |
| `title_raw` | Рабочее название (можно по-русски) | `Казанская Богородица 12×16` | |

> `country_of_origin` и `supplier_declared_dg_hz_regulation` — обязательные поля Amazon. n8n подставляет `not_applicable` для деревянных икон без электроники автоматически.

---

### Шаг 2. Загрузка фотографий — выполняет ВЛАДЕЛЕЦ

```
Google Drive → icon-store-assets → incoming/
→ создать папку с именем артикула: IC-000001
→ загрузить фото внутрь папки
```

> **Имя папки должно совпадать с `merchant_sku` точно.** Например: `IC-000001` — не `IC_000001`, не `ic-000001`, не `000001`. Несовпадение → ошибка `IMG_NO_SUBJECT`.

Минимальные технические требования к фото:

| Параметр | Требование |
|---|---|
| Разрешение | ≥ 2000 px по длинной стороне |
| Формат | JPG или PNG (не HEIC, TIFF, BMP) |
| Фон | Однородный светлый или белый |
| Товар в кадре | ≥ 80% площади, строго фронтально |

Минимальное количество фото по типу иконы:

| Тип | Минимум |
|---|---|
| Плоская доска | front + detail-frame |
| С металлической ризой | front + detail-riza |
| Триптих / складень | front (открыт) + side (закрыт) |
| Путевая икона | front + back + side |

---

### Шаг 3. Запуск пайплайна — выполняет ОПЕРАТОР

После ввода данных и загрузки фото оператор запускает обработку в n8n:

| Шаг | n8n Workflow | Действие |
|---|---|---|
| A | `source-import` | Импорт каталога поставщика (если CSV/Excel) |
| B | `image-ingest` | Обработка фото: удаление фона, деривативы, загрузка в R2 |
| C | `content-generate` | Claude генерирует title, bullets, description, keywords |
| D | `bc-publish` | Публикация карточки в BigCommerce (статус Draft) |
| E | `amazon-syndicate` | Публикация листинга на Amazon через SP-API |

Каждый шаг — после проверки предыдущего. Перед Шагом E — обязательный pre-submit чеклист (раздел 6).

---

## 5. Поля листинга Amazon

> **Источник:** официальный flat file шаблон `WALL_ART_DECORATIVE_SIGNAGE_FINE_ART` (US, март 2026), загруженный из Seller Central. Всего полей в шаблоне: 255. Анализ выполнен по вкладке «Data Definitions».
>
> Шаблон делит поля на три категории: **Required** (Amazon отклонит листинг), **Conditionally Required** (требуется при определённых условиях), **Optional**. Ниже добавлен 4-й уровень: Optional-поля, без которых листинг принимается, но становится Suppressed/неактивным.
>
> ⚠ Flat file и SP-API JSON-схема могут расходиться. Перед реализацией разработчик обязан сверить финальный список с API:
> `GET /definitions/2020-09-01/productTypes/FINE_ART?marketplaceIds=ATVPDKIKX0DER&requirements=LISTING&requirementsEnforced=ENFORCED`

### Уровень 1 — ОБЯЗАТЕЛЬНЫЕ (листинг отклонён или Suppressed)

Всё, что делает листинг недоступным для покупки — либо Amazon отклоняет сразу, либо листинг уходит в Suppressed (скрыт из поиска) или становится неактивным (нет кнопки Buy). Для нас это одинаково плохо — отправлять незаполненным нельзя.

| Поле | Пример | Статус в шаблоне | Кто | Что происходит без него |
|---|---|---|---|---|
| `contribution_sku` | `IC-000042` | Required | АВТО | Amazon не создаст листинг |
| `product_type` | `FINE_ART` | Required | АВТО | Amazon не создаст листинг |
| `item_name` | `Hand-Painted Orthodox Icon of Mother of God, Linden Wood, 5×7 in` | Required | АВТО→ОПЕРАТОР | Amazon не создаст листинг |
| `brand` | `AK Dealer Services` | Required | АВТО | Amazon не создаст листинг |
| `product_id_type` | `ASIN` | Required | АВТО | Amazon не создаст листинг (с GTIN Exemption) |
| `item_type_keyword` | `hand-painted orthodox religious icon on linden wood panel` | Required | АВТО→ОПЕРАТОР | Amazon не создаст листинг / неверная категория |
| `product_description` | Развёрнутое описание, 500–2000 символов | Required | АВТО→ОПЕРАТОР | Amazon не создаст листинг |
| `bullet_point` | 3–5 пунктов | Required | АВТО→ОПЕРАТОР | Amazon не создаст листинг |
| `country_of_origin` | `RU`, `GR`, `UA`, `RS` | Required | ВЛАДЕЛЕЦ | Amazon не создаст листинг |
| `supplier_declared_dg_hz_regulation` | `not_applicable` | Required | АВТО | Amazon не создаст листинг |
| `main_product_image_locator` | URL файла в R2 | Optional* | АВТО | Листинг **Suppressed** — скрыт из поиска |
| `purchasable_offer` → `our_price` | `120.00` | Optional* | АВТО | Листинг неактивен — нет кнопки Buy |
| `condition_type` | `new_new` | Conditionally Req. | АВТО | Без offer-условия листинг неполноценен; для нас всегда `new_new` |
| `fulfillment_availability` → `fulfillment_channel_code` | `DEFAULT` | Conditionally Req. | АВТО | FBM-предложение не создаётся |
| `fulfillment_availability` → `quantity` | `3` | Conditionally Req. | ОПЕРАТОР | «Currently unavailable» — листинг виден, но купить нельзя |
| `merchant_shipping_group` | `icon-store-standard` | Conditionally Req. | АВТО | Amazon применит дефолтный тариф или заблокирует FBM |
| `material` | `Wood` | Conditionally Req. | АВТО→ОПЕРАТОР | Предупреждения по листингу; снижение видимости в категории |
| `wall_art_form` | `Painting` | Conditionally Req. | АВТО | Неверная/отсутствующая подкатегория Wall Art |
| `orientation` | `Portrait` / `Landscape` | Conditionally Req. | АВТО | Снижение видимости; фильтры по ориентации не работают |
| `item_dimensions` (Д × Ш × В) | `7 × 5 × 0.5 in` | Conditionally Req. | АВТО | Покупатель не видит размер; ошибки при расчёте доставки |
| `item_package_weight` | `0.5 lb` | Conditionally Req. | АВТО | Ошибки при расчёте стоимости доставки |
| `keywords` (backend search terms) | до 250 символов | Optional* | АВТО→ОПЕРАТОР | Листинг в конце поиска — покупатели не находят |

> \* В официальном шаблоне помечены Optional, но последствия отсутствия критичны для продаж.

**Примечание:** `item_type_keyword` ≠ `browse_node` — первый обязателен и текстовый, второй опционален и числовой. Хорошие примеры `item_type_keyword`:
```
hand-painted orthodox religious icon on linden wood panel
byzantine style tempera painting on wooden board
russian orthodox icon fine art original on wood
```

Правила для `keywords`: до 250 символов, слова через пробел (не запятые), не повторять слова из `item_name` и `bullet_point` — Amazon их и так индексирует.

---

### Уровень 2 — Optional (улучшают конверсию)

| Поле | Зачем | Кто | Где и как |
|---|---|---|---|
| Галерея (2–8 доп. фото) | Несколько ракурсов → конверсия | АВТО | Python worker создаёт `amazon-gallery` деривативы → n8n |
| `recommended_browse_nodes` | Точнее размещает в дереве категорий | АВТО | n8n → Set node → числовой ID. SC → Help → Browse Tree Guide |
| `list_price` | Зачёркнутая «старая» цена рядом с текущей | АВТО | Опционально; задать как `site_price` до скидки |

---

## 6. Публикация листинга

### Pre-submit чеклист

| Что проверить | Как проверить |
|---|---|
| GTIN Exemption одобрен | SC → Catalog → GTIN Exemption → статус Approved |
| `item_type_keyword` заполнен | Supabase → icons → колонка `item_type_keyword` — конкретный текст, не пусто |
| `amazon_price > 0` и выше минимума | Supabase → icons → `amazon_price` ≥ $10 |
| `compliance_flags` пустые | Supabase → icons → `compliance_flags = []` |
| Фото `amazon-main`: белый фон, без лого | Открыть URL из `derivatives.amazon-main` → визуально проверить |
| Бренд прописан в n8n | n8n → `amazon-syndicate` → Set node → параметр `brand` |
| Shipping template прописан в n8n | n8n → `amazon-syndicate` → `merchant_shipping_group_name = "icon-store-standard"` |

> **⚠ Первый batch — всегда только 1 SKU (canary test).** Один тестовый товар первым, чтобы убедиться в работе цепочки до массовой отправки.

### Процедура подтверждения

1. n8n пришлёт уведомление: «N товаров готово к Amazon-публикации»
2. Supabase → SQL Editor → просмотреть список:
```sql
SELECT merchant_sku, amazon_title, amazon_price, item_type_keyword, country_of_origin
FROM icons
WHERE content_status = 'approved'
  AND bc_status = 'published'
  AND amazon_status = 'not_listed';
```
3. Пройти pre-submit чеклист по каждому SKU
4. n8n → workflow `amazon-syndicate` → Manual Trigger
5. n8n отправит PUT-запрос в SP-API для каждого товара
6. Amazon переведёт листинг в **PENDING** — ожидание 15 мин – 4 часа (норма)
7. При успехе: ASIN записывается в Supabase, `amazon_status = live`
8. При Suppressed: n8n пришлёт alert → исправить по таблице ошибок (раздел 9)

### Визуальная проверка после публикации (checkpoint ⑥)

- [ ] Supabase: `amazon_status = live`, `bc_status = published`, `image_qa_status = approved`
- [ ] Amazon: поиск `IC-XXXXXX` на amazon.com. Фото — белый фон. Title — Title Case. Цена и категория верные.
- [ ] Название бренда совпадает с запланированным
- [ ] Остатки: `site_stock` и `amazon_reserved` ненулевые

---

## 7. Ошибки публикации — таблица решений

| Ошибка | Решение |
|---|---|
| `AMZN_INVALID_ITEM_TYPE` | `item_type_keyword` не распознан. Уточнить — должно быть конкретным описанием физического товара. Обновить в Supabase → перезапустить `amazon-syndicate` |
| `AMZN_PRICE_ALERT` | Цена ниже допустимого минимума. Повысить `amazon_price` → retry |
| `AMZN_IMAGE_REJECTED` | Фото: не белый фон или есть логотипы. Переобработать → retry `amazon-syndicate` |
| `403 Forbidden (SP-API)` | Истёк Refresh Token. SC → Manage Apps → переавторизовать → обновить credentials в n8n |
| Листинг Suppressed | SC → Inventory → Fix stranded listings → исправить → перезапустить `amazon-syndicate` |

---

## 8. Ежедневная и еженедельная рутина

### Каждое утро (5–10 минут)

- [ ] n8n UI → Dashboard → Executions → красные строки (ошибки)
- [ ] Supabase → SQL Editor:
```sql
SELECT merchant_sku, image_qa_status, content_status,
       bc_status, amazon_status, sync_error_code, sync_error_message
FROM icons
WHERE sync_error_code IS NOT NULL
   OR amazon_status = 'suppressed';
```
- [ ] Amazon SC → Inventory → Manage Inventory: статусы Suppressed / Inactive

### Каждую неделю (30–45 минут)

- [ ] SC → Reports → Order Reports → экспорт за прошлую неделю → фильтр SKU `IC-*` → передать владельцу для финансового учёта
- [ ] SC → Performance → Account Health: Order Defect Rate < 1%, Late Shipment Rate < 4%. Если выше — разобраться немедленно
- [ ] Суперпозиция: если `MARGIN_WARNING` флаги в Supabase → обсудить цену с владельцем

---

## 9. Обработка FBM-заказов

FBM (Fulfilled by Merchant) — продавец сам хранит, упаковывает и отправляет.

1. SC → Orders → Manage Orders → статус **Unshipped**
2. Упаковать товар (согласно Amazon packaging rules)
3. SC → Buy Shipping → распечатать этикетку USPS/UPS/FedEx
4. Нажать **Confirm shipment**. Amazon уведомит покупателя. n8n обновит `amazon_reserved - 1` в Supabase

> **⚠ Заказ не подтверждён в течение 7 дней** → Amazon может отменить автоматически.

| Ситуация | Действие |
|---|---|
| Отмена Amazon (до отгрузки) | n8n автоматически вернёт `amazon_reserved + 1` при получении события отмены из Orders API |
| Возврат (после доставки) | SC → Returns → принять. После получения: n8n-форма пополнения → SKU + количество + «return» |

---

## 10. Вариант B — Brand Registry (добавить позже)

Запускаться без него допустимо. Задача на будущее после набора оборотов.

**Путь:** USPTO TEAS Plus, Class 16 и/или 21, $250/класс. Срок: 8–14 месяцев. Serial Number уже даёт доступ к Brand Registry до финального свидетельства. IP Accelerator: ускорение до 4–6 недель, $600–1000.

После одобрения становятся доступны: Brand Store, A+ Content, Brand Analytics, Sponsored Brands.
Подробности — в `amazon-seller-central-requirements.md` разделы 15–17.

---

## 11. Влияние на SP-API (n8n)

Seller ID и credentials не меняются. В workflow `amazon-syndicate`:

```json
"attributes": {
  "brand": [{ "value": "ИМЯ_БРЕНДА_ИКОН", "marketplace_id": "ATVPDKIKX0DER" }]
},
"merchant_shipping_group_name": [{ "value": "icon-store-standard" }]
```

**Как получить актуальные обязательные поля программно:**
```
GET /definitions/2020-09-01/productTypes/FINE_ART
    ?marketplaceIds=ATVPDKIKX0DER
    &requirements=LISTING
    &requirementsEnforced=ENFORCED
```

---

## 12. Финансовое разделение

Amazon выплачивает на один счёт. Разделение в бухгалтерии:
- Теги `AK-Dealer-Auto` и `Icon-Store`
- Фильтр Order Report по SKU-префиксу `IC-`
- Ежемесячный internal transfer доли выручки от икон

---

## 13. Чеклист запуска

**Вариант A (сейчас):**
```
□ Решение об аккаунте зафиксировано в AGENTS.md
□ Название бренда определено
□ GTIN Exemption получен
□ Ungating проверен
□ Shipping Template «icon-store-standard» создан
□ Return Policy настроен
□ amazon_price_factor и margin_floor настроены в Supabase
□ brand + shipping template прописаны в workflow amazon-syndicate
□ Canary test (1 SKU) прошёл
□ Финансовые теги в бухгалтерии настроены
```

**Вариант B (позже):**
```
□ Trademark в USPTO подан
□ Brand Registry активирован
□ Brand Store создана
□ A+ Content добавлен
```

---

## 14. Хронология

```
Неделя 1:    Название бренда, Shipping Template, Return Policy
Неделя 1–2:  GTIN Exemption (3–5 дней)
Неделя 2:    Ungating + бухгалтерские теги + amazon_price_factor
Неделя 2–3:  Canary test (1 SKU) → запуск n8n автоматизации

Параллельно долгосрочно:
Месяц 1:     Заявка на trademark
Месяц 6–14:  Trademark → Brand Registry → Brand Store
```

---

## 15. Словарь Amazon-терминов

| Термин | Значение |
|---|---|
| **SP-API** | Selling Partner API — программный интерфейс Amazon, через который n8n отправляет данные о товарах, получает заказы и обновляет остатки |
| **Refresh Token** | Длительный токен (обычно 1 год), хранится в n8n. Если истёк → n8n получает 403 Forbidden. Переавторизоваться: SC → Manage Apps |
| **GTIN Exemption** | Разрешение публиковать товары без штрих-кода (UPC/EAN/GTIN). Необходимо для изделий ручной работы |
| **FBM** | Fulfilled by Merchant — продавец сам хранит, упаковывает и отправляет |
| **item_type_keyword** | Текстовое описание типа товара для автоматической категоризации. Обязательное поле. |
| **browse_node** | Числовой ID категории Amazon. Опциональное поле. |
| **Canary test** | Запуск одного тестового SKU первым, чтобы убедиться в работе цепочки |
| **Suppressed** | Листинг скрыт Amazon из-за нарушения требований (нет фото, цена ниже минимума и т.п.) |
| **PENDING** | Amazon обрабатывает запрос на публикацию. Норма: 15 мин – 4 ч. Если > 24 ч — alert |
| **Disbursement** | Выплата Amazon продавцу. Обычно раз в 2 недели |
| **Referral fee** | Комиссия Amazon с каждой продажи. Home Décor / Fine Art: **15%** от `amazon_price` |
| **Brand Registry** | Программа для владельцев зарегистрированных торговых знаков. Открывает Brand Store, A+ Content |

---

## 16. Официальная документация

> **С 31 июля 2025 года legacy Feeds API (flat file feeds) устарел.**
> Вся автоматизация через n8n использует **Listings Items API (SP-API)** с JSON-схемой.
> Ручная загрузка через SC (Add Products via Upload) по-прежнему работает.

| Документ | Где найти |
|---|---|
| **Product Type Definitions API** — авторитетный источник обязательных полей | `developer-docs.amazon.com/sp-api/docs/product-type-api-use-case-guide` |
| **Listings Items API v2021-08-01** — создание и обновление листингов | `developer-docs.amazon.com/sp-api/docs/listings-items-api-v2021-08-01-use-case-guide` |
| Маппинг: атрибуты flat file → Listings Items API | `developer-docs.amazon.com/sp-api/docs/mapping-product-attributes` |
| Flat file шаблон для ручной загрузки | SC → Catalog → Add Products via Upload → Download a blank template |
| Browse Tree Guide (числовые node ID) | SC → Help → Browse Tree Guide → Home & Garden |
| Миграция с legacy Feeds API | `developer-docs.amazon.com/sp-api/docs/listing-workflow-migration-tutorial` |
| Brand Registry | `brandregistry.amazon.com` |

---

## 17. Связанные документы

| Документ | Что содержит |
|---|---|
| `MASTER-ARCHITECTURE-v4.md` | Executive layer: роли, принципы, стек |
| `SPEC-1-CATALOG-DATA-MODEL.md` | PostgreSQL DDL, amazon_payload поля |
| `SPEC-3-CONTENT-GENERATION.md` | Amazon-контент: title, bullets, backend_search_terms |
| `SPEC-5-PIPELINES.md` | Pipeline E — SP-API, Shipping Template, листинги |
| `SPEC-7-OPERATOR-PLAYBOOK.md` | Процедуры оператора, статусная матрица |
| `amazon-seller-central-requirements.md` | Полный операционный гайд (поля листинга, рутина, Brand Store) |
