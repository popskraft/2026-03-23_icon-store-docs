# SPEC-3-CONTENT-GENERATION.md

> Version: 1.0
> Status: Draft - parallel workstream
> Language: US English
> Parent: `MASTER-ARCHITECTURE-v4.md`

## 1. Purpose

This document defines the content-generation system for icon product listings across the storefront and Amazon.

The system takes a minimal factual record for each icon and turns it into channel-specific, reviewable, versioned content:

- BigCommerce storefront content
- Amazon listing content
- SEO metadata
- review packs for human approval

This spec focuses on the content lifecycle, not the image pipeline or the final Amazon connector implementation.

## 2. Scope

### In scope

- minimum source input set
- Orthodox terminology and glossary discipline
- deterministic prompt templates and variables
- structured output contracts
- review and approval workflow
- versioning and rollback rules
- batch generation and regeneration
- synchronization rules for BigCommerce and Amazon payloads

### Out of scope

- exact image processing implementation
- final Amazon browse-node validation
- final pricing logic
- storefront UX and theme design

## 3. Confirmed Inputs From Master Architecture

- PostgreSQL is the canonical registry.
- `Icon Master Record` is the source object for content generation.
- BigCommerce is a publication channel, not the source of truth.
- Amazon content and site content are generated independently from the same source facts.
- Human review is mandatory before Amazon publication.
- Final published content defaults to US English.
- `site_ru` is not a launch output.
- Content versions must be stored before publication.

## 4. Research Findings Applied To This Spec

### 4.1 Anthropic Messages and Batches

Anthropic's Messages API is the primary synchronous generation path.

The Message Batches API is the bulk path when the workload is too large or too repetitive for synchronous generation. Official Anthropic documentation states that:

- batches can process large sets of Messages requests asynchronously
- batches support all Messages API features, including tool use and beta features
- a batch can contain up to 100,000 requests or 256 MB total size, whichever comes first
- batch processing can take up to 24 hours
- results are returned as JSONL and are matched back using `custom_id`

### 4.2 Anthropic consistency guidance

Anthropic recommends improving consistency by:

- specifying the desired output format precisely, such as JSON or XML
- pre-filling the assistant response when a specific structure is needed
- using examples of the desired output
- using prompt templates and variables for repeated workloads
- breaking complex tasks into smaller chained prompts

### 4.3 Anthropic tool use guidance

When using tools:

- tools are defined in the top-level `tools` parameter
- the `tool_choice` parameter can be `auto`, `any`, `tool`, or `none`
- the assistant's `tool_use` and the user's `tool_result` must remain in the correct message structure
- no messages should be inserted between an assistant tool-use block and the corresponding tool-result block

These rules matter because this project will likely use tool-like validation, retrieval, or normalization helpers for content generation and review.

### 4.4 BigCommerce constraints relevant to content

BigCommerce official docs establish that:

- product creation should use the V3 Catalog API where possible
- product variants have their own SKU and inventory behavior
- products can be created without categories and categorized later
- product custom fields are limited to 250 characters per value
- product images can be attached using image URLs through the API
- inventory updates are asynchronous and not channel aware
- Standard and Plus plans are rate-limited at 150 requests per 30 seconds

These constraints shape how much content should live in standard fields, custom fields, metafields, and the PostgreSQL registry.

### 4.5 Amazon listing constraints relevant to content

Amazon official seller guidance and Amazon-authored announcements indicate that:

- product titles for most categories may not exceed 200 characters, including spaces
- some special characters are disallowed in titles
- repeated words are constrained
- product bullet points should be concise and policy-safe
- search terms should use synonyms, abbreviations, alternate spellings, and should avoid repetition
- backend search terms are limited to 250 bytes in the seller guidance currently surfaced in Amazon help/community material
- the main image must be a professional photo of the actual product on a pure white background
- the main image should be at least 1000 pixels on one side for zoom
- the main image should not contain logos, watermarks, props, or misleading elements
- product detail page content must be accurate and policy compliant

For this project, content generation must produce policy-aware payloads and must not depend on later manual cleanup.

## 5. Core Principles

1. The source facts are canonical truth.
2. Generated text is derivative data.
3. Every generated output must be structured and machine-validatable.
4. Human review is mandatory before Amazon publication.
5. Manual edits must survive later regeneration.
6. BigCommerce field limits and Amazon field limits must be respected during generation, not fixed after publication.
7. The system should create one reusable prompt template per content family, not one-off prompts per SKU.
8. The same source facts can produce different outputs for site and Amazon, but the outputs must remain semantically aligned.

## 6. Minimum Input Set

The author or operator must provide a minimal factual record for each icon before content generation begins.

### Required minimum fields

- `merchant_sku`
- `title_raw`
- `saint_or_subject`
- `material`
- `technique`
- `size_w_mm`
- `size_h_mm`
- `size_d_mm` or an explicit note that depth is not applicable
- `manufacturer`

### Required when known

- `feast`
- `purpose`
- `style`
- `iconographic_type`
- `construction_type`
- `origin_school`
- `weight_g`

### Optional enrichment fields

- short heritage note
- blessing or provenance note
- gift use note
- room or placement note
- variant notes

### Input rule

The minimum input set must be enough to generate a useful first-pass listing without requiring manual copywriting. If the generated content would be too generic or risky, the item must remain in `needs_review` until more facts are supplied.

## 7. Canonical Glossary And Terminology Discipline

The content system must use a controlled glossary so generated listings are consistent across the catalog.

### Glossary goals

- normalize Orthodox terminology in US English
- avoid contradictory names for the same subject
- preserve theological correctness where the project has already chosen a canonical term
- keep SEO phrasing stable across the catalog

### Examples of glossary-controlled terms

- `Theotokos` vs `Mother of God`
- `Virgin Mary` only where SEO or audience needs justify it
- `Jesus Christ` vs `Christ`
- `Holy Trinity`
- `Archangels`
- `Byzantine icon`
- `Orthodox icon`
- `hand-painted`
- `lithograph`
- `wooden board`
- `wooden icon`

### Glossary rules

- Each canonical subject must have one preferred display name.
- Synonyms may be kept for search terms and internal enrichment, but not mixed into the published title.
- The glossary must explicitly list forbidden or discouraged wording where it risks theological inaccuracy or marketing overstatement.
- The glossary must be versioned like code.

## 8. Output Contracts

### 8.1 Content object

Each generated SKU must produce one machine-readable content object.

```json
{
  "sku": "IC-000001",
  "source_hash": "sha256:...",
  "content_version": 1,
  "generated_at": "2026-03-23T00:00:00Z",
  "review_status": "needs_review",
  "amazon": {
    "title": "",
    "bullets": ["", "", "", "", ""],
    "description_html": "",
    "backend_search_terms": "",
    "browse_node": "",
    "item_type_keyword": "",
    "target_marketplace": "US"
  },
  "site": {
    "title": "",
    "description_html": "",
    "meta_title": "",
    "meta_description": ""
  },
  "seo": {
    "primary_keyword": "",
    "secondary_keywords": []
  },
  "compliance_flags": [],
  "review_notes": ""
}
```

### 8.2 BigCommerce mapping

BigCommerce content should generally map to:

- product name
- product description
- meta title and meta description
- shopper-visible custom fields when the field is intentionally shopper-facing
- metafields for operational metadata and channel-specific payload fragments

BigCommerce custom field values are limited to 250 characters. Content generation must respect that limit if the content will be written to a custom field.

### 8.3 Amazon mapping

Amazon payload must support:

- title
- bullet points
- description
- backend search terms
- browse node or category mapping
- item type keyword
- marketplace target

Amazon titles and backend terms must be generated with platform limits in mind. The generator must fail validation if a field exceeds the known limit or uses clearly disallowed formatting.

## 9. Generation Modes

### 9.1 Synchronous mode

Use Anthropic Messages API for:

- single-SKU generation
- targeted regeneration
- human-in-the-loop drafting
- test runs for new prompts or glossary changes

### 9.2 Batch mode

Use Anthropic Message Batches API for:

- large SKU batches
- reprocessing a catalog segment
- overnight content refreshes
- repeated format-stable jobs

The batch process must include `custom_id` per SKU so results can be reconciled even when result order differs from request order.

### 9.3 Tool-assisted mode

If retrieval, validation, or normalization tools are used in the generation flow, they must be declared explicitly in Anthropic's `tools` parameter and controlled with `tool_choice` as needed.

Use tool-assisted generation only where it reduces error rate or improves validation. Do not add tools just because they are available.

## 10. Review Workflow

### 10.1 Status model

Each content object must pass through:

- `not_started`
- `generating`
- `needs_review`
- `approved`
- `published`
- `rejected`

### 10.2 Review pack

Every batch must produce:

- a JSONL file for machine parsing
- a CSV summary for human review
- a short exception report for failed or ambiguous items

### 10.3 Review rules

- If mandatory fields are missing, generation may be attempted but the record stays `needs_review`.
- If Amazon limits are violated, the item must be rejected or regenerated.
- If the generated content invents facts not present in the source record, the item must be rejected.
- If the content is semantically correct but poorly phrased, the operator may approve with minor edits.

### 10.4 Approval traceability

Every approval must record:

- approver
- approval timestamp
- review notes
- content version
- source hash

## 11. Versioning Model

### 11.1 Content versioning

Each regeneration must increment `content_version` only when the generated payload changes in a meaningful way.

### 11.2 Snapshot retention

The system must retain the last approved content snapshot for rollback and comparison.

### 11.3 Sync state

Publication state must be distinct from content generation state.

- content can be approved but not yet published
- content can be published to BigCommerce before Amazon
- content can be republished without regenerating if only the channel push failed

### 11.4 Rollback rule

Rollback must restore the last approved content snapshot, not the last generated draft.

## 12. Regeneration Rules

### Regenerate automatically when:

- source facts change
- price rules change and channel text contains price-sensitive wording
- a glossary term changes
- Amazon policy updates require title or bullet cleanup
- the approved master image changes and the content includes image-dependent wording

### Do not regenerate automatically when:

- only a non-semantic formatting issue changed
- a human made a stylistic edit that should remain override-locked
- the change affects only one channel and not the other

### Regeneration discipline

- Regeneration should preserve a manual override unless that override is invalidated by policy or source truth changes.
- Automatic regeneration should never silently overwrite an approved human edit without recording why.

## 13. Synchronization Rules

### PostgreSQL to BigCommerce

Write approved content into:

- standard product fields where appropriate
- custom fields only if the value is shopper-facing and under the 250-character limit
- metafields for operational metadata and channel payload fragments

### PostgreSQL to Amazon

Write approved Amazon content into the Amazon payload layer only after the human review gate is passed.

### Cross-field consistency

These values must remain aligned across the master record, BigCommerce, and the Amazon payload:

- canonical title
- subject naming
- approved marketplace description intent
- search terms concept
- compliance status

## 14. Validation Rules

The content generator must reject or flag outputs that:

- exceed Amazon title or backend search term limits
- use obviously prohibited promotional language
- repeat words excessively in title or search terms
- mix channel-specific facts incorrectly
- reference unavailable claims about materials, provenance, or performance
- conflict with the source record
- include HTML or markup where the target field does not support it

## 15. Implementation Guidance

### Prompt templates

Use prompt templates and variables so the content system stays consistent across large batches.

### Output formatting

Use a strict schema and prefill the desired structure where needed so Claude remains consistent.

### Prompt chaining

Separate the job into smaller prompts where helpful:

- prompt 1: normalize source facts
- prompt 2: draft site content
- prompt 3: draft Amazon content
- prompt 4: validate and score

### Tool use

If a tool is used to validate length, glossary, or policy rules, keep the tool interface explicit and deterministic.

## 16. Open Decisions

- What is the final branded tone of voice for the US market?
- Do we allow a controlled `Virgin Mary` synonym in SEO fields, or keep `Theotokos` as the default in all published copy?
- Will BigCommerce custom fields be used for shopper-visible attributes or kept minimal?
- Should Amazon browse-node mapping be generated here or only in the Amazon operations spec?
- Do we want automatic regeneration when glossary terms change, or operator-approved regeneration only?
- Should one content object store both site and Amazon drafts, or should those be separate child records?

## 17. Official References

- Anthropic Message Batches API: [https://docs.anthropic.com/en/api/creating-message-batches](https://docs.anthropic.com/en/api/creating-message-batches)
- Anthropic prompt templates and variables: [https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-templates-and-variables](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-templates-and-variables)
- Anthropic prompt engineering overview: [https://docs.anthropic.com/en/docs/prompt-engineering](https://docs.anthropic.com/en/docs/prompt-engineering)
- Anthropic tool use implementation: [https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use)
- Anthropic output consistency guidance: [https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency](https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency)
- BigCommerce products API: [https://developer.bigcommerce.com/docs/rest-catalog/products](https://developer.bigcommerce.com/docs/rest-catalog/products)
- BigCommerce product variants API: [https://developer.bigcommerce.com/docs/rest-catalog/product-variants](https://developer.bigcommerce.com/docs/rest-catalog/product-variants)
- BigCommerce product metafields: [https://developer.bigcommerce.com/docs/rest-catalog/products/metafields](https://developer.bigcommerce.com/docs/rest-catalog/products/metafields)
- BigCommerce product custom fields: [https://developer.bigcommerce.com/docs/store-operations/catalog/msf-international-enhancements/product-custom-fields](https://developer.bigcommerce.com/docs/store-operations/catalog/msf-international-enhancements/product-custom-fields)
- BigCommerce catalog overview and product images: [https://developer.bigcommerce.com/docs/store-operations/catalog](https://developer.bigcommerce.com/docs/store-operations/catalog)
- BigCommerce API rate limits: [https://developer.bigcommerce.com/docs/start/best-practices/api-rate-limits](https://developer.bigcommerce.com/docs/start/best-practices/api-rate-limits)
- BigCommerce inventory overview: [https://developer.bigcommerce.com/docs/store-operations/catalog/inventory-adjustments](https://developer.bigcommerce.com/docs/store-operations/catalog/inventory-adjustments)
- Amazon title requirements announcement: [https://sellercentral.amazon.com/seller-forums/discussions/t/b2b15728-0d43-453e-974f-59eb63f73059](https://sellercentral.amazon.com/seller-forums/discussions/t/b2b15728-0d43-453e-974f-59eb63f73059)
- Amazon updated bullet requirements announcement: [https://sellercentral.amazon.com/seller-forums/discussions/t/65f8e647-977d-49ac-9036-2049b96720b2](https://sellercentral.amazon.com/seller-forums/discussions/t/65f8e647-977d-49ac-9036-2049b96720b2)
- Amazon search terms guidance: [https://sellercentral.amazon.com/seller-forums/discussions/t/62e0e515a6e991d171d3ccee7be36304](https://sellercentral.amazon.com/seller-forums/discussions/t/62e0e515a6e991d171d3ccee7be36304)
- Amazon image requirements discussion by Amazon staff: [https://sellercentral.amazon.com/seller-forums/discussions/t/7b9fc6009328f1aca4a2ab6ccd59ad78](https://sellercentral.amazon.com/seller-forums/discussions/t/7b9fc6009328f1aca4a2ab6ccd59ad78)
- Amazon product detail page rules reference via Amazon forum posts: [https://sellercentral.amazon.com/seller-forums/discussions/t/2b44a6a45498a9d10e01386314e1f9ad](https://sellercentral.amazon.com/seller-forums/discussions/t/2b44a6a45498a9d10e01386314e1f9ad)

