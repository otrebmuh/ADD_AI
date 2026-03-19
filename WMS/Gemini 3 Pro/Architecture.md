## WMS architecture document

### 1.- Introduction
This document describes the software architecture for the Warehouse Management System (WMS). The WMS is a cloud-native, distributed system designed to manage warehouse operations for a major retail company operating across Canada, the US, and Mexico. The system supports core warehouse functions (receiving, picking, shipping), integrates with external enterprise systems (ERP, Store Systems) and in-warehouse automation, and is designed for high scalability, availability, and multi-region deployment.


### 2.- Context diagram
The following C4 Context diagram illustrates the WMS and its interactions with external users and systems.

```mermaid
C4Context
    title System Context Diagram for Warehouse Management System (WMS)

    Person(warehouse_staff, "Warehouse Staff", "Receiving, Picking, Shipping operators, and Managers.")
    
    System(wms, "Warehouse Management System (WMS)", "Manages warehouse operations including inventory, inbound/outbound flows, and automation.")

    System_Ext(erp, "Corporate Financial System (ERP)", "Handles invoicing and financial reporting.")
    System_Ext(store_system, "Store System", "Manages store inventory and generates replenishment orders.")
    System_Ext(automation_system, "Warehouse Automation System", "Robotic picking, conveyors, and AS/RS systems.")

    Rel(warehouse_staff, wms, "Uses", "HTTPS/Mobile App")
    Rel(store_system, wms, "Sends Replenishment Orders", "HTTPS/API")
    Rel(wms, store_system, "Sends Shipment Confirmations", "HTTPS/API")
    Rel(wms, erp, "Sends Financial Events", "Async Messaging")
    Rel(wms, automation_system, "Sends Picking Tasks", "MQTT/API")
    Rel(automation_system, wms, "Sends Task Confirmations", "MQTT/API")
```

### 3.- Architectural drivers
### 3.1 Primary User Stories
The following are the business-critical user stories that drive the core functionality and architecture of the WMS.

| User Story ID | Description | Priority |
| :---- | :---- | :---- |
| **US-01** | Receiving inbound shipments to capture inventory. | **Business Critical** |
| **US-02** | Put-away of goods into storage locations with optimization. | **Business Critical** |
| **US-03** | Processing replenishment orders from stores (high volume). | **Business Critical** |
| **US-04** | Wave planning and picking route optimization. | **Business Critical** |
| **US-05** | Integration with picking automation systems. | **Business Critical** |
| **US-09** | Shipment confirmations to update store inventory. | **Business Critical** |
| **US-10** | Financial integration for invoicing. | **Business Critical** |
| **US-11** | Cycle counts and inventory reconciliation. | **Business Critical** |

### 3.2 Quality Attribute Scenarios
The following quality attributes are the primary drivers for the system's non-functional requirements.

| ID | Quality Attribute | Scenario Summary | Priority |
| :---- | :---- | :---- | :---- |
| **QA-01** | **Scalability** | Handle 10x peak load (10k orders/hr) with <2s response time. | **Primary** |
| **QA-02** | **Availability** | 99.9% uptime, survive AZ failure, RPO <15m, RTO <4h. | **Primary** |
| **QA-03** | **Offline Availability** | Operate for 3 hours without cloud connectivity; zero data loss; auto-sync. | **Primary** |
| **QA-04** | **Data Integrity** | Exactly-once processing / idempotency for all integrations. | **Primary** |
| **QA-06** | Integration | Decoupled integration with stable APIs/events. | High |
| **QA-07** | Security | 100% secured access, encryption at rest/transit. | High |
| **QA-08** | Isolation | Fault isolation between WMS instances (Cell-based). | High |
| **QA-09** | Observability | Centralized monitoring and alerting per instance. | Medium-High |

### 3.3 System Constraints

| ID | Constraint |
| :---- | :---- |
| **C-01** | Must be deployed in public cloud using managed services. |
| **C-02** | Must support multi-country operations (legal, localization). |
| **C-03** | Independent WMS instances per warehouse. |
| **C-06** | Integration with automation must handle intermittent connectivity. |

### 3.4 Architectural Concerns

| ID | Concern |
| :---- | :---- |
| **AC-01** | Designing for cost-effective cloud scalability. |
| **AC-02** | Service partitioning for independent scaling (microservices). |
| **AC-03** | Reliable, idempotent integration patterns. |
| **AC-04** | Managing multiple independent WMS instances (Control Plane). |
| **AC-05** | Data consistency across distributed systems (Inventory). |
| **AC-07** | End-to-end observability across distributed components. |

### 4.- Domain model
The following is the domain model for the Warehouse Management System (WMS), derived from the Architectural Drivers and System Requirements.

#### Class Diagram

```mermaid
classDiagram
    direction TB

    class RetailCompany {
        +name: String
    }

    class Region {
        +name: String
        +countryCode: String
    }

    class Warehouse {
        +id: String
        +name: String
        +address: Address
        +type: String
    }

    class Store {
        +id: String
        +name: String
        +address: Address
    }

    class Product {
        +sku: String
        +description: String
        +unitOfMeasure: String
        +dimensions: Dimensions
    }

    class Inventory {
        +quantity: Integer
        +status: InventoryStatus
        +lastUpdated: DateTime
    }

    class Location {
        +id: String
        +code: String
        +type: LocationType
        +zone: String
        +capacity: Integer
    }

    class InboundShipment {
        +id: String
        +source: String
        +status: InboundStatus
        +expectedArrival: DateTime
    }

    class InboundLine {
        +quantityExpected: Integer
        +quantityReceived: Integer
    }

    class ReplenishmentOrder {
        +id: String
        +orderDate: DateTime
        +priority: Priority
        +status: OrderStatus
    }

    class OrderLine {
        +quantityRequested: Integer
    }

    class Wave {
        +id: String
        +creationDate: DateTime
        +status: WaveStatus
    }

    class PickTask {
        +id: String
        +status: TaskStatus
        +quantity: Integer
        +priority: Integer
    }

    class Container {
        +id: String
        +type: ContainerType
        +trackingNumber: String
    }

    class OutboundShipment {
        +id: String
        +shipDate: DateTime
        +status: OutboundStatus
        +carrier: String
    }

    class User {
        +id: String
        +username: String
        +role: UserRole
    }

    class Adjustment {
        +id: String
        +reasonCode: String
        +quantityDelta: Integer
        +date: DateTime
    }

    %% Relationships
    RetailCompany "1" -- "*" Region
    Region "1" -- "*" Warehouse
    Region "1" -- "*" Store
    
    Warehouse "1" -- "*" Location : contains
    Warehouse "1" -- "*" Inventory : manages
    
    Location "1" -- "*" Inventory : holds
    Product "1" -- "*" Inventory : categorized_by
    
    Store "1" -- "*" ReplenishmentOrder : places
    ReplenishmentOrder "1" -- "*" OrderLine
    OrderLine "*" -- "1" Product : requests
    
    InboundShipment "1" -- "*" InboundLine
    InboundLine "*" -- "1" Product : contains
    Warehouse "1" -- "*" InboundShipment : receives
    
    Wave "1" -- "*" ReplenishmentOrder : processes
    Wave "1" -- "*" PickTask : generates
    
    PickTask "*" -- "1" Location : picks_from
    PickTask "*" -- "1" Product : picks_item
    PickTask "*" -- "1" Container : packs_into
    
    Container "*" -- "1" OutboundShipment : loaded_onto
    OutboundShipment "1" -- "*" ReplenishmentOrder : fulfills
    Warehouse "1" -- "*" OutboundShipment : ships
    
    User "1" -- "*" PickTask : executes
    User "1" -- "*" Adjustment : performs
    Adjustment "*" -- "1" Inventory : updates
```

#### Domain Elements Description

| Element | Description |
| :--- | :--- |
| **RetailCompany** | The top-level entity representing the organization operating the WMS. |
| **Region** | A geographic operational area (e.g., Canada, US, Mexico) that may contain multiple warehouses and stores. |
| **Warehouse** | A physical facility where goods are received, stored, and shipped. It is the core operational unit of the WMS. |
| **Store** | A retail location that orders products from the warehouse via Replenishment Orders. |
| **Product (SKU)** | A specific item type stored in the warehouse, characterized by a SKU (Stock Keeping Unit), description, and physical attributes. |
| **Inventory** | Represents the quantity of a specific Product held in a specific Location. It creates the link between items and space. |
| **Location** | A defined physical space within a warehouse (e.g., a bin, rack, staging lane, or dock door) identified by a code. |
| **InboundShipment** | A collection of goods arriving at the warehouse from a supplier or as a return. |
| **InboundLine** | A detail line within an Inbound Shipment specifying the expected and received quantity of a Product. |
| **ReplenishmentOrder** | A request from a Store or internal system for a specific quantity of products to be shipped. |
| **OrderLine** | A detail line within a Replenishment Order specifying the product and quantity requested. |
| **Wave** | A logical grouping of orders formed to optimize picking operations (e.g., by route, zone, or carrier). |
| **PickTask** | A specific instruction for an operator or system to move a quantity of product from a storage location to a destination (e.g., packing container). |
| **Container** | A physical unit (e.g., carton, pallet, tote) used to pack items for storage or shipment. Can be identified by an LPN (License Plate Number). |
| **OutboundShipment** | A collection of containers/goods leaving the warehouse destined for a store or other location. |
| **User** | An operator, supervisor, or manager who interacts with the system to perform tasks or view data. |
| **Adjustment** | A record of a change to inventory quantity outside of standard receiving/shipping flows (e.g., cycle count corrections, damage tracking). |


### 5.- Container diagram
The following C4 Container diagram details the high-level containers of the WMS, reflecting the **Cell-Based Architecture**. Each "WMS Cell" is an independent deployment unit.

```mermaid
C4Container
    title Container Diagram for WMS (Cell-Based Architecture)

    Person(warehouse_staff, "Warehouse Staff", "Uses the Web/Mobile UI")

    System_Boundary(wms_cell, "WMS Cell (Single Instance)") {
        Container(web_ui, "Web/Mobile UI", "React/React Native", "Provides user interface for operators.")
        Container(api_gateway, "API Gateway", "Nginx/Kong", "Routes requests, handles auth & rate limiting.")
        
        Container(inbound_svc, "Inbound Service", "Java/Spring Boot", "Handles receiving and put-away.")
        Container(outbound_svc, "Outbound Service", "Java/Spring Boot", "Handles picking, packing, shipping, and wave planning.")
        Container(inventory_svc, "Inventory Service", "Java/Spring Boot", "Manages inventory tracking, locations, and adjustments.")
        Container(integration_svc, "Integration Service", "Java/Spring Boot", "Adapts external system protocols (ERP, Automation).")
        
        ContainerDb(wms_db, "Cell Database", "PostgreSQL", "Stores operational data for this warehouse.")
        ContainerDb(message_bus, "Event Bus", "RabbitMQ/Kafka", "Handles async communication within the cell.")
    }

    System_Boundary(control_plane, "Control Plane") {
        Container(tenant_manager, "Tenant Manager", "Go", "Manages lifecycle of WMS Cells.")
        Container(identity_provider, "Identity Provider", "Keycloak", "Centralized authentication.")
        Container(observability, "Observability Platform", "Prometheus/Loki/Grafana", "Centralized metrics, logs, and traces.")
    }

    Rel(warehouse_staff, web_ui, "Uses", "HTTPS")
    Rel(web_ui, api_gateway, "API calls", "JSON/HTTPS")
    
    Rel(api_gateway, inbound_svc, "Routes", "gRPC/REST")
    Rel(api_gateway, outbound_svc, "Routes", "gRPC/REST")
    Rel(api_gateway, inventory_svc, "Routes", "gRPC/REST")
    
    Rel(inbound_svc, wms_db, "Reads/Writes", "JDBC")
    Rel(outbound_svc, wms_db, "Reads/Writes", "JDBC")
    Rel(inventory_svc, wms_db, "Reads/Writes", "JDBC")
    
    Rel(inbound_svc, message_bus, "Publishes events", "AMQP")
    Rel(outbound_svc, message_bus, "Publishes events", "AMQP")
    
    Rel(integration_svc, message_bus, "Subscribes/Publishes", "AMQP")
    Rel(integration_svc, wms_db, "Reads (for sync)", "JDBC")

    Rel(api_gateway, identity_provider, "Validates Tokens", "OIDC")
    Rel(inbound_svc, observability, "Sends Metrics/Logs", "OTLP")
    Rel(outbound_svc, observability, "Sends Metrics/Logs", "OTLP")
```

| Container | Responsibilities |
| :--- | :--- |
| **Web/Mobile UI** | Provides the graphical interface for warehouse operators to perform tasks. |
| **API Gateway** | Entry point for all client traffic; handles routing, authentication validation, and rate limiting. |
| **Inbound Service** | Manages receiving of shipments, quality control, and put-away logic. |
| **Outbound Service** | Manages order processing, wave creation, picking, packing, and shipping. |
| **Inventory Service** | Source of truth for inventory quantities, locations, and adjustments. |
| **Integration Service** | Handles communication with external systems (ERP, Stores, Automation) to decouple core services. |
| **Cell Database** | Relational database storing all operational data for the specific WMS instance. |
| **Event Bus** | Facilitates asynchronous communication and eventual consistency between services. |
| **Tenant Manager** | (Control Plane) Manages the provisioning, configuration, and monitoring of WMS Cells. |
| **Identity Provider** | (Control Plane) Centralized user management and authentication service (SSO). |

#### Deployment Diagram (Multi-AZ)
The following diagram illustrates the deployment of a single WMS Cell across 3 AWS Availability Zones for High Availability (QA-02), utilizing Kubernetes auto-scaling (QA-01).

```mermaid
C4Deployment
    title Deployment Diagram - WMS Cell (Multi-AZ K8s)

    Deployment_Node(cloud, "Public Cloud Region", "AWS/Azure/GCP"){
        
        Deployment_Node(k8s, "Kubernetes Cluster", "Managed K8s"){
            
            Deployment_Node(az1, "Availability Zone 1", "us-east-1a"){
                Container(inbound_1, "Inbound Pod", "Docker", "Replica")
                Container(outbound_1, "Outbound Pod", "Docker", "Replica")
            }
            
            Deployment_Node(az2, "Availability Zone 2", "us-east-1b"){
                Container(inbound_2, "Inbound Pod", "Docker", "Replica")
                Container(outbound_2, "Outbound Pod", "Docker", "Replica")
            }
            
            Deployment_Node(az3, "Availability Zone 3", "us-east-1c"){
                Container(inbound_3, "Inbound Pod", "Docker", "Replica")
                Container(outbound_3, "Outbound Pod", "Docker", "Replica")
            }
            
            Container(hpa, "HPA Controller", "K8s", "Scales pods based on CPU")
        }
        
        Deployment_Node(db_cluster, "Database Cluster", "RDS/CloudSQL"){
            Deployment_Node(az_db1, "AZ 1"){
                ContainerDb(db_prim, "Primary DB", "PostgreSQL", "Write/Read")
            }
            Deployment_Node(az_db2, "AZ 2"){
                ContainerDb(db_repl1, "Read Replica", "PostgreSQL", "Read Only")
            }
            Deployment_Node(az_db3, "AZ 3"){
                ContainerDb(db_repl2, "Read Replica", "PostgreSQL", "Read Only")
            }
        }
    }
    
    Rel(hpa, inbound_1, "Scales (2-20 replicas)")
    Rel(inbound_1, db_prim, "Writes")
    Rel(inbound_2, db_repl1, "Reads (Reporting)")
```

#### Deployment Diagram (Edge/Hybrid)
To support **Offline Operations (QA-03)**, the WMS employs a Hybrid Edge-Cloud architecture. A lightweight **Edge Node** runs in the warehouse.

```mermaid
C4Deployment
    title Deployment Diagram - Edge Node (Offline Survivability)

    Deployment_Node(warehouse, "Warehouse (On-Premise)", "Physical Location"){
        
        Deployment_Node(edge_server, "Edge Server", "K3s / Docker Compose"){
            Container(edge_inbound, "Edge Inbound Svc", "Local Replica", "Handles receiving when offline.")
            Container(edge_outbound, "Edge Outbound Svc", "Local Replica", "Directs picking automation.")
            Container(sync_agent, "Sync Agent", "Go", "Bi-directional data sync & conflict resolution.")
            ContainerDb(local_broker, "Local Message Broker", "RabbitMQ", "Buffers automation events.")
            ContainerDb(local_db, "Local DB", "SQLite/PostgreSQL", "Sub-set of inventory data.")
        }

        Deployment_Node(workstations, "Workstations"){
            Container(pwa, "Warehouse UI (PWA)", "React", "Cached static assets.")
        }
        
        System_Ext(automation, "Picking Automation", "Conveyors/Robots")
    }

    Deployment_Node(cloud, "Cloud Region", "AWS"){
        Container(cloud_gateway, "Cloud API Gateway")
        Container(cloud_sync, "Cloud Sync Service")
    }

    Rel(pwa, edge_inbound, "Local API Calls", "HTTPS/LAN")
    Rel(edge_outbound, local_broker, "Publishes Tasks", "AMQP")
    Rel(local_broker, automation, "Controls", "MQTT")
    
    Rel(sync_agent, local_db, "Reads/Writes")
    Rel(sync_agent, cloud_sync, "Syncs Deltas (When Online)", "HTTPS/WSS")
```

| Component | Responsibility |
| :--- | :--- |
| **Edge Inbound/Outbound** | Lightweight versions of core services. They process transactions locally against the Local DB. |
| **Sync Agent** | The "Brain" of the offline mode. Tracks local changes (deltas) and pushes them to cloud when connectivity restores. Pulls reference data updates. |
| **Local DB** | Stores a subset of data: expected receipts, open wave orders, and current inventory for that warehouse only. |
| **Local Broker** | Ensures automation systems (robots) can continue to communicate with the WMS logic even if the cloud WAN link is down. |

### 6.- Component diagrams
The following diagram details the **Outbound Service** internals using **Hexagonal Architecture**. This pattern is applied to all complex services (Inbound, Inventory).

```mermaid
C4Component
    title Component Diagram - Outbound Service (Hexagonal Architecture)

    Container(api_adapter, "API Adapter", "Controller", "Handles REST/gRPC requests and maps to commands.")
    
    Container_Boundary(domain, "Domain Layer") {
        Component(wave_service, "Wave Application Service", "Service", "Orchestrates wave creation Use Case.")
        Component(picking_strategy, "Picking Strategy", "Interface", "Defines contract for route optimization.")
        Component(strategy_impl, "ZoneStrategy", "Implementation", "Specific optimization logic.")
        Component(domain_model, "Domain Model", "Entities", "Wave, PickTask, ReplenishmentOrder logic.")
        
        Rel(wave_service, picking_strategy, "Uses")
        Rel(picking_strategy, strategy_impl, "Implemented by")
        Rel(wave_service, domain_model, "Manipulates")
    }

    Container(infra_adapter, "Persistence Adapter", "Repository Impl", "Implements data access.")
    Container(msg_adapter, "Messaging Adapter", "Producer", "Publishes events to Bus.")

    Rel(api_adapter, wave_service, "Calls", "Use Cases")
    Rel(wave_service, infra_adapter, "Uses", "via Interface")
    Rel(wave_service, msg_adapter, "Uses", "via Interface")
```

| Component | Responsibility |
| :--- | :--- |
| **API Adapter** | Driving adapter; converts HTTP/RPC into domain commands. |
| **Wave Application Service** | Coordinator; executes the use case by calling domain objects and ports. Transactions start/end here. |
| **Picking Strategy** | Strategy pattern interface for decoupling optimization algorithms. |
| **Domain Model** | Pure business logic and state validation (Entities, Value Objects). |
| **Persistence Adapter** | Driven adapter; implements repository interfaces to talk to the DB. |


### 7.- Sequence diagrams

#### Sequence: Processing an External Replenishment Order
This diagram shows how the system components collaborate to handle an incoming order from the Store System, emphasizing the **Integration Service** and **Outbound Service**.

```mermaid
sequenceDiagram
    participant StoreSys as Store System
    participant Integ as Integration Service
    participant Bus as Event Bus
    participant Outbound as Outbound Service
    participant Inv as Inventory Service
    participant DB as Cell Database

    StoreSys->>Integ: POST /orders (Replenishment)
    Integ->>Integ: Validate Schema
    Integ->>Bus: Publish OrderReceived Event
    Integ-->>StoreSys: 202 Accepted

    Bus->>Outbound: Subscribe OrderReceived
    Outbound->>Outbound: Create Order Entity
    Outbound->>Inv: Reserve Inventory (Request)
    Inv->>DB: Check & Lock Qty
    DB-->>Inv: Success
    Inv-->>Outbound: Reservation Confirmed
    Outbound->>DB: Persist Order (Status: Created)
    Outbound->>Bus: Publish OrderCreated Event
```

#### Sequence: Receiving Inbound Shipment (US-01)
This diagram illustrates the flow for receiving goods, updating inventory, and notifying the ERP.

```mermaid
sequenceDiagram
    participant UI as Web/Mobile UI
    participant Inbound as Inbound Svc
    participant Inv as Inventory Svc
    participant DB as Cell Database
    participant Bus as Event Bus
    participant Integ as Integration Svc
    participant ERP

    UI->>Inbound: POST /shipments/{id}/receive
    Inbound->>Inbound: Validate Item & Qty
    Inbound->>Inv: Allocate Location (Putaway Strategy)
    Inv-->>Inbound: Location: A-10-2
    
    Inbound->>Inv: Increment Inventory (A-10-2)
    Inv->>DB: Update Stock Records
    
    Inbound->>DB: Update Shipment (Status: RECEIVED)
    Inbound->>Bus: Publish ShipmentReceived Event
    
    Bus->>Integ: Subscribe ShipmentReceived
    Integ->>ERP: Notify Receipt (Financial Event)
    
    Bus-->>UI: Real-time Update
```

#### Sequence: Wave Picking Planning (US-04)
This diagram shows how a planner triggers wave creation, using the **Strategy Pattern** to optimize tasks.

```mermaid
sequenceDiagram
    participant Planner
    participant Outbound as Outbound Svc
    participant Strategy as Picking Strategy
    participant Inv as Inventory Svc
    participant DB as Cell Database

    Planner->>Outbound: POST /waves (Criteria: Next Day Delivery)
    Outbound->>DB: Find Eligible Orders
    DB-->>Outbound: [Order1, Order2...]
    
    Outbound->>Strategy: OptimizeRoute(Orders, Locations)
    Strategy-->>Outbound: Optimized Pick Tasks
    
    Outbound->>Inv: Hard Reserve Inventory
    Inv->>DB: Lock Inventory
    
    Outbound->>DB: Save Wave & Pick Tasks
    Outbound-->>Planner: Wave Created (ID: 1001)
```

#### Sequence: Inventory Update with Optimistic Locking (QA-01)
This diagram details how the **Inventory Service** handles concurrent updates using optimistic locking to avoid database bottlenecks while ensuring consistency.

```mermaid
sequenceDiagram
    participant UI
    participant Service as Inventory Svc
    participant DB as Cell Database

    Note over UI, DB: Scenario: Successful Update
    UI->>Service: POST /inventory/adjust (ItemID, Qty=+10, Ver=5)
    Service->>DB: UPDATE Inv SET Qty=110, Ver=6 WHERE ID=X AND Ver=5
    DB-->>Service: Rows Updated: 1 (Success)
    Service-->>UI: 200 OK

    Note over UI, DB: Scenario: Concurrent Collision (Optimistic Lock)
    UI->>Service: POST /inventory/adjust (ItemID, Qty=+5, Ver=5)
    
    Note right of DB: Another transaction has already updated Ver to 6
    Service->>DB: UPDATE Inv SET Qty=115, Ver=6 WHERE ID=X AND Ver=5
    DB-->>Service: Rows Updated: 0 (Failure)
    
    Note right of Service: Detects 0 rows -> OptimisticLockException
    Service->>Service: Catch Exception -> Retry Logic
    
    Service->>DB: SELECT * FROM Inv WHERE ID=X
    DB-->>Service: Inventory (Qty=110, Ver=6)
    
    Note right of Service: Re-calculate: 110 + 5 = 115
    Service->>DB: UPDATE Inv SET Qty=115, Ver=7 WHERE ID=X AND Ver=6
    DB-->>Service: Rows Updated: 1 (Success)
    Service-->>UI: 200 OK (Transparent to User)
```

#### Sequence: Reliable Event Publication (Transactional Outbox) (QA-04)
This diagram illustrates the **Transactional Outbox** pattern to ensure events are never lost even if the Event Bus is temporarily down.

```mermaid
sequenceDiagram
    participant App as Core Service (e.g. Inbound)
    participant DB as Cell Database (Outbox Table)
    participant Relay as Outbox Relay (BG Process)
    participant Bus as Event Bus

    Note over App, DB: Transaction Scope (Atomic)
    App->>DB: INSERT INTO Operations (Business Data)
    App->>DB: INSERT INTO Outbox (Event Payload)
    App->>DB: COMMIT Transaction
    
    Note over Relay, Bus: Asynchronous Discovery & Publish
    loop Every 500ms
        Relay->>DB: SELECT * FROM Outbox WHERE Status='PENDING'
        DB-->>Relay: [Event1, Event2]
        
        Relay->>Bus: Publish Event1
        Bus-->>Relay: ACK
        
        Relay->>DB: UPDATE Outbox SET Status='PROCESSED' WHERE ID=1
    end
```

#### Sequence: Idempotent External Order Processing (QA-04)
This diagram shows how the system handles duplicate requests (retries) from external systems using an **Idempotency Key**.

```mermaid
sequenceDiagram
    participant Ext as Store System
    participant API as API Gateway
    participant Svc as Integration Svc
    participant DB as Cell Database

    Ext->>API: POST /orders (Idempotency-Key: "UUID-1234")
    API->>Svc: Forward Request
    
    Svc->>DB: SELECT * FROM IdempotencyLog WHERE Key="UUID-1234"
    DB-->>Svc: null (Not found)
    
    Note over Svc, DB: Normal Processing
    Svc->>Svc: Process Order
    Svc->>DB: SAVE Order & INSERT IdempotencyLog("UUID-1234", Result)
    Svc-->>Ext: 201 Created (Order-ID: 555)
    
    Note over Ext, DB: Scenario: Retry (Duplicate)
    Ext->>API: POST /orders (Idempotency-Key: "UUID-1234")
    API->>Svc: Forward Request
    
    Svc->>DB: SELECT * FROM IdempotencyLog WHERE Key="UUID-1234"
    DB-->>Svc: Found (Result: 201, ID: 555)
    
    Svc-->>Ext: 201 Created (Order-ID: 555) [Cached Response]
```

#### Sequence: Offline Mode & Synchronization (QA-03)
This diagram illustrates how the **Edge Node** handles operations during an outage and synchronizes upon recovery.

```mermaid
sequenceDiagram
    participant PWA as Web UI (PWA)
    participant Edge as Edge Node (Inbound)
    participant LocDB as Local DB
    participant Sync as Sync Agent
    participant Cloud as Cloud Sync Svc
    participant CloudDB as Cloud DB

    Note over PWA, Cloud: Phase 1: Offline Operation (WAN Down)
    PWA->>Edge: POST /receive (Item A, Qty 10)
    Edge->>LocDB: INSERT Tx_Log (ID: 101, Op: Receive, Payload...)
    Edge->>LocDB: UPDATE Inventory (Local)
    Edge-->>PWA: 200 OK (Local Mode)
    
    Note over Sync, Cloud: Phase 2: Connectivity Restored
    Sync->>Cloud: Handshake / Authenticate
    Sync->>LocDB: SELECT * FROM Tx_Log WHERE Status='PENDING'
    LocDB-->>Sync: [Tx 101]
    
    Sync->>Cloud: POST /sync/upload (Tx 101)
    Cloud->>CloudDB: Apply Tx 101 (Conflict Check)
    CloudDB-->>Cloud: Success
    Cloud-->>Sync: ACK (Tx 101 Committed)
    
    Sync->>LocDB: UPDATE Tx_Log SET Status='SYNCED' WHERE ID=101
    
    Note over Sync, Cloud: Phase 3: Downward Sync (Ref Data)
    Cloud->>Sync: Push Updates (New Products, Orders)
    Sync->>LocDB: UPSERT Data
```


#### Sequence: Secure User Access with OIDC (QA-07)
This diagram details the authentication flow using the **Authorization Code Flow with PKCE**, ensuring credentials never touch the UI application directly.

```mermaid
sequenceDiagram
    participant User
    participant UI as Web UI
    participant IDP as Identity Provider (Keycloak)
    participant API as API Gateway
    participant Svc as Backend Service

    User->>UI: Click "Login"
    UI->>IDP: Redirect to Login Page (scope=openid)
    IDP-->>User: Login Form
    User->>IDP: Enter Credentials (MFA if needed)
    IDP-->>UI: Redirect with Auth Code
    
    UI->>IDP: Exchange Code for Tokens (ID, Access, Refresh)
    IDP-->>UI: Tokens (JWT)
    
    Note over UI, Svc: Authenticated Request
    UI->>API: GET /orders (Header: Bearer JWT)
    API->>API: Validate Token Signature (Stateless)
    API->>Svc: Forward Request (x-user-id: 123)
    Svc-->>API: Response
    API-->>UI: Data
```

#### Sequence: Observability & Tracing (QA-09)
This diagram illustrates how **OpenTelemetry** traces requests across service boundaries to provide a complete view of a distributed transaction.

```mermaid
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant SvcA as Service A
    participant SvcB as Service B
    participant OTel as OpenTelemetry Collector
    participant Backend as Observability Backend

    Client->>API: Request (Headers)
    Note right of API: Start Trace T1, Span S1
    
    API->>SvcA: Request (Trace-Context: T1, S1)
    Note right of SvcA: Start Span S2 (Parent: S1)
    
    SvcA->>SvcB: Request (Trace-Context: T1, S2)
    Note right of SvcB: Start Span S3 (Parent: S2)
    
    SvcB-->>SvcA: Response
    SvcA-->>OtEL: Async Report (Spans S2, S3)
    
    SvcA-->>API: Response
    API-->>OtEL: Async Report (Span S1)
    API-->>Client: Response
    
    OTel->>Backend: Batch Export (Trace T1)
```


### 8.- Interfaces
The Primary Interfaces are defined by the **API Gateway** (REST/HTTP) for synchronous client interactions and the **Integration Service** for external system coupling.
- **External API:** RESTful JSON.
- **Internal Events:** CloudEvents format over AMQP.

### 9.- Design decisions

| Driver | Decision | Rationale | Discarded alternatives |
| :--- | :--- | :--- | :--- |
| **C-03, QA-08** | **Cell-Based Architecture**<br>Each WMS instance is a self-contained "Cell" with its own services and database. | Ensures strict data and fault isolation. A failure in one cell (e.g., database lockup) does not affect others. Allows independent scaling. | **Multi-tenant Monolith:** Discarded due to risk of "noisy neighbor" issues and complexity of ensuring data isolation. |
| **AC-04** | **Control Plane**<br>Dedicated management layer for provisioning and monitoring WMS Cells. | Essential for managing the lifecycle of many decoupled instances without manual toil. | **Script-based Management:** Not scalable for a large number of instances. |
| **AC-02** | **Microservices / Service Partitioning**<br>Split core domains (Inbound, Outbound, Inventory) into separate services. | Allows independent scaling of high-throughput areas (Outbound) and isolates complexity. | **Monolith:** Discarded because team scale and independent scalability requirements (QA-01) favor decoupling. |
| **QA-06** | **Integration Service**<br>Dedicated adapters for external systems. | Decouples core logic from external API changes and protocols. | **Direct Integration:** Lead to tight coupling and fragile core services. |
| **AC-03** | **Event Bus**<br>Asynchronous communication between services. | Improves reliability and responsiveness; services aren't blocked waiting for others. | **Synchronous HTTP everywhere:** Higher latency and risk of cascading failures. |
| **AC-02** | **Hexagonal Architecture**<br>Structure core services (Inbound, Outbound) with Ports & Adapters. | Decouples business logic from infrastructure, enabling easier testing and technology swaps. | **Layered Architecture:** Prone to dependency leakage. |
| **US-04, US-02** | **Strategy Pattern**<br>Encapsulate algorithms for Picking, Put-away, and Allocation. | Allows runtime selection or configuration of complex algorithms without changing core service logic. | **Hardcoded Logic:** Difficult to extend or customize per tenant. |
| **QA-04, US-10** | **Transactional Outbox / Integration Service**<br>Services emit events to an Outbox; Integration Service relays to external systems. | Guarantees reliability and eventual consistency for external integrations (ERP, Stores) even if failures occur. | **Dual Write:** High risk of inconsistencies. |
| **QA-02** | **Multi-AZ Deployment**<br>Deploy each WMS Cell across 3 Availability Zones with a Load Balancer. | Ensures survival of a complete zone failure. Core services and DB remain available (RPO<15m, RTO<4h). | **Active-Passive / DR Site:** Higher RTO (manual switchover) and wasted resources (idle). |
| **QA-01** | **Horizontal Pod Autoscaling (HPA)**<br>Auto-scale Inbound, Outbound, Inventory pods based on CPU/Metrics. | Allows handling 10x peak load without paying for peak capacity 24/7. | **Static Provisioning:** Cost prohibitive for peak loads. **Vertical Scaling:** downtime required to resize. |
| **QA-01** | **Read Replicas**<br>Direct read-heavy traffic (Reports, Dashboards) to DB synchronous/async replicas. | Prevents reporting queries from locking the primary DB and slowing down core transactions (Receiving/Shipping). | **Sharding:** Too complex for current scale. |
| **QA-01** | **Optimistic Locking**<br>Use version-checking for Inventory updates. | drastically improves throughput compared to DB row locks (Pessimistic). Most inventory updates (different items) don't collide. | **Pessimistic Locking:** High contention risk (deadlocks, timeouts). **Redis:** Persistence risk (QA-04). |
| **QA-04, AC-05** | **Transactional Outbox Pattern**<br>Write events to an `Outbox` table in the same transaction as business data; relay to Event Bus via background process. | Guarantees atomic "Do Operation + Emit Event". Prevents data inconsistencies if Event Bus is down or transaction rolls back. | **Dual Write:** Risk of phantom events or lost events. **2PC:** Too complex/slow. |
| **QA-04** | **Idempotency Keys**<br>Require `Idempotency-Key` header for mutating API calls; store results in `IdempotencyLog`. | Prevents duplicate side-effects (e.g., double billing, double shipping) when clients retry due to network timeouts. | **Stateful De-duplication:** Hard to scale. |
| **AC-03** | **Circuit Breaker**<br>Wrap external integration calls (ERP, Store) with Circuit Breakers (Open/Closed/Half-Open states). | Prevents cascading failures and resource exhaustion when external systems are down. Fails fast. | **Infinite Retries:** Causes thread pool starvation. |
| **AC-03** | **Dead Letter Queue (DLQ)**<br>Route failed messages (after max retries) to a separate DLQ queue. | Ensures system doesn't get stuck processing "poison pill" messages forever. Allows manual intervention. | **Drop Message:** Data loss. |
| **QA-03** | **Hybrid Edge-Cloud Architecture**<br>Deploy a lightweight "Edge Node" in each warehouse running Inbound/Outbound services and a local DB. | Enables critical warehouse operations (Receiving, Picking) to continue for hours/days without internet. Brings compute to the data (low latency). | **Pure Cloud:** Fails QA-03 (Offline). **Full On-Prem:** High management overhead, loses cloud benefits. |
| **QA-03, C-06** | **Local Message Broker (Edge)**<br>Deploy an edge-resident broker (e.g. RabbitMQ) to decouple automation systems. | Allows conveyors/robots to continue receiving tasks and sending confirmations even if WAN to Cloud is down. | **Direct Cloud Connection:** Automation stops immediately on network jitter. |
| **QA-03** | **Sync Agent & Transaction Log**<br>Dedicated agent to capture offline transactions in a log and replay them to cloud when online. | Isolates complex synchronization logic (retry, conflict detection) from business logic. Ensures zero data loss. | **Dual Write:** Fragile. **Database Replication:** Hard to filter/control subsets of data efficiently. |
| **QA-05, QA-03** | **Offline-First PWA**<br>Use Service Workers and IndexedDB in the Web UI. | UI remains responsive during intermittent Wi-Fi drops. Queues requests locally before sending to Edge Node. | **Standard Web App:** Unusable with spotty coverage. |
| **QA-07, AC-06** | **OpenID Connect (OIDC) with Keycloak**<br>Centralized Identity Provider using standard flows (Auth Code + PKCE). | Decouples authentication from services. Enables SSO and centralized user management. | **Custom Auth:** High security risk. **Basic Auth:** Insufficient for modern security needs. |
| **QA-07, AC-06** | **Service Mesh (Linkerd/Istio)**<br>Infrastructure layer for transparent mTLS and traffic control. | Ensures Zero Trust (encryption in transit) between all internal services without code changes. | **Library-level mTLS:** High maintenance burden to manage certificates and implementation per language. |
| **QA-09, AC-07** | **OpenTelemetry (OTel)**<br>Vendor-neutral instrumentation standard for traces, metrics, and logs. | Future-proofs observability. Allows switching backends without rewriting code. Connects traces across boundaries. | **Proprietary Agents:** Vendor lock-in (e.g., Datadog only). |
| **QA-09, AC-07** | **PLG Stack (Prometheus, Loki, Grafana)**<br>Cloud-native stack for metrics and logs aggregation. | Highly scalable and integrates natively with Kubernetes. Efficient log storage (Loki). | **ELK Stack:** Higher resource overhead for log indexing. |
