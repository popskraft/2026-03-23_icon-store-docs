# PROCESS-FLOW: Краткий сквозной процесс

Version: 1.0
Status: canonical quick guide
Language: Russian
Scope: shortest readable map of the operating process, human checkpoints, and handoffs

---

## Часть A. Главное

### 1. Для чего нужен этот документ

This is the shortest canonical explanation of how one SKU moves through the system.
Read this first.
Technical details live in the SPEC files.

### 2. Процесс за одну минуту

1. Supplier sends source photos and product facts.
2. Owner decides the product should go on sale.
3. Owner enters Layer 1 facts into Supabase Table Editor.
4. Operator validates intake, links files to the record, and assigns the working SKU.
5. Image processor or Python worker prepares the approved master image.
6. Operator approves or rejects the image result.
7. System generates Layer 2 content through n8n -> Claude API.
8. Operator reviews batch content in CSV.
9. System publishes approved data to BigCommerce.
10. Operator gives final Amazon publish approval.
11. System submits the Amazon listing through n8n -> SP-API.
12. Orders and inventory updates sync back into PostgreSQL.

### 3. Four rules that matter most

- PostgreSQL on Supabase is the only source of truth.
- Owner intake happens in Supabase Table Editor, not in Google Sheets.
- Batch content review happens in CSV; Table Editor is only for single-SKU fixes and exceptions.
- Amazon launch path is `manual operator approval -> automated n8n -> SP-API submit`; Seller Central manual listing is emergency fallback only.

---

## Часть B. Карта процесса и точки контроля

### 4. Линейная карта

```text
Supplier
  -> source photos + raw facts

Owner
  -> primary go-to-sale decision
  -> Layer 1 facts in Supabase

Operator
  -> intake validation
  -> file-to-record linking
  -> SKU assignment

Image processor / Python worker
  -> approved master
  -> derivatives

Operator
  -> image QA gate

System
  -> Layer 2 content generation
  -> review pack export

Operator
  -> CSV review gate

System
  -> BigCommerce publish

Operator
  -> Amazon publish gate

System
  -> Amazon SP-API submit
  -> inventory and status sync
```

### 5. Человеческие точки контроля

| Checkpoint | Who decides | What must be true |
|---|---|---|
| Product enters pipeline | Owner | Product should be sold; Layer 1 facts are complete enough to continue |
| Image QA | Operator | Front image is usable; approved master is acceptable; missing media is flagged |
| Content QA | Operator | CSV review says `APPROVE`; risky claims removed; record is fit for publication |
| Amazon publish gate | Operator | GTIN exemption confirmed; browse node validated; payload ready; Amazon publish is explicitly approved |

### 6. Publish-gates по каналам

| Channel | Required gates |
|---|---|
| BigCommerce | `images_approved` + `content_approved` + `price_ready` + `inventory_ready` + `bc_publish_approved` |
| Amazon | everything from BigCommerce gate + `amazon_publish_approved` + confirmed GTIN exemption + validated browse node |

### 7. Правила intake и review

- Owner entry: Supabase Table Editor only.
- Supplier bulk import: CSV / Excel / spreadsheet import is allowed only as an intake source for operator workflows.
- Batch review: CSV is canonical.
- Exception handling: Supabase Table Editor is allowed for one SKU, quick correction, or retry preparation.

### 8. Правило image handoff

1. Owner deposits source files.
2. Operator validates intake.
3. Operator links the files to `icon_id` and assigns `merchant_sku`.
4. Processor edits against that record, not against folder names.
5. Operator accepts or rejects the result.
6. System ingests the approved master and generates derivatives.

Identity is held by `icon_id` plus immutable source file IDs.
Folder names and SKU paths are operational aliases, not the source of truth.

### 9. Куда идти дальше

- Data and field ownership: `SPEC-1-CATALOG-DATA-MODEL.md`
- Images and handoff details: `SPEC-2-IMAGE-PIPELINE.md`
- Content review contract: `SPEC-3-CONTENT-GENERATION.md`
- Amazon controls: `SPEC-4-AMAZON-LAUNCH.md`
- Workflow mechanics: `SPEC-5-PIPELINES.md`
- Operator procedures: `SPEC-7-OPERATOR-PLAYBOOK.md`
