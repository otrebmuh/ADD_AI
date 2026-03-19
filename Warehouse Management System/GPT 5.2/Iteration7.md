## Iteration 7 Summary (ADD)

### Goal (ADD Step 2)
**Availability, isolation, operability, and safe change**: refine deployment topology, resilience, DR, observability, and progressive delivery so instances remain isolated and support staff can operate the fleet with controlled rollouts.

### Drivers addressed (ADD Step 2)

| Driver ID | Type | Description |
|---|---|---|
| **QA-02** | Quality (Availability) | When any single compute node or cloud availability zone becomes unavailable, WMS core operations continue per instance, achieving **99.9% uptime**. In a major regional disaster, ensure **RPO < 15 minutes** and **RTO < 4 hours**. |
| **QA-08** | Quality (Tenant isolation) | Issues in one WMS instance must not impact other instances so local outages do not cascade globally. |
| **QA-09** | Quality (Operability and Observability) | Provide centralized monitoring, logs, and alerts per instance and integration to quickly identify and resolve issues across distributed deployment. |
| **QA-10** | Quality (Release management and progressive delivery) | Deploy new versions to a **selected subset of instances** while others remain on prior versions, with gradual rollout and quick rollback without cross-instance downtime or data corruption. |
| **C-01** | Constraint | Deployed in public cloud using managed services where feasible to reduce operational overhead. |
| **C-03** | Constraint | Support independent WMS instances with possible shared services such as monitoring and identity. |
| **AC-04** | Architectural concern | Support multiple WMS instances in the cloud while ensuring isolation and operational simplicity. |
| **AC-07** | Architectural concern | Provide observability that gives end-to-end visibility of order flows and integrations across instances. |

### Elements refined (ADD Step 3)
- **Instance cell as an operable unit**: `WMSInstance isolated deployment` (cell boundaries, shared vs per-instance services).
- **Entry and routing**: `API gateway` (instance routing and resilience posture).
- **Core runtime**: `WMS core application` (health/readiness semantics needed for safe operations).
- **Integration runtime**: `Integration gateway` (operability of integrations, failure isolation and recovery behaviors).
- **Stateful foundations**: `WMS operational database`, `Idempotency and dedup store`, `Outbox publisher`, `Event bus` (HA, backups, recovery, and deploy/failover behavior).
- **Fleet operations**: `Central observability` (per-instance partitioning; signals for troubleshooting and rollout gates).
- **Safe change mechanism**: progressive delivery and rollback control (modeled via `GitOps deployment controller` and related interfaces).

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Multi-AZ “cell” deployment per `WMSInstance`**|Meets **QA-02** for node/AZ failures; preserves **QA-08/C-03** isolation; improves maintenance flexibility|Higher baseline cost per instance; more operational controls needed|Single-AZ per instance; shared multi-tenant runtime for all instances|
|**Managed HA datastore per instance with PITR backups**|Targets **QA-02** availability and recoverability; aligns with **C-01** managed services; reinforces isolation|Cost scales with number of instances; requires standardized ops practices|Self-managed DB; shared multi-tenant database|
|**Region-level DR with warm-standby and async replication**|Directly targets **QA-02** RPO/RTO; preserves **QA-08** by failing over per instance; can be cost-controlled|Async replication complexity; needs routing/secrets cutover and DR drills|Active-active for all instances; backup-only DR|
|**Centralized observability with strict per-instance partitioning**|Meets **QA-09/AC-07** with fleet visibility; supports **C-03** shared services without breaking isolation|Requires governance, taxonomy, and cost controls|Per-instance observability stacks; logs-only ops|
|**Progressive delivery via GitOps with per-instance targeting**|Meets **QA-10** with controlled rollout/rollback; improves auditability and repeatability|Requires repo/process discipline; needs promotion and drift controls|Manual deployments; big-bang upgrades|
|**Canary/blue-green mechanics with automated rollback gates**|Reduces release risk (**QA-10**) using measurable signals (**QA-09**), supporting overall availability (**QA-02**)|More platform complexity; needs stable, well-tuned metrics|Rolling updates without gates; feature-flags-only|
|**Backpressure, circuit breakers, DLQ/replay controls at integration boundary**|Protects availability under partner failures (**QA-02**); improves operability (**QA-09/AC-07**); limits blast radius (**QA-08**)|Requires runbooks/tooling; risk of lag if misconfigured|Unlimited retries/no DLQ; synchronous-only integrations|
|**Cell guardrails for isolation** (quotas, network policies, per-instance keys/secrets)|Strengthens **QA-08/C-03/AC-04** isolation and reduces misconfiguration blast radius|Adds setup overhead; needs strong automation to manage fleet|Flat shared cluster/network with minimal policy|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- A new **Availability, DR, observability, and progressive delivery topology (Iteration 7)** view.
- New sequence diagrams for Iteration 7:
  - **Sequence 15**: progressive delivery rollout to a subset of instances
  - **Sequence 16**: regional DR failover for a single instance
- Interface refinements:
  - Added **Observability telemetry**
  - Added **Release management control**
  - Added **DR orchestration**

|Instantiation decision|Rationale|
|---|---|
|Instantiate multi-AZ as the default `WMSInstance` cell posture|Addresses **QA-02** for node/AZ failure while preserving per-instance blast radius (**QA-08**).|
|Instantiate per-instance database HA + PITR + cross-region replication|Targets QA-02 DR objectives while aligning with managed-service constraint (**C-01**) and maintaining isolation.|
|Instantiate progressive delivery control via GitOps with per-instance targeting|Enables safe rollout/rollback per subset of instances (**QA-10**) without cross-instance downtime.|
|Instantiate DR failover as an explicit operational flow (Sequence 16)|Makes recovery steps explicit and repeatable to meet RPO/RTO and avoid ad-hoc actions (**QA-02**).|
|Instantiate required observability dimensions and centralized aggregation|Improves MTTR and supports rollout gates with per-instance partitioning (**QA-09**, **AC-07**, **C-03**).|

### Recorded design decisions (ADD Step 6)

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|
|QA-02, QA-08, AC-04, C-03|Run each `WMSInstance` cell as a **multi-AZ deployment** with AZ-spread stateless workloads and explicit readiness and health signaling.|Survives loss of a node or an AZ without interrupting core operations (**QA-02**) while keeping failures isolated to the instance cell (**QA-08**, **AC-04**, **C-03**).|Single-AZ instance deployments; shared multi-tenant runtime spanning many instances.|
|QA-02, C-01, QA-08|Adopt a **managed HA operational database per instance** with automated backups, PITR, and **cross-region replication** to a DR region.|Leverages managed services (**C-01**) while meeting availability/DR objectives (**QA-02**) without coupling instances to a shared datastore (**QA-08**).|Self-managed database; backup-only DR without replication; shared database with tenant partitioning.|
|QA-10, QA-08, C-03|Use a **GitOps-based progressive delivery control plane** to target subsets of instances, gate promotions, and enable rapid rollback.|Enables selected-instance rollouts and quick rollback (**QA-10**) while keeping instance cells independent (**QA-08**) and sharing only the control plane (**C-03**).|Manual per-instance deployments; big-bang fleet upgrades; rolling updates without gates.|
|QA-02, QA-08|Define **per-instance DR failover procedures** (replica promotion, dependency restoration, routing switch) as an orchestrated, repeatable runbook.|Achieves predictable recovery within RTO/RPO targets (**QA-02**) while allowing failover for a single affected instance (**QA-08**).|Ad-hoc manual failover; active-active multi-region for all instances.|
|QA-09, AC-07, C-03|Standardize **end-to-end observability telemetry** with required dimensions (`instanceId`, `correlationId`, `integrationTarget`) and central aggregation with per-instance partitioning.|Improves troubleshooting and supports SLO alerting (**QA-09**, **AC-07**) while enabling shared tooling without violating isolation (**C-03**).|Logs-only troubleshooting; per-instance isolated observability stacks.|

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|**QA-02**|**Partially satisfied** — multi-AZ cell posture and DR replication/failover are defined, but explicit RPO/RTO validation assumptions and drill/automation details are not fully specified.|
|**QA-08**|**Satisfied** — the cell-based isolation model is reinforced with multi-AZ deployment and per-instance failover/rollout targeting.|
|**QA-09**|**Partially satisfied** — observability foundations (telemetry + partitioning) are defined, but specific dashboards, alerts, and SLOs are not fully detailed.|
|**QA-10**|**Partially satisfied** — progressive delivery and rollback are defined, but cross-version data/migration and compatibility strategy across mixed-version instance subsets is not fully detailed.|
|**C-01**|**Satisfied** — the design explicitly favors managed services where feasible to reduce operational overhead.|
|**C-03**|**Satisfied** — independent instances are preserved while allowing shared services (identity, observability, deployment control plane).|
|**AC-04**|**Satisfied** — multi-instance support with strong isolation and operational simplicity is strengthened via the cell model plus shared-but-partitioned platform services.|
|**AC-07**|**Partially satisfied** — observability is addressed at the architecture level, but end-to-end operational workflows and standardized troubleshooting artifacts need more refinement.|

