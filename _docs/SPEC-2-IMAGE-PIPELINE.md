# SPEC-2-IMAGE-PIPELINE.md

> Version: 2.0  
> Status: Draft - implementation-facing  
> Language: US English  
> Parent: `MASTER-ARCHITECTURE-v4.md`  
> Scope: human handoff, approved-master storage, preprocessing, derivative generation, QA, and publication handoff

## 1. Purpose

This spec defines the image pipeline for Orthodox icon products from raw handoff to published storefront and marketplace assets.

The pipeline is designed for a real multi-person operating model:

- authors or suppliers provide source files and minimal facts
- the project owner receives and triages the material
- an image processor normalizes or retouches the images
- the store operator publishes the approved assets into commerce channels

The core design rule is simple:

> keep one approved master per asset, derive everything else from that master, and do not create a forest of unmanaged intermediates.

This spec is intentionally implementation-facing. It does not decide the final Amazon connector path, but it does define the image requirements that any connector path must satisfy.

## 2. Design Goals

- Preserve the factual appearance of the icon.
- Support mixed input quality, from already-clean files to heavily flawed source images.
- Keep Google Drive as a human handoff inbox, not as production storage.
- Keep one canonical approved master per SKU or SKU-variant image set.
- Generate derivative image sets deterministically from the approved master.
- Make the pipeline batch-safe, resumable, and auditable.
- Avoid aggressive AI enhancement that can distort sacred imagery.
- Produce image assets that work for both BigCommerce and Amazon.
- Keep the storage model simple enough that outsourced image editors can participate without learning the commerce stack.

## 3. Confirmed Inputs From Master Architecture

The following assumptions are inherited from `MASTER-ARCHITECTURE-v4.md` and should be treated as current project canon:

- PostgreSQL is the canonical icon registry.
- The Icon Master Record is the primary source object for media references.
- BigCommerce is a publication layer, not the source of truth.
- Google Drive is the inbound human exchange layer.
- Approved masters and derivatives live in S3-compatible object storage.
- Cloudinary is not part of the chosen stack.
- Sharp/libvips is the default deterministic processing engine.
- Remove.bg or a local AI background-removal step may be used selectively.
- BigCommerce must ingest approved image assets through its API and then serve its own copy.
- Amazon image assets must obey Amazon image rules, even if the final connector path is implemented later.

## 4. Research Findings Applied Here

### 4.1 S3 and S3-compatible object storage

Amazon S3 stores data as objects in buckets. Object keys are the stable paths used to address those objects, and prefixes act like folders. S3 supports large uploads, multipart uploads, and presigned URLs for temporary access.

Cloudflare R2 is S3-compatible, so the project can use S3 tools and libraries against it by changing the endpoint. R2 also documents presigned URLs for temporary upload/download access and explicitly states there are no egress fees.

Practical implication:

- use S3-compatible object storage as the approved-master archive
- keep the storage API stable and provider-agnostic
- rely on object keys plus database metadata, not on mutable folder names in Google Drive

### 4.2 Sharp and libvips

Sharp is a Node.js image-processing library backed by libvips. Its official docs cover:

- `autoOrient()` for EXIF correction
- `resize()` with fit modes and background handling
- `trim()` for border removal
- `flatten()` for alpha handling
- `normalize()` / `normalise()` for histogram-based contrast stretching
- `modulate()` for controlled brightness/saturation/hue adjustments
- `sharpen()`, `composite()`, and format output control

Sharp’s own documentation states that it is fast, memory-efficient, and suitable for converting large images into smaller web-friendly derivatives.

libvips itself is a low-memory, demand-driven image library with bindings including Python via `pyvips`. ImageMagick remains a valid fallback if a future operator already depends on it, but it should not be the default path for this project.

Practical implication:

- use Sharp/libvips as the default processing worker if the runtime is Node.js
- use pyvips only if the pipeline is intentionally Python-first
- use ImageMagick only as a fallback or legacy bridge, not as the canonical worker

### 4.3 BigCommerce image ingestion

BigCommerce’s Catalog API supports product image creation with `POST /v3/catalog/products/{product_id}/images`.

The API accepts:

- `image_url`
- `image_file` with multipart form-data

BigCommerce documentation also states:

- a product can have only one thumbnail image
- if only one image exists, it becomes both the thumbnail and the main product image
- images can also be added to variants
- product images are visible on storefront channels and locales by default in the multi-storefront model

Practical implication:

- BigCommerce should ingest approved image URLs or uploaded files after approval
- the pipeline should not depend on BigCommerce as the only archive of the approved master
- BigCommerce is the publication layer for the storefront, not the production workspace

### 4.4 Amazon image guidance

Amazon’s official seller guidance and Amazon-authored product-photo guidance indicate that:

- every product must have at least one image
- Amazon recommends at least six images
- images should be between 500 and 10,000 pixels on the longest side
- JPEG, TIFF, PNG, and non-animated GIF are acceptable according to Amazon’s product-photo guidance
- product photos should be accurate, high quality, and well lit
- white-background photography is recommended for product shots
- Amazon prefers images larger than 1,000 pixels on each side to support zoom
- JPEG is Amazon’s preferred format in the seller marketing guidance

Practical implication:

- the internal approved master should be large enough to support Amazon and storefront derivatives
- the Amazon main image must be the strictest version of the asset set
- image QA must be aware of Amazon image suppression risk

### 4.5 Background removal workflows

remove.bg’s official API documentation describes an HTTP API for automatic background removal, with the first 50 API calls per month available on the house.

remove.bg also offers:

- API key-based access
- CLI usage
- commercial tiers for higher volume

Practical implication:

- background removal is best treated as an optional branch, not as the default for all assets
- if the source photo is already clean and white-background ready, skip the extra cost and risk
- if the photo is noisy, cropped poorly, or has background contamination, background removal can be used before the approved master is finalized

## 5. Recommended Stack and Operating Default

### 5.1 Recommended defaults

| Layer | Default recommendation | Notes |
|---|---|---|
| Human handoff inbox | Google Drive | Inbound only, not production storage |
| Approved-master storage | S3-compatible object storage | Default implementation may be Cloudflare R2 or AWS S3 |
| Processing worker | Sharp/libvips | Deterministic normalization and derivative generation |
| Optional background removal | remove.bg or local AI | Use only for problematic batches |
| QA record | PostgreSQL + review pack artifacts | Batch-level traceability |
| Publication layer | BigCommerce API and Amazon listing workflow | Uses approved assets only |

### 5.2 Why this shape

This architecture keeps the system simple:

- Drive handles human exchange.
- Object storage handles durable media.
- PostgreSQL handles state, hashes, and references.
- the worker handles deterministic image operations.
- BigCommerce and Amazon consume the approved outputs.

That separation makes batch updates, outsourcing, and regeneration much safer.

## 6. Roles And Responsibilities

| Role | Responsibility | Does not do |
|---|---|---|
| Author / supplier | Sends source images and minimal factual data | Does not publish to channels |
| Project owner | Accepts batches, resolves ambiguity, approves major exceptions | Does not manually retouch every image |
| Image processor | Normalizes, retouches, and produces approved masters | Does not manage store publishing |
| Store operator | Publishes assets, maintains product records, and syncs channels | Does not invent image facts |
| Outsourced retoucher | Handles difficult images only when needed | Does not change the commerce catalog |

The workflow must keep the image processor and store operator separate. The person fixing pixels should not also be the person deciding catalog publication state.

## 7. Image State Model

The pipeline must use a small number of explicit states.

| State | Meaning | Typical owner action |
|---|---|---|
| `incoming` | File received, not yet triaged | Intake |
| `needs_review` | Requires human inspection before any automation | Owner or processor |
| `retouch_required` | Needs manual correction, geometry repair, or better exposure | Processor or external retoucher |
| `processing` | Auto-normalization is running | Worker |
| `approved_master` | Final approved source asset is locked | Owner approves |
| `derivatives_ready` | Required derivative versions exist | Worker |
| `published` | Asset is linked in BigCommerce and/or Amazon workflow | Operator |
| `rejected` | Asset is not fit for publication and needs reshoot or replacement | Owner |

The state model should be stored in PostgreSQL and mirrored in batch review artifacts.

## 8. Storage Model

### 8.1 Google Drive as handoff inbox

Google Drive is used for:

- inbound folders from suppliers or photographers
- handoff to external retouchers
- return of corrected files
- temporary review sharing

Google Drive is not used as the canonical archive because folder names are mutable, paths are brittle, and the system needs a stable audit trail.

### 8.2 Canonical approved-master storage

Use an S3-compatible bucket for approved masters and derivatives. The storage layer should be provider-agnostic even if one provider is the default.

Recommended key layout:

```text
incoming/{source_system}/{batch_id}/{merchant_sku}/{source_filename}
approved/{merchant_sku}/master.jpg
derived/{merchant_sku}/amazon-main.jpg
derived/{merchant_sku}/amazon-gallery-01.jpg
derived/{merchant_sku}/site-pdp.webp
derived/{merchant_sku}/site-card.webp
derived/{merchant_sku}/site-zoom.jpg
qa/{batch_id}/{merchant_sku}/contact-sheet.jpg
archive/{batch_id}/manifest.json
```

Rules:

- keep `incoming` immutable
- create exactly one approved master per approved image set
- regenerate derivatives from the approved master, not from derivatives
- keep batch manifests and checksums alongside the media

### 8.3 What the master record must store

The Icon Master Record should store media references as structured fields rather than as free-text notes.

Recommended fields:

```text
source_system
source_folder_id
source_file_id
source_filename
received_at
incoming_source_uri
approved_master_uri
approved_master_hash
approved_master_resolution
derivative_uris
image_qa_status
image_qa_notes
last_processed_at
last_image_batch_id
```

The file path is not the source of truth. The database is.

## 9. End-to-End Workflow

### 9.1 Intake and triage

1. The owner receives a batch in Google Drive or via a shared upload link.
2. The intake worker reads the folder/file IDs and creates a batch record.
3. The worker checks:
   - file type
   - file size
   - pixel dimensions
   - duplicate source files
   - SKU mapping
4. The batch is split into:
   - clean enough to normalize automatically
   - needs retouch
   - cannot be used and must be reshot

### 9.2 Normalization

For clean inputs, the worker performs conservative technical normalization:

- auto-orient using EXIF
- standardize color space to sRGB
- correct obvious exposure imbalance only if it does not alter the icon’s appearance
- normalize contrast carefully
- trim stray borders only when safe
- pad or contain onto a white background
- preserve aspect ratio unless a known crop decision exists

The normalization step must not:

- repaint the icon
- invent missing details
- change sacred icon content
- introduce glossy enhancement that makes the image look artificial

### 9.3 Retouch path

If the source image is not ready for direct normalization, it enters `retouch_required`.

Examples:

- perspective distortion
- severe glare
- fingers, shadows, or photographer reflection
- uneven background around frame edges
- cropping issues
- misaligned frame geometry

The processor may send the file to an external retoucher and receive the corrected file back through Google Drive.

### 9.4 Approved master creation

Once an image is visually approved, the worker writes a single approved master to object storage.

Recommended rule:

- the approved master must be the largest useful faithful version of the image
- do not create a small approved master and hope to upscale later
- do not preserve multiple “almost final” masters

The approved master should be the source used for every derivative, every later re-export, and every publication path.

### 9.5 Derivative generation

After the approved master is locked, the worker creates a reproducible derivative set.

Initial default profiles:

| Profile | Default size | Format | Notes |
|---|---:|---|---|
| `amazon-main` | 2000x2000 | JPG | White background, no overlays |
| `amazon-gallery` | 1500x1500 | JPG | Secondary Amazon image |
| `site-pdp` | 800x800 | WebP | Product detail page |
| `site-card` | 400x400 | WebP | Category tile / listing card |
| `site-zoom` | 1600x1600 | JPG | Larger storefront detail view |
| `thumbnail` | 150x150 | WebP | Internal admin and compact UI |

These sizes are defaults, not eternal truth. They should be validated on a pilot batch and adjusted only if the store or marketplace proves a better standard.

### 9.6 QA

QA should happen after derivative generation and before publication.

QA checks:

- image is sharp and not blurred
- icon is fully inside frame
- no photographer reflection
- no accidental props or stray hands
- background is clean and neutral
- frame, gold, silver, and color tones look accurate
- derivative images match the approved master
- Amazon main image is compliant with Amazon’s white-background requirement
- BigCommerce variants and thumbnails are correct

Suggested QA policy:

- 100 percent inspection for any image marked `retouch_required`
- 100 percent inspection for Amazon main images during initial launch batches
- sample-based QA for low-risk repeat batches after the pipeline stabilizes

### 9.7 Publication

#### BigCommerce

BigCommerce should receive approved image URLs or uploaded files through the Catalog API.

The publication flow should:

1. create or update the product record
2. attach the approved storefront images
3. set thumbnail and sort order
4. record the last successful sync hash in PostgreSQL

The storefront should not depend on Google Drive or the raw inbox.

#### Amazon

Amazon image publication must use the same approved asset set, but Amazon has its own rules and listing workflow.

The image pipeline itself does not need to know the final Amazon transport mechanism. It only needs to produce:

- a compliant main image
- clean supporting images
- stable URLs or files for the later Amazon syndication step

If the final Amazon connector path is unavailable, the same approved assets can still be used in a manual bridge workflow.

## 10. Processing Rules

### 10.1 Deterministic first, AI second

The default path should be deterministic image processing:

- resize
- padding
- orientation
- color normalization
- output format conversion

AI should be used only when the deterministic path is not enough.

### 10.2 When to use background removal

Use background removal when:

- the background is not cleanly white
- the edges are contaminated by shadow or clutter
- the product cutout must be isolated more cleanly
- the source image is good enough to justify the cost of enhancement

Do not use background removal when:

- the file already meets the required standard
- the image contains delicate frame edges that may be damaged by cutout errors
- the operation might alter the icon’s color fidelity

### 10.3 Allowed image adjustments

Safe adjustments:

- exposure balancing
- mild contrast normalization
- orientation correction
- sharpening
- format conversion
- resizing
- padding and containment

Restricted adjustments:

- heavy relighting
- generative fill
- synthetic background invention
- content reconstruction
- beautification that changes the product’s real appearance

For Orthodox icons, fidelity matters more than “better than real life” enhancement.

## 11. Batch Behavior And Idempotency

Image processing must be batch-safe.

Recommended batch rules:

- every image batch has a unique `batch_id`
- every input file gets a `source_hash`
- every approved output gets a `target_hash`
- if the source hash did not change, skip the item
- if the approved master exists and is unchanged, regenerate only missing derivatives
- if a derivative preset changes, regenerate just the derivatives, not the approved master

The batch engine should treat image processing as resumable work, not as one giant fragile run.

## 12. Failure Modes And Exceptions

### 12.1 Common failures

- image too small
- blurry or out of focus
- wrong orientation
- background contamination
- heavy glare or reflections
- clipped frame or cropped icon
- too much compression
- wrong SKU mapping

### 12.2 Exception handling

| Failure | Action |
|---|---|
| Minor exposure or background issue | `processing` then re-qa |
| Geometry problem or strong glare | `retouch_required` |
| Unusable source file | `rejected` |
| Amazon main image non-compliant | regenerate main image and re-qa |
| BigCommerce upload failure | retry with same approved asset |

### 12.3 No silent degradation

Do not silently publish an image that failed QA just because the batch needs to move.

If the image is not good enough, it should remain out of publication until the issue is fixed.

## 13. Implementation Notes

### 13.1 Worker choice

Default worker recommendation:

- Node.js worker with Sharp/libvips

Alternative:

- Python worker with pyvips if the wider automation stack becomes Python-first

Legacy fallback:

- ImageMagick if an existing environment already depends on it

### 13.2 Object storage choice

The project should keep the storage interface S3-compatible.

Default implementation recommendation:

- Cloudflare R2 for low-cost S3-compatible storage with no egress fees

Alternative:

- AWS S3 if the project prefers AWS-native tooling

The spec does not require the application code to hard-code a single storage vendor.

### 13.3 Review artifacts

Batch review should generate a human-readable package:

- CSV summary
- JSONL detail file
- contact sheet or simple QA gallery

This package lets the owner approve or reject quickly without opening every source file in the storage bucket.

## 14. Open Decisions

| Decision | Current recommendation | Why it is still open |
|---|---|---|
| Default object storage vendor | Cloudflare R2, with AWS S3 as a valid alternative | Both are viable; the final choice depends on owner preference and ops comfort |
| Background removal default | Optional, not mandatory | Clean images should bypass it; difficult images may need it |
| Approved-master size policy | Largest faithful master, no artificial upscaling by default | Final threshold should be validated on the first real pilot batch |
| AI enhancement scope | Conservative only | Need owner approval for what counts as acceptable image correction |
| BigCommerce ingest method | API upload of approved assets, with URL-based handoff where useful | Exact operational preference should be validated against the store setup |
| Amazon publish transport | Connector or manual bridge using the same approved assets | Final connector path is a separate research item |
| QA sampling rate after stabilization | 100 percent for Amazon main images, sampled QA for low-risk repeat batches | Needs pilot data before it becomes a fixed rule |
| Final derivative presets | Initial defaults in this spec | Should be adjusted only after launch data proves a better set |

## 15. Official References

### Storage

- Amazon S3 User Guide - Overview: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html>
- Amazon S3 User Guide - Uploading objects: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/upload-objects.html>
- Amazon S3 User Guide - Object keys: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-keys.html>
- Cloudflare R2 - How R2 works: <https://developers.cloudflare.com/r2/how-r2-works/>
- Cloudflare R2 - Pricing: <https://developers.cloudflare.com/r2/pricing/>
- Cloudflare R2 - S3 compatibility: <https://developers.cloudflare.com/r2/get-started/s3/>
- Cloudflare R2 - Presigned URLs: <https://developers.cloudflare.com/r2/api/s3/presigned-urls/>
- Cloudflare R2 - Upload objects: <https://developers.cloudflare.com/r2/objects/upload-objects/>

### Image processing

- Sharp - Home: <https://sharp.pixelplumbing.com/>
- Sharp - Resizing images: <https://sharp.pixelplumbing.com/api-resize/>
- Sharp - Image operations: <https://sharp.pixelplumbing.com/api-operation/>
- Sharp - Colour manipulation: <https://sharp.pixelplumbing.com/api-colour>
- libvips - GitHub repository: <https://github.com/libvips/libvips>
- pyvips - GitHub repository: <https://github.com/libvips/pyvips>
- ImageMagick - Command-line tools: <https://imagemagick.org/script/command-line-tools.php>
- ImageMagick - Convert: <https://imagemagick.org/script/convert.php>

### BigCommerce

- BigCommerce Dev Center - Catalog overview: <https://developer.bigcommerce.com/docs/store-operations/catalog>
- BigCommerce Dev Center - Current product images endpoint: <https://developer.bigcommerce.com/docs/rest-catalog/products/images>
- BigCommerce Dev Center - Create product with images and V3 image upload examples: <https://developer.bigcommerce.com/archive/store-operations/v2-products/v2-versus-v3>
- BigCommerce Dev Center - BigCommerce CDN for images: <https://developer.bigcommerce.com/docs/storefront/catalyst/development/optimization>

### Amazon

- Sell on Amazon - How to take product photos in 2025: <https://sell.amazon.com/es/blog/product-photos>
- Amazon Brand Registry / product image requirements are linked from the official Amazon product-photo guidance and Seller Central help pages.

### Background removal

- remove.bg API docs: <https://www.remove.bg/a/api-docs>
- remove.bg pricing: <https://www.remove.bg/pricing>

## 16. Next Spec Dependencies

This spec should be read together with:

- `SPEC-1-CATALOG-DATA-MODEL.md` for the master record and field map
- `SPEC-3-CONTENT-GENERATION.md` for the text generation contract
- `MASTER-ARCHITECTURE-v4.md` for the current canonical project model

The image pipeline is intentionally independent from the final content pipeline and from the final Amazon transport implementation.
