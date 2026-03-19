# Iteration 3: Core Outbound Operations

## Summary

This iteration focused on implementing the primary revenue-generating workflows from order submission to shipment. The design covers the complete outbound flow that enables the WMS to fulfill store replenishment requests, including order management, wave planning, picking, packing, and shipping.

---

## Step 2: Iteration Goal and Drivers

### Goal
Implement the primary revenue-generating workflows from order submission to shipment.

### Drivers

| Driver ID | Type | Description |
|-----------|------|-------------|
| **US-03** | User Story | Submit replenishment orders from store systems - Stores submit orders that the WMS receives, validates, and queues for fulfillment |
| **US-04** | User Story | Allocate and release waves/batches for picking - Warehouse planners group orders into waves, allocate inventory, and optimize picking sequences |
| **US-05** | User Story | Send picking tasks to picking systems - Distribute picking tasks to automated or semi-automated picking systems |
| **US-07** | User Story | Pick confirmation processing - Record pick completions from operators and automation systems, handle short picks and substitutions |
| **US-08** | User Story | Packing and shipping operations - Pack picked items into cartons, prepare shipments, and assign to dock doors for loading |

---

## Step 3: Elements to Refine

| Element | Current State | Refinement Type | Rationale |
|---------|---------------|-----------------|-----------|
| **Outbound Module** | Identified as a high-level module in the WMS Application with sub-modules: Order Management (OM), Wave Planning (WV), Picking (PK), Packing & Shipping (PS) | **Decomposition** (top-down) | Need to decompose into detailed components with controllers, services, repositories, and state machines to support US-03, US-04, US-05, US-07, US-08 |
| **Integration Module - Store Integration (SI)** | Identified as a sub-module within Integration Module | **Decomposition** (top-down) | Need to detail the components that receive replenishment orders from store systems (US-03) |
| **Integration Module - Automation Integration (AI)** | Identified as a sub-module within Integration Module | **Decomposition** (top-down) | Need to detail components for sending picking tasks to and receiving confirmations from picking systems (US-05, US-07) |
| **Inventory Module** | Detailed in Iteration 2 with InventoryCreationService, LocationCapacityService, repositories | **Improvement** | Need to add inventory allocation capabilities to support wave planning and picking (US-04) |
| **Message Broker** | Identified as container for async messaging | **Improvement** | Need to define queues/topics for order intake, picking task distribution, and shipment events |

### Driver-to-Element Mapping

| Driver | Elements Affected |
|--------|-------------------|
| US-03: Replenishment order submission | Store Integration, Order Management |
| US-04: Wave/batch allocation | Wave Planning, Inventory Module (Allocation) |
| US-05: Send picking tasks to systems | Picking, Automation Integration |
| US-07: Pick confirmation processing | Picking, Automation Integration, Inventory Module |
| US-08: Packing and shipping | Packing & Shipping, Order Management |

---

## Step 4: Design Concepts Selected

| Design Concept | Pros | Cons | Discarded Alternatives |
|----------------|------|------|------------------------|
| **State Machine Pattern** for Order and Wave lifecycle management | Explicitly models valid states and transitions; prevents invalid operations; enables clear audit trail; supports pause/resume operations; consistent with Iteration 2 approach for shipments | Requires upfront state definition; state explosion if too many conditions | **Simple status flags**: No transition validation; prone to inconsistent states. **Workflow engine (BPMN)**: Overkill for relatively linear workflows; adds infrastructure complexity |
| **Strategy Pattern** for Wave Planning optimization algorithms | Enables multiple optimization strategies (zone-based, carrier-based, priority-based); runtime strategy selection based on warehouse configuration; easy to add new algorithms; each strategy independently testable | Requires careful interface design; strategy selection logic can become complex | **Single hardcoded algorithm**: Cannot adapt to different warehouse needs; violates Open-Closed principle. **Rule engine**: Higher learning curve; harder to debug complex optimizations |
| **Allocation Service with Soft Reservations** for inventory allocation | Prevents overselling; supports allocation rollback on wave cancellation; enables visibility into committed vs available inventory; handles concurrent wave planning | Adds complexity to inventory queries; requires cleanup of expired reservations | **Hard allocation (immediate deduction)**: Cannot easily undo on wave changes; inventory appears reduced before actual pick. **No reservation (allocate at pick time)**: Risk of stock-outs during picking; poor customer experience |
| **Task Queue Pattern** with priority-based distribution for picking tasks | Load balancing across operators/systems; supports prioritization; fault-tolerant (tasks persist); enables work stealing; natural fit for distributed picking | Requires queue management; potential for task starvation if priorities misconfigured | **Direct assignment to specific operator**: No load balancing; single point of failure. **Broadcast to all**: Inefficient; race conditions on task claiming |
| **Adapter Pattern** for Picking System Integration | Abstracts different picking system protocols; enables support for multiple automation vendors; isolates integration changes; testable with mock adapters | Additional abstraction layer; adapter per system type needed | **Direct integration per system**: Code duplication; changes affect multiple places. **Universal protocol requirement**: Unrealistic; vendors have proprietary APIs |
| **Domain Events** for cross-module communication (OrderReceivedEvent, WaveReleasedEvent, PickCompletedEvent, ShipmentPackedEvent) | Loose coupling between modules; enables parallel processing; supports future extensibility; aligns with event bus from Iteration 1; enables audit logging | Eventual consistency; debugging event flows can be complex | **Synchronous service calls**: Tight coupling; blocking operations; harder to scale. **Shared database tables**: Violates module boundaries; hidden dependencies |
| **Orchestration Pattern** (Saga) for outbound workflow coordination | Manages multi-step process (order → allocate → pick → pack → ship); handles compensation on failures; provides workflow visibility; supports long-running operations | Requires orchestrator component; adds complexity for simple flows | **Choreography only**: Harder to track overall workflow state; complex failure handling. **Two-phase commit**: Not suitable for long-running operations; tight coupling |
| **Bin Packing Algorithm** for carton optimization | Optimizes space utilization; reduces shipping costs; supports multiple carton sizes; can consider weight limits | Computational complexity for large orders; may need heuristics for performance | **Single carton size**: Wasteful; higher shipping costs. **Manual carton selection**: Inconsistent; slower packing process |
| **Optimistic Locking** on Order, Wave, and PickingTask entities | Prevents lost updates during concurrent operations; better performance than pessimistic locking; conflicts are rare and retriable; consistent with Iteration 2 | Requires retry logic; potential for repeated conflicts under high contention | **Pessimistic locking**: Reduces throughput; potential deadlocks. **No locking**: Data corruption risk unacceptable |
| **Message Queue** (RabbitMQ/SQS) for store order intake | Decouples order submission from processing; handles burst traffic; enables retry with dead-letter queue; supports multiple store system formats | Additional infrastructure; message ordering considerations | **Synchronous REST only**: No buffering; store systems blocked during processing; harder to handle spikes. **Direct database insert by stores**: Security risk; tight coupling |

### Design Concept to Driver Mapping

| Driver | Design Concepts Applied |
|--------|------------------------|
| US-03: Replenishment order submission | Message Queue for intake, Domain Events (OrderReceivedEvent), State Machine for Order |
| US-04: Wave/batch allocation | Strategy Pattern for optimization, Allocation Service with Soft Reservations, State Machine for Wave, Orchestration Pattern |
| US-05: Send picking tasks to systems | Task Queue Pattern, Adapter Pattern for picking systems, Domain Events |
| US-07: Pick confirmation processing | Domain Events (PickCompletedEvent), Optimistic Locking, Adapter Pattern |
| US-08: Packing and shipping | Bin Packing Algorithm, State Machine for Shipment, Domain Events (ShipmentPackedEvent) |

---

## Step 5: Instantiation Decisions

| Instantiation Decision | Rationale |
|------------------------|-----------|
| **OrderService** with **OrderStateMachine** managing states: Received → Validated → Allocated → Picking → Picked → Packing → Packed → Shipped | Provides explicit lifecycle management for replenishment orders; validates state transitions; enables tracking order progress through the outbound workflow; supports cancellation at appropriate stages (US-03) |
| **StoreIntegrationAdapter** with **OrderMessageQueue** and **OrderTransformer** | Decouples order intake from processing via message queue; handles burst traffic from 15,000 stores; transforms various store message formats to internal domain model; dead-letter queue for failed messages (US-03) |
| **WavePlanningService** with **WaveStateMachine** (Planning → Allocated → Released → InProgress → Completed) | Manages wave lifecycle; allows iterative wave building; controls when allocation and release can occur; prevents invalid operations like releasing unallocated waves (US-04) |
| **WaveStrategyEngine** with concrete strategies: **ZoneOptimizationStrategy**, **CarrierGroupingStrategy**, **PriorityBasedStrategy**, **WaveBalancingStrategy** | Enables warehouse-specific optimization algorithms; zone strategy minimizes travel distance; carrier strategy aligns with pickup windows; priority strategy ensures urgent orders first; balancing prevents bottlenecks (US-04) |
| **AllocationEngine** with **ReservationService** using Redis for soft reservations | Fast atomic reservations prevent overselling; Redis enables high-throughput concurrent allocations; soft reservations can be released on wave cancellation; database record provides durability (US-04) |
| **InventoryAllocationService** with FIFO/FEFO allocation rules | Determines which specific inventory to allocate; applies rotation rules to minimize expiration waste; integrates with existing Inventory Module from Iteration 2 (US-04) |
| **PickingService** with **PickingTaskStateMachine** (Pending → Assigned → InProgress → Completed/ShortPick) | Manages task lifecycle; supports both automated and manual picking; handles reassignment on short picks; tracks operator assignment (US-05, US-07) |
| **PickingSystemAdapter** with vendor-specific adapters: **VendorAAdapter**, **VendorBAdapter**, **ManualPickAdapter** | Abstracts picking system differences; same interface for automated and manual picking; isolates vendor protocol changes; enables adding new picking systems without core changes (US-05) |
| **PickingSystemQueue** for async task distribution via Message Broker | Reliable delivery to external picking systems; handles system unavailability with retries; confirmation queue for pick completions; decouples WMS from picking system response times (US-05, US-07) |
| Redis-backed task queue for manual picking zones | Fast task distribution to operators; priority-based queue ordering; atomic task claiming prevents duplicate assignments; work stealing across zones if needed (US-05) |
| **ShortPickEvent** triggering **ExceptionHandlingService** for alternative location finding | Automates short pick resolution; finds alternative inventory without manual intervention; creates supplementary tasks; escalates only when no alternatives exist (US-07) |
| **PackingShippingService** with **PackingEngine** and **BinPackingAlgorithm** | Orchestrates packing workflow; bin packing algorithm (First Fit Decreasing) optimizes carton utilization; reduces shipping costs; considers weight and dimension constraints (US-08) |
| **ShipmentStateMachine** managing states: Created → Packing → Packed → Loading → Shipped | Controls shipment progression; prevents completing unpacked shipments; enables tracking packing station progress (US-08) |
| **OutboundOrchestrator** implementing Saga pattern for order-to-ship coordination | Tracks end-to-end workflow state; enables compensation on failures (e.g., wave cancellation releases allocations); provides visibility into order progress; handles long-running workflows |
| Domain Events: **OrderReceivedEvent**, **WaveReleasedEvent**, **PickingTaskCompletedEvent**, **ShipmentCompletedEvent** | Loose coupling between components; enables parallel audit logging; triggers downstream processing (store notification, financial integration); supports future extensibility |
| **Optimistic Locking** on Order, Wave, PickingTask, Carton entities | Prevents lost updates during concurrent operations; better throughput than pessimistic locking; retry logic handles rare conflicts |

---

## Step 6: Design Decisions

| Driver | Decision | Rationale | Discarded Alternatives |
|--------|----------|-----------|------------------------|
| US-03: Store replenishment order submission | Implement **Message Queue-based order intake** with StoreIntegrationAdapter, OrderMessageQueue, and OrderTransformer | Decouples order submission from processing; handles burst traffic from 15,000 stores during peak periods; enables retry with dead-letter queue for failed messages; supports multiple store message formats through transformer | **Synchronous REST API only**: No buffering for traffic spikes; store systems blocked during WMS processing; harder to handle peak loads. **Direct database writes by stores**: Security risk; tight coupling; no validation layer |
| US-03: Store replenishment order submission | Implement **State Machine Pattern** for order lifecycle with states: Received, Validated, Allocated, Picking, Picked, Packing, Packed, Shipped, Cancelled, Rejected | Explicitly models all valid order states and transitions; prevents invalid operations (e.g., cannot ship unpacked order); enables clear progress tracking; supports cancellation at appropriate stages; consistent with Iteration 2 state machine approach | **Simple status flags**: No transition validation; prone to inconsistent states; difficult to enforce business rules. **Workflow engine**: Overkill for order status tracking; adds infrastructure complexity |
| US-03: Store replenishment order submission | Create **OrderService** with validation logic for SKU existence, store validity, and priority calculation | Centralizes order business rules; validates against item master and store registry; calculates priority based on due date and customer tier; publishes domain events for downstream processing | **Validation in adapter**: Would duplicate validation logic; harder to maintain consistent rules. **No validation**: Risk of invalid orders entering the system; downstream failures |
| US-04: Wave/batch allocation | Implement **Strategy Pattern** with WaveStrategyEngine and concrete strategies: ZoneOptimizationStrategy, CarrierGroupingStrategy, PriorityBasedStrategy, WaveBalancingStrategy | Enables warehouse-specific optimization algorithms; zone strategy minimizes picker travel distance; carrier strategy aligns with pickup schedules; priority strategy ensures urgent orders first; easy to add new strategies without modifying existing code | **Single hardcoded algorithm**: Cannot adapt to different warehouse layouts and priorities; one-size-fits-all approach inefficient. **Manual wave building only**: Inconsistent optimization; slower planning; requires expert planners |
| US-04: Wave/batch allocation | Implement **AllocationEngine** with **ReservationService** using Redis for soft reservations | Fast atomic reservations prevent overselling during concurrent wave planning; Redis enables high-throughput operations; soft reservations can be released on wave cancellation without affecting actual inventory; database record provides durability for recovery | **Database-only reservations**: Too slow for high-volume allocation; lock contention during peak planning. **No reservations (allocate at pick)**: Risk of stock-outs during picking; poor customer experience; wasted picking effort |
| US-04: Wave/batch allocation | Create **InventoryAllocationService** with configurable allocation rules (FIFO, FEFO, location priority) | Determines which specific inventory to allocate for each order line; FIFO ensures fair stock rotation; FEFO minimizes expiration waste for perishables; location priority optimizes pick paths; integrates with existing Inventory Module | **Random allocation**: Suboptimal pick paths; no stock rotation; expiration risk. **Oldest inventory only**: May not consider pick path optimization; inflexible |
| US-04: Wave/batch allocation | Implement **WaveStateMachine** with states: Planning, Allocated, Released, InProgress, Completed, Cancelled | Controls wave lifecycle; prevents releasing unallocated waves; enables iterative wave building during Planning state; supports wave cancellation with proper compensation | **No state management**: Risk of releasing incomplete waves; no control over wave lifecycle |
| US-05: Send picking tasks to systems | Implement **Adapter Pattern** with PickingSystemAdapter and vendor-specific adapters (VendorAAdapter, VendorBAdapter, ManualPickAdapter) | Abstracts picking system protocol differences; same interface for automated and manual picking; isolates vendor-specific changes; enables adding new picking systems without modifying core logic; supports heterogeneous picking environments | **Direct integration per vendor**: Code duplication; vendor changes affect multiple places; no abstraction. **Require single protocol**: Unrealistic; vendors have proprietary APIs; limits automation choices |
| US-05: Send picking tasks to systems | Use **Message Queue** (PickingSystemQueue) for async task distribution to external picking systems | Reliable delivery with acknowledgments; handles picking system unavailability with retries; decouples WMS from picking system response times; enables batch task sending for efficiency | **Synchronous API calls**: WMS blocked waiting for picking system; no retry on failure; tight coupling. **Polling by picking systems**: Higher latency; inefficient; picking system must maintain connection |
| US-05: Send picking tasks to systems | Implement **Redis-backed task queue** for manual picking zones with priority ordering | Fast task distribution to operators via handheld devices; priority-based ordering ensures urgent picks first; atomic task claiming prevents duplicate assignments; supports work stealing across zones if needed | **Database polling**: Higher latency; inefficient queries; lock contention. **In-memory queue only**: Lost on restart; no persistence |
| US-07: Pick confirmation processing | Implement **PickingTaskStateMachine** with states: Pending, Assigned, InProgress, Completed, ShortPick, Reassigned, Cancelled | Manages task lifecycle; supports both automated and manual confirmation; handles short picks as explicit state requiring resolution; enables task reassignment; tracks operator assignment for productivity | **Simple complete/incomplete flags**: No support for partial picks; no reassignment workflow; limited visibility |
| US-07: Pick confirmation processing | Create **ShortPickEvent** triggering ExceptionHandlingService for automated alternative location finding | Automates short pick resolution; finds alternative inventory locations without manual intervention; creates supplementary picking tasks; escalates to supervisor only when no alternatives exist; reduces picker waiting time | **Manual supervisor intervention for all shorts**: Slower resolution; supervisor bottleneck; delays picking. **Accept all shorts without resolution**: Reduces fulfillment rate; poor customer experience |
| US-07: Pick confirmation processing | Process confirmations from automated systems via **confirmation message queue** with vendor adapter transformation | Consistent processing regardless of source; vendor adapters handle protocol differences; message queue provides reliable delivery; supports high-volume automated picking environments | **Direct API callbacks**: WMS must expose endpoints to external systems; security concerns; different protocols per vendor |
| US-08: Packing and shipping | Implement **PackingEngine** with **BinPackingAlgorithm** using First Fit Decreasing heuristic | Optimizes carton utilization reducing shipping costs; FFD provides good approximation with O(n log n) performance; considers weight and dimension constraints; supports multiple carton sizes; suggests optimal packing to operators | **Single carton size**: Wasteful for small orders; higher shipping costs. **Manual carton selection**: Inconsistent optimization; slower packing; training burden. **Optimal bin packing**: NP-hard; too slow for real-time packing |
| US-08: Packing and shipping | Implement **ShipmentStateMachine** with states: Created, Packing, Packed, Loading, Shipped | Controls shipment progression; prevents completing unpacked shipments; enables tracking packing station progress; supports dock door assignment for loading | **No state management**: Risk of shipping incomplete orders; no progress visibility |
| US-08: Packing and shipping | Publish **ShipmentCompletedEvent** to trigger store notification and financial integration | Loose coupling with external system integrations; enables parallel notification to stores and financial system; supports future subscribers (analytics, carrier integration); reliable via event bus and message queue | **Synchronous calls to external systems**: Packing blocked by external system latency; failures affect packing workflow. **Batch file export**: Delays in store notification; not real-time |
| US-03, US-04, US-05, US-07, US-08: Workflow coordination | Implement **OutboundOrchestrator** using Saga pattern for order-to-ship coordination | Tracks end-to-end workflow state across multiple services; enables compensation on failures (e.g., wave cancellation releases allocations); provides visibility into order progress; handles long-running workflows spanning hours | **No orchestration**: Difficult to track order progress; manual compensation on failures; no workflow visibility. **Distributed transactions**: Not suitable for long-running operations; tight coupling |
| US-03, US-04, US-05, US-07, US-08: Cross-module communication | Use **Domain Events** (OrderReceivedEvent, WaveReleasedEvent, PickingTaskCompletedEvent, ShipmentCompletedEvent) | Loose coupling between Outbound Module components; enables parallel processing (audit, integration); supports future extensibility; consistent with event-driven approach from Iterations 1 and 2 | **Direct service calls**: Tight coupling; harder to add new subscribers; synchronous blocking |
| US-03, US-04, US-05, US-07, US-08: Data integrity | Apply **Optimistic Locking** on Order, Wave, PickingTask, Carton entities | Prevents lost updates during concurrent operations; better throughput than pessimistic locking; retry logic handles rare conflicts; consistent with Iteration 2 approach | **Pessimistic locking**: Reduces throughput; potential deadlocks. **No locking**: Data corruption risk |
| US-03, US-04, US-05, US-07, US-08: Data access | Implement **Repository Pattern** for OrderRepository, WaveRepository, PickingTaskRepository, OutboundShipmentRepository, CartonRepository | Abstracts database access; enables unit testing with mocks; consistent data access patterns; consistent with Iteration 2 approach | **Direct JPA/JDBC in services**: Tight coupling to data layer; harder to test |

---

## Step 7: Analysis of Design

### Analysis Results

| Driver | Analysis Result | Justification |
|--------|-----------------|---------------|
| **US-03: Submit replenishment orders from store systems** | **Satisfied** | Complete design provided: (1) Message queue-based intake with StoreIntegrationAdapter, OrderMessageQueue for decoupled, reliable order reception; (2) OrderTransformer for multi-format support; (3) OrderService with validation logic; (4) OrderStateMachine for lifecycle management; (5) Sequence diagram 7.8 illustrates the complete flow from store submission through validation and order creation. |
| **US-04: Allocate and release waves/batches for picking** | **Satisfied** | Complete design provided: (1) WavePlanningService orchestrating the planning workflow; (2) WaveStateMachine controlling wave lifecycle (Planning → Allocated → Released); (3) WaveStrategyEngine with four optimization strategies (Zone, Carrier, Priority, Balancing); (4) AllocationEngine with ReservationService using Redis for fast, atomic soft reservations; (5) InventoryAllocationService with FIFO/FEFO rules; (6) Sequence diagram 7.9 illustrates wave creation, allocation, optimization, and release with picking task generation. |
| **US-05: Send picking tasks to picking systems** | **Satisfied** | Complete design provided: (1) PickingService for task generation and management; (2) PickingSystemAdapter with Adapter Pattern abstracting vendor differences; (3) Vendor-specific adapters (VendorAAdapter, VendorBAdapter) for automated systems; (4) ManualPickAdapter with Redis-backed priority queue for manual picking; (5) PickingSystemQueue for async, reliable delivery to external systems; (6) Sequence diagram 7.10 illustrates task distribution to both automated and manual picking systems. |
| **US-07: Pick confirmation processing** | **Satisfied** | Complete design provided: (1) PickingTaskStateMachine with states including ShortPick and Reassigned; (2) Confirmation processing from automated systems via message queue with vendor adapter transformation; (3) Manual confirmation via REST API; (4) ShortPickEvent triggering automated alternative location finding; (5) ExceptionHandlingService for escalation when alternatives unavailable; (6) Inventory updates and reservation release on completion; (7) Sequence diagram 7.11 illustrates both full pick and short pick scenarios with resolution. |
| **US-08: Packing and shipping operations** | **Satisfied** | Complete design provided: (1) PackingShippingService orchestrating the workflow; (2) PackingEngine with BinPackingAlgorithm (First Fit Decreasing) for carton optimization; (3) ShipmentStateMachine controlling shipment progression; (4) Carton management with weight validation; (5) ShipmentCompletedEvent triggering store notification and financial integration; (6) Sequence diagram 7.12 illustrates complete packing and shipping flow including carton packing and external system notification. |

### Summary

| Status | Count | Drivers |
|--------|-------|---------|
| **Satisfied** | 5 | US-03, US-04, US-05, US-07, US-08 |
| **Partially Satisfied** | 0 | — |
| **Not Satisfied** | 0 | — |

---

## Artifacts Produced

The following artifacts were added to `Architecture.md`:

1. **Section 6.2**: Outbound Module Component Diagram with detailed component responsibilities
2. **Section 6.2.1**: Outbound Module Component Responsibilities table
3. **Section 6.2.2**: Integration Module Components (Outbound-Related) table
4. **Section 6.2.3**: Inventory Module Components (Outbound-Related) table
5. **Section 6.2.4**: Order State Machine diagram and state descriptions
6. **Section 6.2.5**: Wave State Machine diagram and state descriptions
7. **Section 6.2.6**: Picking Task State Machine diagram and state descriptions
8. **Section 6.2.7**: Wave Strategy Selection flowchart
9. **Section 6.2.8**: Domain Events - Outbound Operations table
10. **Section 7.8**: Sequence diagram for US-03 (Submit Replenishment Order)
11. **Section 7.9**: Sequence diagram for US-04 (Allocate and Release Wave)
12. **Section 7.10**: Sequence diagram for US-05 (Send Picking Tasks)
13. **Section 7.11**: Sequence diagram for US-07 (Pick Confirmation Processing)
14. **Section 7.12**: Sequence diagram for US-08 (Packing and Shipping)
15. **Section 7.13**: Outbound Workflow Orchestration Overview diagram
16. **Section 9**: Design decisions table for Iteration 3

---

## Items for Future Iterations

| Concern | Target Iteration |
|---------|------------------|
| Store system integration reliability (exactly-once processing) | Iteration 4 |
| Financial system integration for invoicing | Iteration 4 |
| Scalability for 10,000 orders/hour peak load | Iteration 5 |
| Offline operation capability | Iteration 6 |
