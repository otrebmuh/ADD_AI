## Iteration 6 Summary (ADD)

### Goal (ADD Step 2)
**Disconnected operations and synchronization**: design the approach for continuing warehouse operations during loss of connectivity and performing safe reconciliation and synchronization on reconnect (**no loss**, **no duplicates**, bounded divergence with external systems).

### Drivers addressed (ADD Step 2)

| Driver ID | Type | Description |
|---|---|---|
| **QA-03** | Quality (Availability) | Given a loss of connectivity between the warehouse and the cloud during normal warehouse operation hours, warehouse operations can continue for up to **3 hours**, with **zero data loss**, **no duplicate transactions**, and **full synchronization within 30 minutes** after connectivity is restored. Inventory and financial consistency must be preserved across systems; temporary divergence between WMS instances and external systems during the outage is acceptable. |
| **QA-04** | Quality (Reliability and Data integrity) | Given high-volume interactions with picking systems and store systems, when network interruptions or integration failures occur, all messages (orders, picks, shipment confirmations, financial events) must be processed **exactly once or in an idempotent manner** so that inventory and financial data remain consistent across systems. |
| **C-06** | Constraint | The WMS must integrate with in-warehouse automation systems via well-defined APIs or connectors that can tolerate intermittent connectivity in warehouses. |
| **AC-03** | Architectural concern | Design integrations with store systems, financial systems, and warehouse automation using APIs and or event-driven mechanisms to ensure reliability, idempotency, and resilience to network issues. |
| **AC-05** | Architectural concern | Ensure consistent inventory and shipment data across WMS, store systems, and financial system in the presence of asynchronous messaging and failures. |

### Elements refined (ADD Step 3)
- **`API gateway`**: warehouse-local access patterns and routing behavior when cloud connectivity is lost.
- **`WMS core application`** (notably `Inventory module`, `Tasking module`, `Outbound module`): define offline-safe workflow scope and how offline operations are represented for reconciliation.
- **`WMS operational database`**: reconciliation expectations with inventory invariants and ledger-based audit truth.
- **`Integration gateway`**: store-and-forward behavior and safe replay semantics for reconnect scenarios.
- **`Idempotency and dedup store`**: extend dedup to reconnect synchronization (operation ids and outcomes).
- **`Outbox publisher` + `Event bus`**: replay, backpressure, and synchronization behavior after reconnect.
- **New**: **Warehouse edge / offline operations capability** (explicit edge node, local store, and sync agent).

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Edge Offline Operations Node per warehouse** (local entry point and constrained WMS workflows during WAN outage)|Meets **QA-03** by enabling continued execution; improves governance vs per-device offline; clear operational boundary|Requires edge footprint and lifecycle; requires explicit offline-safe scope|Pure cloud-only (cannot meet QA-03); offline in every handheld client (governance and sync complexity)|
|**Local durable operational log** (append-only journal of inventory-affecting and execution operations) + deterministic replay|Zero data loss foundation; supports audit and reconciliation (**QA-04**, **AC-05**)|Requires careful operation modeling and keying; needs batching and compaction|Snapshot-only sync (hard to prove no gaps/duplicates); direct dual-write to cloud (fails in outage)|
|**Inbox/outbox + first-class idempotency** extended to edge and sync|Prevents duplicate effects under retries and reconnect storms; leverages existing patterns in the architecture|Requires clear key scoping and retention; more state to manage|Relying on “exactly once transport” (insufficient); broker semantics alone (insufficient at boundaries)|
|**Resumable bi-directional sync with checkpoints** (store-and-forward, batching, backpressure)|Targets “sync within 30 minutes” by avoiding full rescans; explicit progress tracking; controlled catch-up|Requires ordering and concurrency rules; backlog capacity planning|Multi-master DB replication (high conflict risk); manual reconciliation (slow/error-prone)|
|**Constrained offline domain semantics** (offline-safe operation set) + invariant-based reconciliation and exceptions|Preserves integrity by restricting risky operations; bounded divergence with controlled exception workflows|Some capabilities degraded offline; requires operator feedback and exception handling|Allow all operations offline with best-effort merge (high corruption risk); CRDT-based inventory merging (poor fit for reservations and audit)|
|**Edge automation adapters with local buffering** for C-06|Automation continues locally with buffering; consistent idempotency discipline for retries|Requires local network and adapter availability planning; edge security controls|Push buffering burden to each automation vendor/system; direct VPN to cloud only (still fails for WAN outage)|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- A new **Disconnected operations topology (Iteration 6)** container view introducing: `Edge gateway`, `Edge operations core`, `Edge operational store`, `Edge idempotency store`, `Edge outbox`, and `Sync agent`.
- New sequence diagrams for Iteration 6:
  - **Sequence 13**: offline pick confirmation recorded at the edge and later synchronized without duplicates
  - **Sequence 14**: reconnect synchronization with resumable checkpoints
- Interface refinements:
  - Added `Edge operational API`
  - Added `Edge synchronization interface`

|Instantiation decision|Rationale|
|---|---|
|Introduce a warehouse **edge node** (`Edge gateway`, `Edge operations core`)|Enables continued execution during WAN outages while keeping offline behavior bounded and operable (**QA-03**).|
|Add durable **`Edge operational store`** with offline operation records and projections|Provides **zero data loss** and deterministic reconciliation foundation aligned with ledger-based correctness (**QA-03**, **QA-04**, **AC-05**).|
|Add **`Edge idempotency store`** for edge-facing APIs|Prevents duplicate local effects under retries and intermittent automation connectivity (**QA-04**, **C-06**).|
|Add **`Edge outbox`** for cloud synchronization intent|Makes “to be synchronized” operations durable and replayable, consistent with established outbox patterns (**QA-04**).|
|Add a **`Sync agent`** with checkpoints, batching, and acknowledgements|Provides resumable synchronization and controlled catch-up to meet recovery objectives (**QA-03**, **QA-04**).|
|Instantiate cloud-side application via existing `Integration gateway` + cloud dedup/idempotency|Reuses the integration boundary for resilient processing and cross-system consistency reasoning (**AC-03**, **AC-05**).|
|Define explicit interfaces: `Edge operational API` and `Edge synchronization interface`|Makes offline and reconnect contracts explicit, secure, and idempotent (**QA-03**, **QA-04**).|

### Recorded design decisions (ADD Step 6)
Iteration 6 decisions were recorded in Section 9 of `Design/Architecture.md`, including:

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|
|QA-03, QA-04, C-06, AC-03, AC-05|Introduce a **warehouse edge node** (`Edge gateway`, `Edge operations core`) to support disconnected operations with local APIs and later synchronization to the cloud instance.|Meets continued operation for up to **3 hours** during WAN outage with controlled scope, maintaining integration resilience and cross-system consistency reasoning.|Cloud-only operation; offline implemented in each client device.|
|QA-03, QA-04, AC-05|Adopt a durable **edge operational store** with an **append-only offline operation record** and local projections (ledger and balance), enabling replay and verification during reconciliation.|Ensures **zero data loss** and deterministic reconciliation and audit across disconnect and reconnect cycles, aligned with ledger-based correctness.|Snapshot-only sync without operation journal; multi-master database replication.|
|QA-03, QA-04, AC-03, AC-05|Introduce a **Sync agent** using **resumable checkpoints** and batched submission of operation ids with deduplication and acknowledgements.|Supports predictable catch-up and prevents duplicate effects during reconnect storms via checkpointing and idempotency.|Manual export/import reconciliation; best-effort replay without checkpoints.|
|QA-04, C-06|Require **idempotency keys** for the `Edge operational API` and maintain outcomes in an **Edge idempotency store**.|Prevents duplicate local effects under intermittent automation connectivity and retried requests, preserving integrity during outages.|Exactly-once transport guarantees; retries without recorded outcomes.|
|QA-03, QA-04, AC-03|Define an explicit **Edge synchronization interface** with mutual TLS, checkpoints, batching, and dedup of operation ids.|Makes reconnect contract explicit and secure; ensures resumable and idempotent synchronization to meet “no loss, no duplicates”.|Ad-hoc edge access to cloud DB; per-operation sync without batching/backpressure.|

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|QA-03|**Partially satisfied** — Edge node + durable store + checkpointed sync provide the required backbone. Remaining: explicit **offline-safe operation scope**, backlog sizing for **3 hours**, and validation that sync can complete within **30 minutes** at peak transaction rates.|
|QA-04|**Partially satisfied** — Idempotency is extended to edge and sync using operation ids and recorded outcomes. Remaining: explicit **idempotency key domains**, ordering guarantees per stream, and partial-batch replay semantics.|
|C-06|**Satisfied** — Local automation APIs and buffering on the edge support intermittent warehouse connectivity.|
|AC-03|**Partially satisfied** — Integration boundary and explicit sync interface strengthen resilience. Remaining: sync contract governance (versioning, backpressure semantics) and interaction with existing store/finance/automation delivery workflows.|
|AC-05|**Partially satisfied** — Durable operation records and reconciliation align with ledger-based correctness. Remaining: explicit cross-system reconciliation rules (especially shipping/financial during outage) and conflict and exception detection and resolution procedures.|

