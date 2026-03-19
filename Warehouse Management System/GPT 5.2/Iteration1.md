## Iteration 1 Summary (ADD)

### Goal (ADD Step 2)
**Initial structuring of the system**: define the primary architectural decomposition and boundaries for a single `WMSInstance` and its integrations, establishing the “walking skeleton” containers and responsibility allocation.

### Drivers addressed (ADD Step 2)
- **Architectural concerns**: AC-02, AC-04
- **Constraints**: C-01, C-02, C-03, C-04, C-05, C-06, C-07
- **Quality attributes**: QA-06, QA-08

### Elements refined (ADD Step 3)
- **Whole system**: the `WMSInstance` end-to-end boundary including external integrations (store systems, financial system, warehouse automation).

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Cell-based architecture: single `WMSInstance` = isolated deployment unit** (separate namespace/account/VPC + dedicated primary datastore)|**QA-08/AC-04** strong blast-radius isolation; easier per-instance scaling, lifecycle, and compliance boundaries; clearer cost attribution|More deployments to operate; cross-instance reporting becomes a platform concern; potentially higher baseline cost|Shared multi-tenant runtime + shared database with `tenant_id` (lower cost but weaker isolation and higher “noisy neighbor” risk)|
|**DDD-based modular monolith for core WMS domain (Iteration 1 walking skeleton)** with explicit bounded-context modules (Inbound, Inventory, Outbound, Config, Reporting, etc.)|**AC-02** clear boundaries without distributed-systems overhead; fast iteration; strong consistency where needed; simpler debugging/ops early|Scaling is coarser-grained; later extraction to services requires discipline (module boundaries, contracts)|Microservices from day 1 (better independent scaling, but much higher complexity/ops risk for iteration 1)|
|**Integration boundary via Anti-Corruption Layer + adapter/connector pattern** (Store, Financial, Automation) with canonical internal events/commands|**QA-06/C-04/C-05/C-06** decouples core domain from external contracts; enables adding/changing external systems without core rewrites; isolates protocol variability|More moving parts; requires contract governance and mapping; potential latency and eventual consistency|Point-to-point bespoke integrations directly inside core domain (fast initially but brittle and hard to evolve)|
|**Event-driven integration using a managed broker** (pub/sub topics per instance + contract versioning) plus synchronous APIs for interactive/operator flows|**QA-06** loose coupling; supports async reliability patterns; easier fan-out to multiple consumers; aligns with independent instance topology|Operational complexity (schema/versioning, ordering semantics); eventual consistency for integrations; needs observability|Purely synchronous REST between all systems (tight coupling; poorer resilience to spikes/outages)|
|**Reliability tactics for integrations: Transactional Outbox + Idempotency keys + Dedup store** (per instance)|Supports **idempotent/exactly-once-effective** processing across retries; prevents duplicate effects during failures; improves auditability|Extra persistence and implementation complexity; needs careful key strategy and retention|Best-effort messaging without dedup/outbox (simpler but risks duplicates/data corruption under failure)|
|**Shared platform services with per-instance enforcement**: centralized Identity + Observability, instance-scoped authorization and telemetry partitioning|Meets **C-03/C-07** while keeping instance isolation at the data plane; reduces duplicated ops tooling; enables fleet-level monitoring|Shared-service failure can impact visibility/login if not made highly available; requires strong partitioning to avoid data leakage|Fully separate identity/observability per instance (max isolation but high cost and inconsistent operations)|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- **Context diagram** for a single `WMSInstance`
- **Iteration 1 driver summary**
- **C4 container view** (API gateway, core app, integration gateway, event bus, database, outbox publisher, idempotency store, shared identity/observability)
- **Component view** for the modular core
- **Walking skeleton sequence diagrams** for store order intake, automation pick confirmations, and shipment confirmation fan-out
- **Initial interface list**

|Instantiation decision|Rationale|
|---|---|
|Define `WMSInstance` as an **isolated deployment boundary** with per-instance data plane (DB, event bus) and per-instance routing at the API gateway|Directly satisfies **QA-08 / AC-04** by minimizing blast radius and preventing noisy-neighbor effects; simplifies per-warehouse lifecycle management.|
|Instantiate a **WMS core application** as a modular monolith with bounded modules (Inbound, Inventory, Outbound, Tasking, Config, Audit, Authorization)|Addresses **AC-02** by establishing clear boundaries early while keeping complexity manageable for Iteration 1 and enabling a “walking skeleton.”|
|Introduce an **Integration gateway** as an Anti-Corruption Layer for store, financial, and automation integrations|Meets **QA-06** and constraints **C-04/C-05/C-06** by decoupling external contracts/protocols from core domain logic and enabling evolution without rewriting the core.|
|Add **Transactional outbox + outbox publisher** to emit events to a managed **event bus**|Provides a reliable foundation for decoupled event streams (QA-06) and supports safe publication of integration events without dual-write risks.|
|Add **idempotency and dedup store** on the integration boundary for inbound integration calls|Supports reliable processing under retries/intermittent connectivity (especially relevant to **C-06**) and prevents duplicate side effects in core workflows.|
|Partition **shared platform services** (Identity, Observability) outside the instance boundary with strict per-instance partitioning|Satisfies **C-03/C-07** while keeping operational overhead reasonable and preserving instance isolation at the data plane.|

### Recorded design decisions (ADD Step 6)

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|
|AC-04, QA-08, C-03|Adopt a **cell-based instance model** where each `WMSInstance` is an isolated deployment unit with its own primary datastore and eventing resources; requests are routed to the correct instance at the edge.|Strong fault and performance isolation reduces blast radius and prevents noisy-neighbor effects; enables independent lifecycle management per warehouse instance while still allowing selective shared platform services.|Shared multi-tenant runtime and shared database with tenant partitioning (lower baseline cost but weaker isolation and higher operational and data-leakage risk).|
|AC-02|Implement the core WMS as a **modular monolith** for Iteration 1, with bounded modules for inbound, inventory, outbound, tasking, configuration, audit, and authorization.|Establishes clear internal boundaries without premature distributed-systems complexity; supports a walking skeleton quickly while keeping the design evolvable.|Microservices from day one (better independent scaling but significantly higher complexity, latency, and operational burden early).|
|QA-06, C-04, C-05, C-06|Introduce an **Integration gateway** as an anti-corruption layer for store, financial, and automation systems; centralize protocol adaptation and contract versioning outside the domain core.|Decouples external system variability from the core domain; supports adding new systems or changing protocols without changing core modules; improves maintainability and reduces integration-induced coupling.|Embedding adapters and protocol logic directly inside domain modules (faster initially but brittle and hard to evolve across many integrations).|
|QA-06, C-01|Use a **managed event bus** for asynchronous integration flows and fan-out, complemented by synchronous REST APIs where needed.|Enables stable event streams and loose coupling; supports asynchronous workflows and future extensibility while leveraging managed services to reduce ops burden.|Purely synchronous point-to-point REST for all integrations (tighter coupling and lower resilience to spikes and downtime).|
|QA-06|Adopt **transactional outbox** in the per-instance database with an **outbox publisher** to publish events to the event bus.|Avoids dual-write inconsistencies; ensures events correspond to committed state changes; provides a reliable foundation for decoupled integration messaging.|Direct publish to the event bus inside request processing without outbox (risk of lost or duplicated events under failures).|
|C-06, QA-06|Enforce **idempotency and deduplication** at the integration boundary using idempotency keys and a dedup store per instance.|Warehouse and automation connectivity can be intermittent; idempotency prevents duplicate effects under retries and supports safe replays of integration calls/messages.|Best-effort retries without idempotency (risk of duplicate downstream effects and operational instability).|
|C-07, QA-08|Centralize authentication via a shared **identity provider** while keeping authorization checks **instance-scoped** in the core and integration gateway.|Balances operational simplicity with strict instance isolation: authentication is shared, but access decisions remain bounded by warehouse instance context.|Separate identity per instance (maximum isolation but high operational overhead and inconsistent user experience).|
|C-03|Centralize observability in a shared platform with **strict per-instance partitioning** for logs, metrics, and traces.|Supports fleet operations and troubleshooting at scale while respecting instance isolation requirements.|Per-instance observability stacks (strong isolation but higher cost and fragmented operations).|

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|AC-02|**Partially satisfied** — We defined clear module boundaries (core modular monolith + integration gateway), but we have not yet validated module ownership of key workflows/end-to-end transactions (inbound→inventory, order→allocation→picking→shipping) or identified which boundaries are likely future service seams.|
|AC-04|**Satisfied** — The architecture establishes `WMSInstance` as an isolated deployment boundary (cell model) with per-instance data plane and routing, plus limited shared platform services.|
|C-01|**Satisfied** — Containers assume managed services for event bus and support platform services, aligning with “managed where feasible.”|
|C-02|**Partially satisfied** — We captured locale/currency/time zone at `WMSInstance`, but we haven’t instantiated i18n/l10n mechanisms (translation workflow, formatting rules), nor compliance-driven data residency/legal docs impacts.|
|C-03|**Satisfied** — Independent instances with optional shared identity/observability are explicitly modeled.|
|C-04|**Partially satisfied** — Store integration boundary is defined (REST + events), but protocols, contract versioning rules, and inbound/outbound payload responsibilities are not yet specified.|
|C-05|**Partially satisfied** — Financial integration is represented (events and REST), but corporate pattern specifics (contracts, guarantees, acknowledgements) are not yet instantiated.|
|C-06|**Partially satisfied** — Automation integration is modeled with idempotency support, but intermittent-connectivity handling details (store-and-forward, retry policies, offline windows) are not yet designed.|
|C-07|**Partially satisfied** — Shared identity + instance-scoped authorization is defined, but encryption, RBAC model details, and key management/audit controls are not yet specified.|
|QA-06|**Satisfied** — Stable integration concepts are instantiated via integration gateway + event bus + outbox (decoupled APIs/event streams).|
|QA-08|**Satisfied** — Instance isolation is the primary organizing principle (cell model), with shared services explicitly separated and intended to be partitioned per instance.|

