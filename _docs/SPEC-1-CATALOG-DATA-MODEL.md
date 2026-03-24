# SPEC-1: Catalog & Data Model

Status: Implementation-facing draft  
Baseline: `MASTER-ARCHITECTURE-v4.md`  
Scope: canonical catalog registry, identifier policy, field ownership, channel mapping, import/update contracts

## 1. Purpose

This spec defines the canonical data model for the icon catalog. It is the contract that all downstream workflows must follow:

- source intake
- image processing
- content generation
- BigCommerce publication
- Amazon syndication
- batch updates for price, inventory, and catalog changes

The project treats the catalog as an operational system, not just a storefront. The canonical record lives outside BigCommerce in PostgreSQL. BigCommerce is the publication layer for the owned storefront and the launch commerce channel.

## 2. Design Principles

1. One canonical record per sellable icon SKU.
2. One internal identifier policy across all systems.
3. Supplier identifiers are preserved, but they do not become the system key.
4. BigCommerce stores published commerce fields, not the entire operational ledger.
5. Image originals and approved masters remain outside BigCommerce.
6. All public-facing text is generated from the canonical record and must pass human review before publication.
7. Batch operations must be idempotent and replayable.
8. English is the launch language for project-facing and channel-facing output.

## 3. Platform Constraints That Shape This Spec

These constraints come from current BigCommerce documentation and should be treated as implementation rules for launch.

| Constraint | Implication for this spec |
|---|---|
| Products can be created and managed through the V3 Catalog API. | Use V3 as the default integration surface. |
| Products may be assigned to multiple categories. | Keep an internal primary category, but do not assume a single native primary category in BigCommerce. |
| Category trees are the modern catalog structure and support batch operations. | Use category trees for catalog hierarchy management. |
| Variants are SKU-bearing sellable items and track inventory. | Use variants for sellable size/edition changes. |
| Modifiers do not change the fulfilled SKU and cannot be used to track inventory combinations. | Do not use modifiers for sellable size variants. |
| Custom fields are short shopper-visible attributes. | Use them only for display facts, not operational state. |
| Metafields are key-value operational data and are hidden from the storefront by default. | Use metafields for sync state, source references, and internal channel payloads. |
| Product SEO fields support page title and meta description. | Store these in the canonical record and publish them to BigCommerce. |
| Product images can be uploaded through the Catalog API. | BigCommerce stores published copies; the canonical image source remains elsewhere. |
| Inventory API writes are asynchronous and not channel-aware. | Keep site inventory and Amazon FBA inventory separate and serialize updates. |
| BigCommerce pricing is precise to 4 decimal places. | Price batches must preserve precision and rounding rules. |
| Standard/Plus API rate limits are 150 requests per 30 seconds. | Batch writers must back off and avoid uncontrolled parallelism. |
| Price list bulk upserts must be serialized. | Do not run multiple price-list bulk operations in parallel on the same store. |
| Enterprise-only multi-storefront localization features exist for SEO, custom fields, and variant options. | Do not make launch success depend on locale overrides or Enterprise-only enhancements. |

## 4. Canonical Architecture

### 4.1 System of record

The canonical object is the `Icon Master Record`, stored in PostgreSQL.

### 4.2 System roles

| System | Role |
|---|---|
| PostgreSQL | Canonical catalog registry and audit ledger |
| Google Drive | Human inbox for incoming photos and source files |
| Object storage (S3-compatible) | Approved master images and generated derivatives |
| BigCommerce | Published storefront catalog and checkout layer |
| Amazon Seller Central | Marketplace listing and FBA state |
| n8n | Orchestration and batch control |
| Claude API | Structured content generation |

### 4.3 Data flow

`source files -> canonical record -> image approval -> content generation -> human review -> BigCommerce publish -> Amazon syndication -> inventory/price sync`

## 5. Identifier Policy

### 5.1 Required identifiers

Every icon record must have the following identifiers:

- `icon_id` - internal immutable UUID
- `merchant_sku` - the primary business key for all downstream systems
- `supplier_sku` - the supplier or author SKU, preserved as an external reference
- `product_family_id` - groups sellable variants that belong to the same design family
- `bigcommerce_product_id` - assigned after publication
- `bigcommerce_variant_id` - assigned when a variant exists
- `asin` - assigned only after an Amazon listing exists

### 5.2 Merchant SKU policy

The merchant SKU is the stable identifier used in:

- PostgreSQL
- batch manifests
- CSV imports
- BigCommerce product and variant SKUs
- Amazon listing payloads

Recommended format:

- `IC-000001`
- `IC-000001-S`
- `IC-000001-M`
- `IC-000001-L`

Rules:

- Do not encode theology, saints, materials, or taxonomy into the SKU itself.
- Keep the SKU simple and sequential.
- Keep semantic meaning in fields, not in the identifier string.
- Never reuse a retired merchant SKU for a different product.

### 5.3 Supplier SKU policy

The supplier SKU is always preserved, even if it is messy or inconsistent.

Rules:

- treat supplier SKU as an external reference only
- never use supplier SKU as the primary key
- allow multiple supplier SKUs to point to one merchant SKU when catalog consolidation is required

## 6. Canonical Icon Master Record

The master record is the source of truth for all publishable and operational catalog state.

### 6.1 Identity block

| Field | Description |
|---|---|
| `icon_id` | Internal UUID |
| `merchant_sku` | Primary business key |
| `supplier_sku` | Supplier reference |
| `product_family_id` | Family grouping for variants |
| `asin` | Amazon identifier, nullable until listing exists |
| `bigcommerce_product_id` | BigCommerce product id, nullable until publication |
| `bigcommerce_variant_id` | BigCommerce variant id, nullable until publication |

### 6.2 Source facts block

| Field | Description |
|---|---|
| `title_raw` | Minimal source title from supplier or owner |
| `saint_or_subject` | Who or what is depicted |
| `feast` | Feast or commemoration, if applicable |
| `purpose` | Intended use case or devotional purpose |
| `style` | Stylistic label, e.g. Byzantine |
| `iconographic_type` | Type such as Mother of God, Christ, Saint, Archangel |
| `material` | Base material and finish |
| `technique` | Production technique |
| `construction_type` | Single board, triptych, cased icon, etc. |
| `origin_school` | School or origin tradition |
| `size_w_mm` | Width in millimeters |
| `size_h_mm` | Height in millimeters |
| `size_d_mm` | Depth in millimeters |
| `weight_g` | Weight in grams |
| `manufacturer` | Author, atelier, or manufacturer |
| `notes` | Freeform operational notes |

### 6.3 Media block

| Field | Description |
|---|---|
| `source_system` | `google_drive`, `email`, `s3_upload`, or `manual` |
| `source_folder_id` | Stable folder id from the source system |
| `source_file_id` | Stable file id from the source system |
| `source_filename` | Original incoming filename |
| `received_at` | Timestamp of intake |
| `incoming_hash` | Hash of the incoming file |
| `approved_master_uri` | URI of the approved master image in object storage |
| `approved_master_hash` | Hash of the approved master image |
| `derivatives` | JSON object mapping derivative names to URIs |
| `image_qa_status` | `incoming`, `retouch_required`, `processing`, `media_blocked`, `media_partial`, `media_ready`, `approved`, `rejected` |
| `image_qa_notes` | Human QA notes |
| `last_processed_at` | Timestamp of the last image processing pass |

### 6.4 Content block

| Field | Description |
|---|---|
| `content_version` | Monotonic version number |
| `content_status` | `not_started`, `generating`, `review`, `approved`, `published` |
| `approved_by` | Human approver |
| `approved_at` | Approval timestamp |
| `review_notes` | Human review notes |
| `site_title_en` | Storefront title in English |
| `site_description_html_en` | Storefront long description in English |
| `meta_title_en` | SEO title |
| `meta_description_en` | SEO description |
| `amazon_title` | Amazon title |
| `amazon_bullet_1..5` | Amazon bullets |
| `amazon_description_html` | Amazon description HTML or structured text |
| `amazon_backend_search_terms` | Amazon backend keywords |
| `amazon_browse_node` | Amazon browse node id or mapped node string |
| `amazon_item_type_keyword` | Amazon item type keyword |
| `amazon_target_marketplace` | Target marketplace, e.g. `US` |
| `compliance_flags` | Review flags for risky claims or content |

### 6.5 Pricing block

| Field | Description |
|---|---|
| `base_cost` | Acquisition or landed cost |
| `pricing_rule_id` | Link to pricing rule |
| `site_price` | Published storefront price |
| `amazon_price` | Published Amazon price |
| `map_price` | Minimum advertised price if applicable |
| `price_last_updated` | Timestamp of last pricing change |

### 6.6 Inventory and publication block

| Field | Description |
|---|---|
| `site_sellable_qty` | Quantity available for the owned storefront |
| `fba_sellable_qty` | Quantity available through FBA |
| `reorder_point` | Reorder threshold |
| `supplier_lead_time_days` | Expected replenishment lead time |
| `replenishment_status` | Operational replenishment state |
| `bc_status` | `draft`, `published`, `archived` |
| `amazon_status` | `not_listed`, `pending`, `live`, `suppressed`, `live_manual` |
| `last_hash` | Hash of last published state |
| `last_synced_at` | Last successful sync timestamp |
| `last_batch_id` | Batch id that wrote the last state |
| `sync_error_code` | Current sync error code, if any |
| `sync_error_message` | Current sync error message, if any |

## 7. Field Ownership

The canonical owner is the system that is allowed to author the field first.

| Field group | System of record | BigCommerce storage | Amazon storage | Notes |
|---|---|---|---|---|
| Identity | PostgreSQL | Product or variant SKU | Listing SKU / ASIN reference | `merchant_sku` is primary |
| Supplier reference | PostgreSQL | Metafield only if needed | Not required | Keep as external reference |
| Source facts | PostgreSQL | Shopper-facing subset only | Listing payload subset only | Do not rely on BigCommerce as source |
| Approved master image URIs | Object storage + PostgreSQL | Not the source of truth | Not the source of truth | BigCommerce stores published copies only |
| Derivative image URIs | Object storage + PostgreSQL | Product images after publish | Listing image payload | Derivatives are generated, not hand authored |
| Content | PostgreSQL | Product title, description, SEO fields, metafields | Listing title, bullets, keywords, description | One approved content version per publish cycle |
| SEO | PostgreSQL | `page_title`, `meta_description` | Not applicable | Launch is English only |
| Categories | PostgreSQL | Category assignments | Browse node mapping | Keep internal primary category |
| Inventory | PostgreSQL | Site sellable qty | FBA sellable qty | Site and Amazon pools remain separate |
| Pricing | PostgreSQL | Product price or price list state | Listing price | Publish from rules, not manual edits |
| Sync state | PostgreSQL | `ops.sync` metafields only | Listing status references | Do not store sync state only in the storefront |

## 8. BigCommerce Mapping Contract

### 8.1 Product object

Use the BigCommerce product object for the storefront-visible product shell.

Recommended mapping:

| BigCommerce field | Source field | Notes |
|---|---|---|
| `name` | `site_title_en` or a publish-ready product title | English only for launch |
| `sku` | `merchant_sku` for simple products or base variant SKU for variant products | Must remain stable |
| `type` | `physical` | All icons in this project are physical |
| `price` | `site_price` | Use variant price when variants exist |
| `categories` | category assignment list | Multiple categories allowed |
| `weight` | `weight_g` converted to BigCommerce unit | Keep conversion in the batch writer |
| `page_title` | `meta_title_en` | Product SEO field |
| `meta_description` | `meta_description_en` | Product SEO field |
| `custom_fields` | Shop-facing compact facts | Use only if shopper-visible |
| `metafields` | Operational metadata and channel payloads | Use namespaces defined below |

### 8.2 Variants vs modifiers

Use variants when the choice changes any of the following:

- SKU
- inventory
- price
- fulfillment
- physical weight or shipping behavior

Use modifiers only when the choice is shopper-facing but does not change inventory or fulfillment.

For this project:

- size differences that change sellable stock should be variants
- decorative add-ons that do not change the sellable SKU may be modifiers
- do not model sellable icon size changes as modifiers

### 8.3 Category trees

BigCommerce category trees are the canonical catalog hierarchy for the storefront.

Rules:

- store category logic internally in PostgreSQL
- publish one product to multiple BigCommerce categories when needed
- maintain an internal `primary_category_id`
- keep launch taxonomy simple and SEO-friendly
- do not rely on the storefront menu as the catalog hierarchy
- category assignment is not channel assignment
- a product must be explicitly assigned to a storefront channel to be sellable on that channel
- category placement alone does not make a product available on a channel

### 8.4 Custom fields

Use custom fields for compact shopper-visible facts only.

Recommended examples:

- Material
- Technique
- Size
- Style
- Origin
- Care instructions

Rules:

- keep values short
- custom field values are limited to 250 characters
- do not store operational state here
- do not store hashes, sync ids, or source URIs here
- treat custom fields as display data, not control-plane data

### 8.5 Metafields

Use metafields for operational data that must travel with the product in BigCommerce but should not be treated as storefront content.

Recommended namespaces:

- `ops.review`
- `ops.sync`
- `media.storage`
- `content.amazon`
- `content.site`
- `catalog.source`

Examples of metafield use:

- source SKU mapping
- master record ids
- sync timestamps
- image source references
- content approval state
- Amazon channel payload fragments

Rules:

- do not mirror the entire PostgreSQL record into metafields
- only store what downstream automation actually needs
- keep metafields consistent with the master record
- if a field is shopper-visible, prefer the published BigCommerce product field or custom field instead

### 8.6 Images

BigCommerce should store published copies of product images only.

Rules:

- the canonical image source remains in approved master storage
- the first product image is the thumbnail unless a different order is explicitly required
- one product can have only one thumbnail image
- a single image can serve both thumbnail and main image
- variant images may be added only when the variant actually differs visually

Operational pattern:

1. approved master image is stored in object storage
2. derivative images are generated by the image worker
3. the batch publisher uploads the selected publish images to BigCommerce
4. the master record stores the storage URIs and hashes

### 8.7 SEO

Store and publish these fields:

- `meta_title_en`
- `meta_description_en`

Rules:

- launch content is English only
- locale-specific SEO overrides are out of scope for the initial Standard launch
- keep page titles concise and descriptive

### 8.8 Inventory

BigCommerce inventory must represent the storefront sellable pool only.

Rules:

- if the product has variants, track inventory at the variant level
- use BigCommerce inventory updates for the storefront pool only
- keep Amazon FBA inventory in the master record and Amazon state
- do not treat BigCommerce inventory as the canonical FBA pool
- serialize inventory writes
- avoid running bulk inventory changes in parallel with bulk catalog or order operations

### 8.9 Pricing

Store pricing in the master record first, then publish it to BigCommerce.

Rules:

- `site_price` is the storefront price
- `amazon_price` is the marketplace price
- `base_cost` is internal only
- price precision must respect BigCommerce's 4-decimal behavior
- batch price updates must be idempotent
- manual storefront price changes are exception-only and must be audited

Price Lists are optional for launch. If used later, they must be modeled as a separate pricing rule layer and not as a replacement for the master record.

## 9. Import and Batch Contracts

### 9.1 Source intake formats

Supported input formats:

- CSV
- Excel
- JSONL manifest
- image folders in Google Drive
- ZIP packages from suppliers

### 9.2 Canonical batch types

| Batch type | Purpose |
|---|---|
| `source-import` | Normalize supplier data into the master record |
| `image-ingest` | Intake and approve image assets |
| `content-generate` | Generate site and Amazon content |
| `content-republish` | Rebuild approved content versions |
| `bc-publish` | Create or update BigCommerce products |
| `amazon-syndicate` | Prepare or update Amazon listing payloads |
| `price-update` | Apply rule-driven pricing changes |
| `inventory-sync` | Reconcile site and FBA quantities |
| `category-remap` | Rebuild taxonomy mapping |
| `media-refresh` | Reprocess approved master images |
| `seo-refresh` | Recompute SEO fields |

### 9.3 Canonical import contract

Each batch must carry the following top-level keys:

```json
{
  "batch_id": "2026-03-23-source-import-001",
  "batch_type": "source-import",
  "source_system": "google_drive",
  "source_reference": {
    "folder_id": "drive-folder-id",
    "file_id": "drive-file-id"
  },
  "input_hash": "sha256:...",
  "record_count": 250,
  "created_at": "2026-03-23T12:00:00Z",
  "status": "created"
}
```

### 9.4 Per-SKU normalized manifest

The batch writer should transform supplier input into a normalized per-SKU manifest before publishing anything.

Each row must include:

- `merchant_sku`
- `supplier_sku`
- `title_raw`
- `saint_or_subject`
- `material`
- `technique`
- `size_w_mm`
- `size_h_mm`
- `size_d_mm`
- `source_file_id` or equivalent image reference
- `approved_master_uri` when available
- `content_version`
- `batch_id`
- `source_hash`
- `target_hash`
- `operation`
- `changed_fields`

### 9.5 Idempotency rules

The following rules are mandatory:

- one batch type per batch execution
- repeated execution with the same input hash must be safe
- unchanged rows must be no-ops
- changed rows must be field-level updates, not full destructive rewrites
- if a row fails validation, the row is rejected without invalidating the entire batch unless the failure affects a required shared asset

### 9.6 Validation and gating

Before any publish step, validate:

- `merchant_sku` uniqueness
- presence of required source facts
- units normalized to millimeters and grams
- approved master image hash recorded
- front image present for publishable products
- image QA status is not `rejected`
- content status is `approved`
- price is numeric and within allowed precision
- BigCommerce category assignment exists for publish
- Amazon payload fields are present for Amazon syndication

### 9.7 Batch execution order

Recommended order:

1. source-import
2. image-ingest
3. content-generate
4. bc-publish
5. amazon-syndicate
6. price-update
7. inventory-sync

Reason:

- source facts must exist before content generation
- approved master images must exist before publication
- storefront publication should happen before Amazon sync if the SKU is launch-critical
- price and inventory changes should be applied after the record exists in the target channels

### 9.8 Retry and backoff

Batch writers must:

- respect API rate limits
- back off on 429 responses
- avoid parallel bulk inventory writes on the same store
- serialize price-list bulk upserts
- record retry attempts in the batch log

## 10. Data Validation Rules

### Required for record creation

- `merchant_sku` or a rule that can generate it
- `title_raw`
- `saint_or_subject`
- at least one of `material` or `technique`
- `size_w_mm`, `size_h_mm`
- source reference or intake provenance

### Required for storefront publish

- approved English title
- approved English description
- at least one publishable image
- assigned storefront category
- valid `site_price`
- `bc_status = published`

### Required for Amazon readiness

- approved Amazon title and bullets
- Amazon-safe main image
- browse node mapping
- item type keyword
- compliance review complete

### Hard limits and conventions

- custom fields are short and shopper-visible
- metafields are operational and hidden by default
- price must be stored and published with decimal precision compatible with BigCommerce
- variant SKUs must be explicit when the item is sellable as multiple versions
- do not use modifiers for inventory-bearing choices

## 11. Suggested PostgreSQL Tables

This spec does not prescribe exact migration code, but the following table set is the minimum practical foundation.

- `icon_master_records`
- `icon_source_facts`
- `icon_media_assets`
- `icon_content_versions`
- `icon_pricing_rules`
- `icon_pricing_snapshots`
- `icon_inventory_snapshots`
- `icon_publication_state`
- `icon_sync_jobs`
- `icon_sync_events`
- `batch_runs`
- `batch_items`
- `alert_events`

## 12. Open Decisions

Only unresolved owner/platform decisions belong here.

1. Amazon syndication path on BigCommerce Standard: native connector, feed bridge, or pilot manual bridge.
2. Variant family policy for the launch catalog: one SKU per size or one family with size variants when the design is identical.
3. Whether price lists will be used at launch for storefront promotions or deferred until a later phase.

## 13. Official References

All links below are official BigCommerce docs and should be treated as primary sources for platform behavior.

- [Catalog Overview](https://developer.bigcommerce.com/docs/store-operations/catalog)
- [Products](https://developer.bigcommerce.com/docs/rest-catalog/products)
- [GraphQL Storefront Reference](https://developer.bigcommerce.com/graphql-storefront/reference)
- [Product Variants](https://developer.bigcommerce.com/docs/rest-catalog/product-variants)
- [Product Variant Options](https://developer.bigcommerce.com/docs/rest-catalog/product-variant-options)
- [Product Modifiers](https://developer.bigcommerce.com/docs/rest-catalog/product-modifiers)
- [Category Trees](https://developer.bigcommerce.com/docs/rest-catalog/category-trees)
- [Category Assignments](https://developer.bigcommerce.com/docs/rest-catalog/products/category-assignments)
- [Channel Assignments](https://developer.bigcommerce.com/docs/rest-catalog/products/channel-assignments)
- [Product Metafields](https://developer.bigcommerce.com/docs/rest-catalog/products/metafields)
- [Product Custom Fields](https://developer.bigcommerce.com/docs/rest-catalog/products/custom-fields)
- [Product Images](https://developer.bigcommerce.com/docs/rest-catalog/products/images)
- [Inventory Locations](https://developer.bigcommerce.com/docs/store-operations/catalog/inventory-locations)
- [API Rate Limits](https://developer.bigcommerce.com/docs/start/best-practices/api-rate-limits)
- [Webhook Events](https://developer.bigcommerce.com/docs/integrations/webhooks/events)
- [Prepare Your Migration Data](https://developer.bigcommerce.com/docs/start/migration/prepare-data)
