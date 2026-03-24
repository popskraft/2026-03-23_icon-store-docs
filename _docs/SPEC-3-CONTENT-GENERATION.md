# SPEC-3: Генерация контента

Версия: 2.0
Статус: рабочий черновик (parallel workstream)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.2
Область: жизненный цикл контента, промпты, глоссарий, review workflow, версионирование, batch генерация

---

## 1. Назначение

Система берёт базовые факты иконы (Слой 1) и превращает их в канально-специфичный, версионированный, одобренный оператором контент (Слой 2): site + Amazon + SEO. Слой 1 описан в SPEC-1.

---

## 2. Архитектурный контекст (v4.2)

- PostgreSQL — источник истины. `Icon Master Record` — исходный объект.
- BigCommerce и Amazon — publication channels, не источники истины.
- Контент Amazon и контент сайта генерируются независимо из одних базовых фактов.
- Human review обязателен перед публикацией на Amazon.
- Финальный контент — US English. `site_ru` — не launch output.
- Supabase Table Editor — UI для review: Слой 1 и Слой 2 рядом в одном интерфейсе.

---

## 3. Принципы

1. Базовые факты (Слой 1) — каноническая истина. Сгенерированный текст — производные данные.
2. Каждый generated output — структурированный, machine-validatable.
3. Human review обязателен перед Amazon.
4. Ручные правки переживают регенерацию (кроме случаев инвалидации policy).
5. Ограничения платформ соблюдаются при генерации, не после публикации.
6. Один reusable prompt template на content family.
7. Те же базовые факты дают семантически согласованные outputs для сайта и Amazon.

---

## 4. Минимальный набор входных данных

**Обязательные поля (Слой 1) для запуска генерации:**
`merchant_sku` · `title_raw` · `saint_or_subject` · `material` · `technique` · `size_w_mm`, `size_h_mm` · `manufacturer`

**При наличии добавляют:** `feast` · `purpose` · `style` · `iconographic_type` · `construction_type` · `origin_school` · `weight_g`

Если сгенерированный контент будет слишком generic — запись остаётся `needs_review` до получения дополнительных фактов.

---

## 5. Ограничения платформ

**BigCommerce:** custom fields ≤ 250 символов; rate limit 150 req / 30 сек.

**Amazon:** title ≤ 200 символов; backend search terms ≤ 250 байт; повторение слов в title ограничено; main image — белый фон, без логотипов. Генерировать policy-aware payloads без расчёта на ручную чистку.

---

## 6. Глоссарий

Контролируемый глоссарий обеспечивает консистентность по всему каталогу.

**Канонические термины:** `Theotokos` / `Mother of God` · `Jesus Christ` · `Holy Trinity` · `Archangels` · `Byzantine icon` · `Orthodox icon` · `hand-painted` · `wooden board icon`

**Правила:**
- Один предпочтительный display name на канонический предмет.
- Синонимы допустимы в search terms, но не смешиваются в опубликованный title.
- Глоссарий версионируется как код.

---

## 7. Контракт вывода

```json
{
  "sku": "IC-000001",
  "source_hash": "sha256:...",
  "content_version": 1,
  "generated_at": "2026-03-24T00:00:00Z",
  "review_status": "needs_review",
  "amazon": {
    "title": "",
    "bullets": ["", "", "", "", ""],
    "description_html": "",
    "backend_search_terms": "",
    "browse_node": "",
    "item_type_keyword": "",
    "target_marketplace": "US",
    "payload": {}
  },
  "site": {
    "title": "",
    "description_html": "",
    "meta_title": "",
    "meta_description": ""
  },
  "categories": {
    "primary_category": "",
    "faceted_filters": {}
  },
  "compliance_flags": [],
  "review_notes": ""
}
```

**BigCommerce маппинг:** `site.title` → `name`; `site.description_html` → `description`; `site.meta_title` → `page_title`; `site.meta_description` → `meta_description`.

**Amazon payload** хранится в `amazon_payload` Icon Master Record — готовый объект для `PUT /listings/2021-08-01/items/{sellerId}/{sku}`.

---

## 8. Режимы генерации

| Режим | API | Когда |
|---|---|---|
| Синхронный | Messages API | ≤ 10 SKU, test runs, single regen |
| Batch | Message Batches API | > 10 SKU, ночные refreshes |
| Tool-assisted | Messages API + tools | Только если снижает error rate |

Batch: до 100 000 запросов, обработка до 24 ч, результаты JSONL с `custom_id` на SKU.

---

## 9. Review workflow

### Статусы

`not_started` → `generating` → `needs_review` → `approved` → `published` / `rejected`

Primary review для батчей идёт через CSV summary pack. Supabase Table Editor используется для точечных правок и exception handling по отдельным SKU.

### Review pack (per batch)

- JSONL для machine parsing
- CSV summary для human review (колонки: sku, amazon_title, bullet_1..5, site_title, compliance_flags, decision, review_notes)
- Exception report по failed / ambiguous items

### Правила review

- Нарушены Amazon limits → reject или regen.
- Контент изобретает факты отсутствующие в source → reject.
- Семантически верен, но неудачно сформулирован → approve с минорными правками.

### Трассировка approval

Записывает: approver · timestamp · review_notes · content_version · source_hash.

---

## 10. Версионирование

- Каждая регенерация инкрементирует `content_version` при значимом изменении payload.
- Хранится последний approved snapshot для rollback.
- Publication state отдельен от generation state: контент может быть approved, но не опубликован.
- Rollback восстанавливает последний **approved** snapshot, не последний draft.

---

## 11. Правила регенерации

**Регенерировать при:** изменении базовых фактов (Слой 1); изменении глоссарного термина; обновлении Amazon policy; изменении approved master и image-dependent wording.

**Не регенерировать при:** только форматировании; наличии human override; изменении касающемся только одного канала.

Автоматическая регенерация не перетирает approved human edit без записи причины.

---

## 12. Правила валидации

Генератор обязан reject / flag outputs, которые:
- превышают Amazon title или search term limits
- используют запрещённый promotional language
- чрезмерно повторяют слова в title
- ссылаются на недоступные утверждения о материалах или провенансе
- противоречат source record
- содержат HTML в полях где target его не поддерживает

---

## 13. Открытые решения

- Финальный brand tone of voice для US-рынка?
- `Virgin Mary` как контролируемый SEO-синоним или только `Theotokos`?
- Автоматическая регенерация при изменении глоссария — или только с approve оператора?
