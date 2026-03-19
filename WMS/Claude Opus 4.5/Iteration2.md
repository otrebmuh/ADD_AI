# WMS Architecture Design - Iteration 2 Summary

## Iteration Goal

**Core Inbound Operations**: Support receiving of inbound shipments and put-away workflows with configurable strategies, enabling accurate inventory capture as the foundation for all warehouse operations.

## Drivers Addressed

| ID | Type | Description | Priority Rationale |
|----|------|-------------|-------------------|
| **US-01** | User Story | As a receiving operator, I can receive inbound shipments (from suppliers or returns) and register them in the WMS so that inventory is accurately captured. | **Business Critical**: Receiving is the entry point for all inventory. Accurate inventory capture is fundamental to warehouse operations and directly impacts downstream processes. **Technical Challenge**: Requires real-time inventory updates, integration with supplier systems, and handling of various shipment types and formats. |
| **US-02** | User Story | As a warehouse operator, I can put away received goods into storage locations so that stock is organized and available for picking. Different put-away strategies should be supported (size, rotation, etc.) | **Business Critical**: Put-away directly affects picking efficiency and warehouse space utilization. **Technical Challenge**: Requires sophisticated algorithms for multiple put-away strategies (size-based, rotation-based, etc.), location optimization, and real-time space management. |

## Elements Refined

| Element | Current State (from Iteration 1) | Refinement Type | Rationale |
|---------|----------------------------------|-----------------|-----------|
| **Inbound Module** | Identified as a high-level module within the WMS Application with sub-elements "Receiving" and "Put-Away" | **Decomposition** (top-down) | Decomposed into detailed components showing receiving workflow, put-away task management, and strategy execution |
| **Inventory Module** | Identified with sub-elements "Inventory Management", "Inventory Control", and "Location Management" | **Refinement** (partial decomposition) | Refined to show how inventory records are created during receiving and how location capacity is managed |
| **Internal Event Bus** | Identified as a common service for module communication | **Refinement** (define events) | Defined specific domain events for inbound operations |
| **WMS Database** | Identified as PostgreSQL database for warehouse data | **Refinement** (schema areas) | Identified data entities and relationships specific to inbound operations |

## Design Concepts Selected

| Design Concept | Pros | Cons | Discarded Alternatives |
|----------------|------|------|------------------------|
| **Strategy Pattern** for Put-Away Strategies | Enables runtime selection of put-away algorithms; Easy to add new strategies; Clean separation; Testable in isolation | Requires upfront design of strategy interface; May lead to proliferation of strategy classes | **Rule Engine (Drools)**: Too heavyweight; adds operational complexity. **Hard-coded conditional logic**: Violates Open-Closed principle |
| **State Machine Pattern** for Shipment Lifecycle | Explicit modeling of shipment states; Prevents invalid state transitions; Easy to audit and visualize; Supports pause/resume | Adds complexity for simple linear workflows | **Simple status flags**: Prone to invalid states. **BPMN Workflow Engine**: Overkill for shipment status tracking |
| **Domain Events** for Cross-Module Communication | Loose coupling between modules; Enables audit logging; Supports future extensibility; Aligns with Iteration 1 decision | Eventual consistency; Requires event schema management | **Direct method calls**: Creates tight coupling. **Synchronous API calls**: Same coupling issues |
| **Task Queue Pattern** for Put-Away Task Distribution | Enables work distribution; Supports prioritization; Fault-tolerant; Natural fit for handheld workflows | Adds complexity for task state management | **Direct task assignment**: No load balancing. **Pull-based polling**: Inefficient |
| **Repository Pattern** for Data Access | Abstracts database access; Enables unit testing; Consistent patterns | Additional abstraction layer | **Direct database access**: Couples business logic to schema |
| **Factory Pattern** for Strategy Instantiation | Centralizes strategy creation; Supports configuration-driven selection | Additional indirection | **Manual instantiation**: Scattered creation logic |
| **Optimistic Locking** for Inventory Updates | Prevents lost updates; Better performance than pessimistic; Suitable for read-heavy patterns | Requires retry logic on conflicts | **Pessimistic locking**: Reduces throughput. **No locking**: Data corruption risk |
| **Location Capacity Service** for Space Management | Real-time tracking; Supports put-away decisions; Enables utilization reporting | Requires keeping data synchronized | **Calculate on-demand**: Too slow. **Static assignments**: Inflexible |

## Instantiation Decisions

| Instantiation Decision | Rationale |
|------------------------|-----------|
| Create **ReceivingService** component implementing State Machine pattern | Centralizes receiving workflow logic; manages state transitions; validates business rules |
| Create **ReceivingController** as API endpoint | Provides REST API for web application and handheld devices |
| Create **PutAwayService** component for task management | Orchestrates put-away task creation, assignment, and completion |
| Create **PutAwayStrategyEngine** implementing Factory pattern | Creates appropriate strategy instance based on item attributes and warehouse configuration |
| Create concrete **PutAwayStrategy** implementations: FIFOStrategy, SizeBasedStrategy, ZoneBasedStrategy, RotationStrategy | Each strategy encapsulates specific put-away logic |
| Create **LocationCapacityService** in Inventory Module | Maintains real-time location capacity in Redis cache |
| Create **InventoryCreationService** in Inventory Module | Handles inventory record creation during receiving |
| Define **InboundShipmentRepository** and **PutAwayTaskRepository** | Abstracts data access; enables unit testing |
| Define domain events: ShipmentArrivedEvent, ReceivingLineCompletedEvent, ReceivingCompletedEvent, PutAwayTaskCreatedEvent, PutAwayTaskCompletedEvent | Enables loose coupling; supports audit logging |
| Create **ExceptionHandlingService** for receiving discrepancies | Manages exception registration, escalation, and resolution |
| Use **Optimistic Locking** on Inventory and Location entities | Prevents concurrent update conflicts |

## Components Defined

### Inbound Module Components

| Component | Responsibilities |
|-----------|------------------|
| **ReceivingController** | REST API endpoints for receiving operations; request validation; authentication context extraction |
| **PutAwayController** | REST API endpoints for put-away operations; task assignment queries; task completion endpoints |
| **ReceivingService** | Orchestrates receiving workflow; validates shipment data; manages state transitions; triggers put-away task generation |
| **PutAwayService** | Creates put-away tasks; assigns tasks to operators; processes task completions; coordinates with strategy engine |
| **ExceptionHandlingService** | Registers receiving discrepancies; escalates to supervisors; tracks resolution status |
| **ShipmentStateMachine** | Manages shipment lifecycle states; validates state transitions; enforces business rules per state |
| **PutAwayStrategyEngine** | Factory for creating strategy instances; selects strategy based on item attributes and configuration |
| **FIFOStrategy** | Selects locations to maintain First-In-First-Out inventory rotation |
| **SizeBasedStrategy** | Matches item dimensions to appropriate location sizes |
| **ZoneBasedStrategy** | Assigns items to predefined zones based on product category, velocity, or temperature |
| **RotationStrategy** | Considers expiration dates and lot numbers for FEFO |
| **InboundShipmentRepository** | Data access for InboundShipment and InboundShipmentLine entities |
| **PutAwayTaskRepository** | Data access for PutAwayTask entities |

### Inventory Module Components (Receiving-Related)

| Component | Responsibilities |
|-----------|------------------|
| **InventoryCreationService** | Creates inventory records when receiving is completed; handles lot and expiration tracking |
| **LocationCapacityService** | Maintains real-time location capacity in Redis cache; provides available locations for put-away |
| **InventoryRepository** | Data access for Inventory entities; supports optimistic locking |
| **LocationRepository** | Data access for Location entities |

## Domain Events Defined

| Event | Publisher | Subscribers | Payload |
|-------|-----------|-------------|---------|
| **ShipmentArrivedEvent** | ReceivingService | AuditService | shipmentId, dockDoorId, arrivalTime, expectedLines |
| **ReceivingLineCompletedEvent** | ReceivingService | AuditService, InventoryCreationService | shipmentId, lineId, sku, receivedQty, lotNumber, expirationDate |
| **ReceivingCompletedEvent** | ReceivingService | PutAwayService, AuditService | shipmentId, totalLinesReceived, totalQuantity, completionTime |
| **ReceivingExceptionEvent** | ExceptionHandlingService | AuditService, NotificationService | shipmentId, exceptionType, description, severity |
| **PutAwayTaskCreatedEvent** | PutAwayService | AuditService | taskId, shipmentId, sku, quantity, sourceLocation, targetLocation |
| **PutAwayTaskCompletedEvent** | PutAwayService | InventoryCreationService, AuditService, LocationCapacityService | taskId, sku, quantity, targetLocation, completionTime, operatorId |
| **InventoryCreatedEvent** | InventoryCreationService | AuditService, IntegrationModule | inventoryId, locationId, sku, quantity, status, lotNumber |

## Shipment State Machine

| State | Description | Allowed Actions |
|-------|-------------|-----------------|
| **Expected** | Shipment is anticipated but not yet arrived | Register arrival, Cancel |
| **Arrived** | Shipment has arrived at dock door | Start receiving, Report exception |
| **Receiving** | Actively receiving lines from shipment | Receive line, Complete receiving, Pause, Report exception |
| **PartiallyReceived** | Receiving paused (shift end, issue) | Resume receiving |
| **Received** | All lines received, put-away tasks generated | Automatic transition when put-away completes |
| **Closed** | Shipment fully processed | None (terminal state) |
| **Exception** | Major discrepancy requiring supervisor | Resolve exception, Close with exception |
| **Cancelled** | Shipment cancelled before arrival | None (terminal state) |

## Design Decisions Recorded

| Driver | Decision | Rationale | Discarded Alternatives |
|--------|----------|-----------|------------------------|
| US-01 | **State Machine Pattern** for shipment lifecycle | Explicit state modeling; prevents invalid transitions; supports pause/resume; clear audit trail | Simple status flags; BPMN Workflow Engine |
| US-01 | **Domain Events** for downstream processes | Loose coupling; parallel processing; extensibility | Synchronous direct calls |
| US-01 | **ReceivingService + ReceivingController** layered architecture | Separation of API and business logic; independent testing | Fat controller; CQRS |
| US-01 | **ExceptionHandlingService** for discrepancies | Centralized exception management; supervisor escalation | Inline exception handling |
| US-02 | **Strategy Pattern** with PutAwayStrategyEngine | Runtime algorithm selection; easy extensibility; independent testing | Rule Engine; Hard-coded conditionals |
| US-02 | **Configuration-driven strategy selection** | Warehouse customization without code changes | Item master only; Operator selection |
| US-02 | **LocationCapacityService with Redis cache** | Fast lookups; atomic reservations; reduced database load | Database-only; Calculate on-demand |
| US-02 | **Task Queue Pattern** for put-away distribution | Work distribution; prioritization; fault tolerance | Direct assignment; Pull-based polling |
| US-01, US-02 | **Optimistic Locking** on Inventory and Location | Prevents lost updates; better performance | Pessimistic locking; No locking |
| US-01, US-02 | **Repository Pattern** for data access | Abstraction; unit testing; consistent patterns | Direct JPA/JDBC in services |
| US-01, US-02 | **InventoryCreationService** in Inventory Module | Clear responsibility; respects module boundaries | Inline in ReceivingService |

## Analysis Results

| Driver | Analysis Result | Justification |
|--------|-----------------|---------------|
| **US-01: Receive inbound shipments** | **Satisfied** | Complete support: ShipmentStateMachine, ReceivingService/Controller, Domain events, ExceptionHandlingService, InventoryCreationService, sequence diagrams, repository pattern with optimistic locking |
| **US-02: Put-away with configurable strategies** | **Satisfied** | Comprehensive support: Strategy Pattern with four strategies, PutAwayStrategyEngine, LocationCapacityService with Redis, Task Queue Pattern, sequence diagrams demonstrating workflows |

## Coverage Analysis

| Requirement Aspect | US-01 Coverage | US-02 Coverage |
|-------------------|----------------|----------------|
| **Functional workflow** | ✅ Complete receiving workflow with state transitions | ✅ Complete put-away workflow with strategy selection |
| **API design** | ✅ ReceivingController with endpoints | ✅ PutAwayController with endpoints |
| **Data management** | ✅ Repository pattern; optimistic locking | ✅ Repository pattern; capacity cache in Redis |
| **Exception handling** | ✅ ExceptionHandlingService | ✅ Location mismatch validation; fallback strategies |
| **Integration with other modules** | ✅ Events trigger InventoryCreationService | ✅ Events trigger inventory updates |
| **Audit trail** | ✅ All events subscribed by AuditService | ✅ All events subscribed by AuditService |
| **Configurability** | ✅ State machine supports various scenarios | ✅ Strategy selection via configuration |
| **Sequence diagrams** | ✅ Diagram 7.5 covers full workflow | ✅ Diagrams 7.6, 7.7 cover full workflow |
| **Component definition** | ✅ Components defined in section 6.1 | ✅ Components defined in section 6.1 |

## Iteration Metrics

| Metric | Value |
|--------|-------|
| Primary drivers addressed | 2 of 2 (100%) |
| Primary drivers satisfied | 2 of 2 (100%) |
| Components defined | 16 new components |
| Domain events defined | 7 events |
| Sequence diagrams created | 3 diagrams |
| Design decisions recorded | 11 decisions |

## Artifacts Modified

| Artifact | Section | Changes |
|----------|---------|---------|
| Architecture.md | Section 6 - Component Diagrams | Added Inbound Module component diagram (6.1), component responsibilities tables, state machine diagram, strategy selection flowchart, domain events table |
| Architecture.md | Section 7 - Sequence Diagrams | Added US-01 receiving workflow (7.5), US-02 put-away workflow (7.6), strategy selection detail (7.7) |
| Architecture.md | Section 9 - Design Decisions | Added Iteration 2 design decisions table |

## Conclusion

**Iteration 2 Goal: ACHIEVED**

All design decisions made during this iteration are sufficient to satisfy the drivers US-01 (Receive inbound shipments) and US-02 (Put-away with configurable strategies). The architecture now includes a complete Inbound Module component design, a flexible Strategy Pattern implementation for put-away algorithms, event-driven communication between modules, and comprehensive documentation.

The system is ready to proceed to **Iteration 3: Core Outbound Operations** (US-03, US-04, US-05, US-07, US-08).
