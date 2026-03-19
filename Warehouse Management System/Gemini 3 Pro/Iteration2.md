# ADD Iteration 2: Core Business Workflows

## 1. Iteration Goal
**Goal:** Refine the architecture to support high-priority business processes and complex domain logic.

**Drivers:**
*   **US-01 (Receiving):** Receiving inbound shipments.
*   **US-03 (Replenishment):** Store replenishment orders.
*   **US-04 (Wave Picking):** Wave/batch allocation and picking route optimization.
*   **US-05 (Picking Automation):** Integration with picking systems.
*   **US-10 (Financial Integration):** Financial integration for invoicing.
*   **AC-02 (Service Partitioning - detail):** Partition system into services/modules to allow independent scaling and evolution.

## 2. Elements Refined
The following elements (containers identified in Iteration 1) will be refined to support the core business workflows:
*   **Inbound Service:** To support US-01 (Receiving).
*   **Outbound Service:** To support US-03 (Replenishment), US-04 (Wave Picking).
*   **Inventory Service:** To support inventory updates and tracking across all flows.
*   **Integration Service:** To support US-05 (Picking Automation) and US-10 (Financial Integration).

## 3. Design Concepts Selected

| Design concept | Pros | Cons | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **Hexagonal Architecture (Ports & Adapters)** | Decouples core domain logic (Picking, Allocations) from infrastructure (DB, API). Improves testability and ease of swapping adapters (AC-02). | Increased complexity (more files, indirection) and initial learning curve. | **Layered Architecture:** Prone to dependency leakage (logic depending on DB). **Transaction Script:** Too simple for complex WMS logic. |
| **Strategy Pattern** | Allows switching algorithms (e.g., Picking Route Optimization, Put-away rules) without changing core service logic (US-04, US-02). | Identifying the right abstraction for strategies can be tricky. | **Hardcoded logic:** Rigid and hard to extend. |
| **Transactional Outbox** | Guarantees reliable event publishing (e.g., to Financial System) even if the broker is temporarily down (QA-04, US-10). | Requires a background poller or CDC (Change Data Capture) mechanism. | **Dual Write (DB + Pub):** High risk of data inconsistency if one fails. |
| **REST Adapter (Integration Service)** | Standardizes communication with various external systems (Stores, ERP) via a common internal interface (QA-06). | Implementation effort to map different external schemas to internal domain model. | **Shared Database:** Tight coupling, security risk. |

## 4. Instantiation and Design Decisions
The architecture elements have been instantiated with the following decisions, detailed in `Architecture.md` (Component and Sequence views).

| Instantiation decision | Rationale |
| :--- | :--- |
| **Structure Services as Hexagons** | **AC-02:** Services (Inbound, Outbound, Inventory) are structured with a Domain core, surrounded by an API Adapter (Driving) and Persistence/Messaging Adapters (Driven). This strictly isolates business rules. |
| **Use Strategy Pattern for Wave Algorithm** | **US-04:** The `Outbound Service` domain layer defines a `PickingStrategy` interface. Specific implementations (e.g., `ZoneBasedStrategy`, `HeuristicStrategy`) are injected, allowing the planner to select methods without code changes. |
| **Separate Inventory & Outbound Responsibilities** | **AC-02:** `Outbound` manages Orders and Waves (Process), while `Inventory` manages Stock Quantities (State). Outbound *requests* reservations from Inventory. This avoids a "God Service". |
| **Integration Service as Anti-Corruption Layer** | **QA-06:** The Integration Service translates external formats (ERP XML, Store JSON) into internal **Commands** (e.g., `ReceiveShipment`, `CreateOrder`) before they reach core services. |
| **Async Financial Updates via Event Bus** | **US-10:** Inbound/Outbound services publish `ShipmentReceived` / `OrderShipped` events. Integration Service subscribes and forwards to ERP. This decouples availability of WMS from ERP. |

## 5. Analysis
The design was analyzed against the iteration drivers:

| Driver | Result | Rationale |
| :--- | :--- | :--- |
| **US-01 (Receiving)** | Satisfied | Inbound Service domain logic handles receiving; Hexagonal model ensures validation is testable. Sequence diagrams confirm flow. |
| **US-03 (Replenishment)** | Satisfied | Outbound Service processes orders via defined API; Integration Service decouples external formats. |
| **US-04 (Wave Picking)** | Satisfied | Strategy pattern allows complex optimization logic to be plugged into the Wave process without tightly coupling. |
| **US-05 (Picking Automation)** | Satisfied | Integration Service acts as adapter; Event-driven updates allow async communication with automation. |
| **US-10 (Financial Integration)** | Satisfied | Transactional Outbox ensures financial events are never lost; decoupling via Bus prevents ERP downtime from blocking WMS. |
| **AC-02 (Service Partitioning)** | Satisfied | Hexagonal Architecture + clear Service boundaries (Inbound vs Inventory vs Outbound) ensures code is decoupled and scalable. |
