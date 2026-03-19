# ADD Iteration 3: Scalability and Cloud Availability

## 1. Iteration Goal
**Goal:** Refine the architecture to handle peak loads (10x volume) and ensure high availability against cloud failures.

**Drivers:**
*   **QA-01 (Scalability):** Handle peak season order volume (10,000 orders/hour) and 10x normal load with < 2 seconds response time for 95% of requests.
*   **QA-02 (Availability):** Ensure 99.9% uptime allowing core operations to continue even if a single zone fails. RPO < 15 min, RTO < 4 hours.
*   **AC-01 (Cloud-native Scalability):** Design the system to be cloud-native and scalable to handle peaks without excessive infrastructure cost.

## 2. Elements to Refine
The following elements from the "WMS Cell" container will be refined to address Scalability and Availability:
*   **Infrastructure / Deployment Model:** Definition of how the Cell is deployed across Availability Zones (QA-02).
*   **Cell Database:** Refinement of the storage strategy to handle high throughput (QA-01) and failover (QA-02).
*   **Outbound Service:** Refinement to support horizontal scaling for high-volume order processing (QA-01).
*   **Inventory Service:** Refinement to handle high concurrency/contention on stock records (QA-01).
*   **Event Bus:** Refinement to ensure message durability and throughput (QA-01, QA-02).

## 3. Design Concepts Selected

| Design concept | Pros | Cons | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **Multi-AZ Deployment (K8s Cluster)** | **QA-02:** Ensures system survives failure of a single Availability Zone. **AC-01:** Standard cloud-native pattern using managed Kubernetes (EKS/AKS/GKE). | Cross-zone data transfer costs and slight latency increase for synchronous replication. | **Active-Passive DR:** Slower RTO; resources sit idle. **Single Zone:** Does not meet 99.9% availability safely. |
| **Horizontal Pod Autoscaling (HPA)** | **QA-01:** Automatically scales service replicas based on CPU/Memory/Custom metrics (e.g., queue depth) to handle peaks. | Application must be stateless. Lag in scaling up (requires "warm" start or over-provisioning). | **Vertical Scaling:** Requires downtime or restart; has upper limits. |
| **Database Read Replicas** | **QA-01:** Offloads read-heavy operations (Dashboards, Reporting, Status Checks) from the primary writer node. | Replication lag (Eventual consistency) for readers. | **Database Sharding:** High complexity; single-instance throughput (approx 3-30 ops/sec) doesn't justify sharding yet. |
| **Optimistic Locking** | **QA-01:** Handles concurrent updates to Inventory without long duration database locks, improving throughput. | Transactions fail on collision (requires retry logic); poor performance if contention is extremely high on the *same* record. | **Pessimistic Locking:** severe throughput bottleneck. **Redis as SoT:** Complexity of persistence and consistency guarantees (QA-04). |
| **Compensating Transactions (Sagas)** | **QA-01, QA-04:** Allows long-running processes (e.g., Wave allocation) without holding DB transactions open. | High complexity in implementation and reasoning about state. | **Two-Phase Commit (2PC):** Reducing availability and performance; typically avoided in cloud-native. |

## 4. Instantiation and Design Decisions
The architecture elements have been instantiated with the following decisions, detailed in `Architecture.md` (Deployment and Sequence views).

| Instantiation decision | Rationale |
| :--- | :--- |
| **Deploy Cell on K8s across 3 AZs** | **QA-02:** Ensures high availability. Pods are distributed across 3 Availability Zones. A Load Balancer distributes traffic. If one zone fails, others take over. |
| **Configure HPA for Core Services** | **QA-01:** Inbound, Outbound, and Inventory services will have Horizontal Pod Autoscalers configured to scale replicas (min 2, max 20) based on CPU usage (>70%). |
| **Use PostgreSQL Read Replicas** | **QA-01:** Database will have 1 Primary (RW) and 2+ Read Replicas (RO). Reporting and "Get/List" API calls are routed to Replicas to offload the Primary. |
| **Implement Optimistic Locking on Inventory** | **QA-01:** `Inventory` table will use a `@Version` column. Updates MUST check version. Service layer implements a retry mechanism (max 3 retries) for `OptimisticLockException`. |

## 5. Recorded Decisions
The following decisions have been formally recorded in `Architecture.md`.

| Driver | Decision | Rationale | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **QA-02** | **Multi-AZ Deployment**<br>Deploy each WMS Cell across 3 Availability Zones with a Load Balancer. | Ensures survival of a complete zone failure. Core services and DB remain available (RPO<15m, RTO<4h). | **Active-Passive / DR Site:** Higher RTO (manual switchover) and wasted resources (idle). |
| **QA-01** | **Horizontal Pod Autoscaling (HPA)**<br>Auto-scale Inbound, Outbound, Inventory pods based on CPU/Metrics. | Allows handling 10x peak load without paying for peak capacity 24/7. | **Static Provisioning:** Cost prohibitive for peak loads. **Vertical Scaling:** downtime required to resize. |
| **QA-01** | **Read Replicas**<br>Direct read-heavy traffic (Reports, Dashboards) to DB synchronous/async replicas. | Prevents reporting queries from locking the primary DB and slowing down core transactions (Receiving/Shipping). | **Sharding:** Too complex for current scale. |
| **QA-01** | **Optimistic Locking**<br>Use version-checking for Inventory updates. | drastically improves throughput compared to DB row locks (Pessimistic). Most inventory updates (different items) don't collide. | **Pessimistic Locking:** High contention risk (deadlocks, timeouts). **Redis:** Persistence risk (QA-04). |

## 6. Analysis
The design was analyzed against the iteration drivers:

| Driver | Result | Rationale |
| :--- | :--- | :--- |
| **QA-01 (Scalability)** | Satisfied | **HPA** allows handling 10k orders/hour by adding pods. **Read Replicas** ensure dashboards don't slow down ingestion. **Optimistic Locking** prevents DB contention during high-volume inventory updates. |
| **QA-02 (Availability)** | Satisfied | **Multi-AZ K8s** + **RDS Multi-AZ** ensures survival of a single zone failure (99.9% uptime). RPO is near-zero (sync replication) or < 5s (async). RTO is usually automatic (< 5 mins). |
| **AC-01 (Cloud-Native)** | Satisfied | Usage of Managed K8s (EKS/AKS), RDS, and Auto-scaling aligns with cloud-native cost optimization (pay for what you use). |



