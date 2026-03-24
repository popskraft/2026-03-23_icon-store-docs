# SPEC-4: Операционный запуск Amazon

Версия: 2.0
Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.2
Область: добавление бренда икон в существующий SC-аккаунт, GTIN Exemption, Brand Registry, Shipping Template, SP-API интеграция

---

## Архитектурный контекст (v4.2)

- Amazon Seller Central + FBM — подтверждённый канал (FBA не используется).
- Синдикация: n8n → SP-API напрямую (BC-коннектор не используется).
- Иконный бренд запускается внутри существующего SC-аккаунта AK Dealer Services.
- Все листинги — один SC-кабинет; разделение финансов — на уровне бухгалтерии.
- `merchant_sku` формата `IC-XXXXXX` — идентификатор иконных листингов.
- GTIN Exemption необходим до первой публикации (handmade religious items без UPC/EAN).
- Human review обязателен перед любой публикацией (Пайплайн E, SPEC-5).
- Browse node: подход определён (Home Décor как старт), финальный node ID — после пилота.

---

## 1. Контекст

| Параметр | Магазин 1 | Магазин 2 (иконы) |
|---|---|---|
| SC-аккаунт | AK Dealer Services | Тот же ✅ |
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
| Brand Store | ❌ | ✅ |
| A+ Content | ❌ | ✅ |
| **Рекомендация** | ✅ Запускать сразу | ✅ Добавить позже |

---

## 3. Вариант A — Прямые листинги

**A.1 — Название бренда:** отличается от «AK Dealer Services», стабильное.

**A.2 — GTIN Exemption:**
```
SC → Catalog → Add Products → «I'm adding a product not sold on Amazon»
→ категория: Home Décor
→ Apply for GTIN Exemption
→ Brand name: [бренд икон]
→ Reason: «Handcrafted religious items without standard barcodes.»
→ Прикрепить 2–3 фото + описание бизнеса
→ Срок: 3–5 рабочих дней
```

**A.3 — Ungating:** SC → Add Products → «orthodox icon». Если 🔒 Collectibles — нужны инвойсы $3000+, сайт, 5–14 дней. Home Décor — открытая, ungating не нужен.

**A.4 — Shipping Template:**
```
SC → Settings → Shipping Settings → New Template → «icon-store-standard»
US continental Standard:  $0 или $4.99,  5–8 дней
US continental Expedited: $9.99,          2–3 дня
Alaska/Hawaii Standard:   $7.99,          7–12 дней
```

**A.5 — Return Policy:** адрес возврата, 30-дневное окно.

**Структура листинга (поля из PostgreSQL):**
```
Product name:      [amazon_title]
Brand:             [бренд икон]
SKU:               [merchant_sku — IC-XXXXXX]
Price:             [amazon_price]
Quantity:          [amazon_reserved]
Fulfillment:       FBM
Shipping template: icon-store-standard
Bullet points:     [amazon_bullet_1..5]
Description:       [amazon_description_html]
Keywords:          [amazon_backend_search_terms]
Главное фото:      amazon-main (белый фон, 2000×2000 px)
```

---

## 4. Вариант B — Brand Registry (добавить позже)

Требование: зарегистрированный trademark в USPTO. TEAS Plus, Class 16 и/или 21. $250/класс. Срок: 8–14 месяцев. Serial Number используется для BR пока trademark pending.

IP Accelerator: ускорение до 4–6 недель. Стоимость: $600–1000.

После одобрения: Brand Store → A+ Content → Brand Analytics → Sponsored Brands.

---

## 5. Влияние на SP-API (n8n)

Seller ID и credentials не меняются. В workflow `amazon-syndicate`:

```json
"attributes": {
  "brand": [{ "value": "ИМЯ_БРЕНДА_ИКОН", "marketplace_id": "ATVPDKIKX0DER" }]
},
"merchant_shipping_group_name": [{ "value": "icon-store-standard" }]
```

---

## 6. Финансовое разделение

Amazon выплачивает на один счёт. Разделение в бухгалтерии:
- Теги `AK-Dealer-Auto` и `Icon-Store`
- Фильтр Order Report по SKU-префиксу `IC-`
- Ежемесячный internal transfer доли выручки от икон

---

## 7. Комиссии (FBM)

| | |
|---|---|
| Referral fee (Home Décor) | 15% от цены |
| Professional fee $39.99/мес | Уже оплачен, не удваивается |
| FBA Storage / Fulfillment | ❌ |

---

## 8. Чеклист запуска

**Вариант A:**
```
□ Название бренда определено
□ GTIN Exemption получен
□ Ungating проверен
□ Shipping Template «icon-store-standard» создан
□ Return Policy настроен
□ brand + shipping template прописаны в workflow amazon-syndicate
□ Тестовый листинг (1 SKU) прошёл
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

## 9. Хронология

```
Неделя 1:    Название бренда, Shipping Template, Return Policy
Неделя 1–2:  GTIN Exemption (3–5 дней)
Неделя 2:    Ungating + бухгалтерские теги
Неделя 2–3:  Тестовый листинг → запуск n8n автоматизации

Параллельно долгосрочно:
Месяц 1:     Заявка на trademark
Месяц 6–14:  Trademark → Brand Registry → Brand Store
```

---

## 10. Связанные документы

| Документ | Что содержит |
|---|---|
| `MASTER-ARCHITECTURE-v4.md` | Executive layer: роли, принципы, стек |
| `SPEC-1-CATALOG-DATA-MODEL.md` | PostgreSQL DDL, amazon_payload поля |
| `SPEC-3-CONTENT-GENERATION.md` | Amazon-контент: title, bullets, backend_search_terms |
| `SPEC-5-PIPELINES.md` | Pipeline E — SP-API, Shipping Template, листинги |
| `SPEC-7-OPERATOR-PLAYBOOK.md` | Процедуры оператора, статусная матрица |
