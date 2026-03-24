# AUDIT-v2: Консолидированный аудит архитектуры Icon Store

Версия: 2.0
Дата: 2026-03-24
Статус: рабочий документ — не канон
Область: консолидация внешних и внутренних аудитов, ролевых проверок, архитектурных замечаний и launch-вопросов owner

---

## 1. Как читать документ

Формат каждого пункта:

- `Проблема` — что именно не так
- `Почему это важно` — какой риск создаёт
- `Вариант решения` — что зафиксировать в каноне или спецификациях
- `Где фиксировать` — в каком документе менять

Приоритеты:

- `P1` — блокирует реализацию, создаёт высокий риск поломки канона, данных или marketplace-операций
- `P2` — не блокирует немедленно, но создаёт реальный operational / business risk
- `P3` — техдолг, который лучше закрыть до масштабирования

---

## 2. Executive Summary

- Канон уже сильный на уровне общей архитектуры, но пока недостаточно стабилен как implementation contract.
- Главная проблема — не отсутствие идей, а противоречия между каноническими документами и недоопределённые runtime-контракты.
- Самые опасные зоны: intake path, publish gates, inventory synchronization, image handoff, ownership of fields, Amazon launch controls.
- Для launch нужен один primary operating mode, а не смешение нескольких альтернатив в одном каноне.
- Amazon path должен быть зафиксирован как `manual approval + automated SP-API submit`; ручной Seller Central — только fallback.
- Inventory должен вестись из единого канонического реестра в PostgreSQL, а не расползаться между BigCommerce и Amazon.
- Нужно явно развести `canonical launch architecture` и `lean MVP fallback`, иначе проект останется в состоянии двух одновременно истинных моделей.

---

## 3. Блок A — Критические противоречия канона

### [P1-A1] Intake path противоречит сам себе

Проблема:

- В одном месте owner intake уже прямой через Supabase.
- В другом месте мастер-база ещё Google Sheet.
- В закрытых решениях одновременно существуют обе модели.

Почему это важно:

- Это создаёт два источника правды для Layer 1.
- Любой импорт, review и downstream sync начинают зависеть от того, какой документ открыли первым.

Вариант решения:

- Зафиксировать: `Supabase Table Editor = primary owner intake`.
- `Google Sheet / CSV / Excel = optional supplier bulk-import only`.
- Правило конфликта: `Supabase wins`.
- Убрать любые формулировки, где Google Sheet выступает owner UI.

Где фиксировать:

- `AGENTS.md`
- `ROADMAP.md`
- `OPEN-QUESTIONS.md`
- `MASTER-ARCHITECTURE-v4.md`

### [P1-A2] Review workflow описан двумя несовместимыми способами

Проблема:

- В одном месте review идёт в Supabase Table Editor.
- В другом — через CSV `APPROVE / REJECT / REGEN`.

Почему это важно:

- Это два разных операционных процесса, а не вариации одного.
- Без primary workflow невозможно правильно описать audit trail, approval source и operator SOP.

Вариант решения:

- Зафиксировать: `CSV-based review = canonical MVP batch review`.
- `Table Editor = exception handling / single-record correction`.
- Добавить правило выбора:
  - `batch review -> CSV`
  - `single SKU / quick fix -> Table Editor`
- Записывать `approval_source`.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-3-CONTENT-GENERATION.md`
- `SPEC-7-OPERATOR-PLAYBOOK.md`

### [P1-A3] Image handoff описан противоречиво

Проблема:

- Owner, operator и processor описывают разные пути движения файлов.
- Присутствует переименование папок как часть идентичности.

Почему это важно:

- Появляется риск потерять соответствие между SKU, фото и master record.
- Хрупкая схема handoff ломается на первом же ручном отклонении или повторной обработке.

Вариант решения:

- Добавить единый `Image Handoff Sequence`:
  - `owner deposits`
  - `operator validates intake`
  - `operator assigns SKU`
  - `processor edits`
  - `operator accepts/rejects`
  - `system ingests approved master`
- Идентичность держать на `icon_id` / immutable file ID.
- SKU и папка — alias, не источник истины.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-2-IMAGE-PIPELINE.md`
- `SPEC-7-OPERATOR-PLAYBOOK.md`

### [P1-A4] n8n hosting одновременно confirmed и open question

Проблема:

- В стеке `n8n CE self-hosted` уже подтверждён.
- В open questions он ещё висит как незакрытое решение.

Почему это важно:

- Это меняет infra boundary, backup model, ops burden и credential handling.

Вариант решения:

- Закрыть решение: `n8n CE self-hosted = confirmed`.
- Удалить вопрос из Near-term.
- Если owner сознательно меняет решение — обновлять стек явно как архитектурное изменение.

Где фиксировать:

- `OPEN-QUESTIONS.md`
- `MASTER-ARCHITECTURE-v4.md`

### [P1-A5] Amazon account/legal entity противоречит SPEC-4

Проблема:

- В одном документе указан конкретный Seller Central account.
- В другом это всё ещё blocking question.

Почему это важно:

- Это не косметика: от этого зависит GTIN exemption, credentials, Brand Registry, payout и compliance.

Вариант решения:

- Либо закрыть решение и перенести в Closed.
- Либо пометить упоминание account/entity как `assumption pending owner confirmation`.

Где фиксировать:

- `SPEC-4-AMAZON-LAUNCH.md`
- `OPEN-QUESTIONS.md`

---

## 4. Блок B — Критические runtime и data contracts

### [P1-B1] Human gate не оформлен как runtime-контракт

Проблема:

- Принцип `human gate` есть.
- Но нет явного publish authorization per channel.

Почему это важно:

- Система может неверно трактовать `content_status = approved` как разрешение на полную публикацию.

Вариант решения:

- Добавить поля:
  - `images_approved`
  - `content_approved`
  - `price_ready`
  - `inventory_ready`
  - `bc_publish_approved`
  - `amazon_publish_approved`
  - `approved_payload_hash`
- Правило:
  - workflow стартует только при выполнении полного предиката канала;
  - изменение контента, цены, фото или stock reset’ит publish-ready.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-1-CATALOG-DATA-MODEL.md`
- `SPEC-5-PIPELINES.md`

### [P1-B2] Нет transaction-safe sync-контракта между PostgreSQL и внешними API

Проблема:

- Запись в PostgreSQL и side effect во внешний API не объединены в одну транзакцию.
- При падении `n8n` возможен partial commit.

Почему это важно:

- Возникают “полуопубликованные” объекты: локально статус изменился, а в BC/Amazon — нет.

Вариант решения:

- Ввести промежуточные sync-state:
  - `pending_sync`
  - `sync_in_progress`
  - `sync_succeeded`
  - `sync_failed_transient`
  - `sync_failed_terminal`
- Публикация должна быть driven из durable job queue / batch item state.
- Любой внешний side effect должен завершаться reconciliation step.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-5-PIPELINES.md`
- `SPEC-1-CATALOG-DATA-MODEL.md`

### [P1-B3] Идемпотентность заявлена, но не обеспечена

Проблема:

- Есть принцип restartable batches.
- Но нет step-level idempotency model.

Почему это важно:

- Повторные запуски могут породить duplicate publish, duplicate updates, race conditions.

Вариант решения:

- Добавить:
  - `workflow_run_id`
  - `step_name`
  - `step_attempt`
  - `idempotency_key`
  - `bc_sync_version`
  - `amazon_sync_version`
  - `published_payload_hash`
- Разделить ошибки на:
  - `FAILED_TRANSIENT`
  - `FAILED_TERMINAL`

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-5-PIPELINES.md`

### [P1-B4] Нет field ownership matrix

Проблема:

- Owner, system и operator работают в одной записи.
- Но не определено, кто владеет какими полями и когда они lock’аются.

Почему это важно:

- Возможны тихие конфликты между owner edits, AI enrichment и operator approval.

Вариант решения:

- Добавить ownership matrix по группам полей.
- Зафиксировать:
  - editable owner fields
  - system-only fields
  - operator override fields
  - lock-after-approval fields
- Правило: изменение owner input после content approve автоматически инвалидирует downstream derived content.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-1-CATALOG-DATA-MODEL.md`

### [P1-B5] Нет необходимых database constraints

Проблема:

- Не зафиксированы обязательные `UNIQUE` и schema-level checks на критических идентификаторах и структурах.

Почему это важно:

- Канон без жёстких constraints допускает дубли и тихую порчу реестра.

Вариант решения:

- Добавить:
  - `UNIQUE (merchant_sku)`
  - `UNIQUE (asin) WHERE asin IS NOT NULL`
  - `UNIQUE (bc_product_id) WHERE bc_product_id IS NOT NULL`
- Для `derivatives JSONB` — schema validation / CHECK.
- Для pricing и quantities — CHECK на неотрицательные значения и базовые business guards.

Где фиксировать:

- `SPEC-1-CATALOG-DATA-MODEL.md`

### [P1-B6] Inventory lifecycle не защищает от oversell

Проблема:

- Есть только `site_stock` и `amazon_reserved`.
- Нет reservation lifecycle, TTL, optimistic locking и race resolution.

Почему это важно:

- Для FBM это один из самых опасных operational failure modes.

Вариант решения:

- Добавить модель:
  - `physical_on_hand`
  - `site_reserved`
  - `amazon_reserved`
  - `committed`
  - `released`
  - `available_to_allocate`
  - `inventory_version`
- Race rule: `first successful commit wins`.
- Checkout reserve для сайта — TTL based.
- При critical mismatch — auto-pause listing.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-5-PIPELINES.md`
- `SPEC-1-CATALOG-DATA-MODEL.md`

### [P1-B7] Race condition между inventory workflows и batch engine

Проблема:

- Inventory updates и batch/publish jobs могут конкурировать за одну запись.

Почему это важно:

- Без locking/versioning publish может уйти со stale quantity или stale price/content.

Вариант решения:

- Использовать `SELECT ... FOR UPDATE` или optimistic version field.
- Зафиксировать serialization rule для inventory- и publish-critical updates.

Где фиксировать:

- `SPEC-5-PIPELINES.md`
- `SPEC-1-CATALOG-DATA-MODEL.md`

### [P1-B8] Нет audit trail по статусам и approvals

Проблема:

- Не хватает детального журнала: кто, когда, что и почему поменял.

Почему это важно:

- Без этого невозможно безопасно разбирать suppressed listings, inventory incidents и human overrides.

Вариант решения:

- Добавить отдельную таблицу наподобие `icon_status_audit`.
- Логировать:
  - actor
  - action
  - old_value
  - new_value
  - reason
  - timestamp
  - workflow_run_id

Где фиксировать:

- `SPEC-1-CATALOG-DATA-MODEL.md`
- `MASTER-ARCHITECTURE-v4.md`

---

## 5. Блок C — Amazon и marketplace operations

### [P1-C1] Не зафиксирован один детерминированный launch path для Amazon

Проблема:

- В обсуждениях и документах смешиваются:
  - автоматическая публикация через SP-API
  - ручная публикация через Seller Central
  - гипотетическая загрузка через BigCommerce

Почему это важно:

- Без одного primary path проект застрянет в двойной архитектуре.

Вариант решения:

- Зафиксировать:
  - `Primary path: manual approval + automated submit via n8n -> SP-API`
  - `BigCommerce -> Amazon connector: not used`
  - `Manual Seller Central listing: emergency fallback only`

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-4-AMAZON-LAUNCH.md`
- `SPEC-5-PIPELINES.md`

### [P1-C2] GTIN Exemption должен быть prerequirement, а не runtime surprise

Проблема:

- Сейчас это выглядит как один из шагов запуска, а не hard precondition.

Почему это важно:

- Без него первый реальный listing flow может развалиться уже в момент публикации.

Вариант решения:

- Зафиксировать: `GTIN Exemption approved = hard prerequisite before first publish`.
- Добавить n8n validation, которая не даёт отправить listing без confirmed exemption state.

Где фиксировать:

- `SPEC-4-AMAZON-LAUNCH.md`
- `SPEC-5-PIPELINES.md`

### [P1-C3] Browse node нельзя оставлять “после пилота”

Проблема:

- Формулировка слишком мягкая: browse node как будто можно определить после первых публикаций.

Почему это важно:

- Неверный category mapping и missing required attributes создают suppression risk.

Вариант решения:

- Зафиксировать:
  - browse node strategy должна быть валидирована до первого рабочего batch publish;
  - допустим canary test на 1 SKU;
  - массовая публикация только после успешного category validation.

Где фиксировать:

- `SPEC-4-AMAZON-LAUNCH.md`
- `SPEC-5-PIPELINES.md`

### [P1-C4] Нет жёсткого Amazon risk-control contract

Проблема:

- Нет полного набора проверок, которые отсекают опасный payload до публикации.

Почему это важно:

- Автопубликация без guardrails — главный путь к suppressed listings и account health проблемам.

Вариант решения:

- Перед submit делать:
  - schema validation
  - productType validation
  - required attribute check
  - compliance flags check
  - image gate check
  - price floor / MAP / margin floor validation
  - category / browse node validation
- Запускать первый batch только как canary.
- Любой failed validation = `fail closed`.

Где фиксировать:

- `SPEC-4-AMAZON-LAUNCH.md`
- `SPEC-5-PIPELINES.md`

### [P2-C5] Brand Registry и duplicate listing risk не описаны как marketplace risk

Проблема:

- Без Brand Registry бренд и листинги слабо защищены.

Почему это важно:

- Это не immediate blocker, но стратегически повышает риск duplicate or hijacked listing scenarios.

Вариант решения:

- Зафиксировать как marketplace risk register item.
- Для launch работать без Brand Registry допустимо.
- После traction — поднять как Phase 5 priority.

Где фиксировать:

- `SPEC-4-AMAZON-LAUNCH.md`
- `ROADMAP.md`

### [P2-C6] Статическая модель Amazon fee assumptions

Проблема:

- `amazon_fee_factor` описан слишком грубо.

Почему это важно:

- Fee reality меняется по category и operational conditions; статический коэффициент может разрушить маржу.

Вариант решения:

- Добавить margin floor validation.
- Отдельно учитывать referral fee assumption, shipping burden и manual override cases.
- Пометить `amazon_fee_factor` как default planning simplification, а не вечную формулу.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-1-CATALOG-DATA-MODEL.md`

---

## 6. Блок D — Supplier, media и content operations

### [P1-D1] Нет supplier intake contract

Проблема:

- Поставщик обозначен стартовой точкой, но нет строгого контракта входных данных.

Почему это важно:

- Pipeline A нельзя реализовать безопасно без минимального intake specification.

Вариант решения:

- Добавить `Supplier Intake Contract`:
  - required columns
  - accepted file formats
  - units
  - photo minimums
  - naming
  - accept/reject criteria
  - re-request path for incomplete package

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-5-PIPELINES.md`

### [P1-D2] Verified claims для религиозного контента не определены

Проблема:

- AI и operator review не отделены от evidence-backed religious/provenance claims.

Почему это важно:

- Это buyer trust risk, marketplace compliance risk и репутационный риск.

Вариант решения:

- Добавить поля:
  - `blessing_status`
  - `provenance_status`
  - `handmade_claim_status`
  - `claim_evidence_uri`
  - `claim_review_required`
- Правило:
  - AI не может придумывать religious/provenance claims без evidence.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-3-CONTENT-GENERATION.md`
- `SPEC-1-CATALOG-DATA-MODEL.md`

### [P2-D3] Нет redo path для rejected images и rejected content

Проблема:

- `rejected` не превращается в управляемый rework lifecycle.

Почему это важно:

- Объекты зависают, процесс распадается на ad hoc ручные действия.

Вариант решения:

- Ввести lifecycle:
  - `rejected`
  - `assigned_for_rework`
  - `rework_submitted`
  - `re_review`
- Добавить:
  - `rejection_code`
  - `rework_owner`
  - `attempt_no`
  - `due_at`

Где фиксировать:

- `SPEC-2-IMAGE-PIPELINE.md`
- `SPEC-3-CONTENT-GENERATION.md`
- `SPEC-7-OPERATOR-PLAYBOOK.md`

### [P2-D4] Processor mode decision logic слишком абстрактна

Проблема:

- Неясно, когда item идёт в auto, manual или outsource.

Почему это важно:

- Mode switching станет ситуативным и неаудируемым.

Вариант решения:

- Добавить decision table:
  - признаки clean photo -> auto
  - glare / geometry / difficult edge -> manual
  - repeated auto-fail -> outsource
- Зафиксировать failover rule.

Где фиксировать:

- `SPEC-2-IMAGE-PIPELINE.md`

### [P2-D5] Нет schema/quality contract для `derivatives`

Проблема:

- `derivatives` хранится слишком свободно.

Почему это важно:

- Легко получить неполные, неверно названные или inconsistent media payloads.

Вариант решения:

- Добавить validation schema и required derivative set per channel.
- Ввести media completeness flags.

Где фиксировать:

- `SPEC-1-CATALOG-DATA-MODEL.md`
- `SPEC-2-IMAGE-PIPELINE.md`

---

## 7. Блок E — Customer, storefront и service design

### [P2-E1] Storefront contract слишком пустой для stable architecture

Проблема:

- `SPEC-6` остаётся skeleton, а сайт при этом launch channel.

Почему это важно:

- Без MVP storefront contract нельзя стабильно проектировать content blocks, trust signals, routing rules и support copy.

Вариант решения:

- Поднять в канон минимум:
  - обязательные страницы
  - PDP blocks
  - trust blocks
  - returns / support visibility
  - Amazon routing rules
  - mobile constraints

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-6-STOREFRONT.md`

### [P2-E2] Нет customer policy contract

Проблема:

- Не описаны returns, damaged items, support SLA, shipping promise.

Почему это важно:

- Это критично и для buyer trust, и для FBM operations.

Вариант решения:

- Добавить:
  - `returns_policy_required`
  - `damaged_item_flow`
  - `support_sla`
  - `contact_surface`
  - `refund_escalation_owner`
  - `tracking_upload_deadline`

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-6-STOREFRONT.md`
- `SPEC-4-AMAZON-LAUNCH.md`

### [P2-E3] Не определена channel-routing policy

Проблема:

- Документы говорят, что сайт и Amazon запускаются параллельно, но не фиксируют, когда вести клиента в direct checkout, а когда в Amazon.

Почему это важно:

- Без этого возникнет UX drift, pricing confusion и разные обещания в каналах.

Вариант решения:

- Добавить channel-routing contract:
  - site-primary SKU
  - Amazon-primary SKU
  - when to show `View on Amazon`
  - allowed price divergence
  - allowed copy divergence

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-6-STOREFRONT.md`

---

## 8. Блок F — MVP mode vs full automation

### [P1-F1] Канон не различает primary launch architecture и lean manual fallback

Проблема:

- В обсуждении смешиваются:
  - целевая автоматизированная архитектура
  - ручной MVP-режим

Почему это важно:

- Это создаёт ощущение, что система одновременно автоматическая и ручная.

Вариант решения:

- Добавить две явные модели:
  - `Canonical launch architecture`
  - `Lean MVP fallback`
- Для каждого pipeline A-G указать:
  - automated default
  - allowed manual fallback
  - decision threshold when fallback must be retired

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `ROADMAP.md`

### [P2-F2] Не определено, что именно стоит автоматизировать уже в MVP

Проблема:

- Без этого команда будет спорить о каждом процессе заново.

Почему это важно:

- Можно либо переусложнить MVP, либо слишком рано увязнуть в ручной рутине.

Вариант решения:

- Для MVP автоматизировать в первую очередь:
  - master record
  - content generation
  - media ingest and derivatives
  - inventory sync
  - BC publish
  - Amazon payload assembly
- Оставить ручными:
  - owner approve
  - image QA
  - content review
  - final Amazon publish approval
  - exception triage

Где фиксировать:

- `ROADMAP.md`
- `MASTER-ARCHITECTURE-v4.md`

### [P2-F3] Локальный editor workflow не описан как semi-automatic process

Проблема:

- Не описано, как редактор локально правит фото/контент и как это входит обратно в shared control plane.

Почему это важно:

- Иначе локальная работа будет жить вне audit trail и статусов.

Вариант решения:

- Зафиксировать semi-automatic pattern:
  - local edit
  - upload approved artifact
  - DB status update
  - next pipeline trigger
- Для контента — browser-first через Supabase / CSV.
- Для изображений — локальный editor + managed upload + checksum.

Где фиксировать:

- `SPEC-2-IMAGE-PIPELINE.md`
- `SPEC-3-CONTENT-GENERATION.md`
- `SPEC-7-OPERATOR-PLAYBOOK.md`

### [P2-F4] Gemini image editing не позиционирован в архитектуре

Проблема:

- Появился новый candidate tool, но в каноне он никак не отражён.

Почему это важно:

- Без явной политики появится соблазн превратить генеративный инструмент в production truth source.

Вариант решения:

- Зафиксировать:
  - `Gemini image editing = assistive/manual tool`
  - `Not canonical production pipeline for truth-preserving master images`
  - Любой output из Gemini проходит manual QA before publish
- Deterministic baseline остаётся:
  - background removal
  - crop / resize / format conversion
  - checksum-backed approved master

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-2-IMAGE-PIPELINE.md`

---

## 9. Блок G — Technical debt и scale readiness

### [P2-G1] Backup/restore пока недооценён

Проблема:

- Требования есть, но runbook и restore contract отсутствуют.

Почему это важно:

- PostgreSQL — канонический control plane; его нельзя держать без formal recovery contract.

Вариант решения:

- Добавить:
  - `RPO`
  - `RTO`
  - restore order
  - post-restore reconciliation
  - monthly restore drill

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-7-OPERATOR-PLAYBOOK.md`

### [P2-G2] Observability слишком реактивна

Проблема:

- Есть алерты, но нет полноценной telemetry model.

Почему это важно:

- На 1000+ SKU ручная визуальная проверка не масштабируется.

Вариант решения:

- Добавить:
  - `correlation_id`
  - per-step counters
  - latency buckets
  - alert classes by severity
  - dead-letter queue / bucket
  - proactive Slack alerts

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-5-PIPELINES.md`

### [P3-G3] Multi-supplier модель не оформлена как first-class contract

Проблема:

- Архитектура уже намекает на multi-supplier, но не оформляет его явно.

Почему это важно:

- Это начнёт ломать namespace, replenishment ownership и lead-time logic при росте.

Вариант решения:

- Добавить `supplier_id` как first-class field.
- Зафиксировать folder namespace и supplier ownership rules.

Где фиксировать:

- `SPEC-1-CATALOG-DATA-MODEL.md`
- `MASTER-ARCHITECTURE-v4.md`

### [P3-G4] HTML sanitization и data-integrity policy не формализованы

Проблема:

- Канон не определяет sanitize policy для HTML и allowlist для input/output.

Почему это важно:

- Это и security risk, и content integrity risk.

Вариант решения:

- Добавить:
  - HTML allowlist
  - deny script/iframe
  - enum validation
  - log redaction rules

Где фиксировать:

- `SPEC-3-CONTENT-GENERATION.md`
- `MASTER-ARCHITECTURE-v4.md`

### [P2-G5] Storage provider ещё не зафиксирован детерминированно

Проблема:

- В проекте фигурируют и `R2`, и `S3`, но нет закрытого primary choice.

Почему это важно:

- Это влияет на delivery URLs, cost model, ops playbook и naming in docs.

Вариант решения:

- Зафиксировать:
  - `Cloudflare R2 = launch default`
  - `AWS S3 = compatible fallback / future migration option`
- Не hardcode vendor в code contracts.

Где фиксировать:

- `MASTER-ARCHITECTURE-v4.md`
- `SPEC-2-IMAGE-PIPELINE.md`
- `ROADMAP.md`

---

## 10. Топ-5 первоочередных задач

### 1. Закрыть противоречия канона

Что закрыть:

- intake path
- review workflow
- image handoff
- n8n hosting
- Amazon account assumption

### 2. Добавить publish gate и sync/idempotency contracts

Что закрыть:

- publish predicates
- sync states
- idempotency keys
- status audit

### 3. Закрыть inventory lifecycle

Что закрыть:

- reservation model
- race control
- oversell prevention
- cross-channel sync rules

### 4. Зафиксировать Amazon launch controls

Что закрыть:

- primary publish path
- GTIN as precondition
- browse node validation before batch
- fail-closed validation contract

### 5. Зафиксировать MVP mode и storage decision

Что закрыть:

- hybrid MVP vs full automation
- R2 as default
- Gemini assistive-only policy

---

## 11. Рекомендуемый следующий шаг

Самый правильный рабочий порядок:

1. Исправить Блок A в каноне.
2. Зафиксировать Блок B в DDL и pipeline specs.
3. Закрыть Amazon launch decisions.
4. Поднять storefront/customer policy minimum.
5. Выпустить `AUDIT-v3` как проверку после внесённых правок.

