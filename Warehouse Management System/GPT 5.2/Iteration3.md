## Iteration 3 Summary (ADD)

### Goal (ADD Step 2)
**High-throughput replenishment order intake and planning**: design order ingestion, validation, and allocation/wave planning to sustain peak demand without downtime while preserving inventory consistency.

### Drivers addressed (ADD Step 2)
- **User stories**: US-03, US-04
- **Quality attributes**: QA-01, QA-04
- **Architectural concerns**: AC-01, AC-02

### Elements refined (ADD Step 3)
- **Order intake boundary**: `Integration gateway` + `Store integration API` (contract/versioning, idempotency, durable acceptance)
- **Planning core**: `Outbound module` (`ReplenishmentOrder`, `ReplenishmentOrderLine`, `Allocation`, `Wave`) for validation and planning responsibilities
- **Correctness boundary during planning**: `Inventory module` reservation creation/commit behavior used by allocation to preserve QA-04
- **Data & transactions**: `WMS operational database` transaction boundaries and hot-spot considerations for ingestion and planning at scale
- **Reliable event emission**: `Outbox and event publication module` + `Outbox publisher` for order/planning events
- **Work creation**: `Tasking module` creation/lifecycle of `PickTask` derived from `Wave` planning

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Asynchronous order intake with “accept then process”**: `Store integration API` returns **202 Accepted** after durable persistence; allocation/planning happens asynchronously|**QA-01** smooths spikes via buffering and horizontal worker scaling; avoids tying peak planning time to API latency; supports **no downtime** deployments by draining/continuing async work|More moving parts; introduces eventual completion semantics (store must query status or consume events)|Fully synchronous “submit order + allocate in same request” (simpler but poor latency under spikes and higher downtime risk during deployments)|
|**End-to-end idempotency for order submission** using `externalOrderId` + idempotency key recorded in `Idempotency and dedup store` and linked to `ReplenishmentOrder` / `IntegrationMessage`|**QA-04** safe retries; prevents duplicate orders during network failures; enables at-least-once delivery without corrupting demand state|Retention/expiry policy needed; requires consistent “same request, same result” semantics|Best-effort retries without dedup (duplicates at peak); attempting “exactly-once over the wire” (unrealistic across distributed boundaries)|
|**Split `Outbound module` into two internal responsibilities**: **Order Intake** (validation, normalization, persistence) and **Planning** (allocation + wave building)|**AC-02** clarifies boundaries and scaling hotspots without full microservices; improves maintainability|Extra internal contracts/data handoff to design|One large outbound workflow component (fast to start but becomes bottleneck and hard to evolve)|
|**Staged validation pipeline**: schema/contract validation at `Integration gateway`, business validation in `Outbound module`, and deferred “deep checks” as async steps|**QA-01** keeps p95 API latency low; rejects bad payloads early to reduce wasted work|Deferred failures need compensating statuses/notifications|All validations synchronously before accept (strong immediate feedback but reduces throughput and increases timeout risk)|
|**Partitioned planning work**: workers consume “plan allocation” jobs partitioned by `(WMSInstance, order group)` (e.g., by window/priority/zone)|**QA-01** parallelism and predictable scaling; isolates contention domains; supports backpressure via leases/partitions|Partition strategy must avoid hotspots; partition rebalancing adds complexity|Single planner leader (simple but can’t meet peaks); random distribution (thrashing/lock contention)|
|**Reservation-based allocation**: planning commits `InventoryReservation` (and ledger postings) as atomic correctness boundary with retries|**QA-04** prevents double allocation under concurrent planners; leverages Iteration 2 invariants; safe/auditable retries|Contention risk for hot positions; needs clear reservation expiry/release rules tied to planning states|Allocation by decrementing `availableQty` in outbound tables (fast but breaks source of truth and auditability)|
|**Transactional outbox for demand/planning events** (`order accepted`, `allocation created`, `wave released`) via `Outbox and event publication module`|**QA-04** avoids dual-write inconsistencies; reliable fan-out; supports reprocessing and observability|Outbox lag monitoring/replay tooling needed; eventual delivery semantics|Direct publish inside request thread (risk of lost/duplicated events); polling DB tables from consumers (tight coupling)|
|**Backpressure and admission control at the edge**: rate limits per store, queue-depth-based throttling, circuit breaking around planning dependencies|**QA-01 / AC-01** protects instance from overload and runaway cost; stabilizes latency|Needs tuning/SLOs; may slow legitimate peak traffic|Unbounded acceptance (cascading failure); permanent overprovisioning for peak (high cost)|
|**Write-optimized persistence for high-volume intake**: ingestion-friendly indexes plus append-only `IntegrationMessage` records for traceability|**QA-01** reduces write contention; **QA-04** provides replayable exchange record for recovery|Index/schema tuning critical; storage growth needs retention/archival policies|Over-indexing for queries upfront (hurts writes); logs-only traceability (insufficient for replay)|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- Iteration 3 driver summary in the drivers section
- Updated responsibilities for `Outbound module` to include order intake + planning partitioning responsibilities
- New sequence diagrams for Iteration 3:
  - Async order intake with durable acceptance and planning trigger
  - Allocation and wave planning with reservation-based correctness
- Interface note refinement for `Store integration API` to clarify **202** acceptance and async planning

|Instantiation decision|Rationale|
|---|---|
|Add Iteration 3 driver summary to `Architecture.md`|Keeps traceability to **QA-01, QA-04, US-03, US-04, AC-01, AC-02** for subsequent refinements.|
|Instantiate async order intake as “durably accept then plan” (Sequence 7)|Supports **QA-01** by decoupling peak request volume from planning compute while preserving durability.|
|Instantiate planning as an event-triggered worker inside the instance boundary (Sequence 7)|Addresses **AC-01** peak sustainment and **AC-02** clarity without premature service extraction.|
|Instantiate reservation-based allocation during planning (Sequence 8)|Preserves **QA-04** by reusing inventory invariants/reservation postings to prevent double allocation under retries/concurrency.|
|Instantiate wave creation and pick task generation as part of planning flow (Sequence 8)|Supports **US-04** by showing how planning decisions become executable work (`Wave`, `PickTask`).|
|Instantiate planning outputs via transactional outbox events (Sequence 8)|Prevents dual-write inconsistencies (**QA-04**) while enabling scalable fan-out (**QA-01**).|
|Refine `Store integration API` notes to include 202 acceptance and async planning|Makes ingestion semantics explicit for **US-03/QA-01**.|

### Recorded design decisions (ADD Step 6)

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|
|QA-01, AC-01, US-03|Adopt an **async order intake** model: durably accept replenishment orders (202) after persistence, then trigger downstream planning asynchronously via outbox/event bus.|Decouples peak API traffic from planning compute to sustain **10,000 orders/hour** without downtime (**QA-01**), while preserving durability and recoverability.|Synchronous “submit and allocate” in the API path (higher tail latency and overload risk at peak).|
|QA-01, AC-02, US-04|Partition outbound planning into **Order Intake** and **Planning** responsibilities within the `Outbound module` and execute planning via horizontally-scalable workers.|Improves modularity and scalability at the hotspot without introducing distributed-service complexity; planning throughput can scale independently from ingestion (**QA-01**, **AC-02**).|Single monolithic outbound workflow component (harder to scale and evolve).|
|QA-04, US-04|Make **reservation-based allocation** the correctness boundary during planning: allocation commits `InventoryReservation` and related ledger/balance updates atomically with allocation artifacts.|Prevents double allocation under concurrent planning/retries; leverages inventory invariants and auditability from Iteration 2 to preserve **QA-04** during high-throughput allocation.|Allocating by decrementing availability in outbound tables without inventory postings (inconsistent source of truth, weaker auditability).|
|QA-04, QA-01, US-04|Emit `allocation created` and `wave released` as **transactional outbox events** tied to committed planning state.|Ensures consumers see only committed planning outcomes (**QA-04**) while enabling asynchronous fan-out and scaling (**QA-01**).|Direct bus publish inside planning transaction without outbox (risk of lost/duplicated events).|

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|US-03|**Partially satisfied** — Order intake is durably accepted with idempotency and an async planning trigger, but detailed API contract (validation rules, order status model, status query/notification semantics) is not yet fully specified.|
|US-04|**Partially satisfied** — Allocation→wave→pick task flow is instantiated and tied to reservations, but optimization strategies, prioritization, and replanning/cancellation behaviors are not yet defined.|
|QA-01|**Partially satisfied** — Async intake + worker planning enables horizontal scaling in principle, but concrete throughput controls (partitioning scheme, queue/backpressure limits, DB contention strategy, scaling signals) are not yet specified.|
|QA-04|**Partially satisfied** — Reservation-based allocation and outbox reduce correctness risks, but end-to-end idempotency for planning retries (deduping planning jobs, preventing duplicate waves/pick tasks/events) and retention policies are not yet fully defined.|
|AC-01|**Partially satisfied** — Peak handling is addressed via decoupling and scaling, but explicit cost-control mechanisms (admission control, rate limits, capacity targets) and hotspot mitigation are not yet captured.|
|AC-02|**Partially satisfied** — Responsibilities are clarified (intake vs planning within `Outbound module`), but internal element decomposition/interfaces between intake/planning (and with `Tasking module`) are not fully formalized beyond the sequences.|
