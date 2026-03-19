# **WMS Drivers Document**

## **Business Case**

A retail company operates 15,000 stores across Canada (2000 stores) , the US (10000 stores) and Mexico (3000 stores) and handles thousands of SKUs. Stores are served by warehouses, 3 in Canada, 17 in the US and 5 in Mexico. Each warehouse is operated by a Warehouse Management System instance which supports core warehouse operations (inbound, storage, picking, packing, shipping) and integrates with both in-warehouse automation systems (e.g., picking systems, conveyors) and external enterprise systems (store systems for replenishment orders and shipment confirmations, and a corporate financial system for invoicing). A WMS instance corresponds to a regional or warehouse-scoped deployment with independent data and lifecycle, potentially sharing selected platform services. The solution will be deployed in the cloud to support scalability, high availability, and centralized operations, while allowing geographic regions to operate autonomously and remain isolated from failures in other regions.

## **System Requirements**

### **Primary functionality**

| User story ID | Description |
| :---- | :---- |
| US-01 | As a receiving operator, I can receive inbound shipments (from suppliers or returns) and register them in the WMS so that inventory is accurately captured. |
| US-02 | As a warehouse operator, I can put away received goods into storage locations so that stock is organized and available for picking. Different put-away strategies should be supported (size, rotation, etc…) |
| US-03 | As a store system, I can submit replenishment orders to the WMS so that warehouses can plan and execute picking and shipping to stores. |
| US-04 | As a warehouse planner, I can allocate and release waves or batches for picking so that picking work is optimized and balanced. Other picking route optimization methods should also be supported (simple heuristics, zone-based). |
| US-05 | As a picking system, I can receive picking tasks from the WMS so that automated or semi-automated picking can be performed. |
| US-06 | As a warehouse supervisor, I can register and resolve receiving and picking exceptions so that operations can continue without corrupting inventory. |
| US-07 | As a picker, I can confirm picked quantities (manually or via picking system integration) so that actual picked inventory is accurately recorded. |
| US-08 | As a packing/shipping operator, I can pack picked items into cartons or pallets, stage them for loading, and confirm shipment so that outbound shipments are prepared and tracked. |
| US-09 | As a store system, I can receive confirmation and contents information so that store inventory and expected deliveries are updated. |
| US-10 | As the financial system, I can receive shipment and financial-relevant data so that invoicing can be generated. |
| US-11 | As an inventory manager, I can perform cycle counts and full physical counts by location or item, record counted quantities, and have the WMS reconcile differences (creating adjustments with reason codes and approvals when required) so that on-hand inventory remains accurate. |
| US-12 | As an operations manager, I can access dashboards and reports on warehouse performance, throughput, and inventory so that I can monitor SLA and make decisions. |
| US-13 | As an operations manager, I can manage configuration (locations, units of measure, item attributes, user roles) so that the WMS can be tailored to each warehouse. |
| US-14 | As a warehouse supervisor, I can assign inbound and outbound shipments to dock doors so that congestion is minimized. |
| US-15 | As an inventory manager, I can manage inventory statuses (available, reserved, damaged, quarantined) so that only eligible stock is allocated and shipped. |
| US-16 | As a warehouse planner, I can generate and execute internal replenishment tasks so that picking locations remain stocked. |
| US-17 | As a warehouse supervisor, I can plan, assign and reprioritize tasks across operators and systems so that SLAs are met. |
| US-18 | As an auditor, I can trace inventory-affecting actions to users and systems so that compliance requirements are met. |
| US-19 | As an inventory manager, I can manage returns and recalls by identifying, blocking, and dispositioning affected inventory so that non-eligible stock is never allocated or shipped and compliance is ensured. |
| US-20 | As a quality control operator, I can audit picking and receiving activities and record discrepancies so that operational accuracy and quality levels are continuously monitored. |
| US-21 | As an operations manager, I can view labor productivity and quality dashboards so that staffing, training, and SLA decisions can be made. |

### **Quality attribute scenarios**

| ID | Quality Attribute | Scenario | Associated User story ID |
| :---- | :---- | :---- | :---- |
| QA-01 | Scalability | Given peak season where aggregate order volume across all warehouses reaches 10x normal load, when up to 10,000 replenishment orders per hour are submitted from store systems and picking confirmations are processed in parallel, then the WMS instance should handle the load with \< 2 seconds average API response time for 95% of requests and scale horizontally without downtime. | All |
| QA-02 | Availability | Given normal operating conditions across all warehouses, when any single compute node or cloud availability zone becomes unavailable, WMS core operations (receiving, picking, shipping, inventory updates, and integrations) continue to be available per WMS instance, achieving an overall uptime of 99.9%. In the event of a major regional disaster, the system shall ensure a Recovery Point Objective (RPO) of \< 15 minutes and a Recovery Time Objective (RTO) of \< 4 hours. | All |
| QA-03 | Availability | Given a loss of connectivity between the warehouse and the cloud during normal warehouse operation hours, warehouse operations can continue for up to 3 hours, with zero data loss, no duplicate transactions, and full synchronization completed within 30 minutes after connectivity is restored. Inventory and financial consistency must be preserved across systems, but temporary divergence between WMS instances and external systems is acceptable during failures. | All |
| QA-04 | Reliability / Data integrity | Given high-volume interactions with picking systems and store systems, when network interruptions or integration failures occur, then all messages (orders, picks, shipment confirmations, financial events) must be processed exactly once or in an idempotent manner so that inventory and financial data remain consistent across systems. Note: The system is not required to provide real-time global inventory visibility across all WMS instances. | US-03, US-05, US-07, US-09, US-10 |
| QA-05 | Performance | Given a warehouse operator performing operational tasks, when they search inventory or retrieve work queues (receiving, picking, packing), then screens must load within 1 second for 95% of interactive requests to avoid impacting operator productivity. | All |
| QA-06 | Integration | Given existing automation systems (picking systems, conveyors) and enterprise systems (store systems, corporate financial system), when integrating the WMS, all integration must be decoupled via stable APIs and/or event streams so that new systems or instances can be added without requiring core WMS code changes. | US-03, US-05, US-09, US-10 |
| QA-07 | Security / Compliance | Given that warehouse and financial data includes sensitive commercial information (pricing, quantities, partner data), when users or external systems access the WMS, 100% of access must be secured using strong authentication, authorization by role and warehouse, and encrypted communication in transit and at rest. | All |
| QA-08 | Tenant isolation / regionalization | When a WMS instance experiences issues (load spikes, misconfiguration, failure), all other instances must continue to operate independently so that local outages do not cascade globally. | All |
| QA-09 | Operability / Observability | Given distributed cloud deployment and multiple integrations, when issues occur in order processing or integrations (e.g., failed messages, delayed shipments), then operators and support staff must have centralized monitoring, logs, and alerts per warehouse instance and integration so that they can quickly identify and resolve issues. | All |
| QA-10 | Release management / progressive delivery | Given multiple WMS instances across regions and warehouses, when a new WMS version is released, then it must be possible to deploy and run the new version for a selected subset of instances (e.g., all US warehouses) while other instances remain on the previous version, with the ability to gradually expand rollout and quickly rollback without cross-instance downtime or data corruption. | All |

### **Constraints**

| ID | Constraint |
| :---- | :---- |
| C-01 | The WMS must be deployed in a public cloud environment, using managed services where feasible to reduce operational overhead. |
| C-02 | The WMS must be designed for multi-country operation, ensuring compliance with country-specific regulations, including legal documentation requirements and transportation rules for products. Additionally, the system must provide full internationalization and localization support, including multiple languages, regional date and number formats, currencies, and configurable regional parameters. |
| C-03 | The solution must support independent WMS instances, each with its own configuration and data, while allowing some shared services (e.g., monitoring, identity). |
| C-04 | The WMS must integrate with existing store systems for replenishment orders and shipment confirmations, using standard protocols. |
| C-05 | The WMS must integrate with an existing financial system for invoicing-related data using agreed corporate integration patterns and data contracts. |
| C-06 | The WMS must integrate with in-warehouse automation systems (e.g., picking systems) via well-defined APIs or connectors that can tolerate intermittent connectivity in warehouses. |
| C-07 | The solution must comply with corporate security and data protection policies, including encrypted data in transit and at rest, and role-based access control. |

### **Architectural concerns**

| ID | Concern |
| :---- | :---- |
| AC-01 | How to design the WMS as a cloud-native, scalable system that can handle large order volumes and peaks without excessive infrastructure cost. |
| AC-02 | How to partition the system into services or modules (e.g., inbound, inventory, allocation, picking, shipping, integration) to allow independent scaling and evolution while keeping complexity manageable for the engineering team. |
| AC-03 | How to design integrations with store systems, financial systems, and warehouse automation using APIs and/or event-driven mechanisms to ensure reliability, idempotency, and resilience to network issues. |
| AC-04 | How to support multiple WMS instances in the cloud (e.g., multi-tenant vs. single-tenant per instance) while ensuring data isolation, performance isolation, and operational simplicity. |
| AC-05 | How to ensure consistent inventory and shipment data across WMS, store systems, and financial system in the presence of asynchronous messaging and potential failures. |
| AC-06 | How to implement security, identity, and access control across multiple warehouses and roles, including system-to-system integrations. |
| AC-07 | How to provide observability (logging, tracing, metrics, dashboards) that gives end-to-end visibility of order flows, picking operations, and financial messages across all WMS instances. |
| AC-08 | How to support warehouse-specific configuration (layouts, processes, automation integrations) without creating divergent code bases or overly complex configuration management. |

## **Priorities**

The primary user stories are the following:

| User Story ID | Rationale |
| :---- | :---- |
| US-01 | **Business Critical**: Receiving is the entry point for all inventory. Accurate inventory capture is fundamental to warehouse operations and directly impacts downstream processes. **Technical Challenge**: Requires real-time inventory updates, integration with supplier systems, and handling of various shipment types and formats. |
| US-02 | **Business Critical**: Put-away directly affects picking efficiency and warehouse space utilization. **Technical Challenge**: Requires sophisticated algorithms for multiple put-away strategies (size-based, rotation-based, etc.), location optimization, and real-time space management. |
| US-03 | **Business Critical**: Store replenishment orders are the primary driver of warehouse operations. High volume (10,000 orders/hour peak) directly impacts revenue. **Technical Challenge**: Requires high-throughput API design, integration with multiple store systems, order validation, and handling of peak loads with horizontal scaling. |
| US-04 | **Business Critical**: Wave/batch allocation and picking route optimization directly impact operational efficiency, labor costs, and SLA compliance. **Technical Challenge**: Requires complex optimization algorithms (route optimization, zone-based strategies, heuristics), workload balancing, and real-time decision-making. |
| US-05 | **Business Critical**: Integration with picking systems is essential for automated operations and productivity. **Technical Challenge**: Requires real-time task distribution, handling intermittent connectivity, idempotent message processing, and integration with diverse automation systems. |
| US-09 | **Business Critical**: Shipment confirmations to stores directly impact store operations and customer satisfaction. **Technical Challenge**: Requires reliable integration, data consistency guarantees, handling of network failures, and ensuring exactly-once delivery semantics. |
| US-10 | **Business Critical**: Financial integration is mandatory for invoicing and compliance. Errors can have legal and financial consequences. **Technical Challenge**: Requires strict data consistency, audit trails, idempotent processing, and integration with corporate financial systems following established patterns. |
| US-11 | **Business Critical**: Inventory accuracy is fundamental to warehouse operations. Cycle counts and reconciliation prevent costly errors. **Technical Challenge**: Requires complex reconciliation logic, handling of adjustments with approval workflows, reason code management, and maintaining data integrity during counts. |

Quality attribute scenario priorities is the following:

| Scenario ID | Importance to the customer | Implementation difficulty | Primary |
| :---- | :---- | :---- | :---- |
| QA-01 | High \- Directly impacts business operations during peak seasons. 10x load capacity and 10,000 orders/hour are critical for revenue. | High \- Requires horizontal scaling architecture, load balancing, database sharding/partitioning, caching strategies, and zero-downtime scaling. Performance optimization at scale is complex. | Yes |
| QA-02 | High \- 99.9% uptime is essential for continuous operations. RPO \< 15 min and RTO \< 4 hours are critical for disaster recovery and business continuity. | High \- Requires multi-zone deployment, automated failover, data replication, backup strategies, disaster recovery procedures, and maintaining consistency during failures. | Yes |
| QA-03 | High \- Operations must continue during connectivity loss. Zero data loss and preventing duplicate transactions are critical for inventory and financial accuracy. | Very High \- Requires offline-first architecture, local data storage, conflict resolution algorithms, distributed transaction management, eventual consistency patterns, and sophisticated synchronization mechanisms. | Yes |
| QA-04 | High \- Exactly-once processing is critical for inventory and financial data integrity. Errors can cause significant financial and operational impact. | High \- Requires idempotency patterns, message deduplication, distributed transaction coordination, compensating transactions, and handling partial failures across multiple systems. | Yes |
| QA-05 | Medium-High \- Important for operator productivity but less critical than core availability and data integrity. | Medium \- Achievable through caching, query optimization, database indexing, and proper data modeling. Standard performance optimization techniques. | No |
| QA-06 | High \- Enables system evolution and integration flexibility. Critical for long-term maintainability. | Medium \- Well-established patterns (REST APIs, event streaming, message queues). Requires careful API design and versioning strategy. | No |
| QA-07 | High \- Mandatory for compliance and protecting sensitive commercial data. Regulatory requirement. | Medium \- Standard security practices (encryption, authentication, authorization) but requires careful implementation across all access points and integrations. | No |
| QA-08 | High \- Prevents cascading failures across regions. Critical for multi-instance deployment. | Medium-High \- Requires careful architecture for isolation (data, compute, network), resource quotas, and failure boundaries. Achievable with proper cloud-native design. | No |
| QA-09 | Medium-High \- Important for operations and troubleshooting but not blocking for core functionality. | Medium \- Standard observability tools (logging, metrics, tracing, dashboards). Requires instrumentation and aggregation across distributed system. | No |
| QA-10 | Medium-High \- Important for risk management and safe deployments but not critical for initial operations. | Medium-High \- Requires sophisticated deployment infrastructure, feature flags, database migration strategies, and rollback capabilities. Complex but manageable with proper tooling. | No |
