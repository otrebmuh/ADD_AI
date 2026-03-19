## Iteration 8 Summary (ADD)

### Goal (ADD Step 2)
**Security, governance, and inventory control operations**: design cross-cutting security and compliance controls (RBAC, encryption, audit), plus counting/reconciliation/approvals to preserve inventory accuracy and support audits.

### Drivers addressed (ADD Step 2)
- **User stories**: US-11, US-18, US-13, US-19
- **Quality attributes**: QA-07, QA-04
- **Constraints**: C-02, C-07
- **Architectural concerns**: AC-06, AC-08

### Elements refined (ADD Step 3)
- **Access control**: `Authorization module` (instance and warehouse scoped policy checks)
- **Auditability**: `Audit and traceability module` (immutable `AuditEvent` + linkage to inventory postings)
- **Governed configuration**: `Configuration module` (warehouse-specific config and role changes with governance)
- **Inventory control operations**: `Inventory module` (counting, reconciliation, adjustments, reason codes, approvals, status governance)
- **Ingress enforcement**: `API gateway` (token validation, coarse authorization gates, correlation propagation)
- **System integrations governance**: `Integration gateway` (system-to-system authn posture, durable `IntegrationMessage` trace)
- **Identity lifecycle**: `Identity provider` (shared platform service)
- **Security controls at rest**: `WMS operational database` (encrypted immutable audit trails)

### Design concepts selected (ADD Step 4)

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
|**Centralized IdP using OIDC/OAuth2 with short-lived access tokens** (claims carry instance and role scopes)|Strong standardized AuthN; supports MFA/SSO; reduces custom security surface; aligns with shared-service constraint C-03 pattern|Token validation/rotation complexity; availability dependency mitigated via caching/validation strategy|Custom per-instance identity stores; long-lived API keys only|
|**Policy-based authorization** (RBAC + scoped claims; optional ABAC as needed) in `Authorization module`|Consistent governance; least privilege; supports instance and warehouse scoping; improves maintainability|Policy design/test burden; risk of policy sprawl without governance|Hard-coded checks scattered across modules; coarse RBAC without scopes|
|**API gateway security enforcement** (TLS, token validation, coarse gates, rate limits) plus **service-to-service mTLS**|Defense-in-depth; consistent ingress controls; reduces blast radius of app bugs|Certificate lifecycle overhead; gateway misconfig can impact availability|App-only enforcement with pass-through gateway; network-only trust without mTLS|
|**Encryption at rest via managed KMS** (per-instance key policies) + encryption in transit|Meets corporate security expectations; auditable key usage and rotation; improves isolation posture|Key policy complexity; adds operational discipline|Application-managed keys; single shared key for all instances|
|**Tamper-evident audit trail**: immutable `AuditEvent` + append-only `InventoryLedgerEntry` (optionally hash-chained per instance)|Strong traceability for US-18; supports forensic investigations and replay/verification; aligns with QA-04|Storage/query overhead; hash-chain complicates backfills/ordering constraints|Mutable “audit columns” only; logs-only audit trail|
|**Transaction + Outbox pattern** for audit and integration events|Avoids dual-write gaps; ensures durable event/audit publication; supports replay/recovery|Requires outbox plumbing and monitoring|Best-effort event publishing; distributed transactions across DB and bus|
|**Inventory control workflow** for counts/reconciliation/adjustments with **approval workflow** and reason codes|Dual control for sensitive adjustments; consistent governance; supports audit readiness and reporting|Extra workflow complexity; approval latency can slow operations|Direct adjustments without approvals; email/manual approvals outside system|
|**Configuration-as-data with versioning + change audit** (optionally promoted via GitOps gates)|Prevents “snowflake warehouses”; supports safe rollout/rollback; governance-ready traceability|Schema/version migrations needed; drift risk if bypassed|Divergent code branches per warehouse; unversioned ad-hoc config|
|**Compliance-ready localization model**: `WMSInstance` holds locale/currency/time zone; enforce locale-aware behavior at boundaries|Supports multi-country operations and reduces downstream errors|Test matrix expands; requires disciplined boundary validation|Hard-coded per-UI/service formats; storing localized strings/numbers as authoritative|
|**Recall/returns control via inventory status governance** (block/quarantine + eligibility rules)|Prevents allocation/shipment of non-eligible stock by invariant; auditable status transitions|Operational friction if misused; needs clear exception handling|Ad-hoc “do not ship” flags outside inventory core; outbound-only checks|

### Instantiation outcomes (ADD Step 5)
`Design/Architecture.md` was updated to include:
- `Key management service` in the container view and explicit encryption-at-rest key usage for per-instance stores
- Refined container responsibilities for `API gateway`, `WMS core application`, `Integration gateway`, and `WMS operational database` to reflect security/governance requirements
- New sequence diagrams for Iteration 8:
  - Sequence 17: cycle count → discrepancy → adjustment → optional approval → ledger posting + outbox
  - Sequence 18: configuration/role changes with authorization, versioning, audit, and outbox
  - Sequence 19: recall blocking via inventory status governance preventing allocation

|Instantiation decision|Rationale|
|---|---|
|Add `Key management service` and link it to per-instance data stores for encryption at rest|Addresses **QA-07 / C-07** and supports auditable key controls while preserving per-instance isolation boundaries.|
|Upgrade `API gateway` responsibilities for token validation, coarse gates, and correlation propagation|Enforces consistent ingress security and enables end-to-end traceability for governed inventory operations (**QA-07**, **US-18**).|
|Make policy-based authorization explicit at command boundaries in `WMS core application`|Ensures sensitive workflows (counts/adjustments/approvals/config changes) are consistently controlled and auditable (**US-11**, **US-13**, **US-18**).|
|Strengthen `Integration gateway` to include system-to-system authn posture and durable integration tracing|Improves governance and traceability of integrations while keeping domain logic decoupled (**US-18**, **AC-06**).|
|Add explicit sequences for governed counting/approval, config/role governance, and recall blocking|Defines collaboration/transaction boundaries and makes governance controls concrete for the iteration goal (**US-11**, **US-13**, **US-19**, **QA-04**, **QA-07**).|

### Recorded design decisions (ADD Step 6)

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|
|QA-07, C-07, AC-06|Centralize authentication in a shared IdP using OIDC/OAuth2 with short-lived tokens, and enforce **instance-scoped authorization** (role and warehouse scope) in the `Authorization module` and at the `API gateway` boundary.|Provides strong AuthN while ensuring access decisions remain within the instance isolation boundary; supports least-privilege access and reduces cross-warehouse access risk.|Per-instance identity stores; long-lived API keys only.|
|QA-07, C-07|Adopt **defense-in-depth** ingress security: TLS everywhere, token validation at `API gateway`, coarse authorization gates, and propagation of correlation identifiers for traceability.|Reduces attack surface and ensures correlation for audit/trace of sensitive inventory operations (**US-18**).|Application-only enforcement with a pass-through gateway.|
|QA-07, C-07|Encrypt sensitive operational data at rest using a managed **key management service** with auditable key use and rotation, applied to per-instance `WMS operational database` and `Idempotency and dedup store`.|Meets corporate security expectations and preserves per-instance isolation via scoped key policies.|Single shared encryption key; application-managed keys.|
|US-18, QA-04, QA-07|Strengthen auditability by recording all inventory-affecting operations as immutable `InventoryLedgerEntry` plus `AuditEvent`, and ensure audit/event publication uses the transactional outbox to avoid gaps.|Creates replayable audit evidence and avoids “state changed but missing audit/event” failure modes.|Mutable audit columns only; centralized logs only.|
|US-11, US-18, QA-04|Model inventory counting and reconciliation as explicit workflow state: `InventoryCount`, `InventoryAdjustment`, `ReasonCode`, and `Approval`, with posting gated by authorization and approval rules and recorded in the ledger.|Supports governed adjustments (dual control) with standardized reason codes and auditable postings, preserving integrity under retries/failures.|Direct adjustments without approvals; manual approvals outside the system.|
|US-13, AC-08, QA-07|Treat warehouse configuration and role changes as **configuration-as-data** with versioning and immutable audit events; optionally publish config-change events for downstream consumers.|Avoids warehouse-specific code divergence while enabling controlled change and auditability.|Divergent code branches per warehouse; ad-hoc config edits without versioning/audit.|
|US-19, QA-04, QA-07|Enforce recall and returns controls via **inventory status governance** (block/quarantine) and eligibility checks during allocation and execution, with all status changes ledgered and audited.|Prevents allocation/shipment of non-eligible stock by making status a hard constraint at the inventory boundary; provides compliance evidence.|Outbound-only checks; informal “do not ship” flags outside inventory core.|

### Analysis of iteration goal achievement (ADD Step 7)

|Driver|Analysis result|
|---|---|
|US-11|**Satisfied** — Governed cycle count reconciliation with optional approvals is explicitly modeled and sequenced with ledger/audit/outbox.|
|US-18|**Satisfied** — Traceability is supported via immutable audit events, ledger postings, and correlation identifiers across flows.|
|US-13|**Satisfied** — Configuration and role changes are governed via versioned configuration-as-data with an audit trail and optional events.|
|US-19|**Satisfied** — Recall blocking is enforced via inventory status governance and allocation eligibility checks.|
|QA-07|**Partially satisfied** — Security posture is defined (AuthN/AuthZ, encryption, governance), but needs more detailed claim model, service-to-service auth specifics, key scoping/rotation policies, and access logging requirements.|
|QA-04|**Satisfied** — Idempotency/outbox and immutable audit/ledger patterns preserve integrity under retries and failures for the covered flows.|
|C-02|**Not satisfied** — Multi-country compliance/i18n/l10n instantiation details (regulatory fields and locale-aware boundary validation/formatting) are not yet captured.|
|C-07|**Satisfied** — Encryption at rest via managed KMS and RBAC-based authorization approach align with corporate security/data protection expectations.|
|AC-06|**Partially satisfied** — IdP + authorization + gateway controls are defined, but cross-system identity mapping (service accounts/scopes/trust boundaries) needs more explicit treatment.|
|AC-08|**Satisfied** — Configuration-as-data with versioning/audit supports warehouse-specific tailoring without code divergence.|

