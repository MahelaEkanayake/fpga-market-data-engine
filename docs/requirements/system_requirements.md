# System Requirements Document (SRD)

**Project:** FPGA-Accelerated Real-Time Financial Market Data Processing
Engine\
**Document:** `system_requirements.md`\
**Version:** 0.2\
**Status:** Revised Draft --- Requirements Refinement\
**Target Hardware:** Digilent Arty Z7-20 (Xilinx Zynq-7020)\
**RTL Language:** SystemVerilog

------------------------------------------------------------------------

## Document Control

  -----------------------------------------------------------------------
  Item                                Description
  ----------------------------------- -----------------------------------
  Project type                        FPGA-based market-data processing
                                      prototype

  Primary target                      Arty Z7-20 / Zynq-7020

  Processing partition                FPGA Programmable Logic (PL) with
                                      ARM Cortex-A9 Processing System
                                      (PS)

  Initial instrument count            One

  Initial order capacity              1,024 outstanding orders

  Requirements status                 Provisional; protocol and
                                      architecture decisions remain open
  -----------------------------------------------------------------------

### Revision History

  -----------------------------------------------------------------------
  Version                             Description
  ----------------------------------- -----------------------------------
  0.1                                 Initial engineering requirements

  0.2                                 Clarified project boundaries, event
                                      semantics, analytics, data
                                      integrity, recovery, host
                                      interface, observability, and
                                      verification requirements
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 1. Project Overview

## 1.1 Problem Statement

Financial markets generate continuous streams of market-data messages
describing changes to the state of financial instruments. Depending on
the selected feed protocol, these messages may represent new orders,
order cancellations, order modifications, executions, trades, or other
market events.

A market-data consumer must process the incoming messages in the correct
order and reconstruct the current market state. For an order-level feed,
this includes maintaining the outstanding buy and sell orders that form
an order book.

The reconstructed order book can be used to calculate market information
such as the best bid, best ask, bid-ask spread, market depth, executed
volume, volume-weighted average price (VWAP), and order-book imbalance.

This project aims to develop an FPGA-based hardware accelerator that
consumes an ordered stream of market-data events, maintains an
order-level book for one instrument, and calculates selected market
analytics. The design will be integrated with the ARM processor on the
target Zynq device.

The project is an engineering prototype. It is not intended to implement
a complete exchange or a production trading platform.

## 1.2 Motivation for FPGA Acceleration

General-purpose processors can process market data using software,
multicore execution, SIMD instructions, and optimized data structures.
FPGAs provide a different architectural approach: the processing
pipeline and data paths can be implemented as specialized hardware.

The project will investigate the following potential advantages:

1.  **Pipelining:** Multiple processing stages may operate concurrently
    on different events.
2.  **Hardware parallelism:** Independent calculations and status
    operations may be performed concurrently where dependencies permit.
3.  **Controlled architecture:** The designer can define the processing
    stages, buffering, interfaces, and timing behaviour.
4.  **Specialized data paths:** Hardware can be tailored to the event
    format and selected analytics.
5.  **Predictable processing:** A carefully designed pipeline can
    provide bounded and measurable processing behaviour.

FPGA acceleration is a hypothesis to be evaluated, not an assumed
performance result. The project will measure throughput, latency,
resource utilization, and end-to-end transfer overhead, and compare the
hardware implementation with a software reference implementation.

------------------------------------------------------------------------

# 2. System Scope

## 2.1 Primary Objective

Develop an FPGA accelerator that:

1.  Accepts a stream of market-data events for one financial instrument.
2.  Buffers incoming events and applies backpressure when required.
3.  Decodes and validates event messages.
4.  Detects invalid events and sequence discontinuities.
5.  Reconstructs and maintains an order-level order book.
6.  Processes order additions, cancellations, modifications, and
    executions according to the selected event semantics.
7.  Calculates selected real-time market analytics.
8.  Exposes results, status, configuration, and performance counters to
    a host processor.
9.  Supports simulation-based verification and deployment on the target
    FPGA board.

## 2.2 Initial Scope

The initial implementation is limited to:

-   One financial instrument.
-   One logical order book.
-   An individual-order-level representation.
-   Separate buy and sell sides.
-   Unique order identifiers for active orders.
-   Price ordering and time ordering within each price level.
-   A provisional capacity of at least 1,024 outstanding orders.
-   A defined internal event format, initially independent of any
    particular exchange protocol.
-   A host interface between the FPGA Programmable Logic (PL) and the
    ARM Processing System (PS).
-   Basic order-book analytics and performance monitoring.

The internal event format will be designed first. A real market-data
protocol may be investigated and mapped to this internal format in a
later phase.

## 2.3 Out of Scope

The initial implementation will not:

-   Operate as an actual financial exchange.
-   Accept customer orders and execute trades between buyers and
    sellers.
-   Implement an exchange matching engine.
-   Perform automated or algorithmic trading.
-   Connect directly to a live production exchange feed.
-   Claim production-level reliability, throughput, or latency.
-   Support multiple instruments or multiple independent feeds in the
    initial release.

**Terminology boundary:** The order-book engine reconstructs market
state from market-data events. It does not match incoming customer
orders or decide which orders should trade. Matching-engine
functionality is outside this project's scope.

## 2.4 Assumptions and Constraints

1.  The initial event source is a software-generated feed or testbench.
2.  Events are intended to be applied in feed order.
3.  The selected feed's sequence-number and recovery semantics must be
    understood before claiming correct operation with that feed.
4.  The initial implementation uses one instrument and one order book.
5.  The target is the Digilent Arty Z7-20 board with a Zynq-7020 device.
6.  Performance targets are provisional until the data structures,
    memory architecture, and interface have been evaluated.

------------------------------------------------------------------------

# 3. System Inputs and Event Model

## 3.1 Input Events

The internal event model will initially include the following event
categories:

  -----------------------------------------------------------------------
  Event                   Purpose                 Required effect on the
                                                  order book
  ----------------------- ----------------------- -----------------------
  `ADD`                   Introduces a new order  Create an active order
                                                  with an ID, side,
                                                  price, and quantity

  `CANCEL`                Cancels all or part of  Reduce remaining
                          an existing order       quantity or remove the
                                                  order

  `MODIFY`                Changes attributes of   Apply the
                          an existing order       protocol-defined
                                                  modification rules

  `EXECUTE`               Reports execution       Reduce the order's
                          against an existing     remaining quantity
                          order                   

  `TRADE`                 Reports trade           Update trade statistics
                          information             when the event is
                                                  authoritative and
                                                  applicable
  -----------------------------------------------------------------------

The exact meaning of each event must be defined by the selected
market-data protocol. In particular, `EXECUTE` and `TRADE` must not
cause the same execution to be counted twice.

For the initial internal model, an execution event that references an
existing order is expected to update that order's remaining quantity.
Trade statistics must be updated only from the event or event
combination designated as authoritative by the chosen feed mapping. This
mapping is a protocol-specification decision and must be completed
before RTL implementation.

## 3.2 Provisional Event Fields

The internal event format is expected to contain some or all of the
following fields:

  -----------------------------------------------------------------------
  Field                               Purpose
  ----------------------------------- -----------------------------------
  Event type                          Identifies ADD, CANCEL, MODIFY,
                                      EXECUTE, or TRADE

  Instrument identifier               Identifies the instrument; may be
                                      implicit in the single-instrument
                                      prototype

  Order ID                            Identifies an individual order

  Side                                BUY or SELL, where applicable

  Price                               Order or trade price, where
                                      applicable

  Quantity                            Event quantity

  Sequence number                     Identifies the event's position in
                                      the feed, if provided

  Timestamp                           Source or local timestamp, if
                                      required for measurement
  -----------------------------------------------------------------------

Field widths, signedness, price representation, quantity units, optional
fields, and invalid encodings will be defined in
`market_data_protocol.md`.

## 3.3 Example Events

Example `ADD` event:

``` text
Event Type : ADD
Order ID   : 1001
Side       : BUY
Price      : 100.00
Quantity   : 500
```

Example partial `CANCEL` event:

``` text
Event Type : CANCEL
Order ID   : 1001
Quantity   : 200
```

After the partial cancellation, the order's remaining quantity is
expected to be 300, assuming the event is valid and the selected
protocol defines cancellation in this manner.

These examples describe the conceptual event model, not the final binary
encoding.

------------------------------------------------------------------------

# 4. System Outputs

The system shall make the following information available to the host
processor.

  -----------------------------------------------------------------------
  Output                              Description
  ----------------------------------- -----------------------------------
  Order-book state                    Current active orders on the buy
                                      and sell sides

  Best bid                            Highest active buy price

  Best ask                            Lowest active sell price

  Spread                              Best ask minus best bid, when both
                                      sides are non-empty

  Market depth                        Aggregated remaining quantity at
                                      selected price levels

  Trade information                   Trade price, quantity, and derived
                                      statistics where supported by the
                                      feed mapping

  Order imbalance                     A defined comparison of bid-side
                                      and ask-side quantities

  VWAP                                Volume-weighted average execution
                                      price over a defined measurement
                                      interval

  Event status                        Accepted, rejected, or error status
                                      for processed events

  Feed status                         Sequence continuity and order-book
                                      validity status

  Performance counters                Event counts, FIFO status,
                                      occupancy, and measured processing
                                      statistics
  -----------------------------------------------------------------------

The exact host-access mechanism and register/data layout will be
specified in the hardware architecture and interface documents.

------------------------------------------------------------------------

# 5. Functional Requirements

The following requirements define the expected system behaviour.
Requirements that depend on feed-specific semantics are subject to
confirmation in the Market Data Protocol Specification.

## 5.1 Event Ingress and Validation

  -----------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- -----------------------------------
  FR-01                               The system shall accept market-data
                                      events through the defined input
                                      interface.

  FR-02                               The system shall buffer incoming
                                      events before processing where
                                      required by the architecture.

  FR-03                               The system shall decode and
                                      classify each received event.

  FR-04                               The system shall validate event
                                      type, required fields, and field
                                      ranges before applying an event.

  FR-05                               The system shall detect duplicate
                                      active order IDs on an ADD event.

  FR-06                               The system shall detect CANCEL,
                                      MODIFY, or EXECUTE events that
                                      reference nonexistent orders.

  FR-07                               The system shall reject invalid
                                      events without corrupting the
                                      existing order-book state.
  -----------------------------------------------------------------------

## 5.2 Order-Book Management

  -----------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- -----------------------------------
  FR-08                               The system shall maintain separate
                                      BUY and SELL order collections.

  FR-09                               The system shall add valid orders
                                      to the order book.

  FR-10                               The system shall locate active
                                      orders using their order IDs.

  FR-11                               The system shall support full and
                                      partial cancellation, subject to
                                      the selected event semantics.

  FR-12                               The system shall apply valid order
                                      modifications according to the
                                      selected feed rules.

  FR-13                               The system shall reduce remaining
                                      order quantity when a valid
                                      execution is applied.

  FR-14                               The system shall remove an order
                                      from the active book when its
                                      remaining quantity reaches zero.

  FR-15                               The system shall maintain price
                                      ordering on each side of the book.

  FR-16                               The system shall preserve the
                                      applicable time ordering of orders
                                      at the same price level.
  -----------------------------------------------------------------------

Price-time priority in this document describes how orders are
represented and ordered in the reconstructed book. It does not imply
that the prototype performs trade matching.

## 5.3 Market Analytics

  -----------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- -----------------------------------
  FR-17                               The system shall calculate the best
                                      bid from active BUY orders.

  FR-18                               The system shall calculate the best
                                      ask from active SELL orders.

  FR-19                               The system shall calculate the
                                      bid-ask spread when both sides
                                      contain at least one active order.

  FR-20                               The system shall calculate market
                                      depth by aggregating remaining
                                      quantity at price levels.

  FR-21                               The system shall maintain
                                      executed-volume statistics using
                                      the authoritative execution/trade
                                      event mapping.

  FR-22                               The system shall calculate VWAP
                                      over a defined measurement interval
                                      using executed trades, not
                                      outstanding order quantities.

  FR-23                               The system shall calculate
                                      order-book imbalance using a
                                      defined set of bid and ask depth
                                      levels.

  FR-24                               The system shall make the selected
                                      analytics available to the host
                                      processor.
  -----------------------------------------------------------------------

The formulas, depth window, reset behaviour, numeric precision, and
measurement interval for FR-20 to FR-23 must be finalized in the
architecture or analytics specification. For example, a possible
imbalance definition is
`(bid_quantity - ask_quantity) / (bid_quantity + ask_quantity)`, but the
depth levels included in each quantity must be explicitly defined.
Division-by-zero behaviour must also be specified.

## 5.4 Host Interface and Control

  -----------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- -----------------------------------
  FR-25                               The host processor shall be able to
                                      read system status and order-book
                                      validity.

  FR-26                               The host processor shall be able to
                                      read the supported analytics and
                                      performance counters.

  FR-27                               The host processor shall be able to
                                      configure supported operating
                                      parameters.

  FR-28                               The system shall provide a defined
                                      reset and initialization sequence.

  FR-29                               The system shall expose error and
                                      feed-discontinuity status to the
                                      host processor.
  -----------------------------------------------------------------------

## 5.5 Event Sequencing and Recovery

  -----------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- -----------------------------------
  FR-30                               Where sequence numbers are
                                      supplied, the system shall check
                                      incoming event sequence continuity.

  FR-31                               The system shall detect a missing,
                                      duplicate, or unexpected sequence
                                      number according to the configured
                                      sequence rules.

  FR-32                               When a sequence discontinuity makes
                                      the reconstructed state unreliable,
                                      the system shall mark the order
                                      book invalid.

  FR-33                               The system shall not silently
                                      continue to report an incomplete
                                      order book as valid.

  FR-34                               The system shall expose the
                                      recovery-required condition to the
                                      host processor.

  FR-35                               The system shall support a defined
                                      recovery procedure, initially based
                                      on reset and replay or
                                      snapshot-based reinitialization.
  -----------------------------------------------------------------------

The exact sequence-number width, wraparound behaviour, duplicate
handling, snapshot format, and replay procedure depend on the selected
feed and will be specified later.

------------------------------------------------------------------------

# 6. Data Integrity Requirements

The following invariants shall hold whenever the order book is reported
as valid:

1.  Every active order ID shall be unique.
2.  Every active order shall have a remaining quantity greater than
    zero.
3.  An order whose remaining quantity reaches zero shall not remain
    active.
4.  A successfully applied event shall leave the order book in a valid
    state.
5.  The best bid shall equal the highest active BUY price.
6.  The best ask shall equal the lowest active SELL price.
7.  Orders at the same price level shall preserve the time-ordering
    rules defined for the selected feed.
8.  A rejected event shall not partially modify the order book.
9.  The system shall not report the order book as valid after an
    unrecovered sequence discontinuity.
10. Trade statistics shall not count the same execution more than once.

If one side of the book is empty, the corresponding best price shall be
reported as unavailable using a defined status or validity bit. The
spread shall be marked unavailable unless both best bid and best ask are
valid.

------------------------------------------------------------------------

# 7. Error Handling and Fault Management

## 7.1 Invalid Events

Examples of invalid events include:

-   ADD with an order ID that is already active.
-   CANCEL for a nonexistent order.
-   MODIFY for a nonexistent order.
-   EXECUTE for a nonexistent order.
-   Zero or out-of-range quantities where not permitted.
-   Invalid side or event type.
-   Malformed or incomplete messages.
-   An event whose fields violate the selected protocol rules.

Expected behaviour:

1.  Detect the invalid event.
2.  Reject it without corrupting the order book.
3.  Record an error code or status.
4.  Increment the appropriate error counter.
5.  Make the error status available to the host.
6.  Continue processing later events only when doing so is safe for the
    integrity of the reconstructed state.

## 7.2 FIFO and Backpressure

The input path shall include buffering appropriate to the selected
architecture.

The design shall:

-   Expose FIFO full and empty status internally.
-   Apply backpressure when the downstream processing path cannot accept
    an event.
-   Detect and count overflow conditions.
-   Avoid silently discarding events.
-   Define what happens if the upstream source cannot be stalled and the
    buffer becomes full.
-   Expose relevant FIFO status and overflow information to the host.

If an event is lost, the system shall not continue to present the
resulting order book as valid.

## 7.3 Sequence Discontinuity and Recovery

For a feed that uses monotonically increasing sequence numbers, a gap
shall be detected according to the feed's sequence rules.

For example, receiving sequence numbers 100, 101, 102, and then 104
indicates that sequence 103 may be missing, assuming no wraparound or
protocol-specific exception applies.

When a gap is confirmed:

1.  Mark the order book invalid.
2.  Record the expected and observed sequence information where
    practical.
3.  Notify the host processor.
4.  Stop presenting derived book data as valid.
5.  Recover using the procedure defined for the feed, such as
    reinitialization from a snapshot followed by replay of subsequent
    events.

The initial prototype may implement a simplified reset-and-replay
recovery procedure. Full exchange-grade recovery is not an initial
requirement.

## 7.4 Reset and Initialization

The design shall define:

-   Reset behaviour for all stateful modules.
-   Initialization of order-book storage and valid bits.
-   Reset values for status and performance counters.
-   The condition under which the system asserts that it is ready to
    accept events.
-   The condition under which the order book becomes valid.

------------------------------------------------------------------------

# 8. Performance Requirements

All performance targets in this section are provisional engineering
goals. They must be validated against the selected order-book data
structure, FPGA resource capacity, memory access pattern, and
host-interface overhead.

  -----------------------------------------------------------------------
  Parameter                      Initial Target            Stretch Target
  ------------------- ------------------------- -------------------------
  Sustained event            1 million events/s       10 million events/s
  throughput                                    

  Outstanding orders                      1,024                     4,096

  FPGA clock                            100 MHz       200 MHz, subject to
  frequency                                                timing closure

  Processing latency   Measure and characterize Optimize and characterize

  Processing               Functionally correct           Pipelined where
  architecture                                                 beneficial
  -----------------------------------------------------------------------

## 8.1 Throughput

Throughput is the number of valid input events processed per second over
a defined measurement interval.

The initial target is 1 million events per second. The stretch target is
10 million events per second.

The benchmark shall state the event mix, order-book occupancy, input
pattern, clock frequency, and whether host-transfer overhead is
included.

## 8.2 Latency

Processing latency shall initially be measured from the acceptance of an
event at the defined hardware ingress interface to the point at which
the corresponding order-book update is committed and its completion
status is available internally.

The measurement boundary must be documented. End-to-end latency
including host software and PS-PL transfers shall be measured separately
where applicable.

Average, maximum observed, and distribution-based latency measurements
should be reported. The project shall not claim a worst-case bound
unless the design and measurement method justify it.

## 8.3 Order Capacity

The initial design shall target at least 1,024 outstanding orders. The
architecture should be evaluated for possible expansion to 4,096 orders.

Capacity refers to simultaneously active orders, not the total number of
events processed over time.

## 8.4 Clock Frequency

The initial target clock is 100 MHz. A 200 MHz stretch target may be
investigated if timing closure and resource utilization permit.

A higher clock frequency does not automatically guarantee higher event
throughput. Memory access dependencies, pipeline stalls, and
event-processing cycles must also be considered.

------------------------------------------------------------------------

# 9. Observability and Diagnostics

The system shall provide sufficient diagnostic information to support
verification and performance analysis.

The initial set of counters and status signals should include:

  -----------------------------------------------------------------------
  Diagnostic                          Purpose
  ----------------------------------- -----------------------------------
  Accepted event count                Number of events accepted for
                                      processing

  Processed event count               Number of events successfully
                                      applied

  Rejected event count                Number of invalid or unsupported
                                      events

  Event counts by type                ADD, CANCEL, MODIFY, EXECUTE, and
                                      TRADE counts

  FIFO occupancy                      Current or maximum observed
                                      input-buffer occupancy

  FIFO overflow count                 Number of overflow events

  Sequence error count                Number of detected sequence
                                      discontinuities

  Active order count                  Number of currently active orders

  Processing latency                  Measured event-processing latency

  Order-book valid flag               Indicates whether the reconstructed
                                      state is trusted

  System ready flag                   Indicates that initialization has
                                      completed
  -----------------------------------------------------------------------

The final counter widths, clear-on-read or explicit-clear behaviour, and
host register mapping will be defined in the hardware interface
specification.

------------------------------------------------------------------------

# 10. Target Hardware and Development Environment

  -----------------------------------------------------------------------
  Component                           Selection
  ----------------------------------- -----------------------------------
  FPGA board                          Digilent Arty Z7-20

  FPGA device                         Xilinx Zynq-7020

  RTL language                        SystemVerilog

  RTL simulation                      Verilator

  Waveform analysis                   GTKWave

  Synthesis and implementation        Xilinx Vivado

  Host processor                      ARM Cortex-A9 Processing System

  On-chip memory                      Block RAM, subject to architecture

  Host communication                  AXI interfaces

  Event ingress for initial prototype AXI-Stream, subject to interface
                                      design

  Host control/status interface       AXI4-Lite or another suitable AXI
                                      interface
  -----------------------------------------------------------------------

The FPGA Programmable Logic is expected to perform event ingress,
decoding, order-book updates, analytics, and performance monitoring. The
ARM Processing System is expected to handle configuration, software
control, result access, test orchestration, and reporting.

The exact division of responsibilities and memory architecture will be
finalized in the Hardware Architecture Document.

Development shall begin with simulation and proceed to synthesis,
implementation, and physical-board testing.

------------------------------------------------------------------------

# 11. Verification Requirements

The system shall be verified at module and system level before being
considered complete.

## 11.1 Functional Verification

Verification shall include:

1.  ADD event processing.
2.  Full and partial CANCEL processing.
3.  MODIFY processing for each supported modification case.
4.  EXECUTE processing and remaining-quantity updates.
5.  TRADE processing and trade-statistics updates.
6.  Duplicate order-ID detection.
7.  Unknown order-ID handling.
8.  BUY and SELL book separation.
9.  Price ordering and time ordering at equal price levels.
10. Best bid and best ask calculations.
11. Spread calculation, including empty-side conditions.
12. Market-depth aggregation.
13. VWAP calculation and interval behaviour.
14. Order-imbalance calculation and zero-denominator behaviour.
15. Rejection of malformed or invalid events.
16. Confirmation that rejected events do not corrupt state.

## 11.2 Interface and Fault Verification

Verification shall include:

1.  FIFO full and empty conditions.
2.  Backpressure behaviour.
3.  FIFO overflow handling.
4.  Reset and initialization.
5.  Sequence continuity.
6.  Missing, duplicate, and unexpected sequence numbers.
7.  Order-book invalidation after an unrecovered feed gap.
8.  Recovery and replay behaviour supported by the prototype.
9.  Host register or interface access.
10. Status and diagnostic counter behaviour.

## 11.3 Performance Verification

The design shall be tested for:

-   Sustained throughput under a documented event mix.
-   Processing latency under representative workloads.
-   Performance at different order-book occupancies.
-   FIFO occupancy under burst traffic.
-   Resource utilization after synthesis and implementation.
-   Timing closure at the target clock frequency.
-   End-to-end performance including PS-PL transfer overhead where
    applicable.

## 11.4 Software Reference Model

A software reference implementation should be developed independently of
the RTL. It will process the same event sequences and provide expected
order-book and analytics results.

The RTL simulation results shall be compared against the reference model
for deterministic test sequences and randomized test sequences.

The reference model must use the same explicitly documented event
semantics as the hardware design.

------------------------------------------------------------------------

# 12. Potential Industry Relevance

Low-latency market-data processing is relevant to market-data
infrastructure providers, exchanges, trading firms, and financial
institutions.

Organizations operating in this broad industry include LSEG, Nasdaq,
Cboe Global Markets, and CME Group. Their actual systems, protocols,
performance requirements, and architectures are not assumed to be known
or replicated by this project.

The prototype is intended to demonstrate engineering concepts relevant
to high-performance market-data processing. It must not be described as
an implementation of any named organization's internal system.

------------------------------------------------------------------------

# 13. Future Extensions

Potential future extensions include:

-   Multiple financial instruments.
-   Multiple market-data feeds.
-   Mapping to a real exchange market-data protocol.
-   Ethernet-based event reception.
-   Snapshot and replay recovery.
-   Multi-feed normalization and sequencing.
-   More advanced order-book data structures.
-   Multiple parallel order-book processing engines.
-   External DDR memory integration.
-   Additional hardware-accelerated analytics.
-   Comparison of CPU-only, FPGA-only, and heterogeneous CPU-FPGA
    implementations.
-   More extensive performance and reliability testing.

These extensions are not part of the initial implementation unless
formally added through a requirements revision.

------------------------------------------------------------------------

# 14. Development Milestones

  -----------------------------------------------------------------------
  Milestone               Description             Expected output
  ----------------------- ----------------------- -----------------------
  M1                      Requirements refinement Approved system
                                                  requirements

  M2                      Market-data protocol    Protocol and internal
                          investigation           event specification

  M3                      Hardware architecture   Block diagram and
                          design                  interface definitions

  M4                      Order-book data         Data-structure and
                          structure design        memory-access design

  M5                      Software reference      Executable model and
                          model                   test vectors

  M6                      RTL implementation      Synthesizable
                                                  SystemVerilog modules

  M7                      Functional verification Self-checking
                                                  testbenches and
                                                  comparison results

  M8                      AXI and PS-PL           Host-accessible FPGA
                          integration             subsystem

  M9                      FPGA deployment         Working implementation
                                                  on Arty Z7-20

  M10                     Performance             Measured throughput,
                          optimization            latency, and resource
                                                  report
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 15. Open Decisions and Items to Resolve

The following items must be resolved before the corresponding
implementation work is finalized.

  -------------------------------------------------------------------------
  ID                Open decision     Why it matters      Planned
                                                          resolution
  ----------------- ----------------- ------------------- -----------------
  OD-01             Which market-data Determines event    Protocol
                    protocol will be  semantics,          investigation
                    used as the       sequencing, and     
                    reference?        field mapping       

  OD-02             What is the       Prevents incorrect  Protocol
                    authoritative     remaining           specification
                    relationship      quantities and      
                    between EXECUTE   double-counted      
                    and TRADE         trades              
                    messages?                             

  OD-03             What are the      Some feeds may      Protocol
                    exact MODIFY      treat price or      specification
                    rules?            quantity changes as 
                                      priority-changing   
                                      events              

  OD-04             What price and    Determines field    Internal event
                    quantity          widths, precision,  specification
                    representations   and arithmetic      
                    will be used                          
                    internally?                           

  OD-05             What data         Dominates memory    Data-structure
                    structure will    architecture and    design
                    support order-ID  update latency      
                    lookup and                            
                    price-time                            
                    ordering?                             

  OD-06             How will best bid Affects update      Architecture
                    and best ask be   complexity and      design
                    maintained?       latency             

  OD-07             What depth window Defines the         Analytics
                    will be used for  analytics           specification
                    market imbalance? calculation         

  OD-08             What interval and Defines             Analytics
                    reset policy will trade-statistics    specification
                    be used for VWAP? state               

  OD-09             What is the exact Determines          Interface
                    AXI ingress and   integration and     specification
                    host-control      backpressure        
                    interface?        behaviour           

  OD-10             What recovery     Determines          Recovery design
                    mechanism will be behaviour after a   
                    implemented in    feed gap            
                    the first                             
                    prototype?                            

  OD-11             How will latency  Ensures meaningful  Verification plan
                    be measured in    and reproducible    
                    simulation and    performance claims  
                    hardware?                             
  -------------------------------------------------------------------------

------------------------------------------------------------------------

## 16. Related Project Documents

The following documents will define details that are intentionally not
fixed in this SRD:

  -----------------------------------------------------------------------
  Document                            Responsibility
  ----------------------------------- -----------------------------------
  `market_data_protocol.md`           Internal event format, message
                                      fields, event semantics,
                                      sequencing, and protocol mapping

  `hardware_architecture.md`          PL/PS partition, processing
                                      pipeline, memories, modules, and
                                      AXI interfaces

  `order_book_design.md`              Order storage, ID lookup, price
                                      levels, and time priority

  `verification_plan.md`              Test strategy, test cases,
                                      reference-model comparison, and
                                      coverage

  `performance_report.md`             Benchmark methodology and measured
                                      results
  -----------------------------------------------------------------------

------------------------------------------------------------------------

**Document Status:** Version 0.2 --- Revised Requirements Draft\
**Next Recommended Document:** `market_data_protocol.md`
