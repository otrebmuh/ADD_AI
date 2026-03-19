## Attribute driven design

Attribute Driven Design is an iterative process to design software architectures. The steps of the process are the following.

### Step 2: Establish goal for the iteration by selecting drivers

The goal of the iteration has been defined in the IterationPlan.md document. Review the goal and associated drivers.

### Step 3: Choose one or more elements of the system to refine

Consider the diagrams in the Architecture.md document and identify elements that will need to be refined to satisfy the drivers associated with the goal of the iteration.
Refinement can mean decomposition into finer-grained elements (top-down approach), combination of elements into coarser-grained elements (bottom-up approach), or improvement of previously identified elements. For greenfield development, in the first iteration, the only element to refine is the whole system.

Answer with the list of elements to be refined and wait for user review.

### Step 4: Choose One or More Design Concepts That Satisfy the Selected Drivers
In this step, design concepts are selected to achieve the iteration goal, these design concepts may include:
 - Design patterns: including reference architectures, architectural patterns, design patterns and deployment patterns
 - Externally developed components: including frameworks or specific cloud resources
 - Tactics: These are proven strategies to address particular quality attributes

 Answer to the user the design concepts that you select along with their pros, cons and alternatives that are discarded. Show this in a table in markdown format. Wait for user to review.

|Design concept|Pros|Cons|Discarded alternatives|
|---|---|---|---|
| | | | |


### Step 5: Instantiate Architectural Elements, sketch views, allocate Responsibilities, and Define Interfaces
In this step, the selected design concepts are adapted to address the drivers that compose the goal of the iteration, that is the meaning of instantiation. This may result in new elements created or existing elements changed. 

Modify the diagrams in the Architecture.md file.


For the definition of interfaces, first create sequence diagrams that illustrate how the instantiated elements collaborate to support the user stories or quality attribute scenarios chosen in the goal of the iteration. Add these sequence diagrams to the corresponing section of the architecture document.

Answer with instantiation decisions in a table:

|Instantiation decision|Rationale|
|---|---|

Wait for user review.

### Step 6: Record Design Decisions
Design decisions are documented using table like the following one:

|Driver|Decision|Rationale|Discarded alternatives|
|---|---|---|---|

Record these decisions in section 9 of the Architecture.md file. Wait for user review.

### Step 7:Perform Analysis of Current Design and Review Iteration Goal and Achievement of Design Purpose
For this step, you must analyze if the design decisions made during the iteration were sufficient to address the drivers associated with the iteration goal. Answer with a table like the following:

|Driver|Analysis result|
|---|---|

The analysis result can have different states:
 - Satisfied: If sufficient design decisions have been made to satisfy the driver
 - Partially satisfied: If some design decisions were made but more decisions are necessary
 - Not satisfied: If no design decisions were made during the iteration to satisfy the driver



