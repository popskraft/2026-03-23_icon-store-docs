# SPEC-5: Пайплайны — Операционные Workflows A-G

Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.3
Область: все 7 операционных пайплайнов системы, триггеры, gate conditions, sync and retry rules

---

## Часть A. Главное

### 0.1 Краткое резюме

```text
A intake -> B images -> C content -> D BigCommerce -> E Amazon -> F inventory -> G batch engine
```

- Owner intake is direct in Supabase.
- Supplier files may enter through operator-led bulk import.
- Batch content review is CSV-first.
- Amazon publish is operator-approved and SP-API-driven.
- External sync always uses durable state and reconciliation.

### 0.2 Глобальные правила

- `content_approved` alone is not publish permission.
- Channel workflows start only when the full gate predicate is true.
- `sync_state` must be durable and explicit.
- Retryable failures and terminal failures must be separated.

---

## Часть B. Детальный контракт

## 1. Global channel gates

### 1.1 BigCommerce gate

Required:
- `images_approved = true`
- `content_approved = true`
- `price_ready = true`
- `inventory_ready = true`
- `bc_publish_approved = true`

### 1.2 Amazon gate

Required:
- full BigCommerce gate
- `amazon_publish_approved = true`
- GTIN exemption confirmed
- browse node validated
- approved payload hash present

---

## 2. Pipeline A — Catalog intake

### Главное

- owner input: Supabase Table Editor
- supplier bulk import: CSV / Excel handled by operator workflow
- Supabase wins on conflict

### Output

- canonical record created or patched in PostgreSQL
- `image_qa_status = incoming`
- `content_status = not_started`

### Failure rules

- missing required fields -> reject row
- duplicate `merchant_sku` -> reject row
- invalid size or cost -> reject row

---

## 3. Pipeline B — Image processing

### Trigger

`image_qa_status = incoming`

### Flow

1. operator validates intake
2. processor or worker creates approved master
3. operator accepts or rejects
4. system generates derivatives

### Output

- `image_qa_status = approved` or terminal reject state
- master and derivatives written to R2/S3

---

## 4. Pipeline C — Content generation

### Trigger

`image_qa_status = approved` and `content_status = not_started`

### Flow

1. n8n builds prompt input JSON
2. Claude API generates channel content
3. n8n validates platform limits
4. review CSV pack is exported
5. operator decides `APPROVE`, `REJECT`, or `REGEN`

### Review contract

- batch review -> CSV
- single record correction -> Table Editor only for exception handling
- `approval_source` is mandatory

---

## 5. Pipeline D — Publish to BigCommerce

### Trigger

full BigCommerce gate is true

### Flow

1. create or update BC product
2. attach images
3. write metafields
4. reconcile API result back to PostgreSQL

### Failure rule

- external API success is not final until reconciliation completes

---

## 6. Pipeline E — Amazon syndication

### Trigger

full Amazon gate is true

### Flow

1. operator explicitly approves Amazon publish
2. n8n validates payload, category, and compliance
3. `PUT /listings/.../items/{sellerId}/{sku}`
4. reconciliation writes ASIN, status, and sync result

### Hard rules

- BigCommerce connector is not used
- manual Seller Central listing is fallback only
- first live path should be a canary listing on 1 SKU

---

## 7. Pipeline F — Inventory sync

### Canonical model

- `physical_on_hand`
- `site_stock`
- `site_reserved`
- `amazon_reserved`
- `committed`
- `released`
- `available_to_allocate`
- `inventory_version`

### Rules

- first successful commit wins
- checkout reserve for site is TTL-based
- critical mismatch can auto-pause listing
- publish-critical updates must respect record versioning

---

## 8. Pipeline G — Batch engine

### Required durable fields

- `workflow_run_id`
- `step_name`
- `step_attempt`
- `idempotency_key`
- `sync_state`
- `bc_sync_version`
- `amazon_sync_version`

### Sync states

- `pending_sync`
- `sync_in_progress`
- `sync_succeeded`
- `sync_failed_transient`
- `sync_failed_terminal`

### Rules

- repeat with same input must be safe
- external side effects must be retry-aware
- failed item does not invalidate the whole batch

---

## 9. Error handling

| Error class | Handling |
|---|---|
| transient API failure | retry with backoff |
| validation failure | fail closed, do not publish |
| partial external success | reconcile, then decide retry |
| stale version conflict | re-read record, do not force publish |

---

## 10. n8n workflow groups

- `intake-and-normalize`
- `image-processing`
- `content-generation`
- `bc-publication`
- `amazon-syndication`
- `inventory-sync`
- `batch-control-and-alerting`

---

## 11. Открытые вопросы

1. Exact retry thresholds after pilot.
2. Final browse node after Product Classifier validation.
3. Final variant strategy for BC product model.
