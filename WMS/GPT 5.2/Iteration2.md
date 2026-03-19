## Iteration 2 Summary (ADD)

### Goal (ADD Step 2)
**Foundation for inventory correctness**: design the core inventory model, invariants, and transaction boundaries so that inventory-affecting actions are consistent and auditable across key domain objects (balances, statuses, reservations).

### Drivers addressed (ADD Step 2)
- **User stories**: US-01, US-02, US-15
- **Quality attributes**: QA-04, QA-05
- **Architectural concerns**: AC-05

### Elements refined (ADD Step 3)
- **Inventory correctness core**: `Inventory module` (balances, statuses, reservations, invariants, transaction boundaries)
- **Inbound to inventory**: `Inbound module` → `Inventory module` inventory posting semantics (receiving)
- **Outbound to inventory**: `Outbound module` → `Inventory module` reservation lifecycle tied to demand
- **Audit**: `Audit and traceability module` for inventory-affecting actions
- **Data & transactions**: `WMS operational database` transaction boundaries for inventory postings
- **Idempotent processing controls**: `Integration gateway` + `Idempotency and dedup store` + `IntegrationMessage` for inventory-affecting commands

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Dual-model inventory persistence: append-only Inventory Ledger plus current-state `InventoryBalance` projection**|**QA-04/AC-05** strong auditability and replay; supports traceability for adjustments/receipts/reservations; enables integrity checks; **QA-05** fast reads from balance projection|More storage; needs projection maintenance and backfill; requires clear event taxonomy and immutability discipline|Only maintaining `InventoryBalance` rows (simpler but weak auditability/forensics and harder to reconcile inconsistencies)|
|**Inventory as “Position aggregate” per `Item` + `Location` + `InventoryStatus` with explicit invariants** (e.g., \(availableQty = onHandQty - reservedQty\), non-negative, status eligibility)|**QA-04** makes invariants enforceable in one place; reduces double-allocation risk; clearer ownership inside `Inventory module`|Hot-spot risk for high-churn SKUs/locations; requires concurrency strategy (locking/optimistic)|Spreading inventory rules across inbound/outbound/tasking modules (higher inconsistency risk and harder audits)|
|**Reservation-as-first-class state machine** (`InventoryReservation`: Pending → Committed → Released/Consumed/Expired) tied to `ReplenishmentOrderLine`|**US-15/QA-04** prevents double allocation; supports partials and cancellations; enables idempotent retries without duplicating reservations|More lifecycle complexity; needs cleanup/expiry policies and compensations|Implicit reservations by decrementing `availableQty` only (simple but poor traceability and harder to handle retries/partials safely)|
|**Single-transaction “inventory posting” boundary** for inventory-affecting commands (receive, put-away move, reserve, release, status transfer, adjust) with **optimistic concurrency** (version check) on `InventoryBalance/Position`|**QA-04** atomic updates prevent torn writes; **QA-05** avoids heavy global locking; works well with per-instance DB|Write conflicts under contention; requires retry semantics and careful idempotency|Serializable isolation everywhere / broad pessimistic locks (stronger safety but worse throughput/latency and higher deadlock risk)|
|**Idempotent command handling for inventory-affecting APIs** (idempotency key → recorded outcome, applied at `Integration gateway` and/or `Inventory module` boundary)|**QA-04** safe retries from intermittent clients/systems; prevents duplicate receipts/reservations/adjustments; improves operational recovery|Key strategy and retention policy required; storage overhead; needs consistent “same request, same result” semantics|Best-effort retries without idempotency (duplicate postings and inventory corruption risk)|
|**Audit as a first-class module output**: every inventory posting emits immutable `AuditEvent` + ledger entry (and optionally an integration event via outbox)|Meets “consistent and auditable” goal; supports US-15 governance and later US-18; simplifies investigations and reconciliation|Extra write overhead per transaction; requires standardized event schema and PII/security controls|Ad-hoc logging only (insufficient for compliance-grade traceability and weak for reconciliation)|
|**Read-model shaping for QA-05**: query-oriented projections (e.g., by `Location`, by `Item`, by “available for picking”) maintained from inventory postings|Fast interactive screens/work queues; isolates query complexity from write path; enables indexing tuned to UI patterns|Eventual consistency between write and read models if projections async; added operational complexity|Querying ledger for operational screens (too slow/complex for 1s p95 targets)|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- Iteration 2 driver summary in the drivers section
- `InventoryLedgerEntry` added to the domain model and linked to inventory-affecting entities
- Inventory invariants and a reservation lifecycle state machine
- New sequence diagrams for Iteration 2:
  - Receiving posts inventory with ledger + balance + audit + outbox
  - Reservation posting to prevent double allocation
  - Inventory status transfer as an auditable posting
- Interface list extended with an `Inventory operational API` requiring idempotency for inventory-affecting commands

|Instantiation decision|Rationale|
|---|---|
|Add `InventoryLedgerEntry` as an append-only inventory posting record in the domain model|Establishes an auditable, replayable source for inventory-affecting actions to satisfy **QA-04** and support cross-system consistency analysis (**AC-05**).|
|Clarify `InventoryBalance` as a current-state projection optimized for operational queries|Keeps interactive performance aligned with **QA-05** while allowing correctness to be grounded in ledger postings (**QA-04**).|
|Define explicit inventory invariants (`availableQty = onHandQty - reservedQty`, non-negative quantities per `Item` + `Location` + `InventoryStatus`)|Makes correctness rules explicit and enforceable at the inventory boundary, reducing inconsistency risk (**QA-04**, **AC-05**).|
|Add an explicit reservation lifecycle state machine (Pending → Committed → Consumed/Released/Expired)|Provides a clear, robust model for reservation semantics tied to **US-15**, enabling safe retries and traceability (**QA-04**).|
|Add a receiving sequence to show one transaction (receipt + ledger + balance + audit + outbox)|Demonstrates the intended transaction boundary and auditability for **US-01**, directly supporting **QA-04** and setting a pattern for inventory postings.|
|Add a reservation sequence with ledger/audit and event emission|Shows how to prevent double allocation and keep inventory consistent under retries (**QA-04**) while preparing for outbound planning flows.|
|Add a status transfer sequence (available to quarantined) as a posting|Instantiates inventory status correctness for **US-15** with an auditable trail and consistent balance updates (**QA-04**).|
|Add `Inventory operational API` to Interfaces with idempotency requirement|Makes the external contract intent explicit: commands must be safe under retries to protect inventory integrity (**QA-04**) while still supporting fast reads (**QA-05**).|

### Recorded design decisions (ADD Step 6)

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|
|QA-04, AC-05, US-01, US-02|Adopt an **append-only inventory ledger** (`InventoryLedgerEntry`) as the source of audit truth for all inventory-affecting postings, with `InventoryBalance` maintained as a current-state projection for operational reads.|Provides a consistent, auditable trail for inventory changes and enables verification/replay to detect and correct inconsistencies (**QA-04**, **AC-05**) while preserving fast interactive queries (**QA-05**).|Only persisting mutable `InventoryBalance` rows (simpler but weak auditability and harder to reconcile inconsistencies after failures).|
|QA-04, AC-05, US-15|Make **inventory invariants explicit** at the inventory boundary (e.g., `availableQty = onHandQty - reservedQty`, non-negative quantities per `Item` + `Location` + `InventoryStatus`) and enforce them as part of inventory postings.|Reduces risk of double allocation and inconsistent availability calculations; concentrates correctness rules where inventory changes occur (**QA-04**) and supports cross-system consistency reasoning (**AC-05**).|Embedding availability/reservation logic across multiple modules (higher inconsistency risk and harder auditability).|
|QA-04, AC-05, US-15|Model `InventoryReservation` as a **first-class state machine** (Pending → Committed → Consumed/Released/Expired) and record reservation postings in the ledger.|Supports idempotent retries, partial processing, and safe release/expiry semantics; provides traceability for reservation-driven availability changes (**QA-04**, **AC-05**).|Implicit reservations by decrementing `availableQty` only (harder to audit, brittle under retries/partials).|
|QA-04, QA-05|Require **idempotency for inventory-affecting commands** on the `Inventory operational API` (receipt posting, reserve/release, status change, adjustment), with outcomes recorded for safe retries.|Prevents duplicate effects under intermittent connectivity and retries (**QA-04**) while enabling predictable client behavior and operational recovery without manual reconciliation; keeps interactive flows responsive by avoiding heavyweight locking (**QA-05**).|Best-effort retries without idempotency (duplicate postings and inventory corruption risk).|

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|US-01|**Partially satisfied** — Receipt posting is instantiated as a single inventory-affecting transaction (ledger + balance + audit), but receipt edge cases (partial receipts, over/under, exception handling impacts on balances/statuses) are not yet fully specified.|
|US-02|**Partially satisfied** — The model supports location-anchored balances, but put-away semantics are not yet fully clarified (is it purely a location move vs. also affecting availability/status, and how that is posted/audited).|
|US-15|**Partially satisfied** — Status is modeled and a status-transfer posting sequence exists, but detailed rules are missing (allowed transitions, reservation interactions, preventing reserving quarantined/damaged stock).|
|QA-04|**Partially satisfied** — Ledger + explicit invariants + reservation state machine + idempotent inventory commands address core integrity risks, but the design doesn’t yet define the concrete idempotency key strategy, dedup retention, and how idempotency spans internal vs external callers.|
|QA-05|**Partially satisfied** — Position-based `InventoryBalance` as a projection supports fast reads in principle, but no concrete read-model/indexing strategy is specified yet (e.g., common query shapes for screens/work queues and how projections stay current).|
|AC-05|**Partially satisfied** — Inventory postings are now auditable and replayable and can emit events via outbox, but we haven’t yet specified cross-boundary consistency mechanisms (which inventory events are published, how consumers reconcile, and how shipment/financial consistency ties back to inventory ledger entries).|

