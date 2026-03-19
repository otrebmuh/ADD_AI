## Iteration 5 Summary (ADD)

### Goal (ADD Step 2)
**Outbound shipping confirmations and financial integration**: design shipment lifecycle, store confirmations, and financial events with **strong consistency, traceability, and idempotency** across **asynchronous integrations**.

### Drivers addressed (ADD Step 2)
- **User stories**: US-08, US-09, US-10
- **Quality attributes**: QA-04, QA-06
- **Constraints**: C-04, C-05
- **Architectural concerns**: AC-03, AC-05

### Elements refined (ADD Step 3)
- **`Outbound module`**: shipment lifecycle and shipment confirmation flow, including shipment contents integrity (US-08/09/10, AC-05).
- **`Integration gateway`**: store and financial integration boundary behavior for shipment confirmation and financial events, including contract/versioning rules (QA-06, C-04, C-05, AC-03).
- **`Outbox and event publication module`** (and `Outbox publisher`): strengthen the shipment-confirmed and finance-relevant event publication guarantees tied to committed state (QA-04, AC-05).
- **`Idempotency and dedup store`**: idempotency keys, dedup scope, and “same request same outcome” rules for confirmations and integration retries (QA-04, AC-03).
- **`IntegrationMessage`**: message state model and traceability for store/finance exchanges (QA-04, QA-06).
- **`Audit and traceability module`**: end-to-end trace from shipment confirmation through published events and external acknowledgements (QA-04, AC-05).

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Shipment lifecycle as an explicit state machine** (e.g., Planned → Packed → Staged → Loaded → Confirmed; with Cancelled/Exception paths)|Makes **US-08/09/10** semantics unambiguous; enables “confirm once” rules and safe retries; improves traceability and reporting; provides clear trigger point for integrations (**AC-05**)|More modeling effort; requires strict transition/concurrency rules (e.g., partial shipments, re-open scenarios)|Implicit status via scattered flags/side-effects (simpler initially, brittle under retries and hard to audit)|
|**Confirm-shipment as the single atomic correctness boundary**: persist `Shipment` final state + immutable shipment contents snapshot + audit event + outbox records in one transaction|Directly supports **strong consistency + traceability** (**QA-04**, **AC-05**); prevents “confirmed but not published” and “published but not confirmed” dual-write gaps; enables deterministic replays|Requires careful transactional design and schema for the “contents snapshot”; may increase transaction size|Publishing integration messages directly during request handling without outbox (risk of lost/duplicated events); multi-step updates without atomic boundary (inconsistent states)|
|**Transactional outbox fan-out with per-subscriber event types** (e.g., `ShipmentConfirmed` for store, `FinancialShipmentEvent` for finance) with stable contract versioning|Meets **QA-06** decoupling; allows store and finance to evolve independently; supports async delivery and backpressure; aligns with existing outbox/event bus approach|Requires contract governance (versioning, compatibility, deprecation); event schema discipline|Single “god event” for all consumers (high coupling); synchronous point-to-point calls for all notifications (lower resilience)|
|**Idempotent integration at boundaries for shipping/finance**: idempotency keys + recorded outcomes for (a) shipment confirmation API and (b) inbound acks/callbacks from store/finance where used|Addresses **QA-04** under retries/failures; enables “same request, same result” semantics; reduces duplicate financial postings and duplicate store confirmations|Key strategy + retention needed; operational tooling required for stuck keys and replay|Attempting “exactly once over the wire” (unrealistic); best-effort retries without dedup (duplicate invoices/confirmations risk)|
|**Durable delivery tracking via `IntegrationMessage` as an outbox-backed delivery record** (Pending → Sent → Acked/Failed; DLQ/poison handling + replay controls)|Improves reliability and operability for **AC-03**; supports traceability from shipment to each external delivery attempt; enables controlled retries/backoff and manual replay|Adds persistence + scheduler complexity; requires clear retry limits and operational runbooks|“Fire-and-forget” event publishing with no delivery state (poor traceability); relying only on logs for recovery (high ops risk)|
|**Anti-corruption mapping per external system within `Integration gateway`** (store vs finance adapters; data contracts owned at boundary)|Protects core domain from external contract churn (**QA-06**, **AC-03**); isolates corporate finance patterns and store protocol differences; reduces blast radius of integration changes|Extra adapter design and testing; requires mapping maintenance over time|Embedding store/finance contract logic inside `Outbound module` (fast initially, long-term coupling and change risk)|
|**End-to-end correlation and audit linkage**: propagate `shipmentId` + `correlationId` across outbox events, integration messages, and audit trail|Strengthens traceability requirements and simplifies incident response; supports reconciliation across WMS/store/finance (**AC-05**)|Requires consistent propagation discipline across all hops; may require standardized logging/tracing fields|Ad-hoc correlation via timestamps or free-form notes (poor forensic value)|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- Iteration 5 driver summary in the iteration drivers section
- Addition of AC-03 to the architectural concerns list
- Refinement of `Shipment` semantics to commit an immutable **shipment contents snapshot** on confirmation
- Refinement of `IntegrationMessage` to support durable delivery tracking states
- Refined **Sequence 3** to route outbound notifications through the **Integration gateway** with dedup + delivery tracking
- New **Sequence 12**: retrying outbound deliveries from durable state without duplicating effects
- Refined interfaces to add/clarify **Shipping operational API** idempotency and outbound delivery semantics for store/finance

|Instantiation decision|Rationale|
|---|---|
|Add Iteration 5 drivers to the iteration summary|Maintains traceability from Iteration 5 goal to drivers (**US-08/09/10**, **QA-04/06**, **C-04/05**, **AC-03/05**) for subsequent refinements and decisions.|
|Instantiate shipment confirmation as the single commit point that also persists a shipment contents snapshot + audit + outbox record|Creates a clear correctness boundary for “confirm once, publish many” and prevents dual-write inconsistencies, supporting **QA-04** and **AC-05**.|
|Instantiate outbound notifications as delivered via `Integration gateway` consuming the event bus, not directly from the bus to external systems|Centralizes anti-corruption mapping, protocol adaptation, and integration governance at the boundary (**QA-06**, **AC-03**) while keeping the core domain stable.|
|Instantiate durable outbound delivery tracking using `IntegrationMessage` state for store and finance deliveries|Enables retries, operational visibility, and replay without losing traceability, while aligning with idempotent processing expectations (**QA-04**, **AC-03**).|
|Instantiate dedup/idempotent delivery semantics per target using the idempotency store|Ensures at-least-once delivery infrastructure does not cause duplicate downstream effects under retries/failures (**QA-04**) for both store and finance flows.|
|Define/instantiate interfaces to make idempotency explicit for shipment confirmation and outbound delivery|Clarifies contract expectations and operational semantics for Iteration 5 integrations (**QA-06**, **C-04**, **C-05**) and prevents duplicate effects from client retries (**QA-04**).|

### Recorded design decisions (ADD Step 6)
Iteration 5 decisions were recorded in Section 9 of `Design/Architecture.md`, including:
- Shipment confirmation as the single atomic correctness boundary (confirm + contents snapshot + audit + outbox)
- Store + finance outbound delivery via `Integration gateway`
- Durable delivery tracking via `IntegrationMessage`
- Idempotency/dedup for shipment confirmation and per-target outbound deliveries

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|US-08|**Partially satisfied** — Shipment confirmation is defined as a single correctness boundary, but the full shipment lifecycle/state machine and edge cases (partial shipments, reopen/cancel, exceptions) aren’t fully specified.|
|US-09|**Partially satisfied** — Store confirmations are routed via event/outbox and delivered via the integration boundary with durable tracking, but the detailed store-facing contract (payload shape, versioning, ack semantics) is not fully defined.|
|US-10|**Partially satisfied** — Financial shipment events follow the same reliable pattern and delivery tracking, but the invoicing-relevant data contract and any required validations/ack flows per corporate patterns aren’t fully specified.|
|QA-04|**Partially satisfied** — The design uses outbox + idempotency/dedup + durable delivery records, but key strategy (scope, generation, retention), DLQ/poison handling, replay tooling, and end-to-end guarantees still need explicit definition.|
|QA-06|**Partially satisfied** — Integrations are decoupled via events/APIs and the integration gateway, but contract governance (schema/versioning policy, compatibility rules) is not yet captured.|
|C-04|**Partially satisfied** — Store integration is addressed and decoupled, but “standard protocols” specifics and required behaviors (security, auth, transport, ack) aren’t fully enumerated.|
|C-05|**Partially satisfied** — Finance integration is addressed and decoupled, but the exact corporate pattern/data contract details (and any mandated synchronous steps) aren’t fully specified.|
|AC-03|**Partially satisfied** — Reliable integration patterns are instantiated (gateway, dedup, delivery tracking), but detailed failure-mode handling and operational controls (retry policies, rate limits, backpressure) are not fully defined.|
|AC-05|**Partially satisfied** — The shipment contents snapshot + atomic confirm boundary improves consistency, but cross-system reconciliation semantics (e.g., mismatch handling, replays vs. compensations) remain to be specified.|

