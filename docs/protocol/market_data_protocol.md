# Market Data Protocol Specification

**Project:** FPGA-Accelerated Real-Time Financial Market Data Processing
Engine\
**Document:** `market_data_protocol.md`\
**Version:** 0.5 (Protocol Edge-Case and Serialization Clarification)  
**Status:** Field-level protocol, event mapping, historical P-message compatibility, broken-trade routing, and canonical event-byte serialization specified; provider-specific recovery and implementation verification remain open  
**Date:** 24 November 2026

------------------------------------------------------------------------

## Document Control

| Item | Description |
|---|---|
| Document purpose | Define the market-data protocol and the boundary between the ARM Processing System (PS) and FPGA Programmable Logic (PL). |
| Target feed | Nasdaq TotalView-ITCH 5.0 |
| Transport / delivery | UDP multicast with MoldUDP64 framing and sequencing |
| Initial instrument scope | One selected instrument |
| Primary implementation split | ARM handles network/feed protocol processing; FPGA handles order-book reconstruction and analytics |
| Related documents | `system_requirements.md`, `hardware_architecture.md`, `order_book_design.md`, `verification_plan.md` |

### Revision 0.5 changes

- Added historical compatibility handling for the `P` Buy/Sell Indicator (`B` or legacy `S`) without treating it as an order side.
- Specified message-type-aware routing for global `S` System Event and instrument-specific `B` Broken Trade messages, including Stock Locate validation against the ARM ledger.
- Defined the canonical 59-byte MSB-first logical event serialization and its mapping to ascending 256-bit AXI4-Stream byte lanes.
- Retained the 465-bit logical payload and two-beat 256-bit transfer; deferred width compression until measured resource/performance results justify a versioned interface change.
- Updated parser and ARM–FPGA verification requirements and refined OI-04 as an implementation-conformance task.

### Status conventions

This document distinguishes three kinds of statements:

- **Protocol fact:** Defined by the Nasdaq ITCH or MoldUDP64 specifications.
- **Project decision:** A choice made for this implementation.
- **Open item:** A point that must be verified or decided before implementation is frozen.

This consolidated engineering draft is not a substitute for
the official feed specifications or the feed-specific operational
documentation supplied by a market-data provider.

------------------------------------------------------------------------

# 1. Introduction

## 1.1 Purpose

The purpose of this document is to define the market-data protocol used
by the FPGA-Accelerated Real-Time Financial Market Data Processing
Engine.

The system receives exchange-published market-data messages, interprets
order and trade events, reconstructs the displayed order book for a
selected instrument, and calculates market analytics.

This document describes the selected feed protocol, packet framing,
message formats, data encoding, message interpretation, sequence
management, recovery considerations, and the normalized event interface
between the ARM processor and FPGA.

## 1.2 Scope

The initial implementation focuses on one selected instrument using
Nasdaq TotalView-ITCH 5.0 message semantics and MoldUDP64 delivery
framing.

The protocol-processing scope includes:

-   Receiving UDP multicast datagrams.
-   Decoding MoldUDP64 downstream packet headers and message blocks.
-   Tracking the feed session and message sequence.
-   Detecting missing, duplicate, or unexpected messages.
-   Parsing the selected ITCH 5.0 message types.
-   Filtering messages for the selected instrument without bypassing
    feed-level sequence processing.
-   Converting supported ITCH messages into normalized internal events.
-   Coordinating recovery when feed continuity or order-book validity is
    lost.
-   Transferring normalized events from the ARM to the FPGA.

The FPGA order-book data structures, detailed memory organization, register map, diagnostic subsystem implementation, interrupt topology, and RTL microarchitecture are outside the detailed scope of this document. They belong in `hardware_architecture.md` and `order_book_design.md`. This document specifies only the market-data protocol behaviour and the logical ARM–FPGA event contract.

## 1.3 System boundary

The system is a **market-data consumer**. It reconstructs market state
from events published by the feed.

It is not an exchange matching engine and does not: - Accept customer
orders. - Submit orders to an exchange. - Match incoming customer orders
against resting orders. - Execute trades. - Provide brokerage or
exchange infrastructure.

## 1.4 Proposed processing split

The initial architecture assigns feed and network responsibilities to the ARM PS and order-book computation to the FPGA PL.

| Component | Primary responsibilities |
|---|---|
| ARM PS | UDP reception, MoldUDP64 decoding, feed-session and sequence handling, ITCH parsing, instrument filtering, event normalization, recovery coordination, Match Number ledger in DDR, host communication |
| FPGA PL | Normalized-event acceptance, order-book reconstruction, best bid/ask and selected analytics, book-validity reporting, trade-contribution reports and processing progress reporting |
| Host application | Configuration, monitoring, visualization, logging and performance measurement |

For eligible `E` executions, the FPGA obtains the effective execution price from the referenced resting order and reports the accepted trade contribution to the ARM. The ARM maintains the Match Number ledger required to process later `B` messages. The ordering barrier required before a broken-trade correction is described in Sections 6 and 9.

This split is a project architecture decision, not a requirement imposed by ITCH.

## 1.5 Terminology

| Term | Meaning |
|---|---|
| ITCH | Nasdaq TotalView-ITCH application-level market-data message protocol |
| MoldUDP64 | UDP-based delivery protocol that frames, sequences, and supports re-request of messages |
| Session | A logical sequence of messages identified by the MoldUDP64 session field |
| Message sequence number | Sequence assigned by MoldUDP64 to an individual message in a session |
| Stock Locate | Feed-provided instrument identifier used in ITCH messages |
| Order Reference Number | Feed-provided identifier for an order |
| Match Number | Feed-provided identifier associated with an execution |
| PS | Zynq Processing System, containing the ARM processor |
| PL | FPGA Programmable Logic |
| Normalized event | Project-defined representation of a parsed ITCH message for FPGA processing |
| Book validity | Whether the reconstructed order book is believed to represent a continuous, correctly applied feed state |

------------------------------------------------------------------------

# 2. Protocol Selection

## 2.1 Application protocol: Nasdaq TotalView-ITCH 5.0

**Project decision:** Use Nasdaq TotalView-ITCH 5.0 as the source
message format for the initial implementation.

ITCH is an outbound market-data protocol. It communicates information
about orders, executions, trades, instruments and market events. It is
not an order-entry protocol.

The protocol is suitable for this project because it provides
order-level events that can be applied sequentially to reconstruct a
displayed order book.

The ITCH specification defines message content and semantics. It does
not, by itself, define the network delivery and sequencing mechanism
used by every feed deployment.

## 2.2 Delivery protocol: MoldUDP64

**Project decision:** Use MoldUDP64 as the initial delivery/framing
protocol over UDP multicast.

MoldUDP64 supports one transmitter sending a sequenced data stream to
multiple listeners. It provides: - A session identifier. - A sequence
number for the first message in each downstream packet. - A message
count. - Length-prefixed message blocks. - Heartbeat packets. - A
request mechanism for retransmission of missing messages.

MoldUDP64 does not make UDP itself reliable. The receiver must detect
gaps and use the feed's configured re-request service and recovery
procedures.

## 2.3 Protocol alternatives

Nasdaq's TotalView-ITCH 5.0 specification identifies multiple delivery
options, including SoupBinTCP and MoldUDP64. This project selects
MoldUDP64 for the initial UDP multicast design.

SoupBinTCP is not part of the initial implementation. It may be
considered later as an alternative feed-delivery path or for a separate
recovery/failover design, subject to the actual service configuration.

## 2.4 Protocol layering

The conceptual receive stack is:

``` text
+--------------------------------------+
| Project market-data consumer         |
+--------------------------------------+
| Nasdaq TotalView-ITCH 5.0 messages   |
+--------------------------------------+
| MoldUDP64 downstream packet framing  |
+--------------------------------------+
| UDP                                  |
+--------------------------------------+
| IP multicast                         |
+--------------------------------------+
| Ethernet                             |
+--------------------------------------+
```

Responsibilities are separated as follows:

| Layer | Responsibility |
|---|---|
| Ethernet / IP / UDP | Network delivery of UDP datagrams |
| MoldUDP64 | Session identification, message framing, sequence information, heartbeat, and re-request support |
| ITCH 5.0 | Meaning and binary layout of market-data messages |
| Project normalization | Convert selected ITCH messages into the project's internal event representation |

**Important:** The MoldUDP64 sequence number belongs to the delivery
layer. It is not a field carried inside each ITCH message.

------------------------------------------------------------------------

# 3. MoldUDP64 Packet Format

## 3.1 Downstream packet structure

A MoldUDP64 downstream packet consists of a fixed 20-byte header
followed by zero or more message blocks.

``` text
+-------------------------------+
| Session             10 bytes  |
+-------------------------------+
| Sequence Number      8 bytes  |
+-------------------------------+
| Message Count        2 bytes  |
+-------------------------------+
| Message Block 1                |
|   Message Length     2 bytes  |
|   Message Data       variable |
+-------------------------------+
| Message Block 2                |
|   Message Length     2 bytes  |
|   Message Data       variable |
+-------------------------------+
| ...                           |
+-------------------------------+
```

### 3.1.1 Downstream header

| Field | Offset | Length | Description |
|---|---:|---:|---|
| Session | 0 | 10 bytes | Identifies the MoldUDP64 session |
| Sequence Number | 10 | 8 bytes | Sequence number of the first message in this packet |
| Message Count | 18 | 2 bytes | Number of message blocks in the packet |
| **Total** | **0** | **20 bytes** | Fixed header length |

MoldUDP64 numeric fields are binary big-endian values.

## 3.2 Message blocks

Each message block contains:

| Field | Length | Description |
|---|---:|---|
| Message Length | 2 bytes | Unsigned byte count of the message data that follows |
| Message Data | Variable | Application-level message; in this project, an ITCH message |

The two-byte message-length field does **not** include its own two
bytes.

The first message block starts immediately after the 20-byte downstream
header. Subsequent blocks follow directly after the preceding block.

The receiver must use the message count and each message-length field to
walk the packet safely. It must not assume that all ITCH messages have
the same length.

## 3.3 Sequence numbering

The downstream packet header's sequence number identifies the first
message in the packet.

If a packet contains multiple messages, the messages are numbered
consecutively. For example:

``` text
Packet header:
    Sequence Number = 100
    Message Count   = 3

Message blocks:
    Message 1 -> sequence 100
    Message 2 -> sequence 101
    Message 3 -> sequence 102
```

The receiver maintains the next expected message sequence number for the
active session.

Sequence tracking must occur for the complete feed stream before
filtering messages for the selected instrument. Otherwise, messages for
other instruments could be mistaken for missing feed messages.

## 3.4 Heartbeats and end of session

MoldUDP64 defines special message-count values:

| Message Count | Meaning |
|---:|---|
| `0` | Heartbeat packet |
| `0xFFFF` | End-of-session packet |
| Other values | Number of message blocks in the packet |

Heartbeat packets contain no message blocks and carry the next expected
sequence number. They allow a receiver to detect a gap even when normal
market activity is low.

End-of-session packets indicate that the current session has ended. The
receiver must treat a session transition as a feed lifecycle event and
must not assume sequence continuity across unrelated sessions.

## 3.5 Re-request packets

A receiver can request retransmission by sending a MoldUDP64 request
packet to the configured re-request server.

The request packet contains: - Session: 10 bytes. - First requested
sequence number: 8 bytes. - Requested message count: 2 bytes.

A valid response is a downstream packet sent by unicast to the
requester. If the requested range cannot fit in one response packet,
further requests may be needed.

The exact re-request server addresses, ports, request limits, retention
window and operational rules are feed-deployment configuration items.
They must not be invented by the receiver implementation.

------------------------------------------------------------------------

# 4. ITCH 5.0 Data Encoding

## 4.1 Integer byte order

ITCH integer fields are binary encoded using **big-endian (network byte
order)**. Unless a field is explicitly described otherwise, integer
fields are unsigned.

For a four-byte value:

``` text
Bytes:       00 0F 42 40
Interpret:   0x000F4240
Decimal:     1,000,000
```

The parser must convert multi-byte fields consistently before using them
in the internal representation.

## 4.2 Alpha fields

Alpha fields are ASCII character fields. They are left-justified and
padded on the right with spaces where specified.

The parser must not assume that a space-padded symbol is a
null-terminated C string.

## 4.3 Prices

ITCH price fields are integer values with an implied decimal precision.
A `Price (4)` field has four implied decimal places.

For example:

``` text
Raw integer price: 1,005,000
Precision:         4 decimal places
Displayed price:   100.5000
```

**Implementation rule:** Keep prices as integers in the protocol parser
and FPGA event interface. Do not convert prices to binary floating-point
values for order-book comparisons.

The project must document the selected price scale and use it
consistently in comparisons, analytics and host display conversion.

## 4.4 Timestamps

ITCH timestamps are represented as nanoseconds since midnight.

The ITCH timestamp is an exchange/feed timestamp. It is not the same as
the local packet-reception timestamp measured by the ARM.

The implementation should keep these concepts separate: - Exchange
timestamp from the ITCH message. - Local receive timestamp captured by
the host/ARM for latency and diagnostics.

The local receive timestamp's clock source and synchronization accuracy
belong in the system architecture/performance specification.

## 4.5 Instrument identifiers

ITCH messages contain a Stock Locate field. Nasdaq assigns locate codes
dynamically each day, communicates them through Stock Directory
messages, and keeps a locate code stable intraday. The same locate code
must not be assumed to identify the same instrument on a different
trading day.

The receiver should use the Stock Directory information and the feed's
daily mapping to associate a Stock Locate value with the intended
instrument.

------------------------------------------------------------------------

# 5. Supported ITCH Message Types

## 5.1 Initial message selection

The initial implementation supports the following messages:

| Type | Message | Project treatment |
|---|---|---|
| `S` | System Event | Track feed/day lifecycle; `C` (End of Messages) closes the daily stream. |
| `R` | Stock Directory | Build the current-day Stock Locate-to-symbol mapping. |
| `H` | Stock Trading Action | Track the selected instrument's trading state. |
| `A` / `F` | Add Order / Add Order with MPID | Add a displayed order. |
| `E` | Order Executed | Reduce displayed quantity and contribute to trade statistics. |
| `C` | Order Executed With Price | Reduce displayed quantity; include in trade statistics only when Printable=`Y`. |
| `X` | Order Cancel | Reduce displayed quantity; not a trade. |
| `D` | Order Delete | Remove the remaining displayed quantity. |
| `U` | Order Replace | Remove the old order reference and add the replacement under the new reference. |
| `P` | Trade (Non-Cross) | Update trade statistics; no displayed-book update. |
| `Q` | Cross Trade | Recognize and sequence; exclude from initial trade analytics and do not treat as a displayed-order update. |
| `B` | Broken Trade | Reverse a previously included `E`, `C`, or `P` trade contribution by Match Number; no book update. |

The following known non-core message types are recognized and skipped by the initial application logic when their message framing and lengths are valid. They still consume their normal MoldUDP64 sequence positions.

| Type | Message | Total ITCH message length |
|---|---|---:|
| `Y` | Reg SHO Restriction | 20 bytes |
| `L` | Market Participant Position | 26 bytes |
| `V` | MWCB Decline Level | 35 bytes |
| `W` | MWCB Status | 12 bytes |
| `J` | LULD Auction Collar | 35 bytes |
| `K` | IPO Quoting Period Update | 28 bytes |
| `I` | Net Order Imbalance Indicator (NOII) | 50 bytes |
| `N` | Retail Price Improvement Indicator | 20 bytes |

If a known message later proves relevant to selected-instrument state or the chosen analytics, its handling must be added explicitly.

Unknown message types are not silently treated as harmless. The ARM must stop forwarding the affected stream, record the unsupported type, and apply the conservative feed/book-validity policy in Section 10.3 until the message's impact is understood.

A correctly framed known message may be skipped for FPGA processing while still being counted for feed sequencing. A malformed or truncated message is a different class of error and must not be skipped as though it were valid.

## 5.2 Common message prefix

The selected stock-related ITCH messages generally begin with the
following common fields:

| Field | Offset | Length |
|---|---:|---:|
| Message Type | 0 | 1 byte |
| Stock Locate | 1 | 2 bytes |
| Tracking Number | 3 | 2 bytes |
| Timestamp | 5 | 6 bytes |

The remaining fields depend on the message type.

The Tracking Number is a Nasdaq internal tracking field. It is not the
MoldUDP64 message sequence number.

### 5.2.1 Field encoding conventions

| ITCH field form | Binary representation | Interpretation |
|---|---|---|
| Message Type and other Alpha fields | ASCII byte(s) | Character data; fixed-width Alpha fields are left-justified and space-padded where specified. |
| Stock Locate | Unsigned 16-bit integer | Big-endian. |
| Tracking Number | Unsigned 16-bit integer | Big-endian; feed-internal tracking only. |
| Timestamp | Unsigned 48-bit integer | Big-endian nanoseconds since midnight. |
| Order Reference Number | Unsigned 64-bit integer | Big-endian. |
| Match Number | Unsigned 64-bit integer | Big-endian; uniqueness scope differs by message type as noted in Section 6.6. |
| Shares / quantity | Unsigned 32-bit or 64-bit integer | Width is defined by the individual message field. Big-endian. |
| Price (4) | Unsigned 32-bit integer | Big-endian integer with four implied decimal places. |

All offsets in Sections 5.3–5.15 are byte offsets from the first byte of the ITCH message, including the Message Type byte. Total lengths include that byte. The two-byte MoldUDP64 Message Length prefix is outside the ITCH message and is not included in these offsets or lengths.

## 5.3 Add Order --- `A`

An `A` message adds an unattributed displayed order.

| Field | Offset | Length |
|---|---:|---:|
| Message Type | 0 | 1 byte |
| Stock Locate | 1 | 2 bytes |
| Tracking Number | 3 | 2 bytes |
| Timestamp | 5 | 6 bytes |
| Order Reference Number | 11 | 8 bytes |
| Buy/Sell Indicator | 19 | 1 byte |
| Shares | 20 | 4 bytes |
| Stock | 24 | 8 bytes |
| Price | 32 | 4 bytes |
| **Total message length** | — | **36 bytes** |

Buy/Sell Indicator values:

| Value | Meaning |
|---|---|
| `B` | Buy |
| `S` | Sell |

Processing: 1. Identify the instrument. 2. Apply the selected-instrument
filter. 3. Create an order entry using the Order Reference Number. 4.
Store side, price and displayed quantity. 5. Update the appropriate side
of the order book.

## 5.4 Add Order with MPID Attribution --- `F`

An `F` message has the same fields as `A`, followed by a four-byte
Attribution field.

| Field | Offset | Length | Encoding / meaning |
|---|---:|---:|---|
| Message Type | 0 | 1 byte | ASCII `F` |
| Stock Locate | 1 | 2 bytes | Unsigned integer |
| Tracking Number | 3 | 2 bytes | Unsigned integer |
| Timestamp | 5 | 6 bytes | Nanoseconds since midnight |
| Order Reference Number | 11 | 8 bytes | Unsigned integer |
| Buy/Sell Indicator | 19 | 1 byte | ASCII `B` = buy, `S` = sell |
| Shares | 20 | 4 bytes | Unsigned integer; displayed quantity |
| Stock | 24 | 8 bytes | ASCII symbol, right-padded with spaces |
| Price | 32 | 4 bytes | Price (4), four implied decimal places |
| Attribution | 36 | 4 bytes | ASCII MPID |
| **Total message length** | — | **40 bytes** | — |

The Attribution field identifies the market participant associated with
the order.

For the initial order-book implementation, `F` may use the same
order-book update logic as `A`. Attribution can be retained as metadata
or ignored by the core price/quantity calculations if participant-level
analytics are not in scope.

## 5.5 Order Executed --- `E`

An `E` message reports a full or partial execution of an order already
represented in the displayed book.

| Field | Offset | Length |
|---|---:|---:|
| Message Type | 0 | 1 byte |
| Stock Locate | 1 | 2 bytes |
| Tracking Number | 3 | 2 bytes |
| Timestamp | 5 | 6 bytes |
| Order Reference Number | 11 | 8 bytes |
| Executed Shares | 19 | 4 bytes |
| Match Number | 23 | 8 bytes |
| **Total message length** | — | **31 bytes** |

Processing: 1. Find the referenced order. 2. Subtract Executed Shares
from its remaining displayed quantity. 3. Remove the order if its
remaining quantity becomes zero. 4. Update execution statistics
according to the project's defined trade-statistics policy.

The same order may receive multiple execution messages; the effects are
cumulative.

## 5.6 Order Executed with Price --- `C`

A `C` message reports an execution at a price different from the order's
original display price.

| Field | Offset | Length |
|---|---:|---:|
| Message Type | 0 | 1 byte |
| Stock Locate | 1 | 2 bytes |
| Tracking Number | 3 | 2 bytes |
| Timestamp | 5 | 6 bytes |
| Order Reference Number | 11 | 8 bytes |
| Executed Shares | 19 | 4 bytes |
| Match Number | 23 | 8 bytes |
| Printable | 31 | 1 byte |
| Execution Price | 32 | 4 bytes |
| **Total message length** | — | **36 bytes** |

Printable flag values:

| Value | Meaning |
|---|---|
| `Y` | Printable |
| `N` | Non-printable |

The order-book quantity must be reduced for the execution in either
case.

For time-and-sales or volume statistics, Nasdaq specifies that
non-printable executions should be excluded to avoid double counting
when the shares are represented in a later bulk print. The project must
apply this rule consistently.

## 5.7 Order Cancel --- `X`

An `X` message represents a partial cancellation.

| Field | Offset | Length |
|---|---:|---:|
| Message Type | 0 | 1 byte |
| Stock Locate | 1 | 2 bytes |
| Tracking Number | 3 | 2 bytes |
| Timestamp | 5 | 6 bytes |
| Order Reference Number | 11 | 8 bytes |
| Canceled Shares | 19 | 4 bytes |
| **Total message length** | — | **23 bytes** |

Processing: 1. Find the referenced order. 2. Subtract Canceled Shares
from its remaining displayed quantity. 3. Remove the order if its
remaining quantity becomes zero. 4. Do not count a cancellation as a
trade execution.

## 5.8 Order Delete --- `D`

A `D` message removes an order from the displayed book.

| Field | Offset | Length |
|---|---:|---:|
| Message Type | 0 | 1 byte |
| Stock Locate | 1 | 2 bytes |
| Tracking Number | 3 | 2 bytes |
| Timestamp | 5 | 6 bytes |
| Order Reference Number | 11 | 8 bytes |
| **Total message length** | — | **19 bytes** |

All remaining shares for the order are removed. A delete is not a trade
execution and must not increase executed volume.

## 5.9 Order Replace --- `U`

A `U` message represents a cancel-replace operation. The original order
is removed and the replacement is identified by a new Order Reference
Number.

| Field | Offset | Length |
|---|---:|---:|
| Message Type | 0 | 1 byte |
| Stock Locate | 1 | 2 bytes |
| Tracking Number | 3 | 2 bytes |
| Timestamp | 5 | 6 bytes |
| Original Order Reference Number | 11 | 8 bytes |
| New Order Reference Number | 19 | 8 bytes |
| Shares | 27 | 4 bytes |
| Price | 31 | 4 bytes |
| **Total message length** | — | **35 bytes** |

The side, stock symbol and attribution are inherited from the original
order. They are not supplied again in the `U` message.

Processing: 1. Find and remove the original order. 2. Create the
replacement order using the new reference number, new quantity and new
price. 3. Preserve the original side and other inherited metadata. 4.
Ensure subsequent updates use the new Order Reference Number.

This is not a simple in-place partial modification: the old reference
number is replaced by a new reference number.

## 5.10 Trade (Non-Cross) --- `P`

A `P` message reports a trade involving non-displayed order types. It
does not change the displayed order book.

| Field | Offset | Length | Encoding / meaning |
|---|---:|---:|---|
| Message Type | 0 | 1 byte | ASCII `P` |
| Stock Locate | 1 | 2 bytes | Unsigned integer; identifies the security |
| Tracking Number | 3 | 2 bytes | Unsigned integer; Nasdaq internal tracking field |
| Timestamp | 5 | 6 bytes | Nanoseconds since midnight |
| Order Reference Number | 11 | 8 bytes | Unsigned integer; zero in the documented binary feed since 6 December 2010 |
| Buy/Sell Indicator | 19 | 1 byte | ASCII `B` or `S`; current-feed expectation and historical compatibility are explained below |
| Shares | 20 | 4 bytes | Unsigned integer; matched quantity |
| Stock | 24 | 8 bytes | ASCII symbol, right-padded with spaces |
| Price | 32 | 4 bytes | Price (4); four implied decimal places |
| Match Number | 36 | 8 bytes | Unsigned integer; session-unique for this trade message |
| **Total message length** | — | **44 bytes** | — |

For current TotalView-ITCH 5.0 production data, the Order Reference
Number is expected to be zero. The Buy/Sell Indicator at offset 19 is
expected to be `B` effective 14 July 2014, regardless of the resting side.
Neither field describes a displayed resting order, and neither is mapped
to the normalized event's Side field.

**Historical-capture compatibility:** The parser must accept either
ASCII `B` or ASCII `S` in this one-byte field so that older historical
captures can be decoded. A configured current-feed conformance check may
record `S` as a non-fatal protocol warning because it differs from the
post-14-July-2014 production convention. Any value other than `B` or `S`
is invalid. The compatibility rule does not change the 44-byte message
length and must not cause the parser to reject an otherwise valid `P`
message solely because offset 19 contains `S`.

`P` messages contribute to trade statistics according to the project's
market-statistics policy. The ARM should retain the raw indicator in
parser diagnostics when useful, but the normalized `TRADE` event carries
no meaningful side.

## 5.11 Cross Trade --- `Q`

A `Q` message reports a cross transaction and contains the instrument,
shares, cross price, match number and cross type.

It does not represent an ordinary add, cancel or execution against an
order reference in the displayed order book.

The initial project explicitly excludes `Q` cross-trade volume from its trade analytics. The ARM still recognizes and sequences the 40-byte message. It is not forwarded as a displayed-order update. The message has no effect on the current order book.

| Field | Offset | Length | Encoding / meaning |
|---|---:|---:|---|
| Message Type | 0 | 1 byte | ASCII `Q` |
| Stock Locate | 1 | 2 bytes | Unsigned integer |
| Tracking Number | 3 | 2 bytes | Unsigned integer |
| Timestamp | 5 | 6 bytes | Nanoseconds since midnight |
| Shares | 11 | 8 bytes | Unsigned 64-bit matched cross quantity |
| Stock | 19 | 8 bytes | ASCII symbol, right-padded with spaces |
| Cross Price | 27 | 4 bytes | Price (4) |
| Match Number | 31 | 8 bytes | Unsigned integer; day-unique per ITCH 5.0 description |
| Cross Type | 39 | 1 byte | ASCII: `O` opening, `C` closing, `H` IPO/halt/pause, `I` intraday/post-close Cross Network |
| **Total message length** | — | **40 bytes** | — |

## 5.12 Broken Trade --- `B`

A `B` message is 19 bytes long and identifies a previously reported execution that has been broken. Its Match Number refers to a previously transmitted `E`, `C`, or `P` message. Its Stock Locate identifies the security associated with the broken execution; unlike the global `S` System Event message, `B` is instrument-specific. A trade break is final and cannot be reinstated.

It does not change the current displayed order book. This project processes it through the ARM-owned Match Number ledger described in Section 6.6, and reverses trade statistics only when the original contribution was included. The ARM must validate the `B` message's Stock Locate against the instrument/locate recorded with the matched ledger entry before issuing a reversal.

| Field | Offset | Length | Encoding / meaning |
|---|---:|---:|---|
| Message Type | 0 | 1 byte | ASCII `B` |
| Stock Locate | 1 | 2 bytes | Unsigned integer; identifies the security associated with the broken execution |
| Tracking Number | 3 | 2 bytes | Unsigned integer; Nasdaq internal tracking field |
| Timestamp | 5 | 6 bytes | Nanoseconds since midnight |
| Match Number | 11 | 8 bytes | Match Number of the earlier `E`, `C`, or `P` execution/trade being broken |
| **Total message length** | — | **19 bytes** | — |

## 5.13 System Event --- `S`

The `S` message is 12 bytes long and communicates feed or market lifecycle events. Its event code is important for session/day handling.

| Field | Offset | Length | Encoding / meaning |
|---|---:|---:|---|
| Message Type | 0 | 1 byte | ASCII `S` |
| Stock Locate | 1 | 2 bytes | Unsigned integer; always zero for this message |
| Tracking Number | 3 | 2 bytes | Unsigned integer |
| Timestamp | 5 | 6 bytes | Nanoseconds since midnight |
| Event Code | 11 | 1 byte | ASCII daily event code |
| **Total message length** | — | **12 bytes** | — |

Common daily event codes include:

| Code | Meaning |
|---|---|
| `O` | Start of Messages |
| `S` | Start of System Hours |
| `Q` | Start of Market Hours |
| `M` | End of Market Hours |
| `E` | End of System Hours |
| `C` | End of Messages |

The project should use these events to manage feed lifecycle state. The meaning of an event code must not be confused with the one-byte ITCH Message Type field. `E` (End of System Hours) does not mean that all messages for the day have arrived: Nasdaq states that Broken Trade (`B`) and Order Delete (`D`) messages may still follow. Continue processing valid sequenced messages until `C` (End of Messages), which is the final daily ITCH message. This ITCH daily lifecycle is distinct from the MoldUDP64 `Message Count = 0xFFFF` end-of-session marker.

## 5.14 Stock Directory --- `R`

The Stock Directory message maps a current-day Stock Locate value to a symbol and provides instrument attributes. The complete ITCH 5.0 message is 39 bytes.

| Field | Offset | Length | Notes |
|---|---:|---:|---|
| Message Type | 0 | 1 | `R` |
| Stock Locate | 1 | 2 | Current-day locate code |
| Tracking Number | 3 | 2 | Nasdaq internal tracking field |
| Timestamp | 5 | 6 | Nanoseconds since midnight |
| Stock | 11 | 8 | ASCII symbol, right-padded with spaces |
| Market Category | 19 | 1 | Listing market/category |
| Financial Status Indicator | 20 | 1 | Financial status code |
| Round Lot Size | 21 | 4 | Integer |
| Round Lots Only | 25 | 1 | `Y` or `N` |
| Issue Classification | 26 | 1 | Classification code |
| Issue Sub-Type | 27 | 2 | Sub-type code |
| Authenticity | 29 | 1 | Live/test indicator |
| Short Sale Threshold Indicator | 30 | 1 | Rule 201 status |
| IPO Flag | 31 | 1 | IPO status |
| LULD Reference Price Tier | 32 | 1 | Price-band tier |
| ETP Flag | 33 | 1 | Exchange-traded product indicator |
| ETP Leverage Factor | 34 | 4 | Integer |
| Inverse Indicator | 38 | 1 | Inverse ETP indicator |
| **Total** | — | **39 bytes** | — |

The ARM must validate the complete message length, extract at least Stock Locate and Stock, and maintain the current-day mapping. The locate value is stable intraday but must not be assumed to remain the same across trading days. Other directory attributes may be retained for host display; they are not required by the initial order-book datapath.

## 5.15 Stock Trading Action --- `H`

The Stock Trading Action message is 25 bytes and communicates the trading state of an instrument.

| Field | Offset | Length | Notes |
|---|---:|---:|---|
| Message Type | 0 | 1 | `H` |
| Stock Locate | 1 | 2 | Instrument locate code |
| Tracking Number | 3 | 2 | Nasdaq internal tracking field |
| Timestamp | 5 | 6 | Nanoseconds since midnight |
| Stock | 11 | 8 | ASCII symbol, right-padded with spaces |
| Trading State | 19 | 1 | `H`, `P`, `Q`, or `T` |
| Reserved | 20 | 1 | Reserved by protocol |
| Reason | 21 | 4 | Trading-action reason code |
| **Total** | — | **25 bytes** | — |

Trading State values are: `H` = halted, `P` = paused (Nasdaq-listed securities), `Q` = quotation-only period, and `T` = trading on Nasdaq. The ARM tracks the state for the selected instrument and exposes it to the host. The message does not itself add, cancel, or execute displayed orders.

The exact operational policy for host warnings or analytics during a halt/pause is a system-level decision. The order book must continue to reflect sequenced feed events; a trading-state change must not be mistaken for a sequence reset.

## 5.16 Supported-message length cross-check

The ARM parser must validate each supported ITCH message against its exact fixed length before reading fields or normalizing it. The MoldUDP64 Message Length field remains the packet-framing authority; this table is the ITCH message-level cross-check.

| ITCH type | Message | Total ITCH length |
|---|---|---:|
| `S` | System Event | 12 bytes |
| `R` | Stock Directory | 39 bytes |
| `H` | Stock Trading Action | 25 bytes |
| `A` | Add Order | 36 bytes |
| `F` | Add Order with MPID Attribution | 40 bytes |
| `E` | Order Executed | 31 bytes |
| `C` | Order Executed With Price | 36 bytes |
| `X` | Order Cancel | 23 bytes |
| `D` | Order Delete | 19 bytes |
| `U` | Order Replace | 35 bytes |
| `P` | Trade (Non-Cross) | 44 bytes |
| `Q` | Cross Trade | 40 bytes |
| `B` | Broken Trade | 19 bytes |

A message whose MoldUDP64 block length does not match the exact ITCH length for its type is malformed. It must not be partially parsed or forwarded.

------------------------------------------------------------------------

# 6. Message Processing Rules

## 6.1 Feed sequence before instrument filtering

The ARM must process MoldUDP64 framing and sequence continuity for the
entire feed stream before filtering for the selected instrument.

Reason: the sequence number covers the feed message stream, not only the
selected instrument. Filtering first would make valid messages for other
instruments appear to be missing sequence numbers.

## 6.2 Selected-instrument filtering

The initial project supports one active instrument.

The ARM uses the current-day Stock Locate mapping to determine whether a
parsed message belongs to the selected instrument.

Messages for other instruments are not forwarded to the FPGA, but they
still count as received feed messages for sequence tracking.

Message routing is message-type-aware; the ARM must not apply a single
Stock Locate filter blindly to every message:

- `S` (System Event) has Stock Locate `0` by definition. It is a global
  feed/day lifecycle message and must be processed by the ARM control
  path regardless of the selected instrument. It is not forwarded as an
  ordinary order-book event.
- `B` (Broken Trade) is instrument-specific and carries the Stock Locate
  of the security whose execution was broken. After feed-wide sequence
  accounting, the ARM checks this locate against the active instrument.
  For the selected instrument, the ARM routes the message to the broken-
  trade path and looks up its Match Number in the ledger. Before issuing a
  reversal, it verifies that the ledger entry identifies the same
  instrument/Stock Locate. A `B` with Stock Locate `0`, an unknown Match
  Number, or a locate/ledger mismatch must not cause a guessed reversal;
  record the error and invalidate `TRADE_STATS_VALID` as appropriate.
  A `B` for another instrument is not sent to the selected instrument's
  FPGA path, but it still consumes its normal feed sequence position.
- Other messages with Stock Locate `0`, or messages that are not
  instrument-specific, are handled according to their documented
  semantics and the explicit ARM control/skip policy. They must not be
  mistaken for selected-instrument order events.

## 6.3 Order state transitions

The core displayed-order operations are:

| ITCH message | Order-book effect |
|---|---|
| `A` / `F` | Create a new order |
| `E` | Reduce remaining quantity due to execution |
| `C` | Reduce remaining quantity due to execution at a specified price |
| `X` | Reduce remaining quantity due to partial cancellation |
| `D` | Remove all remaining quantity |
| `U` | Remove old order and create replacement under new reference number |
| `P` | No displayed-book update |
| `Q` | No ordinary displayed-order update |
| `B` | No current displayed-book update |

The order-book engine must preserve the distinction between cancellation
and execution. Cancellations do not contribute to executed volume.

## 6.4 Trade-statistics policy

The initial trade-statistics inclusion policy is:

| Message | Displayed-book effect | Trade-statistics treatment |
|---|---|---|
| `E` | Reduce the referenced resting order | Include the execution. Effective price is obtained from the resting order's stored display price. |
| `C` with Printable=`Y` | Reduce the referenced resting order | Include using the explicit execution price. |
| `C` with Printable=`N` | Reduce the referenced resting order | Exclude from ordinary volume/VWAP; the shares are represented in a later bulk print. Retain its Match Number as an excluded ledger entry so a later `B` is recognized. |
| `P` | No displayed-book update | Include using its explicit price and shares. The Order Reference Number is zero in the documented feed format. Current feeds are expected to carry `B` at the Buy/Sell Indicator; historical compatibility accepts `B` or `S`. Neither value is interpreted as the selected instrument's displayed resting-order side. |
| `Q` | No displayed-order update | Exclude from the initial analytics scope. Recognize and sequence the message. |
| `B` | No displayed-book update | Reverse the original included contribution for the referenced Match Number, if one exists and was included. |

A `B` message refers to a previously transmitted `E`, `C`, or `P` message. It does not refer to a `Q` cross-trade message. A trade break is final and must not be applied twice.

For `P`, the parser accepts ASCII `B` and legacy ASCII `S` in the Buy/Sell Indicator field for historical-capture compatibility. In a current-feed conformance profile, `S` is recorded as a non-fatal deviation from the post-14-July-2014 expectation; it is not grounds by itself to reject the message or to infer a resting-order side.

For VWAP, maintain integer accumulators:

- `notional_sum = Σ(price_integer × quantity)` for included trades;
- `volume_sum = Σ(quantity)` for included trades;
- `VWAP = notional_sum / volume_sum`, converted to display format only at the host boundary.

A broken trade subtracts the original trade's `price × quantity` and quantity from the corresponding accumulators. Define zero-volume behaviour explicitly in the analytics implementation (for example, report VWAP as unavailable rather than divide by zero).

## 6.5 Integer arithmetic

Prices and quantities must remain integer values in the protocol and FPGA processing path. Any conversion to decimal text or floating-point display values should occur at the host/display boundary, not in the order-book comparison or accumulator logic.

## 6.6 Match Number ledger and broken-trade ordering

The ARM owns a Match Number ledger in DDR. Each entry records at least the Match Number, trading-day/feed context, originating MoldUDP64 session, source ITCH type, feed sequence, Stock Locate (and the resolved instrument identity), effective price when known, quantity, whether the trade was included in statistics, and whether it has already been broken.

The ITCH 5.0 specification describes Match Numbers for `E` and `C` executions as day-unique, and Match Numbers for `P` trades as session-unique. `Q` also carries a day-unique Match Number, but `Q` is excluded from this project's initial analytics. A `B` message carries the Stock Locate and Match Number of the broken execution, but does not identify the original message type or its session. Therefore, the implementation must not assume that a bare 64-bit Match Number is globally unique across all event types and sessions. The selected feed's assignment behaviour and the practical relationship between these namespaces must be confirmed. Until then, the ledger must retain day, session, message type, sequence, and instrument context; lookup must detect ambiguous candidates rather than silently choosing one.

For included `E` executions, the FPGA resolves the effective price from the referenced resting order and returns a trade-contribution report to the ARM. For included `C` and `P` messages, the price is present in the ITCH message, but the ARM must still associate the ledger entry with successful FPGA processing before treating the contribution as committed. `C` with Printable=`N` is recorded as excluded and does not require an FPGA statistics contribution.

Because AXI4-Stream acceptance (`TVALID && TREADY`) is not proof that the FPGA has completed an event, the ARM must not process a later `B` correction against an incomplete ledger entry. When a `B` is received, the ARM holds that correction until the FPGA's ordered processing-progress indication confirms that all earlier selected-instrument events have been applied and their trade-contribution reports have been delivered. The ARM then looks up the Match Number and, if the original contribution was included, sends a `TRADE_REVERSAL` normalized event containing the original price and quantity. The FPGA subtracts that contribution from the trade accumulators.

If a `B` references a known excluded contribution (for example, `C` with Printable=`N`), mark the ledger entry broken without changing statistics. If the Match Number is unknown, or if an expected contribution report is missing or inconsistent, set `TRADE_STATS_VALID=0`, record the error, and do not invent a reversal amount. `BOOK_VALID` may remain asserted if the order book itself is unaffected. A duplicate `B` must not reverse a contribution twice.

Retain the ledger through the ITCH `S` event code `C` (End of Messages), not merely `E` (End of System Hours). Nasdaq specifies that Broken Trade and Order Delete messages may still arrive after End of System Hours; End of Messages is the final daily message. Archive or persist the ledger before clearing it for the next trading day.

------------------------------------------------------------------------

# 7. Sequence Management

## 7.1 Expected sequence tracking

The ARM maintains: - Active MoldUDP64 session identifier. - Next
expected message sequence number. - Last contiguous message sequence
number processed. - Feed-valid state. - Any buffered out-of-order or
future messages.

Sequence state is scoped to the MoldUDP64 session. A new session must be
handled as a session transition, not assumed to continue the prior
session's sequence without feed-specific confirmation.

## 7.2 Normal packet processing

For a packet containing `N` messages: 1. Validate the packet header and
session. 2. Read the first message sequence number. 3. Walk each
length-prefixed message block. 4. Assign consecutive sequence numbers to
messages in the packet. 5. Compare each message sequence with the
expected sequence. 6. Process only messages that are eligible for
ordered application. 7. Advance the expected sequence only according to
the defined contiguous-processing rule.

## 7.3 Sequence gap

A sequence gap means one or more messages expected by the receiver have
not yet been received.

Example:

``` text
Expected sequence: 103
Received sequence: 105

Missing sequence:  103 and 104
```

The ARM must not silently apply message 105 to the order-book stream
while 103 and 104 are missing.

Proposed response: 1. Mark the feed as not contiguous. 2. Stop
forwarding later messages to the FPGA for application. 3. Buffer later
messages within a bounded recovery buffer where possible. 4. Request the
missing message range from the configured re-request service. 5.
Validate and apply recovered messages in sequence. 6. Resume normal
forwarding only after continuity is restored.

The buffer capacity, request timeout, retry limits and escalation policy
are implementation parameters to be set and tested. They are not fixed
by the generic MoldUDP64 packet format.

## 7.4 Duplicate messages

A message whose sequence number has already been accepted for the active
session must not be applied to the order book a second time.

The receiver should identify duplicates using session and message
sequence number, not by comparing only the ITCH payload.

## 7.5 Out-of-order arrival

If a message arrives ahead of the next expected sequence number, the
receiver may buffer it while requesting or waiting for the missing
range.

It must not apply the later message before the missing earlier messages.

The maximum reorder/recovery window and timeout are project
configuration items.

## 7.6 Heartbeat processing

A heartbeat has a message count of zero. It carries the next expected
sequence number.

The receiver should use heartbeat sequence information to detect a gap
during periods with no ordinary data messages.

## 7.7 Session transition

The receiver must detect changes in the MoldUDP64 session field and
process end-of-session indications.

A new session requires a defined startup/synchronization procedure. The
implementation must not assume that the previous in-memory order book
remains valid for the new session without confirming the feed's recovery
procedure.

------------------------------------------------------------------------

# 8. Recovery and Snapshot Synchronization

## 8.1 Recovery objective

Recovery must restore a trustworthy order-book state after: - A feed
sequence gap that cannot be repaired by message re-request. - Receiver
restart or loss of feed state. - Recovery-buffer overflow. - A detected
order-book consistency failure. - A session transition requiring a new
synchronized state.

## 8.2 Incremental retransmission

The first recovery mechanism is to request missing messages from the
feed's configured re-request service, when those messages remain
available.

Recovered messages must be checked for: - Correct session. - Correct
requested sequence range. - Valid MoldUDP64 framing. - Valid message
lengths. - Correct sequence order.

They must be applied in the correct order and must not be applied twice.

## 8.3 Snapshot recovery

If the missing range is unavailable or incremental recovery cannot
safely restore continuity, the system requires a feed-supported
snapshot/recovery mechanism.

The Nasdaq ITCH specification references the GLIMPSE service for
obtaining current book state in the relevant TotalView-ITCH environment.
The exact service, snapshot format, entitlement, transport, and
synchronization rules must be confirmed for the feed used by this
project.

**Important:** A snapshot must be associated with an unambiguous feed
synchronization point. The system must know which sequence state is
already represented by the snapshot and which subsequent messages must
be applied.

## 8.4 Snapshot cutover rule

The project convention is that the snapshot synchronization sequence denotes the **last feed message already represented by the snapshot**. The first subsequently applied message is therefore `snapshot_sequence + 1`.

This convention is valid only when the actual snapshot/recovery service documents an equivalent sequence boundary. Before implementation, confirm the service's exact cutover semantics. Messages arriving during snapshot acquisition must be buffered and reconciled against that boundary. The FPGA order book remains invalid until the complete snapshot has been loaded, entry count and integrity checks pass, and all required post-snapshot messages have been applied contiguously.

Bulk snapshot loading uses a dedicated snapshot-loading interface or memory region, not the normal event stream. The snapshot transfer must include an expected entry count and integrity value; the ARM and FPGA independently calculate the checksum over an agreed set of fields. The checksum algorithm, field coverage, transfer framing and completion handshake are specified in the hardware/interface design documents.

## 8.5 Book validity

The system should distinguish: - **Feed validity:** Whether the ARM has
a contiguous, correctly identified feed stream. - **Book validity:**
Whether the FPGA's order book is known to represent the correct state at
the current feed position.

A contiguous feed does not automatically prove that the FPGA book is
correct if an internal processing error occurred.

During recovery, analytics may be retained for diagnostics or display,
but they must be marked stale/invalid and must not be presented as
current valid market state.

## 8.6 Recovery limitations

Recovery is feed-specific. This document defines the required behaviour
at a high level, but the actual snapshot and re-request workflow must be
verified against the service documentation available for the selected
feed.

------------------------------------------------------------------------

# 9. ARM--FPGA Normalized Event Interface

## 9.1 Purpose

The ARM parses the external ITCH message format and converts supported
messages into a project-defined normalized event representation.

This separates external feed details from the FPGA order-book engine.

The FPGA should not need to parse UDP, MoldUDP64 or the original ITCH
variable-length messages in the initial architecture.

## 9.2 Proposed data path

```text
UDP multicast
     |
     v
ARM: UDP reception and MoldUDP64 parsing
     |
     v
ARM: feed-wide sequence management and recovery
     |
     v
ARM: ITCH parsing and instrument filtering
     |
     v
ARM: normalized event generation + Match Number ledger
     |
     | AXI4-Stream
     v
FPGA: order-book engine + trade statistics
     |
     +---- trade-contribution reports / processing progress ----> ARM
```

The ARM performs transport framing, feed sequencing, ITCH parsing, instrument filtering, and normalization. The FPGA does not parse UDP, MoldUDP64, or raw ITCH messages in the initial architecture. The FPGA reports accepted trade contributions so the ARM can maintain the Match Number ledger and process later trade breaks safely.

## 9.3 Normalized event fields and logical bit layout

The ARM parses raw ITCH messages and produces a project-defined logical event. The logical payload is exactly 465 bits and is represented as `event_payload[464:0]`. Concatenate fields in the order shown below, from the most-significant end of the vector toward the least-significant end. Multi-bit numeric fields are unsigned; the most-significant bit of each field occupies the highest bit number in that field's range. This defines the canonical logical bit order. Section 9.5 defines how this logical vector is converted into a 59-byte image and mapped to AXI4-Stream byte lanes; `hardware_architecture.md` must implement and verify that contract.

| Field | Width | Logical bit range | Purpose / validity |
|---|---:|---:|---|
| Event type | 8 bits | `[464:457]` | Project event code from Section 9.4. |
| Order Reference Number | 64 bits | `[456:393]` | Affected displayed order; zero when not applicable. |
| New Order Reference Number | 64 bits | `[392:329]` | Replacement order ID for `REPLACE`; zero otherwise. |
| Side | 1 bit | `[328]` | `0` = BUY, `1` = SELL; valid only for `ADD`, zero otherwise. |
| Price | 32 bits | `[327:296]` | Display price for `ADD` or replacement display price for `REPLACE`; zero otherwise. |
| Quantity | 32 bits | `[295:264]` | Event quantity according to the validity table below. |
| Execution Price | 32 bits | `[263:232]` | Explicit match price for `EXECUTE_WITH_PRICE` and `TRADE`; original match price for `TRADE_REVERSAL`; zero for `EXECUTE` input. |
| Match Number | 64 bits | `[231:168]` | Match correlation for execution/trade events and reversal. Zero otherwise. |
| Event Flags | 8 bits | `[167:160]` | Bit definitions in Section 9.3.1. |
| Feed Sequence Number | 64 bits | `[159:96]` | MoldUDP64 message sequence assigned to the source ITCH message. |
| Exchange Timestamp | 64 bits | `[95:32]` | Original 48-bit ITCH timestamp zero-extended to 64 bits; nanoseconds since midnight. |
| Internal Stream Session ID | 32 bits | `[31:0]` | ARM-owned identifier for the current ARM-to-FPGA stream epoch; distinct from the 10-byte MoldUDP64 session. |
| **Total** | **465 bits** | **`[464:0]`** | **Logical payload before byte padding.** |

### 9.3.1 Event flag definitions

| Bit | Name | Meaning |
|---:|---|---|
| 0 | `PRINTABLE` | Used only by `EXECUTE_WITH_PRICE`: `1` means ITCH Printable=`Y`; `0` means Printable=`N`. Must be zero for all other event types. |
| 1 | `TRADE_STATS_UPDATE` | `1` means this event changes trade-statistics accumulators. It is set for `EXECUTE`, printable `EXECUTE_WITH_PRICE`, `TRADE`, and `TRADE_REVERSAL`. It is zero for non-printable `EXECUTE_WITH_PRICE` and all book-only events. The event type determines whether the arithmetic adds or subtracts. |
| 7:2 | Reserved | Must be transmitted as zero; receivers must reject non-zero reserved bits until a versioned interface extension defines them. |

### 9.3.2 Event field-validity rules

Fields marked unused must be set to zero by the ARM. The FPGA must not infer meaning from a non-zero value in an unused field.

| Event type | Order ID | New order ID | Side | Price | Quantity | Execution price | Match Number | Required flags |
|---|---|---|---|---|---|---|---|---|
| `ADD` | Valid | Zero | Valid | Display price | Added shares | Zero | Zero | All zero |
| `EXECUTE` | Valid | Zero | Zero | Zero | Executed shares | Zero on ingress; FPGA resolves from stored order | Valid | `TRADE_STATS_UPDATE=1` |
| `EXECUTE_WITH_PRICE` | Valid | Zero | Zero | Zero | Executed shares | Explicit ITCH execution price | Valid | `PRINTABLE` reflects `Y`/`N`; `TRADE_STATS_UPDATE=1` only for `Y` |
| `CANCEL` | Valid | Zero | Zero | Zero | Canceled shares | Zero | Zero | All zero |
| `DELETE` | Valid | Zero | Zero | Zero | Zero; FPGA removes all remaining shares | Zero | Zero | All zero |
| `REPLACE` | Original ID | New ID | Zero; side inherited from original FPGA order entry | New display price | New displayed shares | Zero | Zero | All zero |
| `TRADE` | Zero | Zero | Zero | Zero | Matched shares | Explicit ITCH `P` price | Valid | `TRADE_STATS_UPDATE=1` |
| `TRADE_REVERSAL` | Zero | Zero | Zero | Zero | Original trade quantity | Original trade price from ARM ledger | Valid | `TRADE_STATS_UPDATE=1`; `PRINTABLE=0` |

For `EXECUTE`, the FPGA must find the referenced resting order, capture its stored display price as the effective execution price, then apply the quantity reduction and issue the trade-contribution report to the ARM. The inbound execution-price field remains zero so that an absent source field cannot be mistaken for a zero-priced trade.

For `P` Trade (Non-Cross), Nasdaq specifies an Order Reference Number of zero in the binary feed and a Buy/Sell Indicator of `B` from the stated effective date. The parser also accepts historical `S` values as described in Section 5.10. Neither value describes a displayed resting order. Both are parsed/validated on the ARM and the normalized `TRADE` event carries no order ID and no meaningful side.

Instrument identifier is omitted because the initial design supports one selected instrument and ARM filters before forwarding. Runtime switching invalidates/reinitializes the stream and assigns a new internal stream session ID. If multiple instruments are later supported concurrently, the event format must be extended with an instrument identifier.

The local ARM receive timestamp is captured separately using the ARM's monotonic clock. It is not included in this logical event payload; it is retained in ARM-side receive/latency telemetry. The exchange timestamp remains the original ITCH nanoseconds-since-midnight value and must not be substituted with local time.

The external MoldUDP64 session identifier is 10 bytes and is owned by the feed protocol. The internal stream session ID is 32 bits and is owned by the ARM–FPGA interface. They are not interchangeable.

**Payload-width decision:** Revision 0.5 retains the 465-bit logical event format. The project does not narrow the native 48-bit timestamp, pack Side into Event Flags, or reduce the 32-bit internal stream session ID in this revision. Width optimization is deferred until synthesis, timing, buffering, and end-to-end throughput measurements demonstrate a meaningful benefit. Any future field-width or field-layout change requires an explicit normalized-interface version change and updated ARM/RTL golden vectors.

## 9.4 Project-defined event codes

| Code | Event | Source / effect |
|---:|---|---|
| `0x01` | `ADD` | ITCH `A` or `F`; insert displayed order. |
| `0x02` | `EXECUTE` | ITCH `E`; reduce order and include trade contribution. |
| `0x03` | `EXECUTE_WITH_PRICE` | ITCH `C`; reduce order, with Printable flag controlling statistics inclusion. |
| `0x04` | `CANCEL` | ITCH `X`; reduce remaining displayed quantity. |
| `0x05` | `DELETE` | ITCH `D`; remove remaining displayed quantity. |
| `0x06` | `REPLACE` | ITCH `U`; replace old order ID with new order ID. |
| `0x07` | `TRADE` | ITCH `P`; update trade statistics only. |
| `0x08` | `TRADE_REVERSAL` | Project event generated from ITCH `B`; reverse a previously included trade contribution. |

These values are project-defined and are not the ASCII ITCH message-type bytes. `Q` is recognized and sequenced by the ARM but is not forwarded as a normal book/trade-statistics event in the initial scope. `S`, `R`, and `H` are handled by the ARM control/state path and host status, not as ordinary order-book events.

### 9.4.1 ITCH-to-normalized-event mapping

The ARM assigns the MoldUDP64 sequence number to each ITCH message before instrument filtering. The source fields below are mapped only after the message has passed framing, length, encoding, and selected-instrument validation.

| Source message | Project event | Source-to-normalized mapping | FPGA effect / special handling |
|---|---|---|---|
| `A` | `ADD` (`0x01`) | Order Reference Number → Order ID; Buy/Sell Indicator → Side (`B`→0, `S`→1); Shares → Quantity; Price → Price; Timestamp → Exchange Timestamp; MoldUDP64 sequence → Feed Sequence Number. | Insert displayed order. |
| `F` | `ADD` (`0x01`) | Same mapping as `A`; parse the four-byte Attribution field but omit it from the initial normalized payload because participant analytics are out of scope. | Insert displayed order. |
| `E` | `EXECUTE` (`0x02`) | Order Reference Number → Order ID; Executed Shares → Quantity; Match Number → Match Number; Timestamp and MoldUDP64 sequence → event metadata. Execution Price is zero on ingress. | Reduce resting quantity; resolve effective price from the FPGA order entry; report applied trade contribution to ARM. |
| `C` Printable=`Y` | `EXECUTE_WITH_PRICE` (`0x03`) | Order Reference Number → Order ID; Executed Shares → Quantity; Execution Price → Execution Price; Match Number → Match Number; Printable=`Y` → `PRINTABLE=1`; set `TRADE_STATS_UPDATE=1`. | Reduce resting quantity and include explicit-price trade contribution. |
| `C` Printable=`N` | `EXECUTE_WITH_PRICE` (`0x03`) | Same order fields as `C(Y)`; Printable=`N` → `PRINTABLE=0`; set `TRADE_STATS_UPDATE=0`. | Reduce resting quantity but do not add volume/VWAP contribution. Keep an excluded Match Number ledger entry so a later `B` is recognized. |
| `X` | `CANCEL` (`0x04`) | Order Reference Number → Order ID; Canceled Shares → Quantity; timestamp and feed sequence → metadata. | Subtract canceled quantity; no trade statistics. |
| `D` | `DELETE` (`0x05`) | Order Reference Number → Order ID; all other operation fields zero. | Remove the order's entire remaining quantity. |
| `U` | `REPLACE` (`0x06`) | Original Order Reference Number → Order ID; New Order Reference Number → New Order ID; Shares → Quantity; Price → Price. | Replace old order with new ID, preserving the original side and inherited attributes. |
| `P` | `TRADE` (`0x07`) | Shares → Quantity; Price → Execution Price; Match Number → Match Number; Order Reference Number → zero/unused; Buy/Sell Indicator is parsed as `B` or historical `S` for compatibility but is not mapped; order ID and side set to zero. | Update trade statistics only; no displayed-book operation. |
| `B` | Conditional `TRADE_REVERSAL` (`0x08`) | Stock Locate → selected-instrument check and ledger sanity check; Match Number → ledger lookup. If the original contribution was included, ledger price/quantity → Execution Price/Quantity; preserve Match Number. | Subtract the original contribution only after the ledger instrument/Stock Locate matches the `B` message. If the original was excluded (for example `C(N)`), mark it broken in ARM ledger without sending a reversal event. |
| `Q` | No normalized event | Validate its full 40-byte format, instrument identity, sequence position, and cross type. | Recognize/sequence only; excluded from initial book and trade-statistics path. |
| `S` | ARM control event only | Event Code and Timestamp update feed/day lifecycle state. | Not sent to the ordinary FPGA order-book event stream. |
| `R` | ARM instrument-directory update | Stock Locate and Stock, plus any retained attributes, update current-day instrument map. | Not sent as an order-book event. |
| `H` | ARM instrument-status update | Stock Locate, Trading State, Reason, and Timestamp update the selected instrument's trading state. | Not sent as an order-book event. |

The ITCH Tracking Number is not mapped into the 465-bit payload. The ARM may retain it in diagnostic metadata, but it must never be substituted for the MoldUDP64 Feed Sequence Number. The Stock Locate and Stock fields are consumed by ARM-side instrument selection and are omitted from the initial single-instrument event payload.

## 9.5 AXI4-Stream transfer semantics and canonical byte serialization

The proposed event-data path uses AXI4-Stream. A transfer is accepted on
a clock edge when `TVALID=1` and `TREADY=1`. If `TVALID` is asserted
while `TREADY` is low, the producer must keep `TVALID`, `TDATA`, `TKEEP`,
and `TLAST` stable until the transfer is accepted.

The AXI4-Stream handshake confirms transfer acceptance only. It does not
mean that the FPGA has completed the order-book state update. Processing
completion and errors are reported through the separate interface
described in Section 9.7.

### 9.5.1 Canonical serialized event image

The 465-bit logical event payload is serialized as a 59-byte,
most-significant-byte-first image. Seven zero padding bits are appended
after logical bit 0:

```systemverilog
logic [464:0] event_payload;
logic [471:0] event_image;

assign event_image = {event_payload[464:0], 7'b0};
```

For serialized byte index `i`, where `0 <= i < 59`:

```systemverilog
serialized_byte[i] = event_image[471 - 8*i -: 8];
```

Therefore, byte 0 contains `event_payload[464:457]`, and bytes proceed
from the most-significant end of the logical payload toward the least-
significant end. The final serialized byte contains the remaining seven
payload bits followed by the single least-significant padding bit. The
padding bit must be zero.

This is a project serialization convention. It is deliberately distinct
from the numeric byte order used by the ITCH parser: ITCH multi-byte
fields are decoded from big-endian network byte order into logical
integer values first; the complete normalized event is then serialized
according to the rule above.

### 9.5.2 Mapping serialized bytes to AXI4-Stream lanes

For a 256-bit `TDATA` interface, each AXI byte lane carries one
consecutive serialized byte. Byte lane 0 is `TDATA[7:0]`, lane 1 is
`TDATA[15:8]`, and so on. The first serialized (most-significant) event
byte is therefore placed in the least-significant AXI byte lane:

```systemverilog
// Beat 0: serialized bytes 0 through 31
TDATA_beat0[8*i +: 8] = serialized_byte[i];       // 0 <= i <= 31

// Beat 1: serialized bytes 32 through 58
TDATA_beat1[8*i +: 8] = serialized_byte[32 + i];  // 0 <= i <= 26
```

The transfer uses two beats:

| Beat | Serialized byte indexes | Valid bytes | `TKEEP` | `TLAST` |
|---|---|---:|---|---:|
| 0 | 0–31 | 32 | `0xFFFFFFFF` | `0` |
| 1 | 32–58 | 27 | `0x07FFFFFF` | `1` |

On beat 1, `TKEEP[26:0]` is asserted and `TKEEP[31:27]` is deasserted.
The five unused byte lanes must be driven to zero. All seven padding
bits in the final serialized byte must also be zero. The AXI lane mapping
is not native little-endian integer serialization: it is the explicitly
defined mapping of the first serialized byte to `TDATA[7:0]`.

The ARM C/C++ implementation must use explicit byte-wise packing and
unpacking according to this contract. It must not rely on native C++
structure layout, compiler padding, or host endianness. The corresponding
SystemVerilog implementation must use the same byte-index convention.
`hardware_architecture.md` must repeat this mapping in the interface
specification and include shared golden test vectors for ARM and RTL.

## 9.6 Control and status path

AXI4-Lite is used for configuration and status, including `FEED_VALID`, `BOOK_VALID`, `TRADE_STATS_VALID`, recovery state, error status, and performance counters. The detailed register map, interrupt topology, and diagnostic subsystem implementation belong in the hardware architecture document.

`FEED_VALID` is owned by the ARM because it knows whether the feed session and sequence are contiguous. `BOOK_VALID` is owned by the FPGA because it knows whether all accepted events have been applied consistently. `TRADE_STATS_VALID` is tracked separately because a trade-ledger or broken-trade problem can invalidate statistics without necessarily invalidating the displayed order book.

## 9.7 Event ordering and processing reports

The ARM preserves the order of selected-instrument events. The FPGA applies them in that same order; no later event may overtake an earlier event.

The AXI-Stream handshake confirms transfer acceptance only. The FPGA therefore provides a separate ordered processing-progress indication and a trade-contribution report path to the ARM. A trade-contribution report includes at least feed sequence, Match Number, effective price, quantity, and processing result. For an `E` event, the FPGA obtains the effective price from the order entry before applying the execution.

The ARM may stream ordinary events continuously. When it receives a `B` message, it must hold the correction and stop forwarding later selected-instrument events until the processing-progress indication confirms that all preceding selected-instrument events have completed and all corresponding trade-contribution reports have been delivered. The ARM may continue receiving feed datagrams, but must buffer subsequent events within a bounded queue. It then resolves the Match Number in its DDR ledger and sends a `TRADE_REVERSAL` event only when the original contribution was included. Later selected-instrument events resume only after the reversal has been accepted and its processing result is known. This barrier prevents an asynchronous FPGA report from arriving after the ARM has already tried to reverse the trade.

The precise report-channel encoding, queue depth, progress-watermark semantics, timeout behaviour, and backpressure handling are interface-design items for `hardware_architecture.md`.

------------------------------------------------------------------------

# 10. Error Handling

## 10.1 Error categories

The implementation should distinguish errors by where they occur.

| Category | Example | Responsible area |
|---|---|---|
| Network/transport | UDP datagram loss | Network/ARM receive path |
| MoldUDP64 framing | Invalid message count or truncated message block | ARM protocol parser |
| Sequence | Missing or duplicate message sequence | ARM sequence manager |
| ITCH parsing | Unsupported type, invalid length, or invalid field encoding | ARM ITCH parser |
| Instrument mapping | Unknown or stale Stock Locate mapping | ARM instrument directory/filter |
| Normalized interface | Invalid event code or malformed normalized event | ARM–FPGA interface |
| Order-book consistency | Unknown order on cancel/execute, duplicate add, or impossible quantity transition | FPGA order-book engine |
| Recovery | Snapshot mismatch or recovery timeout | ARM recovery coordinator and FPGA state |

## 10.2 Malformed MoldUDP64 packet

The ARM validates the complete downstream datagram before forwarding any message from it. Validation includes the fixed header, message count, every two-byte message-length field, each complete message block, and exact consumption of the datagram payload.

If any part is malformed or truncated, reject the entire datagram; do not forward earlier message blocks from that same datagram. Record the session, packet starting sequence, declared message count, and failure reason where available. Do not parse beyond the validated packet boundary. Reconcile the affected sequence range using subsequent packets/heartbeats and the configured recovery service; mark `FEED_VALID=0` and `BOOK_VALID=0` whenever continuity cannot be proven.

## 10.3 Unsupported ITCH message

If a message type is on the explicitly recognized skip list (`Y`, `L`, `V`, `W`, `J`, `K`, `I`, `N`) and its MoldUDP64 message boundary and expected ITCH length are valid, preserve feed sequence accounting and skip forwarding it to the FPGA.

For an unknown message type, stop forwarding later events, record the type and sequence, and mark the feed as untrusted. Because the unknown message may alter order-book state, conservatively clear `BOOK_VALID` until its semantics are understood and a recovery/snapshot establishes a trustworthy state. A known, correctly framed skip-list message is not itself a sequence gap.

If any message boundary or length is not trustworthy, reject the entire malformed MoldUDP64 datagram before forwarding any message from it. Do not parse beyond validated packet boundaries. Enter recovery if continuity or message integrity can no longer be established.

## 10.4 Invalid order-book operation

Examples include: - ADD for an order reference that already exists. -
CANCEL for an unknown order. - EXECUTE for an unknown order. -
Cancellation or execution quantity greater than the known remaining
quantity. - REPLACE for an unknown original order.

The FPGA must not perform a state update that would create an invalid
quantity or corrupt the book.

The initial policy is: 1. Reject the invalid state transition. 2.
Preserve the current order-book contents. 3. Report the error and
associated event context. 4. Determine whether the event indicates
feed/parser corruption, lost event history, or an FPGA implementation
defect. 5. Mark the book invalid and initiate recovery if the
correctness of the reconstructed state can no longer be guaranteed.

An invalid order-book transition is not automatically a network sequence gap. The FPGA rejects the state transition, preserves the last known state for diagnostics, reports the event context, and clears `BOOK_VALID` whenever correctness of the reconstructed state can no longer be guaranteed. The ARM then coordinates recovery. The implementation must classify errors rather than treating every unknown-order condition as packet loss.

## 10.5 Error reporting

At minimum, the system should make it possible to identify: - Error
category and code. - Associated feed sequence number, when available. -
Associated order reference number, when available. - Whether the feed is
valid. - Whether the book is valid. - Whether trade statistics are valid. - Whether recovery is in progress.

The physical diagnostic transport, interrupt structure and register
allocation are implementation details for the hardware architecture
document.

------------------------------------------------------------------------

# 11. Instrument Selection and Switching

## 11.1 Initial single-instrument mode

The initial system supports one active instrument.

The ARM resolves the selected symbol using the current day's Stock
Directory information and filters messages using the corresponding Stock
Locate value.

Sequence tracking remains global to the received feed stream and must
not be limited to the selected instrument.

## 11.2 Runtime instrument switching

Runtime instrument switching is supported only as a controlled restart/reconstruction operation:

1. Stop forwarding events for the old instrument.
2. Mark `BOOK_VALID=0` and make the old book unavailable as current state.
3. Resolve the new instrument's current-day Stock Locate value from `R` messages.
4. Start a new internal ARM–FPGA stream session ID.
5. Obtain a feed-supported synchronized starting state for the new instrument.
6. Load the snapshot or otherwise reconstruct the new book, then apply subsequent selected-instrument events in the correct sequence.
7. Set `BOOK_VALID=1` only after synchronization and contiguous application complete.

Feed-wide MoldUDP64 sequence tracking continues throughout the switch; it is not reset merely because the selected instrument changes. If the actual feed/snapshot service cannot provide a safe synchronization point, switching must be implemented as a full controlled restart rather than an unsynchronized in-place change.

------------------------------------------------------------------------

# 12. Verification Requirements

The protocol implementation must be verified independently at the ARM
parser, normalized interface and FPGA event-processing boundaries.

## 12.1 MoldUDP64 test cases

At minimum, test: - Valid packet with one message. - Valid packet with
multiple messages. - Correct message-length parsing. - Heartbeat packet
with message count zero. - End-of-session packet. - Sequence gap. -
Duplicate sequence. - Out-of-order packet/message arrival. - Re-request
response. - Wrong or unexpected session. - Truncated header. - Truncated
message block. - Invalid message count or inconsistent packet length.

## 12.2 ITCH parser test cases

At minimum, test: - Valid `S`, `R`, `H`, `A`, `F`, `E`, `C`, `X`, `D`, `U`, `P`, `Q`, and `B` messages. - Correct field offsets and lengths, including `R` (39 bytes) and `H` (25 bytes). - Big-endian
decoding. - ASCII and space-padded fields. - Price scaling. -
Nanoseconds-since-midnight timestamp decoding. - `P` Buy/Sell Indicator=`B` in current-feed mode. - Historical `P` message with Buy/Sell Indicator=`S` accepted in compatibility mode and not mapped to Side. - Invalid `P` indicator values rejected or flagged as malformed according to parser policy. - Unsupported message
type. - Invalid message length. - Boundary values for quantity, price
and order reference fields.

## 12.3 Order-book event test cases

At minimum, test: - Add buy order. - Add sell order. - Partial cancel. -
Full delete. - Partial execution. - Full execution. - Execution with a
different execution price. - Non-printable execution handling. - Order
replacement and new order reference number. - Non-displayed `P` trade that must not modify the displayed book but must update eligible statistics. - `Q` recognition without inclusion in initial statistics. - `B` reversal for prior `E`, printable `C`, and `P` contributions. - `B` for excluded non-printable `C`, duplicate `B`, unknown Match Number, and `B` arriving before the FPGA trade report is committed. - Unknown order
reference. - Duplicate add. - Quantity underflow attempt. - Correct best
bid and best ask after each event.

## 12.4 Sequence and recovery test cases

At minimum, test: - Normal contiguous feed. - Missing sequence followed
by successful re-request. - Missing sequence unavailable for
retransmission. - Buffer limit reached. - Snapshot recovery. - Snapshot
synchronization boundary. - Duplicate recovered message. - Session
transition. - ARM restart or receiver state loss. - Book remains invalid
until synchronization is complete.

## 12.5 ARM--FPGA interface test cases

At minimum, test: - Correct normalized event encoding and padding. - Canonical MSB-first 59-byte serialization. - First serialized byte placed on `TDATA[7:0]`; subsequent bytes placed on ascending byte lanes. - Correct two-beat 256-bit AXI4-Stream framing, `TKEEP`, and `TLAST`. - ARM and RTL agree on shared golden vectors. - Correct sequence and internal stream-session preservation. - `TVALID`/`TREADY` handshake and stable payload during backpressure. - No event reordering. - Invalid event code. - FPGA processing error reporting. - Ordered processing-progress reporting. - Trade-contribution reports for `E`, `C(Y)`, and `P`. - `B` barrier and reversal ordering. - Independent `FEED_VALID`, `BOOK_VALID`, and `TRADE_STATS_VALID` transitions.

## 12.6 Reference-model comparison

A software reference model should process the same ordered
normalized-event stream as the FPGA.

After each event or selected checkpoint, compare: - Active order
count. - Per-order remaining quantity. - Best bid. - Best ask. -
Spread. - Selected market-depth values. - Trade statistics, if
enabled. - Book-validity state.

The software model is a verification reference, not the production
source of truth.

------------------------------------------------------------------------

# 13. Open Items and Assumptions

The following items remain open after consolidation. They are not silently assumed to be solved by the protocol standards.

| ID | Open item | Required follow-up |
|---|---|---|
| OI-01 | Exact feed product and service configuration | Confirm multicast groups, ports, re-request servers, entitlement, maximum packet size, request limits, and retention window with the actual provider. |
| OI-02 | Snapshot/recovery service | Confirm whether GLIMPSE or another service is available, its snapshot format, synchronization marker semantics, and cutover procedure. |
| OI-03 | Recovery buffering and timeout parameters | Select buffer capacity, reorder window, timeout, retry policy, and snapshot escalation behaviour from measured traffic and service limits. |
| OI-04 | AXI4-Stream packing conformance | Implement the canonical MSB-first serialized-byte mapping from Section 9.5 in `hardware_architecture.md`; verify `TDATA` lanes, `TKEEP`, `TLAST`, padding, and ARM/RTL golden vectors. The logical byte order and lane convention are specified here. |
| OI-05 | Processing-report interface | Define report queue depth, report encoding, sequence-progress watermark, timeout, and the precise `B` barrier protocol. |
| OI-06 | Match Number ledger identity and persistence | Confirm provider-specific Match Number namespace behaviour (especially `P` session-unique versus `E`/`C` day-unique); size the DDR hash table; define collision handling, retention, archival, restart persistence, and daily rollover. |
| OI-07 | Snapshot transfer interface | Define snapshot entry layout, loading protocol, checksum algorithm/coverage, completion signalling, and interaction with normal event ingress. |
| OI-08 | Trade-statistics scope | Confirm whether additional analytics beyond cumulative volume and VWAP are required, including rolling windows and any separate cross-volume metric. |
| OI-09 | Timestamp and latency measurement | Define ARM monotonic clock source, timestamp capture point, clock synchronization assumptions, and how exchange/local timestamps are compared. |
| OI-10 | Isolated FPGA state-error policy | Finalize which detected order-book errors always invalidate the book and which can be safely rejected without further state loss. |
| OI-11 | Runtime switch behaviour | Verify that the selected feed's snapshot/recovery service can support synchronized instrument switching; otherwise require a full controlled restart. |
| OI-12 | Unknown message policy | Maintain an explicit versioned list of recognized-but-skipped message types and update it when the selected feed specification changes. |

------------------------------------------------------------------------

# 14. References

The official specifications take precedence over this project's summaries if a discrepancy is found.

1. Nasdaq, **TotalView-ITCH 5.0 Interface Specification** (Nasdaq Trader-hosted copy): https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification_5.0.pdf
2. Nasdaq, **TotalView-ITCH 5.0 Interface Specification** (PDF dated 14 September 2026): https://assets.ctfassets.net/mx0rke14e5yt/5Uz6MGJxbo4wRPou8KveFs/0c3143e52b4e9e586db7d9194c444326/9-14-26_TVITCH_5.0.pdf
3. Nasdaq, **MoldUDP64 Protocol Specification, Version 1.00**: https://nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf
4. AMD/Xilinx AXI4-Stream, Zynq-7000, and 7-series documentation: consult the documents matching the selected Vivado release and Zynq-7020 device when the hardware interface is frozen.

------------------------------------------------------------------------

# Appendix A --- Worked MoldUDP64 Example

Assume a downstream packet contains three ITCH messages and the packet
header sequence number is 500.

| Message block | ITCH message | MoldUDP64 sequence |
|---:|---|---:|
| 1 | Add Order (`A`) | 500 |
| 2 | Order Cancel (`X`) | 501 |
| 3 | Order Executed (`E`) | 502 |

The receiver: 1. Reads the MoldUDP64 header. 2. Sees that the first
message sequence is 500. 3. Reads the first two-byte message length and
extracts the first ITCH message. 4. Assigns sequence 500 to that
message. 5. Repeats for the next message blocks, assigning 501 and 502.
6. Verifies that the message count and packet boundaries are consistent.
7. Processes the messages in sequence order.

The sequence numbers are supplied by MoldUDP64 framing; they are not
embedded in the ITCH messages.

# Appendix B --- Worked Order-Book Example

Assume the selected instrument receives these normalized events in
order:

| Sequence | Event | Order ID | Side | Price | Quantity |
|---:|---|---:|---|---:|---:|
| 100 | ADD | 1001 | BUY | 100.5000 | 200 |
| 101 | ADD | 1002 | SELL | 100.7000 | 150 |
| 102 | CANCEL | 1001 | — | — | 50 |
| 103 | EXECUTE | 1002 | — | — | 50 |

Expected remaining orders after processing:

| Order ID | Side | Price | Remaining quantity |
|---:|---|---:|---:|
| 1001 | BUY | 100.5000 | 150 |
| 1002 | SELL | 100.7000 | 100 |

Expected top of book:

| Metric | Expected value |
|---|---:|
| Best Bid | 100.5000 |
| Best Ask | 100.7000 |
| Spread | 0.2000 |

The example is illustrative. A complete test vector must include the
exact binary ITCH message bytes and MoldUDP64 packet framing.


------------------------------------------------------------------------

# Appendix C --- Worked ITCH Bytes-to-Normalized-Event Examples

The following vectors are synthetic protocol test vectors. They demonstrate byte offsets, big-endian decoding, MoldUDP64 sequence assignment, and the logical normalized-event values. They are not captured exchange traffic.

## C.1 Example 1: MoldUDP64 packet carrying an Add Order (`A`)

Assume the MoldUDP64 session is the ten-byte ASCII value `SIM2026A01`, the first message sequence number is 500, and the packet contains one message. The ITCH message is an Add Order for symbol `ABC`, Stock Locate 7, order reference `0x0102030405060708`, buy side, 1,000 shares, and price `100.0000`.

### MoldUDP64 packet header and block prefix

| Field | Bytes (hex) | Interpretation |
|---|---|---|
| Session | `53 49 4D 32 30 32 36 41 30 31` | ASCII `SIM2026A01` |
| Sequence Number | `00 00 00 00 00 00 01 F4` | 500; sequence of first message |
| Message Count | `00 01` | One message block |
| Message Length | `00 24` | 36 ITCH message bytes; excludes this two-byte prefix |

### ITCH `A` message bytes

| Offset | Field | Bytes (hex) | Decoded value |
|---:|---|---|---|
| 0 | Message Type | `41` | `A` |
| 1 | Stock Locate | `00 07` | 7 |
| 3 | Tracking Number | `12 34` | `0x1234` (not the MoldUDP64 sequence) |
| 5 | Timestamp | `00 00 3B 9A CA 00` | 1,000,000,000 ns since midnight |
| 11 | Order Reference Number | `01 02 03 04 05 06 07 08` | `0x0102030405060708` |
| 19 | Buy/Sell Indicator | `42` | ASCII `B` = buy |
| 20 | Shares | `00 00 03 E8` | 1,000 |
| 24 | Stock | `41 42 43 20 20 20 20 20` | `ABC` followed by five spaces |
| 32 | Price | `00 0F 42 40` | 1,000,000 × 10⁻⁴ = `100.0000` |
| — | **Total** | **36 bytes** | — |

### Resulting normalized `ADD` event

Assume the ARM has assigned internal stream session ID 1.

| Normalized field | Value | Explanation |
|---|---|---|
| Event type | `0x01` | `ADD` |
| Order Reference Number | `0x0102030405060708` | From ITCH offset 11 |
| New Order Reference Number | `0` | Unused for `ADD` |
| Side | `0` | BUY |
| Price | `1,000,000` | Raw Price (4) integer; display as `100.0000` |
| Quantity | `1,000` | From Shares |
| Execution Price | `0` | Unused for `ADD` |
| Match Number | `0` | Unused for `ADD` |
| Event Flags | `0x00` | No flags set |
| Feed Sequence Number | `500` | Assigned by MoldUDP64 header/block position |
| Exchange Timestamp | `1,000,000,000` | Zero-extended 48-bit ITCH timestamp |
| Internal Stream Session ID | `1` | ARM-owned stream epoch |

The FPGA inserts the order into the buy side of the displayed order book. The ITCH Stock Locate and Stock fields are consumed by the ARM's instrument filter and are not repeated in the single-instrument normalized payload.

## C.2 Example 2: Order Executed (`E`) and FPGA price resolution

Assume the order from Example C.1 is still present with 1,000 shares at `100.0000`. The next ITCH message reports that 250 shares were executed. The MoldUDP64-assigned sequence is 501.

### ITCH `E` message bytes

| Offset | Field | Bytes (hex) | Decoded value |
|---:|---|---|---|
| 0 | Message Type | `45` | `E` |
| 1 | Stock Locate | `00 07` | 7 |
| 3 | Tracking Number | `12 35` | `0x1235` |
| 5 | Timestamp | `00 00 77 35 94 00` | 2,000,000,000 ns since midnight |
| 11 | Order Reference Number | `01 02 03 04 05 06 07 08` | `0x0102030405060708` |
| 19 | Executed Shares | `00 00 00 FA` | 250 |
| 23 | Match Number | `11 22 33 44 55 66 77 88` | `0x1122334455667788` |
| — | **Total** | **31 bytes** | — |

### Resulting normalized `EXECUTE` event

| Normalized field | Value | Explanation |
|---|---|---|
| Event type | `0x02` | `EXECUTE` |
| Order Reference Number | `0x0102030405060708` | Referenced resting order |
| New Order Reference Number | `0` | Unused |
| Side | `0` | Unused; must be zero |
| Price | `0` | Unused |
| Quantity | `250` | Executed shares |
| Execution Price | `0` on ingress | The ITCH `E` message contains no execution price |
| Match Number | `0x1122334455667788` | From ITCH message |
| Event Flags | `0x02` | `TRADE_STATS_UPDATE=1` |
| Feed Sequence Number | `501` | MoldUDP64 sequence |
| Exchange Timestamp | `2,000,000,000` | Zero-extended ITCH timestamp |
| Internal Stream Session ID | `1` | Same stream epoch as Example C.1 |

The FPGA looks up the order and resolves the effective execution price from its stored display price: `1,000,000` (`100.0000`). It reduces the remaining order quantity to 750 and sends the ARM a trade-contribution report containing the feed sequence, Match Number, effective price, quantity, and processing result. The ARM commits this information to its Match Number ledger only after the report confirms successful processing.

## C.3 Example 3: Broken Trade (`B`) and normalized reversal

Assume the ARM has received and committed the successful trade-contribution report from Example C.2. A later `B` message references the same Match Number. The ARM waits for its ordered processing barrier, looks up the committed ledger entry, and constructs a reversal. The MoldUDP64 sequence is 502.

### ITCH `B` message bytes

| Offset | Field | Bytes (hex) | Decoded value |
|---:|---|---|---|
| 0 | Message Type | `42` | `B` |
| 1 | Stock Locate | `00 07` | 7 |
| 3 | Tracking Number | `12 36` | `0x1236` |
| 5 | Timestamp | `00 00 B2 D0 5E 00` | 3,000,000,000 ns since midnight |
| 11 | Match Number | `11 22 33 44 55 66 77 88` | `0x1122334455667788` |
| — | **Total** | **19 bytes** | — |

### Resulting normalized `TRADE_REVERSAL` event

| Normalized field | Value | Explanation |
|---|---|---|
| Event type | `0x08` | `TRADE_REVERSAL` |
| Order Reference Number | `0` | A trade reversal does not modify the current displayed order |
| New Order Reference Number | `0` | Unused |
| Side | `0` | Unused |
| Price | `0` | Unused |
| Quantity | `250` | Original trade quantity from ARM ledger |
| Execution Price | `1,000,000` | Original effective execution price from ARM ledger (`100.0000`) |
| Match Number | `0x1122334455667788` | Match being reversed |
| Event Flags | `0x02` | `TRADE_STATS_UPDATE=1`; event code defines subtraction |
| Feed Sequence Number | `502` | Sequence of the `B` message |
| Exchange Timestamp | `3,000,000,000` | Timestamp of the `B` message, zero-extended |
| Internal Stream Session ID | `1` | Same stream epoch |

The FPGA subtracts `250` from the trade volume accumulator and `1,000,000 × 250` from the notional accumulator. The order book remains unchanged by the reversal. The ARM marks the ledger entry broken so a duplicate `B` cannot reverse it twice. If the original ledger entry were for an excluded `C` message with Printable=`N`, the ARM would mark it broken without sending a `TRADE_REVERSAL` event.


------------------------------------------------------------------------

**End of document --- Version 0.5**
