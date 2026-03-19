# Iteration 6: Availability and Resilience

## Goal

**Ensure continuous operations under various failure scenarios** - Achieve 99.9% uptime with multi-zone deployment, implement disaster recovery (RPO <15min, RTO <4hrs), and enable offline warehouse operations during connectivity loss (up to 3 hours).

## Drivers Addressed

| Driver ID | Type | Description |
|-----------|------|-------------|
| **QA-02** | Quality Attribute (Primary) | **Availability**: Given normal operating conditions across all warehouses, when any single compute node or cloud availability zone becomes unavailable, WMS core operations (receiving, picking, shipping, inventory updates, and integrations) continue to be available per WMS instance, achieving an overall uptime of 99.9%. In the event of a major regional disaster, the system shall ensure a Recovery Point Objective (RPO) of < 15 minutes and a Recovery Time Objective (RTO) of < 4 hours. |
| **QA-03** | Quality Attribute (Primary) | **Offline Operations**: Given a loss of connectivity between the warehouse and the cloud during normal warehouse operation hours, warehouse operations can continue for up to 3 hours, with zero data loss, no duplicate transactions, and full synchronization completed within 30 minutes after connectivity is restored. Inventory and financial consistency must be preserved across systems, but temporary divergence between WMS instances and external systems is acceptable during failures. |
| **QA-08** | Quality Attribute | **Tenant Isolation / Regionalization**: When a WMS instance experiences issues (load spikes, misconfiguration, failure), all other instances must continue to operate independently so that local outages do not cascade globally. |
| **AC-05** | Architectural Concern | **Data Consistency**: How to ensure consistent inventory and shipment data across WMS, store systems, and financial system in the presence of asynchronous messaging and potential failures. |

## Elements Refined

| Element | Current State | Refinement Needed | Related Driver |
|---------|---------------|-------------------|----------------|
| **Deployment Architecture** | Multi-region deployment with regional Kubernetes clusters; single availability zone implied | Refine to **multi-zone deployment** within each region; add zone-aware pod scheduling and data replication for zone failure tolerance | QA-02 |
| **WMS Database (PostgreSQL)** | Primary with async read replica for reporting; partitioned tables; PgBouncer pooling | Refine for **high availability**: synchronous standby in different AZ, automatic failover, WAL archiving for point-in-time recovery, cross-region backup for DR | QA-02 |
| **Redis Cache** | Managed Redis for L2 cache and session storage | Refine to **Redis Cluster** or **Sentinel** for HA with automatic failover; multi-AZ deployment | QA-02 |
| **Message Broker (RabbitMQ/SQS)** | Used for async messaging and integration queues | Refine for **HA cluster configuration** with mirrored queues (RabbitMQ) or use SQS with regional redundancy | QA-02 |
| **WMS Application Pods** | Stateless with HPA scaling; basic health checks | Refine with **Pod Disruption Budgets (PDBs)**, enhanced liveness/readiness probes, zone-aware scheduling (pod anti-affinity), graceful shutdown handling | QA-02, QA-08 |
| **Integration Module** | Circuit breakers, retry handlers, outbox pattern, idempotency | Refine with **bulkhead pattern**, timeout configuration, fallback strategies, and enhanced circuit breaker thresholds | QA-08, AC-05 |
| **Platform Services** | Shared services across all instances | Refine for **HA deployment** with multi-AZ, regional failover, and degraded-mode operation capability | QA-02, QA-08 |
| **Instance Isolation Model** | Namespace isolation, separate databases, network policies | Refine with **resource quotas**, **network fault injection testing**, and **failure domain boundaries** | QA-08 |

## New Elements Introduced

| New Element | Purpose | Related Drivers |
|-------------|---------|-----------------|
| **Edge Gateway Component** | Local component deployed at each warehouse to enable offline operations when cloud connectivity is lost; provides local API endpoint, local data store, and sync capabilities | QA-03 |
| **Local Data Store (Edge)** | SQLite database at the warehouse edge to store operational data during offline periods | QA-03, AC-05 |
| **Sync Service** | Component responsible for bidirectional data synchronization between Edge Gateway and cloud WMS; handles conflict resolution and ensures zero data loss | QA-03, AC-05 |
| **Conflict Resolution Engine** | Logic to detect and resolve conflicts when the same data is modified both offline and online; implements business-rule-based resolution strategies | AC-05 |
| **Disaster Recovery Configuration** | Cross-region backup strategy, recovery procedures, and automated failover runbooks | QA-02 |

## Design Concepts Selected

### Design Concepts for QA-02: High Availability

| Design Concept | Pros | Cons | Discarded Alternatives |
|---|---|---|---|
| **Multi-AZ Deployment Pattern** | Survives complete AZ failure; automatic failover; native cloud support; meets 99.9% SLA | Higher infrastructure costs; increased network latency between AZs | Single-AZ with fast recovery; Multi-Region Active-Active (too complex) |
| **PostgreSQL HA with Patroni/Cloud-Managed** | Automatic failover in <30 seconds; zero data loss (sync replication); point-in-time recovery | Sync replication adds ~2-5ms write latency; standby resources cost | Async replication only (data loss); Manual failover (slow RTO); Multi-master (too complex) |
| **Redis Sentinel with Multi-AZ Replicas** | Automatic failover; read scaling via replicas; managed options available | Async replication means potential cache data loss on failover (acceptable) | Redis Cluster mode (too complex); Single Redis instance (no HA) |
| **Message Broker HA (SQS or RabbitMQ Cluster)** | SQS: Fully managed, 99.999999999% durability. RabbitMQ: Message mirroring, automatic failover | SQS: Vendor lock-in. RabbitMQ: Operational complexity | Single RabbitMQ node (no HA); Kafka (overkill) |
| **Cross-Region Disaster Recovery (Active-Passive)** | Meets RPO <15min with async replication; RTO <4hrs with automated recovery | DR infrastructure costs; recovery testing complexity | Active-Active Multi-Region (too complex); Backup-only DR (RTO too long) |

### Design Concepts for QA-03: Offline Operations

| Design Concept | Pros | Cons | Discarded Alternatives |
|---|---|---|---|
| **Edge Gateway Pattern** | Enables full offline operation; local low-latency operations; warehouse autonomy | Additional infrastructure at warehouse; sync complexity | Thick Client Only (limited); Queued Operations Only (no queries) |
| **Embedded Database (SQLite) for Edge** | Full query capability offline; ACID transactions; proven technology | Data subset management; storage limitations | In-Memory Store (volatile); Document Store (unfamiliar) |
| **Store-and-Forward Pattern** | Zero data loss; preserves operation order; works with existing idempotency | Queue can grow large during extended offline | Immediate retry (network exhaustion); Batch upload (loss of order) |
| **Timestamp-Based Sync with Conflict Detection** | Well-understood pattern; efficient delta sync; clear conflict identification | Timestamp skew issues; conflict resolution can be complex | Vector Clocks (too complex); Event Sourcing for Sync (major change) |

### Design Concepts for QA-08: Instance Isolation

| Design Concept | Pros | Cons | Discarded Alternatives |
|---|---|---|---|
| **Bulkhead Pattern** | Prevents resource exhaustion cascades; isolates failures; predictable degradation | More complex resource configuration; potential underutilization | Shared resource pools (cascade risk); No isolation |
| **Pod Disruption Budgets (PDBs)** | Prevents accidental downtime during deployments; native Kubernetes | Only affects voluntary disruptions | No PDBs (full outage risk); Manual coordination |
| **Resource Quotas and LimitRanges** | Hard boundaries prevent noisy neighbor; predictable allocation; cost control | Requires capacity planning; may reject legitimate scaling | No quotas (resource starvation); Cluster-level limits only |
| **Timeout and Fallback Patterns** | Prevents thread blocking; bounded response times; graceful degradation | Tuning timeouts is difficult | No timeouts (resource exhaustion); Very long timeouts (poor UX) |

### Design Concepts for AC-05: Data Consistency

| Design Concept | Pros | Cons | Discarded Alternatives |
|---|---|---|---|
| **Eventual Consistency with Business-Rule Conflict Resolution** | Enables offline operation; deterministic resolution; business-aligned outcomes | Some scenarios require manual resolution | Strong consistency (impossible offline); Last-write-wins always (inappropriate) |
| **Idempotency Keys for All Write Operations** | Zero duplicate transactions; works with existing infrastructure | Storage for keys; TTL management | No idempotency (duplicates); Sequence numbers only (gaps) |
| **Operation Log with Replay** | Complete audit trail; deterministic replay; debugging capability | Storage overhead; replay can be slow | Event Sourcing full (too complex); State-only sync (no history) |
| **Conflict Detection with Notification** | Transparency into data issues; human oversight for edge cases | Manual resolution delays; notification fatigue risk | Silent resolution only (trust issues); Block on all conflicts (poor UX) |

## Instantiation Decisions

| Instantiation Decision | Rationale |
|------------------------|-----------|
| **Multi-AZ Kubernetes Deployment** with pod anti-affinity rules spreading WMS replicas across 3 availability zones; zone-aware scheduling for all stateful components | Ensures WMS instance survives complete AZ failure; automatic pod rescheduling to healthy zones; addresses QA-02 99.9% uptime requirement |
| **PostgreSQL HA with Managed Multi-AZ** (RDS Multi-AZ / Cloud SQL HA) or **Patroni cluster** with synchronous streaming replication to standby in different AZ; automatic failover in <30 seconds | Zero data loss during failover (synchronous replication); meets RPO requirement; automatic failover meets RTO requirement |
| **WAL Archiving to Cross-Region Storage** with continuous archiving to S3/GCS in DR region; 15-minute backup frequency for base backups | Enables point-in-time recovery; meets RPO <15 minutes for regional disaster; supports DR recovery procedures |
| **Redis Sentinel with Multi-AZ Replicas** (3 nodes: 1 primary, 2 replicas across AZs) or managed Redis HA (ElastiCache Multi-AZ); automatic failover | Cache remains available during AZ failure; Sentinel provides automatic failover; addresses QA-02 for cache layer |
| **Amazon SQS** as primary message broker (inherently multi-AZ) or **RabbitMQ Cluster** with mirrored queues across 3 AZ nodes | SQS provides 99.999999999% durability with no operational overhead; addresses QA-02 for messaging layer |
| **Edge Gateway Component** deployed at each warehouse on local hardware; runs containerized WMS-Edge application with local API endpoints | Enables warehouse operations during cloud connectivity loss (QA-03); supports 3-hour offline operation requirement |
| **SQLite Database for Edge** storing operational subset: active inventory, pending tasks, item master, location master, in-progress transactions | Lightweight embedded database suitable for edge deployment; full SQL capability; sufficient for 3-hour operation window |
| **Edge Operation Log** recording all write operations with timestamps, operation type, payload, and idempotency keys in local SQLite | Enables reliable sync after reconnection; preserves operation order; supports exactly-once processing during sync |
| **Sync Service Component** running on both Edge Gateway and cloud WMS; handles bidirectional delta synchronization based on timestamps | Efficient sync using only changed records; completes full sync within 30 minutes per QA-03 |
| **Conflict Resolution Engine** with configurable business rules: inventory conflicts use minimum quantity; task status uses latest timestamp; location capacity recalculates from transactions | Deterministic conflict resolution without manual intervention for most cases; business-aligned outcomes |
| **Pod Disruption Budgets (PDB)** configured as minAvailable=50% for WMS application pods | Ensures availability during deployments, upgrades, and node maintenance; works with existing HPA |
| **Resource Quotas per Namespace**: CPU limit 8 cores, memory limit 32GB, pod limit 20 per WMS instance namespace | Prevents any single instance from consuming excessive cluster resources; ensures fair resource distribution (QA-08) |
| **Bulkhead Configuration** for Integration Module: separate thread pools per external system (store: 20 threads, financial: 10 threads, automation: 30 threads) | Prevents slow external system from exhausting shared resources; failures isolated to specific integration |
| **Enhanced Timeout Configuration**: external API calls 10s, database queries 30s, cache operations 500ms | Prevents thread blocking on unresponsive services; bounded response times; consistent timeout policy |
| **Connectivity Monitor Service** on Edge Gateway detecting cloud connectivity status; triggers offline mode when heartbeat fails for 30 seconds | Fast detection of connectivity loss; automatic mode switching; enables seamless transition to offline operation |
| **Disaster Recovery Runbook** with automated scripts for database restore, DNS failover, application deployment in DR region | Documented and tested recovery procedures; automation reduces RTO; supports RTO <4 hours requirement |

## Design Decisions

| Driver | Decision | Rationale | Discarded Alternatives |
|--------|----------|-----------|------------------------|
| QA-02: High availability | Deploy **Multi-AZ Architecture** with WMS pods across 3 AZs using pod anti-affinity | Survives complete AZ failure; automatic failover; meets 99.9% SLA | Single-AZ deployment; Multi-Region Active-Active |
| QA-02: Database HA | Implement **PostgreSQL HA with synchronous streaming replication** to standby in different AZ; automatic failover in <30 seconds | Zero data loss on failover; automatic failover without manual intervention | Async replication only; Manual failover; Multi-master |
| QA-02: Disaster recovery | Implement **Cross-Region DR with WAL archiving** to S3/GCS; continuous WAL shipping every 5 minutes | Meets RPO <15 minutes; RTO <4 hours with automated recovery | Active-Active Multi-Region; Backup-only DR |
| QA-02: Cache HA | Deploy **Redis Sentinel with Multi-AZ replicas** or managed Redis HA; automatic failover in <15 seconds | Cache available during AZ failure; Sentinel provides automatic election | Single Redis instance; Redis Cluster mode |
| QA-02: Message broker HA | Use **Amazon SQS** or **RabbitMQ cluster with mirrored queues** across 3 AZ nodes | Fully managed HA; messages replicated across AZs automatically | Single RabbitMQ node; Kafka |
| QA-02: Application availability | Configure **Pod Disruption Budgets (PDB)** with minAvailable=50% | Ensures minimum availability during maintenance; Kubernetes-native | No PDBs |
| QA-03: Offline operations | Implement **Edge Gateway Pattern** with local edge server running containerized WMS-Edge application | Enables full warehouse operations during connectivity loss; 3-hour offline window | Thick client; Queue-only |
| QA-03: Edge data storage | Deploy **SQLite database on Edge Gateway** storing operational subset | Lightweight; full SQL capability; ACID transactions; ~500MB typical size | Redis at edge; Full PostgreSQL |
| QA-03: Offline recording | Implement **Operation Log pattern** with all writes recorded with idempotency keys | Enables reliable sync; preserves operation order; exactly-once semantics | No operation log; Event sourcing |
| QA-03: Data sync | Implement **Timestamp-based delta sync** with SyncService; completes within 30 minutes | Efficient delta sync; handles both directions | Full resync; Event sourcing for sync |
| QA-03: Connectivity detection | Implement **Connectivity Monitor** with heartbeat-based detection; offline mode after 30 seconds | Fast detection; automatic mode switching | Manual toggle; First-failed-request detection |
| AC-05: Data consistency | Implement **Eventual Consistency with Business-Rule Conflict Resolution** | Enables offline operation; deterministic resolution; business-aligned outcomes | Strong consistency; Last-write-wins always |
| AC-05: Conflict resolution | Define **Business Rule-based Conflict Resolution** per entity type (Inventory: MINIMUM_QUANTITY, Tasks: TERMINAL_STATE_WINS) | Deterministic; business-appropriate; conservative for inventory | Single rule for all; Always manual resolution |
| AC-05: Idempotent sync | Extend **Idempotency Keys to all offline operations** | Zero duplicate transactions; exactly-once semantics | No idempotency; Sequence numbers |
| QA-08: Bulkhead isolation | Implement **Bulkhead Pattern** with isolated resource pools per external system | Prevents resource exhaustion cascade; failures isolated | Shared thread pool; No resource limits |
| QA-08: Resource quotas | Configure **Kubernetes Resource Quotas per namespace** (CPU: 8 cores, memory: 32GB) | Hard boundaries prevent noisy neighbor; cost control | No quotas; Cluster-level limits only |
| QA-08: Network isolation | Implement **Network Policies** restricting inter-namespace traffic | Prevents network-level cascade; security boundary | Flat network |
| QA-08: Failure boundaries | Implement **Enhanced Timeout Configuration** (API: 10s, DB: 30s, cache: 500ms) | Prevents thread blocking; bounded response times | No timeouts; Very long timeouts |
| QA-08: Circuit breaker | Configure **Enhanced Circuit Breaker thresholds** per external system | Prevents cascade failures; automatic recovery | Same thresholds for all; No circuit breakers |
| QA-02: Health monitoring | Implement **Comprehensive Health Checks** with Kubernetes probes | Fast failure detection; automatic pod restart | Basic health check only; No health checks |
| QA-02, AC-05: Platform HA | Deploy **Shared Platform Services with HA** across multiple AZs | Platform available during AZ failure; degraded-mode operation | No HA for platform; Fully decentralized |

## Analysis Results

| Driver | Analysis Result | Justification |
|--------|-----------------|---------------|
| **QA-02: Availability** (99.9% uptime, RPO <15min, RTO <4hrs) | **Satisfied** | Multiple complementary decisions address availability: (1) **Multi-AZ deployment** ensures survival of complete AZ failure; (2) **PostgreSQL HA with synchronous replication** provides zero data loss for AZ failures; (3) **WAL archiving** achieves RPO <15 minutes for regional disasters; (4) **Automated DR runbooks** enable RTO <4 hours; (5) **Redis Sentinel** and **SQS/RabbitMQ HA** ensure all stateful components are highly available; (6) **PDBs** and **health checks** ensure application availability during maintenance. |
| **QA-03: Offline Operations** (3-hour capability, zero data loss, sync within 30 minutes) | **Satisfied** | Comprehensive decisions address offline operation: (1) **Edge Gateway Pattern** enables full warehouse operations during connectivity loss; (2) **SQLite database** provides persistent local storage with full SQL capability; (3) **Operation Log pattern** ensures zero data loss through local persistence with idempotency keys; (4) **Timestamp-based delta sync** achieves synchronization within 30 minutes; (5) **Connectivity Monitor** provides fast detection and automatic mode switching. |
| **QA-08: Instance Isolation** (prevent cascading failures) | **Satisfied** | Multiple isolation mechanisms prevent cascades: (1) **Bulkhead Pattern** isolates thread pools per external system; (2) **Resource Quotas** prevent any instance from starving cluster resources; (3) **Network Policies** prevent cross-instance network traffic; (4) **Enhanced timeouts** bound response times; (5) **Circuit breakers** fail fast when external systems are down. All mechanisms work together to contain failures within single instance or integration. |
| **AC-05: Data Consistency** (consistent data during failures) | **Satisfied** | Data consistency achieved through: (1) **Eventual consistency model** explicitly accepting temporary divergence; (2) **Business-rule-based conflict resolution** provides deterministic, appropriate outcomes per entity type; (3) **Idempotency keys** guarantee exactly-once processing during sync; (4) **Audit logging** of all conflict resolutions for compliance and transparency. |

## Summary

| Driver | Status |
|--------|--------|
| QA-02: Availability (99.9% uptime, RPO <15min, RTO <4hrs) | ✅ Satisfied |
| QA-03: Offline Operations (3-hour capability, zero data loss) | ✅ Satisfied |
| QA-08: Instance Isolation (prevent cascading failures) | ✅ Satisfied |
| AC-05: Data Consistency (consistent data during failures) | ✅ Satisfied |

## Expected Outcomes Verification

| Expected Outcome | Status | Evidence |
|-----------------|--------|----------|
| Multi-zone deployment architecture | ✅ Delivered | Section 5.8: Multi-AZ Deployment Architecture with HA configuration |
| Failover and recovery procedures | ✅ Delivered | Section 5.8.3-5.8.4: Database HA and Disaster Recovery Architecture |
| Offline-first architecture for warehouse edge | ✅ Delivered | Section 5.9: Edge Computing Architecture with Edge Gateway |
| Data synchronization and conflict resolution | ✅ Delivered | Section 8.6: Sync Service interfaces and Conflict Resolution Rules |
| Circuit breaker and bulkhead patterns | ✅ Delivered | Section 5.10: Instance Isolation and Resilience Patterns |

## Architecture Artifacts Modified

The following sections were added or modified in Architecture.md:

1. **Section 5.8** - High Availability Architecture
   - 5.8.1 Multi-AZ Deployment Architecture diagram
   - 5.8.2 High Availability Configuration table
   - 5.8.3 Database High Availability Detail with Patroni diagram
   - 5.8.4 Disaster Recovery Architecture diagram

2. **Section 5.9** - Edge Computing Architecture for Offline Operations
   - 5.9.1 Edge Gateway Architecture diagram
   - 5.9.2 Edge Gateway Components table
   - 5.9.3 Data Replication Strategy table
   - 5.9.4 Offline Mode State Machine diagram
   - 5.9.5 Supported Offline Operations table

3. **Section 5.10** - Instance Isolation and Resilience Patterns
   - 5.10.1 Bulkhead Pattern Implementation diagram
   - 5.10.2 Bulkhead Configuration table
   - 5.10.3 Circuit Breaker States diagram
   - 5.10.4 Enhanced Circuit Breaker Configuration table
   - 5.10.5 Kubernetes Resource Isolation (YAML examples)
   - 5.10.6 Failure Domain Boundaries table

4. **Section 7.20** - Sequence diagram: QA-02 Availability Zone Failover Scenario

5. **Section 7.21** - Sequence diagram: QA-03 Offline Operation and Sync Scenario

6. **Section 7.22** - Sequence diagram: QA-08 Instance Isolation During Cascading Failure

7. **Section 7.23** - Sequence diagram: AC-05 Conflict Resolution During Sync

8. **Section 8.5** - Edge Gateway Interfaces (ConnectivityMonitor, OperationLog, EdgeDataStore)

9. **Section 8.6** - Sync Service Interfaces (SyncService, ConflictResolutionEngine, Conflict Resolution Rules, Cloud Sync API)

10. **Section 9** - Design decisions for Iteration 6

## Conclusion

**Iteration 6 is complete.** All drivers have been satisfied and all expected outcomes have been delivered. The architecture now supports the availability and resilience requirements with:

- **99.9% availability** through multi-AZ deployment with automatic failover for all components
- **Disaster recovery** meeting RPO <15 minutes and RTO <4 hours through WAL archiving and automated recovery
- **3-hour offline warehouse operations** through the Edge Gateway pattern with local SQLite storage
- **Zero data loss** with full synchronization within 30 minutes after reconnection
- **Instance isolation** preventing cascading failures through bulkhead patterns, resource quotas, and network policies
- **Data consistency** through eventual consistency with deterministic business-rule-based conflict resolution
