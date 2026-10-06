# artdaq-core-mu2e — interfaces

The interface of this package is its public headers. Everything downstream —
`artdaq-mu2e` generators and art modules, every `otsdaq-mu2e*` art module,
`mu2e-pcie-utils` (`DTCLib`/`CFOLib` hardware access and the `mu2esim` DTC
simulator) and Mu2e Offline — includes these headers and links
`artdaq-core-mu2e::Overlays` or `artdaq-core-mu2e::Data`. A change of a field
width or offset here is a wire-format change for all of them.

**Baseline.** Line numbers are for `cf86c07` (version 9.02.00,
`CMakeLists.txt:13`), the commit the superproject pins. It sits on the fork
`normanajn/artdaq-core-mu2e` and is 68 commits behind (2 ahead of)
`Mu2e/artdaq-core-mu2e` `develop`, which is 9.07.00 at `7ae4622`
(2026-10-06). Every header was diffed against upstream `develop` with
whitespace ignored. Upstream is mostly a clang-format pass. No layout in
§2–§9 changed, apart from three semantic differences: `STMFragment` is
rewritten (§7.1), the parse loops no longer spin (§5.3), and the JSON
printers add hex values. Upstream also drops the `auto_approve` workflow and
adds `.github/copilot-instructions.md`.

Reviewed under the mega-review Tier B rule: `CMakeLists.txt`, every header under `Data/` and
`Overlays/`, registration macros, `test/` and `.github/` file names. Where a
header declares a pack/unpack function and does not itself define the bytes,
only the few lines of that named function were read; those are cited as
`*.cpp:line`. Paths below are relative to `artdaq-core-mu2e/` (the source
directory inside the repository) unless they start with `doc/` or `test/`.

The packet specification is **Mu2e-doc-4914** ("Mu2e data format"). The
headers cite it in `Data/EventHeader.hh:7`, `Data/RunHeader.hh:7`,
`Data/SubRunHeader.hh:7`, `Data/EWT.hh:7` and
`Overlays/Decoders/CRVDataDecoder.hh:23`; none of the `DTC_Packets/` headers
cites a document. The CFO event record cites **Mu2e-doc-19095 p. 22**
(`Data/EventHeader.hh:8`). Spec comparisons below use V53 (21 Feb 2026), the
copy shipped as `mu2edaq-dataformat-viewer/doc/mu2e-dataformat.pdf`, as
tabulated by that package's review (`mu2edaq-dataformat-viewer/doc/INTERFACES.md` §4).

---

## 1. Build products

| Target | Source | Contents |
|---|---|---|
| `artdaq-core-mu2e::Overlays` | `Overlays/CMakeLists.txt:1-35` | Fragment types, fragment overlays, `DTCLib::DTC_*` packets and types, `CFOLib::CFO_*` packets, subsystem decoders. Links `artdaq-core::Data`. |
| `artdaq-core-mu2e::Data` | `Data/CMakeLists.txt:1-10` | art data products (`EventHeader`, `RunHeader`, `SubRunHeader`, `TimeStamp`, `Mu2eEventHeader`). Links `::Overlays`. |
| `artdaq-core-mu2e_Data_dict` | `Data/CMakeLists.txt:14`, `Data/classes_def.xml` | ROOT dictionary for the products and their `art::Wrapper`s, built with `NO_CHECK_CLASS_VERSION`. |
| `artdaq-core-mu2e_Overlays_root_dict` | `Overlays/root/CMakeLists.txt:4`, `Overlays/root/classes_def.xml` | Optional ROOT dictionary for 30 `DTCLib` classes; built only when the environment sets `BUILD_ROOT_INTERFACE` (`Overlays/CMakeLists.txt:50-53`). |
| `artdaq-core-mu2e_BuildInfo` | `BuildInfo/CMakeLists.txt:9` | `GetPackageBuildInfo`, with the build time stamped at configure time. |
| plugin `Mu2e` `artdaq::fragmentNameHelper` | `Overlays/CMakeLists.txt:44` | `Mu2eFragmentNameHelper` (`Overlays/Mu2e_fragmentNameHelper.cc`), which maps the type numbers of §2 to names for artdaq tools. |

No executable is built for installation. The two Boost.Test programs
(`test/Overlays/CMakeLists.txt:1,6`) carry `INSTALL_BIN`, so they land in
`bin/` of an install; they are unit tests, not tools.

Namespaces: `mu2e` (fragment overlays, decoders, data products), `DTCLib`
(DTC packets and types), `CFOLib` (CFO packets). The `DTCLib`/`CFOLib` classes
live here, not in `mu2e-pcie-utils`, so that Offline can decode raw data
without the PCIe driver library.

## 2. artdaq Fragment types

`Overlays/FragmentType.hh:9-25`. artdaq-core defines `FirstUserFragmentType = 1`
and `EmptyFragmentType = 231` (`artdaq-core/Data/detail/RawFragmentHeader.hh:51,62`).

| Name | Value | Overlay that reads it | Notes |
|---|---|---|---|
| `EMPTY` | 231 | — | artdaq system type |
| `MISSED` | 1 | — | |
| (`DTC`, `MU2E`, `MU2EEVENT`) | 2, 3, 4 | — | commented out as DEPRECATED (`:13-15`); the numbers stay reserved |
| `TRK` | 5 | — | tracker; no overlay class in this package |
| `CAL` | 6 | — | calorimeter; no overlay class |
| `CRV` | 7 | — | cosmic ray veto; no overlay class |
| `DBG` | 8 | — | debug packets |
| `DTCEVT` | 9 | `mu2e::DTCEventFragment` (§5) | the event-building fragment: one `DTC_Event` |
| `STM` | 10 | `mu2e::STMFragment` (§7.1) | |
| `TRKDTC` | 11 | `mu2e::TrkDtcFragment` (§7.2) | register dump, "hardware debug info" |
| `CFO` | 12 | `mu2e::CFOEventFragment` (§6) | one CFO event record |
| `INVALID` | 13 | — | sentinel; `static_assert` that 12 is still a user type (`:28`) |

`names` (`:33-46`), `toFragmentType`, `fragmentTypeToString` and
`makeFragmentTypeMap` (`:48-55`) give the string mapping. Subsystem data
(TRK/CAL/CRV) travels inside `DTCEVT` fragments as DTC sub-events and is
decoded with the decoders of §8, not with a per-type overlay.

## 3. Conventions used in the layout tables

- **Packets are 16 bytes** (`DTC_DataPacket.h:13`, constructor `:29-30`).
  A *word* is 16 bits; word *w* is bytes 2*w* (bits 7:0) and 2*w*+1
  (bits 15:8). This is the little-endian reading every pack/unpack function
  here uses (`in.GetData()[n]` byte indexing), and it matches the spec's
  word tables.
- **Event-building headers are `uint64_t` bit-field structs** copied with
  `memcpy` from the DMA buffer (`DTC_Event.cpp:16`, `DTC_SubEvent.cpp:18`,
  `CFO_Event.cpp:14`). Bit positions given for them assume what GCC and Clang
  do on x86-64: bit-fields allocated LSB-first within each 64-bit unit,
  little-endian host. The C++ standard leaves bit-field allocation
  implementation-defined and no header asserts the struct size or byte
  order — see ACM-003.
- Offsets are as **the header defines them**. Where the spec and the header
  disagree, both are shown.

## 4. DTC DMA packets (`DTCLib`, `Overlays/DTC_Packets/`)

### 4.1 Packet types

`DTC_PacketType.h:11-19`:

| Value | Name | Class |
|---|---|---|
| 0 | `DCSRequest` | `DTC_DCSRequestPacket` |
| 1 | `Heartbeat` | `DTC_HeartbeatPacket` (a.k.a. "Readout Request") |
| 2 | `DataRequest` | `DTC_DataRequestPacket` |
| 4 | `DCSReply` | `DTC_DCSReplyPacket` |
| 5 | `DataHeader` | `DTC_DataHeaderPacket`, also `RocDataHeaderPacket_t` |
| 0x10 | `Invalid` | software sentinel |

Spec types 3 (Prefetch), 6 (Data Payload), 7 and 8 (DCS block write/read
payload) have no enum value; payload packets have no DMA header.

### 4.2 Common DMA header — `DTC_DMAPacket`

`DTC_DMAPacket.h:18-159`; unpack `DTC_DMAPacket.cpp:35-48`, pack `:11-33`.

| Word | Bits | Field | Unpacked from | Packed to |
|---|---|---|---|---|
| 0 | 15:0 | byte count (`byteCount_`) | bytes 0–1 (`:45`) | bytes 0–1 (`:15-18`) |
| 1 | 3:0 | "hop count" (spec: reserved) | byte 2 [3:0] (`:38`) | byte 2 [3:0] (`:19`) |
| 1 | 7:4 | packet type | byte 2 [7:4] (`:39`) | byte 2 [7:4] (`:20`) |
| 1 | 10:8 | link ID (`DTC_Link_ID`) | byte 3 [2:0] (`:41`) | byte 3 [2:0] (`:21`) |
| 1 | 14:11 | DTC errors (spec) | not read by the base class; `DTC_DCSReplyPacket.cpp:51` reads them | **subsystem ID written to [14:12]** (`:21`) — see §4.6 |
| 1 | 15 | valid | byte 3 bit 7 (`:42`) | byte 3 bit 7 (`:21`) |
| 2 | 15:13 | subsystem ID (Data Header only) | byte 5 [7:5] (`:43`) | not written by the base class |

`DTC_Link_ID` (`DTC_Types/DTC_Link_ID.h:10-22`): 0–5 ROC links, 6 = CFO,
7 = EVB, 8 = `Unused` (software), 255 = `ALL`.

### 4.3 Data Header packet (type 5) — two overlays for one packet

This is the first packet of every ROC data block. Two independent
definitions exist:

- `DTC_DataHeaderPacket` (`DTC_DataHeaderPacket.h:19-140`), byte-indexed
  unpack `DTC_DataHeaderPacket.cpp:12-38` and pack `:70-81`. Used by
  `DTC_DataBlock`, `DTC_SubEvent`, the decoders and `mu2esim`.
- `RocDataHeaderPacket_t` (`DTC_RocDataHeaderPacket.h:6-51`), a global-namespace
  `uint16_t`/`uint8_t` bit-field struct for direct pointer overlay. Used by
  `otsdaq-mu2e-tracker` (`Ui/DtcInterface.hh`, `ArtModules/TrackerDQM_module.*`).

| Word | Bits | Spec V53 p. 11 | `DTC_DataHeaderPacket` | `RocDataHeaderPacket_t` |
|---|---|---|---|---|
| 0 | 15:0 | byte count | `byteCount_` | `byteCount:16` |
| 1 | 3:0 / 7:4 / 10:8 / 14:11 / 15 | reserved / type / link / DTC errors / valid | per §4.2 | `unused:4`, `packetType:4`, `linkID:3`, `DtcErrors:4`, `valid:1` |
| 2 | 10:0 | packet count (data packets after the header) | `packetCount_` = byte 4 + byte 5 [2:0] (`:24`) | `packetCount:11` |
| 2 | 12:11 | reserved | — | `unused2:2` |
| 2 | 15:13 | subsystem ID | via `DTC_DMAPacket` byte 5 [7:5] | `subsystemID:3` |
| 3–5 | | event window tag, 48 bits | `DTC_EventWindowTag(arr, 6)`, bytes 6–11 (`:25`) | `eventTag[3]`; `ewtag()` (`:43-50`) |
| 6 | 7:0 | status | byte 12 (`:26`) | `status:8` |
| 6 | 15:8 | data format version | byte 13 (`:27`) | `version:8` |
| 7 | 7:0 | DTC ID | byte 14 (`:28`) | `dtcID:8` |
| 7 | 15:8 | on-spill [8], subrun [10:9], event mode [15:11] | byte 15 as **`evbMode_`** (`:29`) | `onSpill:1`, `subrun:2`, `eventMode:5` |

The two overlays name word 7 [15:8] differently: `DTC_DataHeaderPacket` calls
the whole byte "EVB mode", `RocDataHeaderPacket_t` and the spec split it into
on-spill, subrun and event-mode bits. `mu2esim` fills it from the
`CFOEmulation_EventMode1` register (`mu2e-pcie-utils/dtcInterfaceLib/mu2esim.cpp:765`).

Consistency check: the unpacking constructor throws `DTC_WrongPacketTypeException`
if the type is not 5 (`:14-21`) and `DTC_WrongPacketSizeException` unless
`(packetCount + 1) * 16 == byteCount` (`:31-37`). `IsDataHeaderPacket`
(`:88-144`) is the fast check used to resynchronise after corruption: it masks
the first two `uint64_t` words for type 5, valid, and optionally EWT, link
and subsystem (subsystem at bits 47:45 of the first word, i.e. word 2
[15:13] — agreeing with the unpacker); the DTC-ID comparison only logs
(`:119-131`).

**Status byte** (`DTC_DataStatus.h:10-18`, `RocDataHeaderPacket_t::empty()`
… `overflow()`, `:35-41`): bit 0 event window has data, 1 invalid event
window request, 2 ROC corrupt, 3 timeout, 4 overflow, 7:5 reserved. The only
named constant is `DTC_DataStatus_Valid = 0`.

**Format version** (word 6 [15:8]) is how subsystem decoders select a payload
layout: the tracker decoder upgrades V0 packets to the current layout
(§8.1), the calorimeter decoder rejects versions above 1
(`Decoders/CalorimeterDataDecoder.cc:18`).

**Subsystem ID values** — the overlay and the spec disagree on three codes
(`DTC_Types/DTC_Subsystem.h:8-18`):

| Value | Overlay `DTC_Subsystem` | Spec V53 p. 11 |
|---|---|---|
| 0 | `Tracker` | Tracker |
| 1 | `Calorimeter` | Calorimeter |
| 2 | `CRV` | CRV |
| 3 | `Other` | reserved |
| 4 | `STM` | STM |
| 5 | `ExtMon` | ExtMon |
| 6 | `MobileSync` | reserved |
| 7 | `Unused` (implicit), used as "match any" by `IsDataHeaderPacket` | **TDAQ** |

The overlay therefore cannot name, or filter for, a TDAQ (7) data block.
`mu2edaq-dataformat-viewer` follows the spec (its DFV-002). Recorded under
ACM-005.

### 4.4 Heartbeat packet (type 1) — `DTC_HeartbeatPacket`

`DTC_HeartbeatPacket.h:19-87`; unpack `DTC_HeartbeatPacket.cpp:36-42`, pack `:78-80`.

| Word | Bits | Field |
|---|---|---|
| 0–1 | | DMA header (§4.2) |
| 2–4 | | event window tag, bytes 4–9 |
| 5 | 7:0 / 15:8 | event mode byte 0 (`mode0`) / byte 1 (`mode1`) |
| 6 | 7:0 / 15:8 | event mode byte 2 / byte 3 |
| 7 | 7:0 | `mode4`: on-spill [0], subrun [1], predictive subrun [2] (`DTC_Types/DTC_EventMode.h:26-28`) |
| 7 | 15:8 | delivery-ring RF-0 marker TDC (`deliveryRingTDC`) |

Overlay and spec agree (dataformat-viewer §4.2). `DTC_EventMode`
(`DTC_EventMode.h:8-30`) holds the five mode bytes plus the TDC.

### 4.5 Data Request packet (type 2) — `DTC_DataRequestPacket`

`DTC_DataRequestPacket.h:19-108`; unpack `DTC_DataRequestPacket.cpp:24-27`, pack `:59-62`.

| Word | Bits | Field |
|---|---|---|
| 0–1 | | DMA header |
| 2–4 | | event window tag, bytes 4–9 |
| 6 | 0 | debug flag |
| 6 | 7:4 | debug type (`DTC_DebugType`, `DTC_Types/DTC_DebugType.h:9-17`: 0 special sequence, 1 external serial, 2 external serial with FIFO reset, 3 RAM test, 4 DDR test) |
| 7 | 15:0 | debug packet count (spec: 11 bits [10:0], [15:11] reserved) |

Overlay and spec agree on positions; the spec defines debug types 0–2 only.

### 4.6 Subsystem ID does not round-trip through `DTC_DMAPacket` (ACM-001)

The base-class packer writes the subsystem ID into byte 3 bits 6:4, i.e.
**word 1 bits 14:12**, inside the spec's DTC-errors field
(`DTC_DMAPacket.cpp:21`). The unpacker reads it from byte 5 bits 7:5, i.e.
**word 2 bits 15:13** (`DTC_DMAPacket.cpp:43`), as do the spec,
`RocDataHeaderPacket_t` (`DTC_RocDataHeaderPacket.h:19-21`) and
`IsDataHeaderPacket` (`DTC_DataHeaderPacket.cpp:116-117`).
`DTC_DataHeaderPacket::ConvertToDataPacket` writes byte 5 with the packet
count's top three bits only (`DTC_DataHeaderPacket.cpp:74`), so word 2
[15:13] stays 0.

A Data Header built by this code decodes as subsystem 0 (Tracker) with a
non-zero DTC-errors nibble. `mu2esim` builds its emulated calorimeter and CRV
headers this way (`mu2e-pcie-utils/dtcInterfaceLib/mu2esim.cpp:842-843`,
`:855-856`); the calorimeter decoder then skips every such block as "from
different subsystem" (`Decoders/CalorimeterDataDecoder.cc:38-41`), and
`DTC_Event::GetSubsystemBlocks` (`DTC_Event.h:105-122`) misfiles them.
Hardware-produced headers are unaffected: they are only unpacked.
`DTC_DataHeaderPacket::Equals` compares two packed images and so does not
notice. Staged as ACM-001 (S2).

### 4.7 DCS Request (type 0) and DCS Reply (type 4)

`DTC_DCSRequestPacket.h:20-170`, unpack `DTC_DCSRequestPacket.cpp:32-51`,
pack `:186-215`; `DTC_DCSReplyPacket.h:18-125`, unpack
`DTC_DCSReplyPacket.cpp:51-98`, pack `:189-211`.

| Word | Bits | Request | Reply |
|---|---|---|---|
| 1 | 14:11 | — | DTC error bits (`:51`) |
| 2 | 2:0 / 3 | op type (`DTC_DCSOperationType`) / request ack | op type [3:0] incl. double-op bit 2 / ack bit 3 |
| 2 | 4 | increment address | DCS receive FIFO empty |
| 2 | 5 | — | ROC corrupt |
| 2 | 15:6 | block packet count (10 bits) | block packet count |
| 3 / 4 | | address 1 / data 1 | address 1 / data 1 |
| 5 / 6 | | address 2 / data 2 (double op), or block-write words 0–2 | address 2 / data 2, or block-read words |
| 7 | | block-write word or 0 | block-read word |

`DTC_DCSOperationType` (`DTC_Types/DTC_DCSOperationType.h:13-24`): 0 read,
1 write, 2 block read, 3 block write, 4 double read, 5 double write;
0xC invalid-S2C, 0xE timeout, 0xF unknown (software use).

## 5. Event-building containers — `DTCEVT` fragments

A `DTCEVT` fragment's payload is one **DTC event**: an event header followed by
sub-events, one per DTC; each sub-event is a sub-event header followed by ROC
data blocks, each a Data Header packet (§4.3) plus `packetCount` 16-byte data
packets.

```
DTC_Event      = DTC_EventHeader(24 B)    + DTC_SubEvent × num_dtcs
DTC_SubEvent   = DTC_SubEventHeader(48 B) + DTC_DataBlock × num_rocs
DTC_DataBlock  = DataHeader(16 B)         + 16 B × packetCount
```

### 5.1 `DTC_EventHeader` — 24 bytes (`DTC_EventHeader.h:11-65`)

| Word (64-bit) | Bits | Field |
|---|---|---|
| 0 | 23:0 | `inclusive_event_byte_count` (24 bits, max 16 MiB) |
| 0 | 31:24 | reserved |
| 0 | 63:32 | `event_tag_low` |
| 1 | 15:0 | `event_tag_high` (EWT is 48 bits) |
| 1 | 23:16 | `num_dtcs` |
| 1 | 63:24 | `event_mode` (40 bits = event mode bytes 0–4) |
| 2 | 7:0 / 15:8 / 23:16 / 31:24 | `dtc_mac` / `partition_id` / `evb_mode` / `evb_id` |
| 2 | 39:32 / 47:40 | `evb_status` (`DTC_Types/DTC_EVBStatus.h`: bit 7 error, bit 0 fragment timeout) / `emtdc` |
| 2 | 63:48 | reserved |

No version field.

### 5.2 `DTC_SubEventHeader` — 48 bytes (`DTC_SubEventHeader.h:14-138`)

| Word (64-bit) | Bits | Field |
|---|---|---|
| 0 | 24:0 | `inclusive_subevent_byte_count` (25 bits, max 32 MiB — wider than the event's 24-bit count) |
| 0 | 31:25 / 63:32 | reserved / `event_tag_low` |
| 1 | 15:0 / 23:16 / 63:24 | `event_tag_high` / `num_rocs` / `event_mode` (40 bits) |
| 2 | 7:0 / 15:8 / 23:16 / 31:24 | `dtc_mac` / `partition_id` / `evb_mode` / `source_dtc_id` |
| 2 | 49:32 | `link0_subsystem` … `link5_subsystem`, 3 bits each |
| 2 | 63:50 | reserved |
| 3 | 47:0 | `link0_status` … `link5_status`, 8 bits each |
| 3 | 55:48 | **`subevent_format_version`** |
| 3 | 63:56 | `emtdc` |
| 4 | 15:0 / 31:16 / 63:32 | `link4_drp_rx_latency` / `link5_drp_rx_latency` / reserved |
| 5 | 63:0 | `link0` … `link3_drp_rx_latency`, 16 bits each |

**Version handling.** `CURRENT_SUBEVENT_FORMAT_VERSION` is a macro, 1
(`DTC_SubEventHeader.h:10`), used as the default for generated headers.
`DTC_SubEvent::REQUIRED_SUBEVENT_FORMAT_VERSION` is 1 (`DTC_SubEvent.h:21`,
defined `DTC_SubEvent.cpp:7`); the constructor and `SetupSubEvent` throw
`DTC_WrongPacketTypeException` on any other value (`DTC_SubEvent.cpp:21-30`,
`:85-89`), with the message "Check that your DTC FPGA version matches the
software expectation". There is exactly one accepted version and no
upgrade path.

### 5.3 Parsing and its failure modes

`DTCEventFragment::getData()` (`DTCEventFragment.hh:59-67`) constructs
`DTC_Event(dataBeginBytes())` and calls `SetupEvent()`. `SetupEvent`
(`DTC_Event.cpp:26-72`) walks sub-events until the header's
`inclusive_event_byte_count`; `SetupSubEvent` (`DTC_SubEvent.cpp:78-…`) walks
data blocks until `inclusive_subevent_byte_count`, checking each block's
link index and EWT against the sub-event and resynchronising with
`IsDataHeaderPacket` when the next header is not where the byte count says.
`IsCorrupt()` reports any of this; `DTCEventFragment::IsCorrupt()` separately
reads a 1-byte fragment metadata flag (`DTCEventFragment.hh:28-57`,
`CURRENT_VERSION = 1`).

Two header-visible properties matter to every consumer:

- **No buffer bound.** Neither constructor takes a size and `getData()` does
  not pass `dataSizeBytes()`; the walk is bounded only by the byte counts in
  the data. `DTC_DataBlock` guards its accessors with `assert`
  (`DTC_DataBlock.h:55,63,68`), compiled out in release builds. ACM-002.
- **Retry without advance.** When the `DTC_SubEvent` constructor throws
  `DTC_WrongPacketTypeException` (format version ≠ 1), `SetupEvent` catches it
  and `continue`s with the same pointer and byte count
  (`DTC_Event.cpp:52-57`); when a `DTC_DataBlock` constructor throws inside
  `SetupSubEvent` (type ≠ 5, or `(packetCount+1)*16 ≠ byteCount`), both catch
  blocks fall through to the same loop condition (`DTC_SubEvent.cpp:107,113,201,240`).
  Either loops forever at `cf86c07`. **Fixed upstream** by 5225ef0
  (2026-09-21, `SetupEvent` now `break`s) and 8bf66e6 (2026-09-27, both
  `SetupSubEvent` handlers now `break`); the upstream comment records a
  2026-09-22 bench run of 254k identical passes and 1.5 GB of log. The
  superproject still pins the unfixed commit — ACM-006.

`DTC_Event::MAX_DMA_SIZE` is 0x8000 (`DTC_Event.h:32`).

### 5.4 Selecting a subsystem

`DTC_Event::GetSubsystemData(subsys)` returns sub-events whose
`linkN_subsystem` matches on any link (`DTC_Event.h:92-103`);
`GetSubsystemBlocks(subsys)` additionally filters blocks by the Data Header
subsystem ID (`:105-122`) — the field affected by §4.6. `GetSubEventCount(subsys)`
and `GetBlockCount(subsys)` look at link 0's subsystem only (`:47-68`).

## 6. CFO event record — `CFO` fragments

`CFOEventFragment::getData()` (`CFOEventFragment.hh:40-48`) returns a
`CFOLib::CFO_Event`, whose constructor copies `sizeof(CFO_EventRecord)` bytes
from `dataBeginBytes()` (`CFO_Packets/CFO_Event.cpp:14`). The record is a
56-byte `uint64_t` bit-field struct (`CFO_Packets/CFO_EventRecord.h:10-81`),
following Mu2e-doc-19095 p. 22 per `Data/EventHeader.hh:8`:

| Word (64-bit) | Bits | Field |
|---|---|---|
| 0 | 47:0 / 55:48 / 63:56 | `event_tag` / reserved / **`record_format_version`** |
| 1 | 31:0 / 63:32 | `linux_timestamp` (s) / reserved |
| 2 | 47:0 / 63:48 | `event_mode` (48 bits) / `event_duration` |
| 3 | 7:0 | `DR_marker_N_est` |
| 4 | 7:0 | `DR_marker_Nplus1_est` |
| 5 | 7:0 / 31:8 / 63:32 | `DR_marker_N_meas` / reserved / `TDC_marker_N_from_spill` |
| 6 | 7:0 | `DR_marker_Nplus1_meas` |

`record_format_version` is carried but nothing in the headers checks it, and
the 56-byte copy does not consult the fragment size (ACM-002). Event mode is
48 bits here against 40 bits in the DTC headers.

`CFO_DMAPacket`/`CFO_DataPacket` (`CFO_Packets/`) are reduced copies of the DTC
classes for the CFO DMA channel; `CFO_PacketType` defines only `DataHeader = 5`
and `Invalid = 0x10`.

## 7. Other fragment overlays

### 7.1 `STMFragment` — `STM` (`Overlays/STMFragment.hh:28-75`)

**At `cf86c07`:** an `int16_t` view of the payload with fixed offsets and an
"STM-TODO" note that the layout is provisional (`:10-25`): `EvNum` at int16
index 8, `DataType` 22 (comment: assumed raw), `EvLen` 23, `detID` 28, data
from 32. No version, no length check.

**On upstream `develop` (rewritten, 332 lines):** the dataset is carried in
the Fragment ID (`stm::Dataset`: 100/101/102/103 = HPGe raw / zero-suppressed /
pulse-height / container, 200–203 the same for LaBr). Raw data starts with a
22-word `int16` header (`stm::RawHeader`): anchor `0xCAFE` at word 0 and 21,
EWT in words 1–3, ADC clock 4–7, channel + DTC clock 8–11, event mode
12–14 (word 14 bit 0 = on-spill), prescale 15, raw length 16, ZS region count
17, ZS length 18, pulse-height count 19, data flags 20. Each zero-suppressed
pulse has a 2-word header (raw index, pulse length). Payload size comes from
`dataSizeBytes()`. This review documents the upstream layout from the header
only and did not compare it with the STM firmware.

### 7.2 `TrkDtcFragment` — `TRKDTC` (`Overlays/TrkDtcFragment.hh:8-60`)

A list of `{uint32_t address; uint32_t value;}` register pairs; `nReg()` is
`dataSizeBytes()/8` and `getRegisterEntry` bounds-checks the index. Metadata
is one `int32_t version`, `CURRENT_VERSION = 1`; `size_words = 4` is
commented "in bytes".

## 8. Subsystem payload decoders (`Overlays/Decoders/`)

All derive from `mu2e::DTCDataDecoder` (`DTCDataDecoder.hh:23-87`), which
holds a **reference** to a `DTC_SubEvent` and exposes `block_count()`,
`blockSizeBytes(i)`, `dataAtBlockIndex(i)`; the sub-event must outlive the
decoder. Each decoder has a `std::vector<…>` typedef (`TrackerDataDecoders`
etc.) for use as an art product; the dictionaries live outside this package.

### 8.1 Tracker — `TrackerDataDecoder.hh`

`TRACKER_FORMAT_VERSION = 1` (`:15`). Version 0 packets (`TrackerDataPacketV0`,
32 bytes, `:19-135`: straw index, TDC0, TDC1, TOT0/TOT1, fifteen 12-bit ADC
samples, preprocessing flags) are upgraded in software to the current
layout (`Upgrade`, `:285`).

`TrackerDataPacket`, 16 bytes (`:137-195`):

| Word | Bits | Field |
|---|---|---|
| 0 | 15:0 | `StrawIndex` |
| 1 / 2 [7:0] | | `TDC0` (24 bits: `TDC0A` + `TDC0B<<16`) |
| 2 | 11:8 / 15:12 | `TOT0` / `EWMCounter` |
| 3 / 4 [7:0] | | `TDC1` (24 bits) |
| 4 | 11:8 / 15:12 | `TOT1` / `ErrorFlags` |
| 5 | 5:0 / 15:6 | `NumADCPackets` / `PMP` |
| 6–7 | | ADC samples 0–2, 10 bits each (`ADC01` split 6+4) |

Followed by `NumADCPackets` × `TrackerADCPacket` (16 bytes, twelve 10-bit
samples, `:197-278`). These are overlaid directly on the data (bit-field
structs); see ACM-003.

### 8.2 Calorimeter — `CalorimeterDataDecoder.hh`

The calorimeter payload uses **swapped 16-bit word pairs and MSB-first bit
numbering** (32-bit big-endian words), unlike every other layout here.
`CalorimeterHitDataPacket` is *not* an overlay: `mapFromRaw` extracts by bit
offset with `extractBits` (`:84-112`):

| Bits (MSB-first, after pair swap) | Field |
|---|---|
| 0–11 | reserved |
| 12–19 | `BoardID` |
| 20–22 | `DetectorID` |
| 23–27 | `ChannelID` |
| 28–43 | `Time` |
| 44–59 | `InPayloadEventWindowTag` |
| 60–71 | `Baseline` |
| 72–81 | `IndexOfMaxDigitizerSample` |
| 82–85 | `ErrorFlags` |
| 86–95 | `NumberOfSamples` |

Waveform samples are 12-bit, unpacked by `Data12bitReader` (`:15-66`), with
21 samples per two packets in debug mode. `CalorimeterHitTestDataPacket`
(`:115-130`) is the test-pattern format with markers 0xAAA (begin) and 0xFFF
(last sample). `CalorimeterFooterPacket` (16 bytes, `:133-148`): detector
type [2:0] (0 CALO, 1 CAPHRI, 2 TRAD, 3 LASER), board ID [10:3], channel
status flags, then six DIRAC words. Accepted Data Header versions: 0 and 1.
The `Get*` accessors return heap-allocated `std::vector*` that the caller
owns (`:157-163`; `CalorimeterDataDecoder.cc:28`).

### 8.3 CRV — `CRVDataDecoder.hh`

FEB-I structures (`:18-116`) are marked "soon to be obsolete" (open issue
Mu2e/artdaq-core-mu2e#120). FEB-II (`:121-306`):

| Struct | Size | Layout |
|---|---|---|
| `CRVROCStatusPacketFEBII` | 16 B | event word count, trigger count, active-FEB flags (24 bits over 3 bytes), micro-bunch status (32), EWT (32) |
| `CRVHitInfoFEBII` | 4 B | `fpgaChannel:5`, `fpgaNumber:2`, `portNumber:5`, reserved:4; `hitTime:16` |
| `CRVHitADCBlockFEBII` | 6 B | four 12-bit samples |
| `CRVHitRawFEBII` | 22 B | info + 3 ADC blocks = 12 samples; `static_assert` on size (`:274`) |
| `CRVGlobalRunInfo` | 16 B | EWT count, marker count, last EWT, lock/PLL/CRC, injection window and time |

`GetCRVHitRangeFEBII` returns a zero-copy range of `CRVHitRawFEBII` cast
directly onto the payload (`:281-306`); `error()` signals a payload that is
not a multiple of 22 bytes.

### 8.4 MobileSync — `MobileSyncDataDecoder.hh`

16-byte packets (`static_assert`, `:27`). A data-ID tag in the header maps to
a payload version and a sync-firmware git tag; payload versions 0 and 1 have
hit formats `kV0`/`kV1` (`:61-62`); any other version yields `nullptr`
(`:65-76`).

### 8.5 CFO — `CFODataDecoder.hh`

A thin holder of a `CFO_Event` reference (`:22-29`).

## 9. art data products (`Data/`)

Persisted in art files; dictionary in `Data/classes_def.xml`.

| Class | Header | Contents |
|---|---|---|
| `mu2e::EventHeader` (+ `EventHeaders`) | `EventHeader.hh:29-95` | `ewt`, `mode` (32 bits = event-mode bytes 0–3), `rfmTDC_est`, `flags` (on-spill = bit 0), `eventDuration`, `rfmTDC_measured`, four 1-bit check flags (`rnr_check`, `ndtc_check`, `dtc_check`, `ewt_check`) |
| `mu2e::RunHeader` | `RunHeader.hh` | `startTime` (`TimeStamp`), `ewtFirst` |
| `mu2e::SubRunHeader` | `SubRunHeader.hh` | same two fields |
| `mu2e::TimeStamp` | `TimeStamp.hh` | `uint32_t` seconds since the Unix epoch |
| `mu2e::Mu2eEventHeader` | `Mu2eEventHeader.hh` | legacy: `uint64_t EventWindowTag`, `uint8_t EventMode` |
| `mu2e::EWT` | `EWT.hh` | `typedef uint64_t`; 48 significant bits |

The heartbeat fields come from §4.4, the CFO fields from §6. Open issue
Mu2e/artdaq-core-mu2e#101 asks for an `EventWindowMarker` translation for
Offline.

## 10. Header-level review summary

| Concern | Where | Assessment |
|---|---|---|
| Unchecked length fields | `DTC_Event`/`DTC_SubEvent` walk (§5.3), `CFO_Event` 56-byte copy, `DTC_DataBlock` asserts; `STMFragment` at the pin only | ACM-002 |
| Parse loop does not advance after an exception | `DTC_Event.cpp:52-57`, `DTC_SubEvent.cpp:201,240` at `cf86c07` | fixed upstream (5225ef0, 8bf66e6); pin lacks it — ACM-006 |
| Pack/unpack asymmetry | subsystem ID, §4.6 | ACM-001 |
| Type punning | bit-field structs overlaid with `reinterpret_cast` (`RocDataHeaderPacket_t`, tracker packets, `CRVHitRawFEBII`); `reinterpret_cast<const uint64_t*>` in `IsDataHeaderPacket` (`DTC_DataHeaderPacket.cpp:96`); event headers use `memcpy` (defined) | ACM-003 |
| Endianness | every pack/unpack and overlay assumes a little-endian host; calorimeter payload is the one big-endian-word format and is handled explicitly | ACM-003 |
| Bit-field allocation | implementation-defined; size `static_assert`s exist only for `CRVHitRawFEBII`, `MobileSyncPacket`, `DTCEventFragment::Metadata`, `TrkDtcFragment::Metadata` | ACM-003 |
| Version handling | sub-event format: exactly 1, else throw; data-header version: per-decoder; CFO record version: unchecked; `DTCEVT`/`CFO` fragment `CURRENT_VERSION = 1` not compared on read; STM: none at the pin, dataset in Fragment ID upstream | §5.2, §6, ACM-002 |
| Include guards | `DTCEventFragment.hh` and `CFOEventFragment.hh` share one guard | ACM-004 |

## 11. Independent encodings of these layouts

`mu2edaq-dataformat-viewer/formats/*.yaml` re-encodes the ROC packets for a
GUI decoder. Its review compared the YAML, the spec and these headers: the
spec and the overlays agree on Data Header, Heartbeat and Data Request; the
YAML disagrees on Heartbeat, Data Request and Debug Header layouts (its
DFV-001) and on subsystem-ID labels (DFV-002). Nothing in either repository
checks one encoding against the other. A golden-bytes test here (ACM-003)
would give the YAML something to be checked against.
