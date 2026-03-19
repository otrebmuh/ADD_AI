# ADD Iteration 1: Initial System Structure

## 1. Iteration Goal
**Goal:** Define the top-level system structure, deployment model, and service decomposition to support core functionality and multi-instance requirements.

**Drivers:**
*   **C-01 (Cloud Deployment):** Public cloud environment.
*   **C-03 (Independent Instances):** Independent WMS instances with some shared services.
*   **QA-06 (Integration):** Decoupled integration.
*   **QA-08 (Tenant Isolation):** Independent operation of instances.
*   **AC-02 (Service Partitioning):** Partitioning into services/modules.
*   **AC-04 (Multi-instance Support):** Supporting multiple WMS instances.

## 2. Elements Refined
*   **WMS System** (Greenfield development - entire system context).

## 3. Design Concepts Selected

| Design concept | Pros | Cons | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **Cell-Based Architecture** | High isolation (QA-08), reduced blast radius, simple scaling unit. | Ops complexity of managing many cells. | **Multi-tenant Monolith:** High risk of noisy neighbors and cascading failures. |
| **Control Plane** | centralized lifecycle management for decoupled cells (AC-04). | Requires building a dedicated management service. | **Manual/Scripted Ops:** Doesn't scale. |
| **Microservices** | Independent scaling (AC-02), fault isolation. | Distributed system complexity. | **Monolith:** Doesn't satisfy scaling/team drivers. |
| **API Gateway** | Single entry point, security, routing. | Additional hop. | **Direct Access:** Security nightmare. |
| **Event Bus** | Decoupling (QA-06), reliability via async processing. | Eventual consistency complexity. | **Synchronous-only:** Tight coupling. |

## 4. Instantiation and Design Decisions
The architecture was instantiated into **Context** and **Container** views (added to `Architecture.md`).

**Key Decisions:**

| Driver | Decision | Rationale |
| :--- | :--- | :--- |
| **C-03, QA-08** | **Cell-Based Architecture** | Ensures strict isolation (data/faults) and supports independent scaling of instances. |
| **AC-04** | **Control Plane** | Vital for automating the lifecycle (provisioning, updates) of multiple decentralized cells. |
| **AC-02** | **Microservices** | Decouples high-throughput areas (Outbound) from complex ones (Inventory) and allows independent scaling. |
| **QA-06** | **Integration Service** | Isolates core logic from external system protocols and changes. |
| **AC-03** | **Event Bus** | Enables asynchronous, reliable communication and eventual consistency between services. |

## 5. Analysis
The design was analyzed against the iteration drivers:

| Driver | Result | Rationale |
| :--- | :--- | :--- |
| **C-01** | Satisfied | Cloud-native design (Containers/K8s compatible). |
| **C-03** | Satisfied | Cells leverage cloud isolation. |
| **QA-06** | Satisfied | Integration Service + Message Bus ensures decoupling. |
| **QA-08** | Satisfied | Cells prevent cascading failures. |
| **AC-02** | Satisfied | Services partitioned by domain. |
| **AC-04** | Satisfied | Control Plane handles multi-instance complexity. |
