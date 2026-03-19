## Iteration 4 Summary (ADD)

### Goal (ADD Step 2)
**Picking execution and automation integration**: design pick task distribution and pick confirmations, including resilient/idempotent integration patterns with picking systems and exception handling to keep operations running.

### Drivers addressed (ADD Step 2)
- **User stories**: US-05, US-07, US-06
- **Quality attributes**: QA-04, QA-06, QA-05
- **Constraints**: C-06
- **Architectural concerns**: AC-03

### Elements refined (ADD Step 3)
- **Automation integration boundary**: `Automation integration API` (pick task distribution + pick confirmations) — contract shape, versioning, idempotency requirements, acknowledgements, retry semantics.
- **Connectivity-tolerant integration logic**: `Integration gateway` automation adapter — buffering/outbound delivery, retry/backoff, dedup/idempotency enforcement, intermittent-connectivity handling (C-06).
- **Picking orchestration**: `Tasking module` — `PickTask` lifecycle/state machine, assignment/dispatch to automation vs humans, work queue retrieval behavior (QA-05).
- **Picking domain entities**: `PickTask` and `PickConfirmation` — correlation identifiers, partial/multi-confirmations, reconciliation rules, and linkage to downstream work.
- **Inventory correctness touchpoints**: `Inventory module` — how pick confirmations consume reservations and post auditable inventory updates (QA-04).
- **Exception handling**: `ExceptionCase` (picking) — capture, triage, resolution flows and their effects on tasks and inventory (US-06).
- **Reliable messaging and publication**: `Outbox and event publication module` + `Outbox publisher` + `Event bus` — events around task issued/confirmed/exception with idempotent processing guarantees (QA-04, QA-06).
- **Idempotency + operational traceability**: `Idempotency and dedup store` + `IntegrationMessage` — idempotency key strategy scope for automation interactions and observability of message/task processing.
- **Read performance for operators/supervisors**: query/read-model shaping for pick work queues and exception queues to meet **QA-05**.

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Pick task lifecycle as an explicit state machine** (e.g., Created → Released → Dispatched → InProgress → Confirmed; with Cancelled/Exception paths)|**US-05/07/06** makes execution semantics unambiguous; supports retries and reconciliation; enables clear exception routing and reporting; improves operability and testability|More modeling effort; requires careful rules for transitions and concurrency|Implicit “status by side-effects” across tables/services (simpler but brittle, hard to audit/debug)|
|**At-least-once integration with automation + strict idempotency** for both directions (task dispatch and confirmation intake) using **idempotency keys + recorded outcomes**|Directly addresses **QA-04** under intermittent connectivity (**C-06**); safe retries without duplicate picks/inventory postings; supports replay/recovery|Requires key strategy, retention, and “same request, same result” discipline; needs operational tooling for stuck keys|Trying to guarantee “exactly once over the wire” (unrealistic); best-effort retries without dedup (duplicate effects risk)|
|**Store-and-forward “Automation Connector” pattern inside the instance boundary** (integration gateway persists outbound task messages and manages delivery/retry/backoff)|Improves resilience to automation downtime (**C-06**, **QA-04**); isolates vendor/protocol variability (**QA-06**, **AC-03**); enables backpressure and throttling to protect core|Adds persistence + scheduler complexity; requires monitoring for backlog/lag|Pure synchronous push to automation with immediate failure to the core (tight coupling, low resilience)|
|**Inbox/Outbox pattern for automation messages**: inbound confirmations recorded as `IntegrationMessage` first (inbox), outbound tasks emitted via transactional outbox|Avoids dual-write inconsistencies; improves auditability and recoverability; enables deterministic reprocessing; aligns with existing outbox approach|More tables/flows; requires replay semantics and operational controls|Direct bus publish or direct processing without durable recording (harder to recover and audit)|
|**Correlation model for pick execution**: stable identifiers across WMS and automation (e.g., `pickTaskId`, `externalTaskId`, `correlationId`) and support for partial/multiple confirmations|Enables robust reconciliation and exception handling (**US-06**); prevents misattribution of confirmations; supports partial picks and retries safely (**QA-04**)|More contract complexity; requires consistent identifier propagation across systems|Relying on timestamps/implicit matching (high operational risk and poor traceability)|
|**Exception-as-first-class workflow** (`ExceptionCase`) tied to pick tasks and inventory postings with controlled resolution actions (e.g., short pick, damage, re-pick, relocate)|Keeps operations running without corrupting inventory (**US-06**, **QA-04**); creates auditable trail; supports supervisor intervention and downstream adjustments|Extra workflow states and UI/ops handling; needs clear authorization and guardrails|Ad-hoc manual fixes outside the system (fast locally but high integrity and compliance risk)|
|**Inventory posting on confirmation via the Inventory module boundary** (consume reservations + append ledger entry + update balance projection as one correctness boundary)|Preserves **QA-04** and leverages Iteration 2 invariants/ledger; ensures picks have auditable inventory impact; supports safe retries if confirmation is idempotent|Potential contention hotspots for high-churn locations/SKUs; requires concurrency strategy and retry policy|Updating inventory in automation adapter or tasking module directly (leaks invariants, increases inconsistency risk)|
|**API contract versioning + anti-corruption mapping for automation** (stable canonical pick APIs internally; adapters per vendor/protocol externally)|Improves maintainability and decoupling (**QA-06**, **AC-03**); enables onboarding new automation with minimal core impact; supports long-lived warehouses with heterogeneous systems|More upfront integration design; contract governance needed|Per-vendor bespoke endpoints embedded in core domain modules (fast initially, high long-term coupling/cost)|
|**Performance-oriented pick work queries**: query projections for pick queues/exception queues (optionally cached) tuned to p95 < 1s|Targets **QA-05** directly; reduces load on write-path entities; improves operator/supervisor UX|Projection lag considerations; extra operational complexity for projection maintenance|Querying transactional tables with heavy joins only (simpler but risks missing 1s targets under load)|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- Iteration 4 driver summary in the drivers section
- Refined **Sequence 2** for automation confirmations to include inbox-first durability and idempotency recording
- New sequence diagrams for Iteration 4:
  - Sequence 9: pick task distribution using store-and-forward delivery
  - Sequence 10: manual pick confirmation with inventory posting boundary
  - Sequence 11: picking exception register/resolve workflow
- Refined `Automation integration API` interface notes to include correlation identifiers and idempotency expectations

|Instantiation decision|Rationale|
|---|---|
|Add Iteration 4 to iteration driver summary|Maintains traceability from the iteration goal to drivers (**US-05/06/07**, **QA-04/05/06**, **C-06**, **AC-03**) for subsequent refinements and decisions.|
|Instantiate automation confirmations as **inbox-first** (`IntegrationMessage` persisted before domain processing) + idempotency recording|Improves resilience and recoverability under intermittent connectivity and retries (**QA-04**, **C-06**) while preserving decoupled integration handling (**AC-03**, **QA-06**).|
|Instantiate **store-and-forward** pick task distribution (Sequence 9)|Ensures task dispatch continues reliably even when automation endpoints are intermittently unavailable (**C-06**) and supports safe retries without duplicating work (**QA-04**).|
|Instantiate manual confirmation path (Sequence 10) with inventory postings at the inventory boundary|Preserves inventory correctness by consuming reservations and writing ledger entries in a single consistency boundary (**QA-04**) while keeping interactive confirmation flow direct and responsive (**QA-05**).|
|Instantiate exception capture and resolution workflow (Sequence 11) tied to `ExceptionCase` and `PickTask`|Keeps operations running with controlled, auditable actions that prevent inventory corruption (**US-06**, **QA-04**) and provides supervisor control points.|
|Refine `Automation integration API` notes to require correlation identifiers and idempotency|Strengthens the external contract to support robust reconciliation, retries, and cross-system traceability (**QA-04**, **QA-06**, **AC-03**).|

### Recorded design decisions (ADD Step 6)

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|
|QA-04, QA-06, C-06, AC-03, US-05|Adopt a **store-and-forward automation connector** at the integration boundary: persist outbound pick task messages and manage delivery with retries/backoff and acknowledgements.|Tolerates intermittent warehouse automation connectivity (**C-06**) while keeping integrations decoupled (**QA-06**) and preventing lost/duplicated task dispatch effects under retries (**QA-04**); isolates protocol/vendor variability (**AC-03**).|Pure synchronous “push tasks” from core to automation (tight coupling, fragile under connectivity issues); direct point-to-point calls without durable send state (hard to recover).|
|QA-04, QA-06, AC-03, US-07|Require **idempotent pick confirmations** from automation (and manual confirmation APIs) using idempotency keys and recorded outcomes, with stable correlation identifiers (`pickTaskId`, `correlationId`).|Prevents duplicate inventory postings and duplicated confirmations under retries (**QA-04**); improves reconciliation and traceability across systems (**QA-06**, **AC-03**).|Attempting exactly-once delivery “over the wire” (unrealistic); best-effort retries without dedup (inventory corruption risk).|
|QA-04, AC-05, US-07|Handle pick confirmations by **posting through the Inventory module boundary** (consume reservations, append `InventoryLedgerEntry`, update `InventoryBalance` projection) as a single correctness boundary.|Reuses the inventory correctness foundation to ensure picks consistently affect inventory with auditability and replayability (**QA-04**, **AC-05**), regardless of source (automation or manual).|Updating inventory directly in tasking/integration code paths (leaks invariants and increases inconsistency risk).|
|US-06, QA-04|Treat picking exceptions as **first-class workflow** (`ExceptionCase`) with controlled resolution actions and audit events, tied to `PickTask` state transitions.|Keeps operations running while preventing ad-hoc inventory changes; provides auditable, supervised resolution that preserves correctness under disruptions (**US-06**, **QA-04**).|Manual out-of-band fixes and spreadsheet reconciliation (fast locally but high integrity/compliance risk); burying exceptions in logs only (poor operational control).|

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|US-05|**Partially satisfied** — Pick task distribution to automation is instantiated (store-and-forward + correlation), but the detailed `PickTask` state machine, dispatch/assignment rules (human vs automation), and cancellation/re-dispatch semantics are not fully specified.|
|US-07|**Partially satisfied** — Pick confirmations (automation + manual) are instantiated with idempotency and inventory posting boundary, but partial picks, multi-confirm events, conflict handling (e.g., duplicate/out-of-order confirmations), and reconciliation rules are not fully detailed.|
|US-06|**Partially satisfied** — Exception capture and resolution workflow is instantiated (`ExceptionCase` + audit + outbox), but the catalog of exception types and the allowed/guarded resolution actions (and their precise inventory/task impacts) are not yet fully defined.|
|QA-04|**Partially satisfied** — Inbox-first processing for confirmations + idempotency + outbox patterns significantly reduce duplicate/lost effects, but concrete idempotency key strategy, dedup retention, replay tooling, and end-to-end guarantees across task dispatch + confirm flows are not fully specified.|
|QA-06|**Partially satisfied** — Integration is decoupled via `Integration gateway` and versioned `Automation integration API`, but contract governance (schema/versioning policy), vendor adapter strategy details, and backward compatibility approach are not yet captured.|
|QA-05|**Partially satisfied** — The design recognizes fast pick/exception queue reads via projections, but the specific read models, indexing, and p95 latency strategy for work-queue endpoints are not yet defined.|
|C-06|**Partially satisfied** — Store-and-forward + retries addresses intermittent automation connectivity, but warehouse-side connectivity edge cases (long outages, backlog draining, operator fallbacks) and operational limits are not fully specified.|
|AC-03|**Partially satisfied** — Core integration patterns are selected/instantiated (anti-corruption layer, inbox/outbox, idempotency, correlation), but detailed failure-mode handling (poison messages, dead-lettering, manual replay), observability for integration flows, and SLA/error budgets are not yet defined.|

