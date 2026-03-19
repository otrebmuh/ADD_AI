# ADD Iteration 5: Offline Warehouse Operations

## 1. Iteration Goal
**Goal:** Enable warehouse operations to continue during connectivity loss with the cloud.

**Drivers:**
*   **QA-03 (Availability - Offline):** Operations (receiving, picking, shipping) must continue for up to 3 hours with zero data loss during WAN outages.
*   **C-06 (Automation Constraint):** Integration with automation systems must tolerate intermittent connectivity.
*   **AC-03 (Resilience Concern):** Design for reliability and idempotency in integrations.

## 2. Elements to Refine
The following elements were refined to support offline capabilities:
*   **WMS Cell Architecture:** Split into Cloud and Edge components.
*   **Web/Mobile UI:** Enhanced for offline-first interaction.
*   **Integration Service:** Decoupled layout for local automation.

## 3. Design Concepts Selected

| Design concept | Pros | Cons | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **Hybrid Edge-Cloud Architecture** | **QA-03:** Guarantees survivability of operations ensuring compute is available locally. | Complexity of managing distributed edge nodes. | **Pure Cloud:** Fails offline requirement. **Full On-Prem:** Violates cloud-first strategy. |
| **Local Message Broker** | **C-06:** Decouples automation from cloud reachability. | Operational overhead of local middleware. | **Direct Cloud Connection:** Automation fails on network jitter. |
| **Sync Agent & Log** | **QA-03:** Robust "Store-and-Forward" mechanism ensuring data integrity (zero loss). | Synchronization logic is complex. | **Dual Write / Database Replication:** Harder to manage state conflicts. |
| **Offline-First PWA** | **QA-05:** Ensures UI availability and responsiveness. | Device storage limitations. | **Standard Web App:** Unusable offline. |

## 4. Instantiation and Design Decisions
The architecture was instantiated with a **Edge Node** deployment unit.

| Instantiation decision | Rationale |
| :--- | :--- |
| **Deploy 'Edge Node' (K3s/Docker) in Warehouse** | Run lightweight versions of Inbound/Outbound services locally to process transactions against a Local DB. |
| **Local Transaction Log** | Capture user intents (commands) locally instead of just state; allows easier conflict resolution when replaying to cloud. |
| **Sync Agent Component** | Dedicated process to handle the complex handshake, upload, download, and ACK cycle for synchronization. |

## 5. Recorded Decisions
The following decisions were added to `Architecture.md`:
*   **Hybrid Edge-Cloud Architecture**
*   **Local Message Broker (Edge)**
*   **Sync Agent & Transaction Log**
*   **Offline-First PWA**

## 6. Analysis
The design meets the iteration drivers:

| Driver | Result | Rationale |
| :--- | :--- | :--- |
| **QA-03 (Offline)** | **Satisfied** | Edge Node + Local DB allows operations to persist indefinitely (limited only by disk/local state) without Cloud. Sync Agent ensures eventual consistency. |
| **C-06 (Automation)** | **Satisfied** | Local Broker bridges the gap between local robots and the Local WMS. |
| **AC-03 (Resilience)** | **Satisfied** | Offline-First PWA and Store-and-Forward patterns provide end-to-end resilience. |
