# Staged issues — artdaq-core-mu2e

Six findings from the September 2026 DAQ mega-review
(`mu2e-sept-mega-review`, 2026-10-06).

**Baseline.** Reviewed at `cf86c07` (9.02.00), the commit the superproject
pins. That commit is on the fork `normanajn/artdaq-core-mu2e`; it is 68
commits behind and 2 ahead of `Mu2e/artdaq-core-mu2e` `develop`
(`7ae4622`, 9.07.00, 2026-10-06). Because the issues go to the upstream
repository, every finding was re-checked against upstream `develop`: each
cited line was located again there, and every header was diffed with
whitespace ignored. Line numbers are for `cf86c07`; where upstream differs,
its line is given as `develop:N`. One defect found at `cf86c07` is already
fixed upstream and is **not** staged as an issue: the `SetupEvent` /
`SetupSubEvent` loops that never terminate after a parse exception (fixed by
5225ef0, 2026-09-21, and 8bf66e6, 2026-09-27). The superproject pin still
lacks that fix; it is evidence in ACM-006.

**Routing.** The superproject submodule URL is `normanajn/artdaq-core-mu2e`, a
fork with issues disabled. ACM-001 to ACM-005 go to **`Mu2e/artdaq-core-mu2e`**
(public, issues enabled). ACM-006 is about the superproject and goes to
**`Mu2e/mu2edaq-main`**.

**Method.** Tier B: `CMakeLists.txt`, all public headers under `Data/` and
`Overlays/`, registration macros, `test/` and `.github/` file names. `.cpp`
bodies were read only for the functions a header names and the review needed
(pack/unpack constructors, `SetupEvent`, `SetupSubEvent`); every such read
is cited by line. GitHub writes are denied in this session, so nothing has
been filed.

**Dedup.** `gh issue list -R Mu2e/artdaq-core-mu2e --state all --limit 300`
returned 10 issues (#6, #8, #9, #20, #23, #34, #59, #65, #101, #120); none
carries a mega-review fingerprint and none overlaps a finding below by two or
more significant title tokens. Open #120 (remove FEB-I CRV structures) and
open #101 (EventWindowMarker from EventHeader) are not duplicated. Closed #65
("Handle cases where SetupSubEvent fails") concerned the parse loops that
upstream has since fixed.

**Fingerprints** are `sha1("<repo>:<path>:<rule_id>")[:8]`; inputs are
tabulated at the end.

6 findings: 1 × S2, 4 × S3, 1 × S4.

---

## ACM-001

**Title:** `[overlays] DTC_DMAPacket packs the subsystem ID into the DTC-errors field, so Data Header packets do not round-trip`

**Severity:** S2 — production breakage or silent wrong behaviour; workaround exists
**Category:** bug
**Location:** `artdaq-core-mu2e/Overlays/DTC_Packets/DTC_DMAPacket.cpp:21`
**Found by:** mega-review (`mu2e-sept-mega-review`), 2026-10-06

## What
`DTC_DMAPacket::ConvertToDataPacket` writes the subsystem ID into byte 3
bits 6:4 (word 1 bits 14:12). The unpacking constructor reads it from byte 5
bits 7:5 (word 2 bits 15:13). `DTC_DataHeaderPacket::ConvertToDataPacket`
then writes byte 5 with only the packet count's top three bits. A Data Header
built by this library therefore decodes as subsystem 0 (Tracker), and its
DTC-errors nibble is non-zero for every other subsystem.

## Evidence
```
$ sed -n 21p artdaq-core-mu2e/Overlays/DTC_Packets/DTC_DMAPacket.cpp       # pack
	uint8_t word1B = static_cast<uint8_t>(linkID_ & 0x7) + (valid_ ? 0x80 : 0x0) + ((subsystemID_ & 0x7) << 4);
$ sed -n 43p artdaq-core-mu2e/Overlays/DTC_Packets/DTC_DMAPacket.cpp       # unpack
	subsystemID_ = (in.GetData()[5] >> 5) & 0x7;
$ sed -n 74p artdaq-core-mu2e/Overlays/DTC_Packets/DTC_DataHeaderPacket.cpp
	output.SetByte(5, static_cast<uint8_t>((packetCount_ & 0x0700) >> 8));
```
Every other definition agrees with the unpacker, not the packer:
- `DTC_RocDataHeaderPacket.h:13-21`: word 1 `unused:4, packetType:4, linkID:3, DtcErrors:4, valid:1`; word 2 `packetCount:11, unused2:2, subsystemID:3`.
- `DTC_DataHeaderPacket.cpp:116-117` (`IsDataHeaderPacket`): subsystem mask `0x0000E00000000000`, i.e. word 2 [15:13].
- `DTC_DCSReplyPacket.cpp:51`: DTC errors are byte 3 [6:3], word 1 [14:11].
- Mu2e-doc-4914 V53 p. 11 (as tabulated in `mu2edaq-dataformat-viewer/doc/INTERFACES.md` §4.1): word 1 DTC errors [14:11], word 2 subsystem ID [15:13].

Callers that build Data Headers with a non-Tracker subsystem:
```
mu2e-pcie-utils/dtcInterfaceLib/mu2esim.cpp:842  DTC_DataHeaderPacket header(..., DTCLib::DTC_Subsystem_Calorimeter, ...);
mu2e-pcie-utils/dtcInterfaceLib/mu2esim.cpp:843  memcpy(&buffer[0], header.ConvertToDataPacket().GetData(), 16);
mu2e-pcie-utils/dtcInterfaceLib/mu2esim.cpp:855  DTC_DataHeaderPacket header(..., DTCLib::DTC_Subsystem_CRV, ...);
mu2e-pcie-utils/dtcInterfaceLib/mu2esim.cpp:856  memcpy(&buffer[0], header.ConvertToDataPacket().GetData(), 16);
```
and a consumer that filters on the field:
```
artdaq-core-mu2e/Overlays/Decoders/CalorimeterDataDecoder.cc:38  if (dataBlock->GetHeader()->GetSubsystem() != DTCLib::DTC_Subsystem_Calorimeter)
artdaq-core-mu2e/Overlays/Decoders/CalorimeterDataDecoder.cc:40      ... "this block is from different subsystem" ... skipping
```

## Why it matters
Emulated calorimeter and CRV data from the DTC simulator arrive labelled as
Tracker with DTC error bits 12 (calorimeter) or 13 (CRV) set. The calorimeter
decoder skips every block, `DTC_Event::GetSubsystemBlocks` (`DTC_Event.h:105-122`)
files them under the wrong subsystem, and a DQM or verifier run on simulated
data reports no calorimeter/CRV hits plus spurious DTC errors, with no
exception. Simulation-mode integration tests of the calorimeter and CRV
chains therefore cannot pass for the right reason. Hardware-produced headers
are only unpacked and are decoded correctly, which is why this is S2 and not
S1. `DTC_DataHeaderPacket::Equals` compares two packed images and so never
detects it.

## Suggested fix
In `DTC_DMAPacket::ConvertToDataPacket` drop `((subsystemID_ & 0x7) << 4)`
from byte 3; in `DTC_DataHeaderPacket::ConvertToDataPacket` write byte 5 as
`((packetCount_ >> 8) & 0x7) | ((subsystemID_ & 0x7) << 5)`. Add a unit test
that packs and unpacks a Data Header for every `DTC_Subsystem` value and
compares all fields, including that bits 14:11 of word 1 are zero.

## Related
- Mu2e/mu2edaq-dataformat-viewer DFV-001 (pending): YAML, spec and overlay comparison that surfaced this
- `doc/INTERFACES.md` §4.6

<!-- mega-review-fingerprint: ccbd3de4 -->

---

## ACM-002

**Title:** `[overlays] fragment overlays parse event payloads with no buffer bound; header byte counts are trusted`

**Severity:** S3 — correctness or maintainability risk, no current outage
**Category:** bug
**Location:** `artdaq-core-mu2e/Overlays/DTCEventFragment.hh:63` (develop:64)
**Found by:** mega-review (`mu2e-sept-mega-review`), 2026-10-06

## What
Every overlay hands the parser a bare pointer; the only limit on how far it
reads is a length field inside the data. Nothing compares that field with
`artdaq::Fragment::dataSizeBytes()`.

## Evidence
```
cf86c07 line (develop line)
DTCEventFragment.hh:63 (64)  event_ptr_.reset(new DTCLib::DTC_Event(artdaq_Fragment_.dataBeginBytes()));
DTC_Event.h:25               explicit DTC_Event(const void* data);           // no size parameter
DTC_Event.cpp:34 (34)        while (byte_count < header_.inclusive_event_byte_count)   // 24-bit, from the data
DTC_SubEvent.cpp:107 (111)   while (byte_count < header_.inclusive_subevent_byte_count) // 25-bit, from the data
DTC_DataBlock.h:37 (38)      byteSize = hdr.GetByteCount();                  // 16-bit, from the data
DTC_DataBlock.h:68 (73)      assert(byteSize > 16);                          // compiled out under NDEBUG
CFOEventFragment.hh:44 (45)  new CFOLib::CFO_Event(artdaq_Fragment_.dataBeginBytes())
CFO_Event.cpp:14 (14)        memcpy(&record_, data, sizeof(record_));         // 56 bytes regardless of payload size
```
Version fields are not consulted on read either: `CFO_EventRecord::record_format_version`
(`CFO_EventRecord.h:22`, develop:23) and `DTCEventFragment::CURRENT_VERSION` /
`CFOEventFragment::CURRENT_VERSION` (`DTCEventFragment.hh:26`, develop:27;
`CFOEventFragment.hh:25`, develop:26) are defined but never compared.
Upstream's rewritten `STMFragment` (develop) does size its payload from
`dataSizeBytes()`; the DTC and CFO overlays do not.

## Why it matters
A truncated fragment, a fragment of the wrong type handed to the wrong
overlay, or a corrupted byte count makes the parser read past the end of the
fragment's heap buffer. The likely outcomes are a segfault in an art module,
or plausible-looking garbage decoded from the next allocation. This happens in
the DQM and data-logging processes that see the most varied data. No outage
is attributed to it today, so S3.

## Suggested fix
Add `size_t` size parameters to `DTC_Event(const void*, size_t)` and
`DTC_SubEvent(const void*, size_t)`, pass `dataSizeBytes()` from the
overlays, and stop with `corruption_detected_` when a header count exceeds
the remaining bytes. Make `CFO_Event` check `dataSizeBytes() >= 56` and the
record version. Replace the `assert`s in `DTC_DataBlock` with checks that
survive release builds.

## Related
- Upstream 5225ef0 / 8bf66e6 stopped the parse loops from spinning after an
  exception; they do not bound the reads.

<!-- mega-review-fingerprint: d5befae7 -->

---

## ACM-003

**Title:** `[overlays] wire-layout bit-field structs rely on compiler bit allocation and a little-endian host, with no size/byte-order asserts or golden-byte tests`

**Severity:** S3 — correctness or maintainability risk, no current outage
**Category:** enhancement
**Location:** `artdaq-core-mu2e/Overlays/DTC_Packets/DTC_SubEventHeader.h:14` (develop:15)
**Found by:** mega-review (`mu2e-sept-mega-review`), 2026-10-06

## What
The event-building headers and several payload formats are C++ bit-field
structs that are `memcpy`d or `reinterpret_cast` onto DMA bytes. Bit-field
allocation order and packing are implementation-defined ([class.bit]/1); the
layouts are correct only for GCC/Clang on a little-endian host. Most of these
structs have no `static_assert` on their size, and none on byte order. The
unit tests parse three captured binaries; no test packs and unpacks any
packet or checks a struct against known bytes.

## Evidence
```
$ grep -rn static_assert artdaq-core-mu2e/Overlays      # cf86c07 (develop)
DTCEventFragment.hh:35 (36)             sizeof(Metadata) == Metadata::size_bytes
TrkDtcFragment.hh:36 (37)               sizeof(Metadata) == Metadata::size_words
Decoders/CRVDataDecoder.hh:274 (282)    sizeof(CRVHitRawFEBII) == hitSize
Decoders/MobileSyncDataDecoder.hh:27 (28)  sizeof(MobileSyncPacket) == kPacketBytes
FragmentType.hh:28 (30)                 isUserFragmentType(INVALID - 1)
```
The same five, and no others, are present on upstream `develop`.
No assert for: `DTC_EventHeader` (24 B, `DTC_EventHeader.h:11`),
`DTC_SubEventHeader` (48 B, `DTC_SubEventHeader.h:14`), `CFO_EventRecord`
(56 B, `CFO_EventRecord.h:10`), `RocDataHeaderPacket_t` (16 B, mixed
`uint16_t`/`uint8_t` bit-fields, `DTC_RocDataHeaderPacket.h:6`),
`TrackerDataPacket` / `TrackerADCPacket` (16 B each,
`TrackerDataDecoder.hh:137`, `:197`), `CRVROCStatusPacketFEBII` (16 B).
Direct punning: `reinterpret_cast<const uint64_t*>(ptr)` in
`DTC_DataHeaderPacket.cpp:96`; `*((uint32_t *)(&(ptr[i])))` throughout
`DTC_SubEvent.cpp` logging. Tests: `test/Overlays/DTC_Event_t.cc` and
`DTC_SubEvent_t.cc`, 8 `BOOST_AUTO_TEST_CASE`s, all parsing `*.bin` files.

## Why it matters
A compiler, flag or platform change alters these layouts silently. MSVC, for
example, does not pack bit-fields of different declared types into one
unit, which changes `RocDataHeaderPacket_t`. ACM-001 shows that pack/unpack
asymmetries survive today's tests unnoticed. `mu2edaq-dataformat-viewer`
re-encodes the same layouts in YAML and has drifted (its DFV-001/DFV-002);
there is no reference here to check it against. No outage today, so S3.

## Suggested fix
Add `static_assert(sizeof(T) == N)` for each struct above and one
`static_assert` that the host is little-endian (C++20 `std::endian`, or
`__BYTE_ORDER__` under C++17) in a common header. Add a test file with
golden 16-byte images (from the spec or captured hardware) for each packet
type and header struct, checking every field via both the pack and unpack
paths. Export the same images as a fixture that `mu2edaq-dataformat-viewer`
can test its YAML against.

## Related
- ACM-001
- Mu2e/mu2edaq-dataformat-viewer DFV-001, DFV-002 (pending)

<!-- mega-review-fingerprint: 11221bcb -->

---

## ACM-004

**Title:** `[overlays] DTCEventFragment.hh and CFOEventFragment.hh share one include guard`

**Severity:** S3 — correctness or maintainability risk, no current outage
**Category:** bug
**Location:** `artdaq-core-mu2e/Overlays/CFOEventFragment.hh:1`
**Found by:** mega-review (`mu2e-sept-mega-review`), 2026-10-06

## What
Both headers open with `#ifndef artdaq_core_Data_Mu2eEventFragment_hh`. In
any translation unit that includes both, the second one is empty.

## Evidence
```
$ grep -rn 'artdaq_core_Data_Mu2eEventFragment_hh' artdaq-core-mu2e/Overlays
Overlays/CFOEventFragment.hh:1:#ifndef artdaq_core_Data_Mu2eEventFragment_hh
Overlays/CFOEventFragment.hh:2:#define artdaq_core_Data_Mu2eEventFragment_hh
Overlays/DTCEventFragment.hh:1:#ifndef artdaq_core_Data_Mu2eEventFragment_hh
Overlays/DTCEventFragment.hh:2:#define artdaq_core_Data_Mu2eEventFragment_hh
```
No file in the superproject includes both today.

## Why it matters
A module that consumes both `DTCEVT` and `CFO` fragments, such as an event
verifier that compares the CFO record's EWT and mode with the DTC event's, fails
to compile with "`mu2e::CFOEventFragment` has not been declared" (or the DTC
equivalent, depending on include order). The cause is not obvious from the
message.

## Suggested fix
Rename the guards to `artdaq_core_mu2e_Overlays_DTCEventFragment_hh` and
`artdaq_core_mu2e_Overlays_CFOEventFragment_hh`, or use `#pragma once`.

## Related
- (none)

<!-- mega-review-fingerprint: 90bc5125 -->

---

## ACM-005

**Title:** `[headers] header hygiene: broken deprecated umbrella, non-self-contained headers, off-by-one guards, spec-divergent subsystem codes`

**Severity:** S4 — cosmetic, stale, hygiene
**Category:** bug
**Location:** `artdaq-core-mu2e/Overlays/CFO_Packets.h:6`
**Found by:** mega-review (`mu2e-sept-mega-review`), 2026-10-06

## What
A rollup of small header defects found while documenting the interface. Each
is independent and individually minor. All ten are present on upstream
`develop` (7ae4622) as well as at `cf86c07`.

## Evidence
1. `Overlays/CFO_Packets.h:6` includes `CFO_Packets/CFO_EventHeader.h`, which
   was deleted in e3cd292; including the umbrella is a hard error. Nothing
   includes it (`mu2e-pcie-utils/cfoInterfaceLib/CFO.h:8` has it commented out).
2. `DTC_Packets/DTC_RocDataHeaderPacket.h` is not self-contained: no
   `#include <cstdint>`, uses the glibc-only `ulong` (`:43-48`), declares
   `RocDataHeaderPacket_t` in the global namespace, and its comment says
   "8 16-byte words" for a 16-byte packet (`:9`).
3. `Decoders/CalorimeterDataDecoder.hh:27-28` (develop:29-30) uses `uint`
   (glibc typedef). `DTC_Types/DTC_Link_ID.h:33,44` (develop:45,…) uses `std::runtime_error`/`std::to_string`
   without `<stdexcept>`/`<string>`.
4. `Decoders/DTCDataDecoder.hh:39,50` (develop:40,…): `if (blockIndex > block_count())`
   should be `>=`; at `blockIndex == block_count()` the intended `0`/`nullptr`
   return is skipped and `DTC_SubEvent::GetDataBlock` throws `std::out_of_range`.
5. `CFO_Packets/CFO_DataPacket.h:158-160` (develop:161-163) declares stray public bit-fields
   `event_tag_low`, `event_tag_high`, `event_mode` in a pointer-wrapper class.
6. `TrkDtcFragment.hh:20` (develop:21): `size_words = 4` is commented "in bytes".
7. "Getters" that write through `const_cast`: `DTC_EventMode::GetEventMode(const uint8_t*)`
   (`DTC_Types/DTC_EventMode.h:17-24`, develop:18-…), `DTC_EventWindowTag::GetEventWindowTag(const uint8_t*, int) const`.
8. `CalorimeterDataDecoder::Get*` return owning raw `std::vector<...>*`
   (`CalorimeterDataDecoder.hh:157-163`, develop:180-…; allocated with `new` at
   `CalorimeterDataDecoder.cc:28`). Each caller must `delete` them per event.
9. `DTC_Types/DTC_Subsystem.h:13-17` (develop:14-18) defines 3 = `Other`, 6 = `MobileSync`,
   7 = `Unused` (used as "match any" by `IsDataHeaderPacket`); Mu2e-doc-4914
   V53 p. 11 defines 7 = TDAQ and reserves 3 and 6. The overlay cannot name or
   filter a TDAQ block. Either the spec or the enum needs updating.
10. `DTC_SubEventHeader.h:10`: `CURRENT_SUBEVENT_FORMAT_VERSION` is a
    preprocessor macro in a widely included header.

## Why it matters
None causes a failure in the current build. Together they make the headers
harder to reuse outside the exact Spack/GCC environment, and items 4, 8 and 9
are traps for the next consumer.

## Suggested fix
Delete `CFO_Packets.h` or drop the dead include; add the missing includes and
replace `ulong`/`uint` with `<cstdint>` types; fix the two comparisons; remove
the stray members; return `std::unique_ptr` or a value from the calorimeter
getters; reconcile `DTC_Subsystem` with Mu2e-doc-4914 (or update the spec);
make the version macro a `constexpr`.

## Related
- Mu2e/mu2edaq-dataformat-viewer DFV-002 (pending): the YAML follows the spec's subsystem codes

<!-- mega-review-fingerprint: 19ebb664 -->

---

## ACM-006

**Title:** `[submodules] .gitmodules points artdaq-core-mu2e at a personal fork with issues disabled`

**Severity:** S3 — correctness or maintainability risk, no current outage
**Category:** bug
**Location:** `.gitmodules:40-42` (file this against `Mu2e/mu2edaq-main`)
**Found by:** mega-review (`mu2e-sept-mega-review`), 2026-10-06

## What
The superproject's `artdaq-core-mu2e` submodule URL is the fork
`normanajn/artdaq-core-mu2e`, not the collaboration repository
`Mu2e/artdaq-core-mu2e`. The fork has issues disabled. Every other
`artdaq-*`/`otsdaq-mu2e*` submodule points at `Mu2e/`.

## Evidence
```
$ sed -n 40,45p .gitmodules
[submodule "artdaq-core-mu2e"]
	path = artdaq-core-mu2e
	url = git@github.com:normanajn/artdaq-core-mu2e.git
[submodule "artdaq-mu2e"]
	path = artdaq-mu2e
	url = git@github.com:Mu2e/artdaq-mu2e.git
$ gh repo view normanajn/artdaq-core-mu2e --json hasIssuesEnabled,isFork,parent
{"hasIssuesEnabled":false,"isFork":true,"parent":{"name":"artdaq-core-mu2e","owner":{"login":"Mu2e"}}}
$ git -C artdaq-core-mu2e remote -v
origin	git@github.com:normanajn/artdaq-core-mu2e.git (fetch)
upstream	git@github.com:Mu2e/artdaq-core-mu2e.git (fetch)
$ git ls-tree HEAD artdaq-core-mu2e
160000 commit cf86c07691b9e1845ce8ff817593860a4012b76d	artdaq-core-mu2e
$ gh api repos/Mu2e/artdaq-core-mu2e/compare/cf86c07...develop --jq '{status,ahead_by,behind_by}'
{"ahead_by":68,"behind_by":2,"status":"diverged"}
```
The pinned commit is version 9.02.00 (`CMakeLists.txt:13`); upstream
`develop` is 9.07.00. The two fork-only commits are a clang-format change
and a merge. Among the 68 upstream commits the pin lacks are 5225ef0
(2026-09-21, "Break parse loop on exceptions that don't advance the read
pointer") and 8bf66e6 (2026-09-27). Before those, `DTC_Event::SetupEvent` and
`DTC_SubEvent::SetupSubEvent` re-parse the same bytes forever after a
sub-event format-version mismatch or a bad first ROC header
(`DTC_Event.cpp:52-56`, `DTC_SubEvent.cpp:201`, `:240` at the pin). The
upstream comment records a 2026-09-22 bench run: 254k identical passes and
1.5 GB of log before the process was killed.

## Why it matters
A fresh `git clone --recursive` of this superproject checks out a 9.02.00
data-format library that diverges from what `artdaq-mu2e`, `mu2e-pcie-utils`
and the `otsdaq-mu2e*` packages build against upstream, and that still has
the parse-loop hang above. Commits pushed from the submodule go to a personal
account. Tooling that derives the issue tracker from `origin` lands on a
repository that cannot take issues; this review needed a manual override.
Rated S3 rather than S2 because I could not establish that any production
build is made from this superproject's gitlinks. If one is, raise to S2.

## Suggested fix
`git submodule set-url artdaq-core-mu2e git@github.com:Mu2e/artdaq-core-mu2e.git`,
then `git submodule sync`, and move the gitlink to a current upstream
`develop` commit (at least 8bf66e6). Upstream the fork's clang-format commit
by PR first if it is still wanted.

## Related
- `docs/mega-review/CONTRACT.md` §6 routing table

<!-- mega-review-fingerprint: 268897cf -->

---

## Fingerprint inputs

| ID | Input to `sha1(...)[:8]` | Fingerprint |
|---|---|---|
| ACM-001 | `Mu2e/artdaq-core-mu2e:artdaq-core-mu2e/Overlays/DTC_Packets/DTC_DMAPacket.cpp:subsystem-id-roundtrip` | ccbd3de4 |
| ACM-002 | `Mu2e/artdaq-core-mu2e:artdaq-core-mu2e/Overlays/DTCEventFragment.hh:unbounded-payload-parse` | d5befae7 |
| ACM-003 | `Mu2e/artdaq-core-mu2e:artdaq-core-mu2e/Overlays/DTC_Packets/DTC_SubEventHeader.h:bitfield-layout-unchecked` | 11221bcb |
| ACM-004 | `Mu2e/artdaq-core-mu2e:artdaq-core-mu2e/Overlays/CFOEventFragment.hh:include-guard-collision` | 90bc5125 |
| ACM-005 | `Mu2e/artdaq-core-mu2e:artdaq-core-mu2e/Overlays/CFO_Packets.h:header-hygiene` | 19ebb664 |
| ACM-006 | `Mu2e/mu2edaq-main:.gitmodules:submodule-personal-fork` | 268897cf |
