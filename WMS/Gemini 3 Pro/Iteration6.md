# Iteration 6: Security and Observability

## 1. Goal of the Iteration

The goal of this iteration is to incorporate security controls and monitoring mechanisms across the distributed system. This involves securing user and service access and ensuring that the system is observable (logs, metrics, traces) to support operations.

## 2. Selected Drivers

The following drivers were selected for this iteration:

*   **QA-07 (Security / Compliance):** Ensure 100% of access is secured using strong authentication, authorization by role and warehouse, and encryption in transit and at rest.
*   **QA-09 (Operability / Observability):** Provide centralized monitoring, logs, and alerts per warehouse instance and integration for quick issue resolution.
*   **AC-06 (Security):** Implement security, identity, and access control across multiple warehouses and roles, including system-to-system integrations.
*   **AC-07 (Observability):** Provide observability (logging, tracing, metrics, dashboards) for end-to-end visibility of order flows and operations.

## 3. Elements to Refine

The following elements were identified for refinement:

*   **Identity Provider & API Gateway:** To handle authentication and authorization (OIDC).
*   **WMS Cell Containers:** To include instrumentation for observability.
*   **Service-to-Service Communication:** To ensure encryption (mTLS).
*   **Control Plane:** To include the observability infrastructure.

## 4. Design Concepts

The following design concepts were selected to address the drivers:

| Design concept | Description | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **OpenID Connect (OIDC) & OAuth 2.0** | Standard protocol for authentication and authorization. | Standardizes auth, enables SSO, decouples identity logic. | Complexity of flows. |
| **Service Mesh (e.g., Linkerd/Istio)** | Infrastructure layer for service-to-service communication. | Transparent mTLS (Zero Trust), traffic control, golden signal metrics. | Operational complexity. |
| **OpenTelemetry (OTel)** | Vendor-neutral standard for telemetry data (logs, metrics, traces). | Future-proof, rich ecosystem, unifies tracing context. | Additional infrastructure (collector) needed. |
| **PLG Stack (Prometheus, Loki, Grafana)** | Cloud-native observability stack. | Scalable, efficient log storage, native K8s integration. | Management of storage retention. |

## 5. Instantiation Decisions

The selected concepts were instantiated as follows:

| Instantiation decision | Rationale |
| :--- | :--- |
| **Keycloak as Identity Provider** | Used for centralized user management and issuing JWT tokens via OIDC Authorization Code flow with PKCE. |
| **Linkerd as Service Mesh** | Implements mTLS between all internal services in the WMS Cell, ensuring encryption in transit without code changes. |
| **OpenTelemetry Agents** | Added to all Service Containers (Inbound, Outbound, etc.) to auto-instrument code and send telemetry to the collector. |
| **Centralized Observability Platform** | Prometheus (metrics), Loki (logs), and Grafana (dashboards) deployed in the Control Plane to aggregate data from all cells. |

## 6. Diagrams

The **Architecture.md** document was updated with:

1.  **Updated Container Diagram:** Now includes the **Observability Platform** and **Identity Provider** in the Control Plane, and shows data flows for telemetry.
2.  **Sequence Diagram: Secure User Access:** Illustrates the OIDC Authorization Code Flow.
3.  **Sequence Diagram: Observability & Tracing:** Illustrates how OpenTelemetry traces requests across microservices.

## 7. Analysis

| Driver | Analysis result |
| :--- | :--- |
| **QA-07 (Security)** | **Satisfied.** OIDC and mTLS provide comprehensive coverage for authentication and encryption. |
| **QA-09 (Observability)** | **Satisfied.** The PLG stack + OpenTelemetry provides the required visibility into system health and distributed transactions. |
| **AC-06 (Security)** | **Satisfied.** Centralized identity management scales across multiple warehouses and roles. |
| **AC-07 (Observability)** | **Satisfied.** The solution provides end-to-end visibility required for operations. |
