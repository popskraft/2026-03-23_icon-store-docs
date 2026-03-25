# SPEC-7: Плейбук оператора

Версия: 3.0
Статус: рабочий черновик (implementation-facing)
Язык: русский
Базис: `MASTER-ARCHITECTURE-v4.md` v4.3
Область: end-to-end процедуры оператора, human gates, daily routine, incident handling

---

## Часть A. Главное

### 0.1 Краткое резюме

Operator owns:
- intake validation
- image QA
- content review
- channel publish approvals
- exception handling
- daily system monitoring

Read `PROCESS-FLOW.md` first for the shortest route through the system.

### 0.2 Non-negotiable rules

- PostgreSQL is the source of truth.
- Batch content review is CSV-first.
- Table Editor is for one-SKU fixes and exceptions.
- Amazon publish needs explicit operator approval.
- Manual Seller Central listing is fallback only.

---

## Часть B. Детальная процедура

## 1. Global status logic

```text
new SKU
  -> image intake
  -> approved image
  -> generated content
  -> approved content
  -> BigCommerce publish
  -> Amazon publish approval
  -> Amazon live
```

### Required gates

| Channel | Required |
|---|---|
| BigCommerce | `images_approved`, `content_approved`, `price_ready`, `inventory_ready`, `bc_publish_approved` |
| Amazon | all BC gates + `amazon_publish_approved`, GTIN exemption confirmed, browse node validated |

---

## 2. One-time setup before first SKU

1. Deploy VPS with self-hosted n8n CE.
2. Apply DDL from Spec-1.
3. Create R2/S3 bucket structure.
4. Configure credentials in n8n.
5. Connect Google Drive inbox.
6. Obtain GTIN exemption.
7. Run one end-to-end canary SKU without mass publish.

---

## 3. Step-by-step operating flow

### 3.1 Intake

- confirm product is approved by owner for sale
- verify Layer 1 facts exist in Supabase
- if supplier file import is used, import into PostgreSQL and verify result

### 3.2 Images

- validate source files
- link files to `icon_id`
- assign or confirm SKU
- send to processor or worker
- accept or reject result

### 3.3 Content

- trigger generation when images are approved
- review batch in CSV
- use `APPROVE`, `REJECT`, `REGEN`
- use Table Editor only for exception cases

### 3.4 BigCommerce

- confirm full BC gate
- let n8n publish
- verify reconcile result if workflow reports success

### 3.5 Amazon

- verify GTIN exemption
- verify browse node and category readiness
- explicitly approve Amazon publish
- visually check listing after SP-API submit

---

## 4. Daily routine

Morning checklist:
- review failed n8n executions
- review failed or partial batch items
- review low-stock alerts
- review suppressed Amazon listings

New order checklist:
- ship FBM order
- enter tracking
- verify order is marked shipped

Stock update checklist:
- update canonical stock through the approved workflow
- verify sync completed

---

## 5. Incident handling

| Incident | Operator action |
|---|---|
| image issue | reject or send to retouch, never silently publish |
| content issue | reject or regen, then re-review |
| BC publish mismatch | reconcile canonical record before retry |
| Amazon suppression | fix canonical record, then resubmit |
| inventory mismatch | stop relying on stale channel data; reconcile from PostgreSQL |

---

## 6. Quick SQL targets

Use SQL for:
- validating recent imports
- checking publish flags
- inspecting failed batch items
- confirming ASIN / BC IDs after publish

This document intentionally does not keep a large SQL snippet library; add only reusable queries that survive repeated use.

---

## 7. Открытые вопросы

1. Final workflow for entering FBM tracking numbers.
2. Backup and restore runbook for PostgreSQL control plane.
3. Final exception-triage ownership if more than one operator appears later.
