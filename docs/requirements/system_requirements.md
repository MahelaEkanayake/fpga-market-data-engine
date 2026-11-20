# System Requirements Document (SRD)

**Project:** FPGA-Based Financial Market Data Processing Engine  
**Version:** 0.1  
**Status:** Initial Draft  
**Target Hardware:** Digilent Arty Z7-20 (Xilinx Zynq-7020)  
**RTL Language:** SystemVerilog

---

# 1. Project Overview

## 1.1 Problem Statement

Financial exchanges generate enormous volumes of order-related events. These events must be processed to maintain an accurate representation of the market's outstanding orders and calculate market information with low latency.

In financial markets, participants submit orders to buy or sell financial instruments at specified prices and quantities.

- A bid represents buying interest.
- An ask represents selling interest.
- A trade occurs when compatible buying and selling orders are executed.

Financial exchanges continuously receive new orders, cancellations, modifications and execution reports.

Processing these events requires efficient data structures, memory management and computational architectures.

This project aims to develop an FPGA-based hardware accelerator capable of reconstructing an order book from individual market-data events and calculating real-time market analytics.

## 1.2 Motivation for FPGA Acceleration

Traditional software implementations typically use general-purpose processors (CPUs) to process financial-market data.

Modern CPUs support parallel execution through multiple cores, SIMD instructions and other architectural techniques.

However, FPGAs offer the ability to construct specialized hardware architectures optimized for particular workloads.

The primary motivations for using an FPGA are:

1. **Hardware parallelism:** Multiple independent operations can execute concurrently.
2. **Pipelining:** Different processing stages can operate on different events simultaneously.
3. **Deterministic architecture:** Processing behaviour and timing can be controlled through hardware design.
4. **Specialized processing:** Hardware resources can be dedicated to specific operations.
5. **Low-latency processing:** Carefully designed hardware pipelines can minimize processing delays.

The project will investigate how these characteristics can improve financial-market data processing.

---

# 2. System Scope

## 2.1 Primary Objective

Develop an FPGA accelerator that:

1. Receives individual market-data events.
2. Parses and validates incoming messages.
3. Reconstructs and maintains an order book.
4. Processes order additions, cancellations, modifications and executions.
5. Calculates real-time market statistics.
6. Makes processed information available to a host processor.

## 2.2 Initial Scope

The initial implementation will focus on:

- A single financial instrument.
- One order book.
- Individual order-level events.
- Price-time priority representation.
- Order ID lookup.
- Buy and sell order management.
- Basic market analytics.

Multiple instruments and multiple market-data feeds will be considered future extensions.

## 2.3 Out of Scope

The initial implementation will NOT:

- Operate as an actual financial exchange.
- Accept customer trading orders for execution.
- Connect directly to a live production exchange.
- Perform automated financial trading.
- Guarantee production-level financial infrastructure performance.

---

# 3. System Inputs

The system will receive market-data messages representing individual order-related events.

The initial internal event categories are:

| Event | Description |
|---|---|
| ADD | Introduces a new order |
| CANCEL | Cancels an existing order or part of its quantity |
| MODIFY | Modifies an existing order |
| EXECUTE | Reports execution of an existing order |
| TRADE | Reports trade information |

The internal event representation will eventually be mapped to the actual message types of the selected market-data protocol.

### Example input

ADD event:

    Event Type: ADD
    Order ID: 1001
    Side: BUY
    Price: 100.00
    Quantity: 500

CANCEL event:

    Event Type: CANCEL
    Order ID: 1001
    Quantity: 200

The exact message encoding and field widths will be defined in the Market Data Protocol Specification.

---

# 4. System Outputs

The system will provide the following information.

| Output | Description |
|---|---|
| Order Book | Current outstanding buy and sell orders |
| Best Bid | Highest available buying price |
| Best Ask | Lowest available selling price |
| Spread | Difference between best ask and best bid |
| Market Depth | Available quantities at different price levels |
| Trade Information | Reported order executions |
| Event Status | Processing success, rejection or error information |
| Order Imbalance | Difference between buying and selling interest |
| VWAP | Volume-weighted average execution price |

The output interface will allow a host processor to access relevant market information.

---

# 5. Potential Industry Applications

Potential users of this technology include:

- London Stock Exchange Group (LSEG).
- Nasdaq.
- Cboe Global Markets.
- CME Group.
- High-frequency trading firms.
- Financial market-data infrastructure providers.
- Investment banks.

These organizations have different technical requirements, but low-latency market-data processing is relevant across several of these industries.

The project is intended as an engineering prototype exploring technologies relevant to these applications.

---

# 6. Performance Requirements

The initial performance targets are provisional and will be evaluated against the capabilities of the target FPGA.

| Parameter | Initial Target | Stretch Target |
|---|---:|---:|
| Sustained throughput | 1 million events/s | 10 million events/s |
| Outstanding orders | 1,024 | 4,096 |
| Processing latency | Measure and characterize | Optimize |
| FPGA clock | 100 MHz | 200 MHz, subject to timing closure |
| Processing architecture | Functionally correct | Pipelined |

## 6.1 Throughput

Throughput represents the number of market-data events that can be processed per second.

The initial objective is to sustain one million events per second.

## 6.2 Latency

Latency represents the time required to process an individual event.

Latency will be measured from the acceptance of an input event to the completion of its corresponding order-book update.

Both average and worst-case latency will be investigated.

## 6.3 Order Capacity

The initial implementation should support at least 1,024 outstanding orders.

The architecture should allow future expansion to 4,096 outstanding orders.

## 6.4 Clock Frequency

The initial target operating frequency is 100 MHz.

A stretch target of 200 MHz will be investigated depending on resource utilization and timing closure.

---

# 7. Functional Requirements

The system must implement the following functionality.

| ID | Requirement |
|---|---|
| FR-01 | Accept incoming market-data events |
| FR-02 | Decode and classify incoming events |
| FR-03 | Validate incoming event fields |
| FR-04 | Add new orders to the order book |
| FR-05 | Locate orders using their unique order IDs |
| FR-06 | Cancel existing orders |
| FR-07 | Modify existing orders |
| FR-08 | Update remaining quantities after executions |
| FR-09 | Maintain separate buy and sell books |
| FR-10 | Maintain price-time priority |
| FR-11 | Calculate best bid and best ask |
| FR-12 | Calculate bid-ask spread |
| FR-13 | Calculate market depth |
| FR-14 | Provide processed information to the host processor |

---

# 8. Data Integrity Requirements

The order book must maintain a consistent internal state.

The following invariants must hold:

1. An active order ID must be unique.
2. An order's remaining quantity must never become negative.
3. Fully executed or cancelled orders must not remain active.
4. The best bid must represent the highest active bid price.
5. The best ask must represent the lowest active ask price.
6. Orders at the same price level must respect the applicable time-priority rules.
7. Every successfully processed event must leave the order book in a valid state.

These requirements will be used during functional verification and hardware testing.

---

# 9. Error Handling and Fault Management

The system must detect invalid or unexpected events without unnecessarily interrupting normal processing.

### 9.1 Invalid Events

Examples include:

- Cancellation of a nonexistent order.
- Modification of a nonexistent order.
- Duplicate order IDs.
- Invalid quantities.
- Malformed messages.

Expected behaviour:

1. Detect the invalid event.
2. Generate an error status.
3. Record relevant error information.
4. Avoid corrupting the order book.
5. Continue processing subsequent valid events.

### 9.2 Input Buffer Management

The system must use buffering to accommodate temporary bursts of incoming events.

The architecture should investigate:

- FIFO buffering.
- Backpressure.
- FIFO overflow detection.
- Input event sequencing.
- Recovery from missing market-data messages.

Market-data events must not be silently discarded.

If an event is lost, the system must detect the condition and prevent an incomplete order book from being treated as valid.

---

# 10. Target Hardware and Development Environment

| Component | Selection |
|---|---|
| FPGA Board | Digilent Arty Z7-20 |
| FPGA Device | Xilinx Zynq-7020 |
| RTL Language | SystemVerilog |
| Simulation | Verilator |
| Waveform Analysis | GTKWave |
| Synthesis and Implementation | Xilinx Vivado |
| Host Processor | ARM Cortex-A9 |
| Internal Memory | Block RAM |
| Host Communication | AXI interfaces |

The design will initially be developed and verified through simulation before deployment to the physical FPGA.

---

# 11. Verification Requirements

The implementation must be verified through simulation and hardware testing.

Verification should include:

1. Correct processing of ADD events.
2. Correct processing of CANCEL events.
3. Correct processing of MODIFY events.
4. Correct processing of EXECUTE events.
5. Correct maintenance of price-time priority.
6. Correct best bid and ask calculations.
7. Correct spread calculations.
8. Detection of invalid events.
9. Handling of simultaneous or conflicting requests.
10. FIFO overflow and backpressure testing.
11. Throughput measurements.
12. Latency measurements.

A software reference implementation may be developed to compare the FPGA's results against an independently implemented order book.

---

# 12. Future Extensions

Potential future improvements include:

- Multiple financial instruments.
- Multiple market-data feeds.
- Ethernet-based market-data reception.
- More advanced order-book structures.
- Parallel order-processing engines.
- Hardware-accelerated market analytics.
- External DDR memory integration.
- Multi-feed market-data normalization.
- Performance comparison between CPU and FPGA implementations.

---

# 13. Development Milestones

| Milestone | Description |
|---|---|
| M1 | System requirements |
| M2 | Market-data protocol investigation |
| M3 | Hardware architecture design |
| M4 | Order-book data structure design |
| M5 | SystemVerilog implementation |
| M6 | Functional verification |
| M7 | AXI integration |
| M8 | FPGA deployment |
| M9 | Performance optimization and benchmarking |

---

**Document Status:** Version 0.1 — Initial Engineering Requirements

**Next Document:** `market_data_protocol.md`
