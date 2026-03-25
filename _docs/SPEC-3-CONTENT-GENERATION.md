# SPEC-3: Генерация контента

Версия: 3.0
Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.3
Область: жизненный цикл контента, prompt contract, review workflow, версионирование, batch generation

---

## Часть A. Главное

### 0.1 Краткое резюме

- Layer 1 facts are canonical; generated text is derived data.
- Batch review is CSV-first.
- Table Editor is only for single-SKU fixes, overrides, and exception handling.
- Amazon and storefront content are generated independently from the same facts.
- Human approval is required before publish.

### 0.2 Каноническое правило review

| Scenario | Canonical tool |
|---|---|
| Batch review | CSV review pack |
| One SKU quick fix | Supabase Table Editor |
| Exception handling | Supabase Table Editor |

`approval_source` must record which route was used.

---

## Часть B. Детальный контракт

## 1. Назначение

Система берёт Layer 1 facts и превращает их в channel-specific, versioned, operator-approved content for site and Amazon.

---

## 2. Архитектурный контекст

- PostgreSQL is the source of truth.
- BigCommerce and Amazon are publication targets.
- Final launch content is US English.
- `site_ru` is not a launch output.
- Content approval and publish approval are separate decisions.

---

## 3. Principles

1. Source facts win over generated text.
2. Generated output must be structured and machine-validatable.
3. Human review is mandatory before publish.
4. Manual operator edits must survive regeneration unless explicitly invalidated.
5. Platform limits are enforced before publish, not after.

---

## 4. Minimum input set

Required to start generation:
- `merchant_sku`
- `title_raw`
- `saint_or_subject`
- `material`
- `technique`
- `size_w_mm`
- `size_h_mm`
- `manufacturer`

If facts are too weak, record stays in `needs_review` instead of publishing generic content.

---

## 5. Output contract

```json
{
  "sku": "IC-000001",
  "source_hash": "sha256:...",
  "content_version": 1,
  "generated_at": "2026-03-25T00:00:00Z",
  "review_status": "needs_review",
  "approval_source": null,
  "amazon": {
    "title": "",
    "bullets": ["", "", "", "", ""],
    "description_html": "",
    "backend_search_terms": "",
    "browse_node": "",
    "item_type_keyword": "",
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

---

## 6. Review workflow

### 6.1 Statuses

`not_started` -> `generating` -> `needs_review` -> `approved`

Publication state is tracked separately from generation state.

### 6.2 Batch review

Primary review path:
- n8n generates JSONL + CSV pack
- operator reviews CSV
- operator sets `APPROVE`, `REJECT`, or `REGEN`
- system writes back result and `approval_source = csv_batch`

### 6.3 Single-SKU correction

Allowed only when:
- one record needs a quick fix
- batch review already happened
- operator is resolving an exception

Then `approval_source = table_editor_exception`.

### 6.4 Approval trace

Track:
- approver
- timestamp
- approval source
- content version
- source hash
- review notes

---

## 7. Platform limits

### 7.1 BigCommerce

- custom fields <= 250 chars

### 7.2 Amazon

- title <= 200 chars
- backend search terms <= 250 bytes
- no prohibited promotional language
- no invented facts
- main image rules are enforced upstream in Spec-2

---

## 8. Regeneration rules

Regenerate when:
- Layer 1 facts changed
- glossary changed
- platform policy changed
- approved image changed and wording depends on image

Do not auto-overwrite approved human edits without recording why.

---

## 9. Validation rules

Reject or flag output when it:
- exceeds platform limits
- repeats keywords unnaturally
- invents provenance or material claims
- contradicts source facts
- uses unsupported HTML

---

## 10. Открытые вопросы

1. Final US tone of voice for launch copy.
2. Controlled synonym policy for `Theotokos` vs `Virgin Mary`.
3. Auto-regeneration policy after glossary updates: automatic or operator-confirmed.
