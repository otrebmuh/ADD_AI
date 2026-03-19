# ADD Iteration 4: Data Integrity and Reliability

## 1. Iteration Goal
**Goal:** Design mechanisms for data consistency, idempotency, and reliable integration with external systems.

**Drivers:**
*   **QA-04 (Reliability / Data Integrity):** Ensure data consistency and integrity, particularly across distributed services and external integrations.
*   **AC-03 (Integration Reliability):** Reliable integration with external systems (ERP, Stores) handling failures gracefully.
*   **AC-05 (Data Consistency):** Maintain consistency across service boundaries, potentially using eventual consistency patterns where appropriate.

## 2. Elements to Refine
The following elements will be refined to address Data Integrity and Reliability:
*   **Integration Service:** Refinement of how it handles incoming/outgoing messages and failures (QA-04, AC-03).
*   **Event Bus / Messaging:** Refinement of message delivery guarantees (AC-05).
*   **Core Services (Inbound, Outbound, Inventory):** Refinement of how they emit events and handle distributed transactions (QA-04).

## 3. Design Concepts Selected

| Design concept | Pros | Cons | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **Transactional Outbox Pattern** | **QA-04, AC-05:** Guarantees "at-least-once" delivery of events. Ensures atomic update of local DB state and event emission. | Adds complexity (Outbox table, Relay process). | **Dual Write:** High risk of inconsistencies on failure. **Two-Phase Commit:** Reduces availability and performance; typically avoided in microservices. |
| **Idempotency Keys** | **QA-04:** Prevents duplicate processing of side-effect operations (e.g., creating orders) during retries/network glitches. | Client must generate and send unique keys. | **Stateful De-duplication:** Harder to scale horizontally. |
| **Circuit Breaker** | **AC-03:** Prevents the WMS from being overwhelmed by slow/down external systems (ERP). Allows "fail fast". | Requires tuning of timeouts and thresholds. | **Unbounded Retries:** Can cause cascading failures and resource exhaustion. |
| **Dead Letter Queue (DLQ)** | **AC-03:** Captures "poison pill" messages or persistent failures for manual inspection, preventing queue blockage. | Requires operational process to review and replay DLQs. | **Drop Message:** Data loss. **Infinite Loop:** Blocks the queue. |

## 4. Instantiation and Design Decisions
The architecture elements have been instantiated with the following decisions, detailed in `Architecture.md` (Sequence views).

| Instantiation decision | Rationale |
| :--- | :--- |
| **Add 'Outbox' Table to Cell Database** | **QA-04:** Each core service database (Inbound, Outbound, Inventory) will have an `outbox_events` table. Services insert events into this table in the SAME transaction as business data change. |
| **Implement Outbox Relay in Core Services** | **QA-04:** A background thread (or separate process like Debezium, though simple polling thread is chosen for simplicity initially) will poll the `outbox_events` table and publish to the Event Bus. |
| **Require Idempotency-Key Header** | **QA-04:** The API Gateway and Integration Service will enforce the presence of `Idempotency-Key` header for non-safe methods (POST, PUT) from external clients. |
| **Idempotency Log Table** | **QA-04:** Integration Service will store processed keys and their responses in an `idempotency_log` table to return cached responses for duplicates. |
| **Configure Circuit Breakers on Integration Service** | **AC-03:** Use a library (e.g., Resilience4j) to wrap calls to external ERP/Store APIs. Open circuit after 50% failure rate; wait 30s before half-open. |

## 5. Recorded Decisions
The following decisions have been formally recorded in `Architecture.md`.

| Driver | Decision | Rationale | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **QA-04, AC-05** | **Transactional Outbox Pattern**<br>Write events to outbox table locally; relay asynchronously. | Guarantees consistency between local state and emitted events. | **Dual Write:** Unsafe. **2PC:** Too slow/blocking. |
| **QA-04** | **Idempotency Keys**<br>Enforce unique keys on mutating APIs; cache results. | Ensures safety during retries (at-least-once delivery results in exactly-once processing). | **Stateful/Session-based:** Hard to scale. |
| **AC-03** | **Circuit Breaker**<br>Fail fast when external dependencies are down. | Protects system resources from exhaustion. | **Infinite Retry:** Dangers of cascading failure. |
| **AC-03** | **Dead Letter Queue (DLQ)**<br>Route permanently failed messages to DLQ. | Prevents queue blockage by poison messages. | **Drop Message:** Data loss. |

## 6. Analysis
The design was analyzed against the iteration drivers:

| Driver | Result | Rationale |
| :--- | :--- | :--- |
| **QA-04 (Reliability)** | Satisfied | **Transactional Outbox** ensures no events are lost (At-least-once). **Idempotency Keys** ensure duplicates are handled safely (Exactly-once semantics). |
| **AC-03 (Integration)** | Satisfied | **Circuit Breakers** prevent WMS from hanging when ERP is down. **DLQs** ensure we don't lose failed messages while keeping queues moving. |
| **AC-05 (Data Consistency)** | Satisfied | **Outbox** guarantees eventual consistency between services. |
