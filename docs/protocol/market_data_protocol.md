# Market Data Protocol Specification

**Project:** FPGA-Accelerated Real-Time Financial Market Data Processing Engine  
**Document:** `market_data_protocol.md`  
**Status:** Working draft — Decisions 1–80 captured  
**Revision:** Draft 0.2

---

## 1. Purpose and scope

This document records the agreed protocol and interface decisions for the market-data consumer. The system receives exchange-published market-data messages, reconstructs an order book, calculates market analytics, and reports status and diagnostics.

> **Scope boundary:** This project is a market-data consumer, not an exchange matching engine. It reconstructs the displayed order book from published feed events; it does not match customer orders or execute trades.

The initial scope is one instrument and one order book. Multiple instruments and feed support may be considered later.

### 1.1 Protocol layering

| Layer | Selected technology | Responsibility |
|---|---|---|
| Feed delivery | UDP multicast | Delivers feed packets to subscribers. |
| Transport framing and sequencing | MoldUDP64 | Carries session information, starting sequence number, message count, and length-prefixed messages. |
| Market-data message format | Nasdaq TotalView-ITCH 5.0 | Defines message types, fields, and market-data semantics. |
| ARM-to-FPGA event interface | AXI4-Stream | Transfers normalized events from ARM to FPGA. |
| FPGA control and status | AXI4-Lite | Configuration, status, recovery control, and diagnostics. |

ITCH defines the market-data message semantics. MoldUDP64 defines the UDP delivery framing and packet-level sequencing. These are separate protocol layers.

### 1.2 Initial message scope

The initial implementation focuses on order-book messages and essential market-status/instrument messages required by the selected feed scope.

| ITCH message | Purpose |
|---|---|
| `A` | Add Order (no MPID attribution) |
| `F` | Add Order with MPID attribution |
| `E` | Order Executed |
| `C` | Order Executed With Price |
| `X` | Order Cancel |
| `D` | Order Delete |
| `U` | Order Replace |
| `P` | Trade (Non-Cross) |

The exact supported message subset and treatment of additional system/status messages must be finalized against the selected ITCH specification and implementation requirements.

### 1.3 Important market-data semantics

- The order book stores active displayed orders. The matching engine is outside this project's scope.
- `A`/`F` adds a displayed order.
- `X` reduces an order's remaining quantity.
- `D` removes the remaining quantity of an order.
- `U` replaces an existing order using a new order reference number and replacement attributes.
- `E` and eligible `C` messages reduce the resting order quantity because an execution occurred.
- `P` represents a non-cross trade that is not associated with a displayed order; it may contribute to trade analytics but does not add or remove a displayed order.
- VWAP is based on eligible executed-trade data, not outstanding order quantity.
- Execution and trade messages must be interpreted according to ITCH semantics so trade volume and VWAP are not double-counted. Non-printable execution cases must be handled according to the feed specification.
- ITCH price fields use integer representation with implied decimal precision. Use integer arithmetic rather than floating-point arithmetic.
- ITCH fields are encoded in network byte order (big-endian).
- ITCH timestamps and local packet-reception timestamps are distinct. Local reception time is initially captured on the ARM side.

---

## 2. Protocol and feed design decisions

### 2.1 Protocol selection and message scope — Decisions 1–4

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 1 | Nasdaq TotalView-ITCH 5.0 | Use a documented, established market-data message format. |
| 2 | UDP multicast with MoldUDP64 | Use multicast for feed delivery and MoldUDP64 for packet framing and sequencing. |
| 3 | Core order-book messages plus essential market-status/instrument messages | Implement the subset needed for the initial single-instrument consumer. |
| 4 | Skip unsupported but correctly framed messages | Preserve feed progression. Handle malformed or truncated data separately. |

### 2.2 Instrument selection and book lifecycle — Decisions 5–8

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 5 | ARM-side instrument filtering | Filter after transport-level sequencing and required feed-level processing. |
| 6 | One active instrument initially | Maintain one active instrument and one order book in the first implementation. |
| 7 | Support runtime instrument switching | Allow the selected instrument to be changed while running. |
| 8 | Reconstruct the book after an instrument switch | Obtain a synchronized snapshot/reconstruction before declaring the new book valid. |

### 2.3 Sequence tracking and recovery — Decisions 9–16

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 9 | Support incremental and snapshot recovery | Use feed-supported incremental recovery where available; snapshot recovery is the fallback. |
| 10 | Preserve the existing book but mark it invalid during recovery | Old state may remain available for diagnostics, but must not be represented as current and valid. |
| 11 | Display stale analytics with an explicit status | Host software may show last-known analytics only when clearly marked stale/invalid. |
| 12 | Preserve feed sequence information in normalized events | Keep sequence information associated with each event from ARM to FPGA. |
| 13 | Detect and discard confirmed duplicates | Never apply a duplicate message to the order book more than once. |
| 14 | Buffer out-of-order messages | Use a bounded reorder buffer and a defined timeout/window. |
| 15 | Configurable recovery timeout | Any initial timeout value is provisional and must be validated through testing. |
| 16 | Use snapshot recovery after recovery-buffer overflow | Invalidate the book and use snapshot recovery if the recovery buffer overflows. |

**Feed-specific caveat:** Incremental recovery, snapshot synchronization, duplicate handling, and reordering rules must be validated against the actual feed/recovery service. Do not assume a generic snapshot cutover rule.

### 2.4 Validation, timestamps, and normalized events — Decisions 17–20

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 17 | Two-level validation | ARM performs comprehensive protocol/message validation; FPGA defensively validates normalized events and order-book operations. |
| 18 | Keep local reception timestamp on ARM initially | Keep local receive time distinct from the exchange timestamp. |
| 19 | Fixed numeric event-type codes | Define a stable numeric code map for normalized event types. |
| 20 | Reject invalid FPGA event codes and increment a counter | Invalid codes must not modify the order book; record the condition in diagnostics. |

### 2.5 ARM–FPGA event transfer and validity ownership — Decisions 21–26

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 21 | AXI-Stream handshake confirms transfer; processing status is separate | A transfer is accepted on `TVALID && TREADY`; this does not confirm successful order-book processing. |
| 22 | FPGA order-book errors trigger invalidation and recovery handling | A consistency error may invalidate the book and require recovery or investigation. |
| 23 | Separate error counters | Maintain distinct protocol, sequence, order-book, and other relevant error counters. |
| 24 | AXI4-Lite diagnostic registers plus interrupts | Use registers for status/details and interrupts for important notifications. |
| 25 | ARM owns `FEED_VALID`; FPGA owns `BOOK_VALID` | ARM is authoritative for feed validity; FPGA independently tracks book validity. |
| 26 | Continuous event streaming; do not wait for per-event processing completion | Preserve event order and stream while AXI-Stream permits. Monitor processing status separately. |

---

## 3. In-flight event handling and snapshot recovery

### 3.1 In-flight event handling — Decision 27

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 27 | Stop book updates after an event-processing failure | Invalidate the book and discard or quarantine subsequent in-flight events so later events cannot compound a state error. |

### 3.2 Snapshot synchronization and loading — Decisions 28–31

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 28 | Snapshot includes an explicit synchronization sequence number | Define it as the **last event already included** in the snapshot. The first event applied afterward is `recovery_sequence + 1`. |
| 29 | Dedicated snapshot-loading interface or memory region | Keep bulk snapshot initialization separate from the normal AXI-Stream event path. |
| 30 | Validate snapshot entry count and checksum | FPGA verifies the expected entry count and an independently calculated checksum. |
| 31 | ARM and FPGA independently calculate the snapshot checksum | Independent calculations help detect transfer corruption. The algorithm and covered fields remain to be defined. |

### 3.3 Snapshot status reporting — Decision 32

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 32 | AXI4-Lite status registers plus interrupt notification | FPGA reports snapshot status through registers and notifies ARM on success/failure. ARM must read status; an interrupt alone does not imply success. |

Suggested snapshot status information: state, expected and received entry counts, calculated checksum, recovery sequence number, and error code. The exact register map is a later interface-design task.

---

## 4. Interrupt architecture and diagnostic counters

### 4.1 Interrupt architecture and acknowledgement — Decisions 33–36

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 33 | Dedicated interrupt signal for each event type | Use separate interrupt lines instead of one shared interrupt. Finalize the complete list later. |
| 34 | ARM explicitly acknowledges interrupts through a clear register | Keep each interrupt asserted until acknowledged. Clearing one source must not clear unrelated sources. |
| 35 | Defer non-critical errors until recovery reaches a safe point | Critical errors may abort recovery when necessary. |
| 36 | Independent pending-status bit for each interrupt source | Multiple pending sources must not overwrite one another. |

### 4.2 Interrupt counters — Decisions 37–40

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 37 | Maintain an event counter for each interrupt source | Count repeated occurrences even if ARM has not yet serviced the source. |
| 38 | Cumulative counters | Counters continue accumulating until explicitly reset. Acknowledgement does not reset them. |
| 39 | Individual counter-reset commands | ARM can reset selected counters independently through AXI4-Lite. |
| 40 | 32-bit saturating counters | Counters stop at `0xFFFFFFFF` rather than wrapping to zero. |

---

## 5. Diagnostic event history and FIFO

### 5.1 Diagnostic history and FIFO capacity — Decisions 41–43

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 41 | Maintain a hardware diagnostic FIFO | Store event type, order ID where applicable, feed sequence number where applicable, and FPGA timestamp. |
| 42 | 256 diagnostic entries | Initial FIFO capacity is 256 records. |
| 43 | Initially selected: overwrite the oldest record when full | **Superseded by Decision 49.** The current policy is to preserve existing records and reject new records when full. |

### 5.2 Diagnostic transfer and buffering — Decisions 44–48

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 44 | AXI DMA (S2MM) to transfer records to ARM-accessible DDR | Avoid reading every diagnostic record through individual AXI4-Lite accesses. |
| 45 | Threshold-based DMA trigger | Start a transfer at a configurable occupancy threshold; initial threshold is 64 records. |
| 46 | Transfer all currently available records in the captured batch | Capture a consistent batch boundary. Records arriving afterward belong to a subsequent batch. |
| 47 | Double buffering in DDR | DMA fills one buffer while ARM processes the other. |
| 48 | Backpressure when both DDR buffers are occupied | Stop consuming FIFO records until a buffer becomes available. |

### 5.3 FIFO full policy and loss counter — Decisions 49–50

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 49 | Reject new records when the diagnostic FIFO is full | Preserve existing records. This supersedes Decision 43's overwrite-oldest policy. |
| 50 | Cumulative, saturating, independently resettable 32-bit diagnostic-loss counter | Increment whenever a new diagnostic record is rejected because the FIFO is full. Do not reset automatically when DMA resumes. |

---

## 6. Diagnostic record format and BRAM-to-DMA datapath

### 6.1 Diagnostic record format and storage — Decisions 51–53

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 51 | Fixed-size diagnostic records | Every record uses the same structure and width. |
| 52 | 256-bit (32-byte) diagnostic record | Field allocation is shown in the table below. |
| 53 | Implement the diagnostic FIFO using BRAM | Confirm actual BRAM mapping and resource use in Vivado synthesis. |

#### Diagnostic record layout

| Field | Width | Purpose |
|---|---:|---|
| Error/event type | 8 bits | Identifies the diagnostic event. |
| Order ID | 64 bits | Identifies the affected order, where applicable. |
| Feed sequence number | 64 bits | Identifies the associated market event, where applicable. |
| FPGA timestamp | 64 bits | Records when the event occurred. |
| Reserved/future fields | 56 bits | Future expansion and event-specific information. |
| **Total** | **256 bits** | **32 bytes per record.** |

The event-specific meaning and validity of each field must be documented. Some event types may not use the order ID or feed sequence fields.

### 6.2 BRAM-to-AXI-Stream path — Decisions 54–56

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 54 | Custom FIFO-to-AXI-Stream controller | Custom RTL reads diagnostic records from BRAM and streams them into AXI DMA. |
| 55 | 256-bit AXI-Stream data width | One diagnostic record per beat. Use `TDATA[255:0]`, `TVALID`, `TREADY`, `TLAST`, and `TKEEP[31:0]`. |
| 56 | Dedicated hardware interface for DMA completion status | Controller receives DMA completion/error status through hardware. Verify the exact available interface in the selected AXI DMA IP configuration. |

**AXI-Stream behaviour:** The controller must hold `TVALID` and `TDATA` stable while `TREADY` is low. It must not advance the FIFO read pointer until the beat is accepted (`TVALID && TREADY`). `TLAST` marks the final beat of a DMA transaction. For a full 32-byte record, all `TKEEP` bits are asserted.

### 6.3 DMA failure and retry policy — Decisions 57–60

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 57 | Retry the entire DMA transaction after a transfer failure | Retain the original records until a successful transfer is confirmed. ARM must not process a partial DDR write as a valid batch. |
| 58 | Maximum of three total DMA attempts | Three attempts means one initial attempt plus at most two retries. |
| 59 | After three failed attempts, disable further DMA transactions but continue capture into BRAM | Preserve records until the FIFO becomes full; then apply Decision 49 and count rejected records. |
| 60 | Require explicit ARM intervention to re-enable DMA | ARM investigates and issues a re-enable command through AXI4-Lite. FPGA verifies that the DMA interface is ready. |

---

## 7. Diagnostic DMA batches and integrity

### 7.1 Batch identity and metadata — Decisions 61–64

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 61 | Unique, incrementing batch ID for each logical transaction | Retries retain the same batch ID; a new logical batch receives a new ID. |
| 62 | 64-bit batch ID | Do not silently wrap or reuse IDs. Define a safe exhaustion/reinitialization procedure. |
| 63 | Transfer batch metadata in a header alongside diagnostic records | ARM receives metadata and its associated records together in DDR. |
| 64 | 256-bit (32-byte) batch header | Proposed field allocation is shown below. |

#### Proposed batch header layout

| Field | Width | Purpose |
|---|---:|---|
| Batch ID | 64 bits | Identifies the logical transaction. |
| Record count | 16 bits | Number of diagnostic records in the batch. |
| Attempt number | 8 bits | Identifies the transfer attempt. |
| Transaction status | 8 bits | Encoded transaction state. |
| Timestamp | 64 bits | Batch creation timestamp. |
| CRC-32 | 32 bits | Batch integrity value. |
| Reserved | 64 bits | Future extension. |
| **Total** | **256 bits** | **32 bytes per header.** |

**Header validity caveat:** A header transferred at the beginning cannot, by itself, prove that the complete batch arrived. Define a separate completion/validity indication associated with successful DMA completion and ARM verification.

### 7.2 Batch integrity — Decisions 65–66

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 65 | Calculate and verify CRC-32 for every DMA batch | FPGA calculates the checksum; ARM independently recalculates it after receiving the batch. |
| 66 | CRC-32/ISO-HDLC | Use the standard variant with the parameters below. |

#### CRC-32/ISO-HDLC parameters

| Parameter | Value |
|---|---|
| Polynomial (normal representation) | `0x04C11DB7` |
| Initial value | `0xFFFFFFFF` |
| Reflected input/output | Yes |
| Final XOR | `0xFFFFFFFF` |

The checksum coverage must be defined precisely. Current intent is to cover the relevant batch metadata and diagnostic records while excluding the checksum field itself (or treating it as zero during calculation). Exact byte order and test vectors remain to be specified.

---

## 8. Batch acknowledgement, retransmission, and diagnostic reset

### 8.1 Batch ownership and acknowledgement — Decisions 67–70

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 67 | Retain FIFO records until ARM confirms successful CRC verification | DMA completion alone does not release records. |
| 68 | ARM acknowledges a batch by writing its 64-bit batch ID to an AXI4-Lite register | FPGA verifies the ID matches the outstanding batch before releasing associated records. |
| 69 | ARM explicitly requests retransmission after CRC verification failure | Request includes the batch ID; FPGA verifies it and retransmits the original batch. |
| 70 | Maximum of three CRC-verification attempts per batch | After three unsuccessful attempts, raise a fatal diagnostic-integrity error, preserve records, stop automatic retransmission, and wait for ARM intervention. |

DMA transfer retries (Decision 58) and CRC-verification attempts (Decision 70) are separate mechanisms.

### 8.2 Diagnostic recovery and reset — Decisions 71–76

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 71 | ARM-controlled recovery after a fatal integrity error | ARM investigates, issues a recovery command, and requests retransmission of the retained batch. No full FPGA reset is required. |
| 72 | Independent software-controlled diagnostic subsystem reset through AXI4-Lite | Reset diagnostic control/DMA state without resetting the order-book engine, market-data processing, or feed sequence tracking. |
| 73 | Require a predefined reset key | Proposed key: `0xD1A60001`. FPGA validates the key before initiating reset. |
| 74 | Dedicated reset-completion interrupt | Raise `IRQ_DIAGNOSTIC_RESET_COMPLETE`; ARM reads reset status and acknowledges the interrupt. |
| 75 | Preserve cumulative diagnostic counters across diagnostic subsystem resets | Only explicit individual counter-reset commands clear counters. |
| 76 | Preserve diagnostic FIFO contents across diagnostic subsystem resets | Separate FIFO memory from control reset; ensure pointers and occupancy remain consistent and records remain accessible. |

### 8.3 Outstanding batch and reset continuity — Decisions 77–80

| No. | Agreed decision | Implementation note |
|---:|---|---|
| 77 | Mark outstanding DMA batch as interrupted during reset and retransmit it afterward | Preserve the original batch ID and diagnostic records. ARM must not accept an uncertain/partial pre-reset transfer as complete. |
| 78 | Preserve the 64-bit batch ID counter across diagnostic subsystem resets | Interrupted batches retain their original ID; subsequent new batches continue with the next ID. |
| 79 | Record diagnostic subsystem resets in the diagnostic event history | Add a special reset record with reason, timestamp, reset identifier, and related batch ID where applicable. Record it after FIFO control is operational again. |
| 80 | Use fixed enumerated reset-reason codes | Use stable numeric codes instead of free-form values. The provisional codes are listed below. |

#### Provisional reset reason codes

| Code | Reset reason |
|---:|---|
| `0x01` | DMA fatal error |
| `0x02` | CRC verification failure |
| `0x03` | ARM-requested reset |
| `0x04` | Internal diagnostic controller fault |
| `0x05` | DMA interface fault |
| `0x06` | Other hardware-detected fault |

These codes are provisional and should be finalized with the register map and RTL definitions.

---

## 9. Known open points and consistency checks

The following points must be resolved before treating this document as an implementation-ready specification.

| ID | Open point | Required follow-up |
|---|---|---|
| O-01 | FIFO full policy | Decision 43 initially selected overwrite-oldest; Decision 49 later selected reject-new/preserve-existing. Decision 49 is the current policy. |
| O-02 | FIFO retention versus circular FIFO operation | Define how outstanding batch records are retained until ARM acknowledgement while new records are accepted, and how retained records are protected from overwrite. |
| O-03 | Double-buffer capacity and ownership | Specify DDR buffer sizes, ownership state machine, and buffer-full/backpressure signalling. |
| O-04 | DMA completion interface | Verify the AXI DMA IP status/interrupt connectivity available in Vivado. |
| O-05 | DMA batch boundary | Define the snapshot point for “all currently available records”; records arriving during a transfer must belong to the next batch. |
| O-06 | Partial DMA write handling | Define how ARM invalidates incomplete DDR data and how a retry avoids stale partial data being mistaken for a complete batch. |
| O-07 | Batch header validity | Define how completion is indicated. A header transferred first cannot alone prove that the full transfer completed. |
| O-08 | CRC coverage and byte order | Define exact byte coverage, byte order, checksum-field treatment, and CRC test vectors. |
| O-09 | Reset and retained FIFO state | Define how pointers, occupancy, outstanding metadata, and in-progress transfer state are preserved/reconstructed across reset. |
| O-10 | Reset event capture | Ensure the reset event is not erased by the reset it records; define when it is inserted after reset. |
| O-11 | Interrupt list and priority | Finalize sources, critical/non-critical classification, pending-bit behaviour, counter association, and acknowledgement semantics. |
| O-12 | Diagnostic record field interpretation | Define event-specific meanings and validity rules for reserved bits and fields that may not apply to every event. |
| O-13 | Snapshot checksum | Choose the algorithm and define the exact snapshot fields covered. |
| O-14 | Snapshot interface | Specify snapshot interface, framing, completion signalling, and synchronization with normal event ingress. |
| O-15 | 256-bit AXI-Stream/DMA configuration | Confirm that the selected AXI DMA configuration supports the required data width and byte-lane configuration. |
| O-16 | Counter semantics | Define whether counters track diagnostic event occurrences, interrupt assertions, or both. |

---

## 10. Implementation and verification notes

These decisions define intended behaviour; they are not proof that the design has already been implemented or validated.

Before RTL sign-off:

1. Verify ITCH 5.0 message layouts and semantics against the official specification.
2. Verify MoldUDP64 packet framing and sequence handling against the official specification.
3. Define normalized-event field validity for each event type.
4. Define AXI-Stream event framing and error-reporting details.
5. Define AXI4-Lite register addresses, access permissions, reset values, and write-one-to-clear semantics.
6. Define snapshot transfer framing and checksum test vectors.
7. Define diagnostic FIFO and DMA state machines, including ownership and retry behaviour.
8. Build self-checking testbenches for normal transfer, backpressure, FIFO full, DMA failure, CRC failure, retransmission, reset, and interrupted-batch recovery.
9. Verify synthesis resource usage and timing for the target Vivado version and Zynq-7020 device.
10. Benchmark the complete system, including ARM parsing, AXI transfer overhead, FPGA processing, and diagnostic activity.

### 10.1 Key invariants

| Invariant | Required behaviour |
|---|---|
| Feed sequence integrity | A sequence gap invalidates feed/book state until feed-specific recovery establishes a valid synchronization point. |
| Book validity | FPGA must not apply later events after an earlier failure makes the book state unreliable. |
| Snapshot acceptance | Accept a snapshot only after complete transfer and successful integrity/structural checks. |
| Snapshot synchronization | Snapshot sequence denotes the last event included; next applied event is sequence plus one. |
| AXI-Stream transfer | A beat transfers only when `TVALID && TREADY`; producer holds data stable while stalled. |
| Processing status | Interface transfer acceptance is not the same as successful event processing. |
| Diagnostic retention | Do not release diagnostic records until ARM acknowledges a CRC-verified batch. |
| DMA retry | A retry retransmits the same logical batch with the same batch ID. |
| Diagnostic reset isolation | Diagnostic reset must not reset the order-book engine or feed sequence tracking. |
| Counter preservation | Diagnostic subsystem reset preserves cumulative counters and the batch ID counter. |

---

## 11. References

- Nasdaq TotalView-ITCH 5.0 specification: <https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification_5.0.pdf>
- MoldUDP64 specification: <https://nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf>

Consult the relevant AMD/Xilinx AXI DMA, AXI4-Stream, Zynq-7000, and 7-series documentation when finalizing hardware interfaces and implementation details.
