# DM-32UV Block Inventory

The definitive inventory of every 4 KB metadata page in the DM-32UV main config region
(`0x001000-0x0C8FFF`), and an honest count of what is still unknown.

This document is the **census**. For byte-level record layouts see
[05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md); for addressing rules and the V-frame region map see
[04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md).

**Implementation reference**: `src/radios/dm32uv/memory.ts`, `src/radios/dm32uv/constants.ts`
(repository: [neonplug](https://neonplug.app))

---

## Provenance and confidence

| Marker | Meaning |
|---|---|
| *(no marker)* | **CONFIRMED** — observed on real hardware, or implemented and exercised against a real radio |
| `⚠️ DERIVED` | Implemented / inferred, but never verified against hardware |
| `❓ UNKNOWN` | Purpose not established |
| `NEVER OBSERVED` | The byte value has never appeared in any capture and is not named in any implementation |

Two OEM-CPS serial captures against real hardware are the top authority for this document:

| Capture | File | Radio | Firmware | CPS |
|---|---|---|---|---|
| Full read | [`serial_capture_example.txt`](serial_capture_example.txt) | DP570UV (DM-32UV rebadge) | `DM32.01.01.046` | DMR CPS V1.41 |
| Full write | [`serial_capture_write_example.txt`](serial_capture_write_example.txt) | same | same | same |

Where these captures disagree with the reference implementation or with earlier revisions of this
spec, **the captures win**. Every such override is called out inline.

### ⚠️ Sample bias — read this before quoting any number here

The captured radio was at **factory defaults**: 25 channels, 1 zone, 2 scan lists, 5 quick messages,
5 DMR radio IDs, 8 talk groups, 3 roam zones. It is *not* a full codeplug (4000 ch / 250 zones /
800 TGs). Therefore:

- **Allocation counts are a lower bound, not the steady state.** A fully-loaded radio will have more
  live pages and fewer `0xFF` pages.
- Per-block entry *counts* observed here say nothing about maximum capacity. Entry *strides* do.
- What the captures **do** prove definitively is the **logical-ID space**, because the OEM CPS write
  pass walks every slot it knows about whether or not that slot is populated on this radio.

---

## A. Main config memory map

**Range**: `0x001000-0x0C8FFF` — 819,200 bytes = **200 pages of 4 KB**.
Discovered at runtime from V-frame `0x0A`
(`src/radios/dm32uv/protocol.ts:397-404`, `src/radios/dm32uv/memory.ts:18-19`).

```
0x001000 ┌──────────────────────────────────────────────────────────┐
         │  Main config region — 200 × 4 KB pages                   │
         │                                                          │
         │   71 pages carry a live logical block ID  (35.5%)        │
         │   15 pages tagged 0x00  ⚠️ DERIVED: invalidated/superseded│
         │  114 pages tagged 0xFF  ⚠️ DERIVED: free / erased        │
         │  ───                                                     │
         │  200 pages total                                         │
0x0C8FFF └──────────────────────────────────────────────────────────┘

  one page:
  ┌──────────────────────────────────────────────────────────────────┐
  │ 0x000 .. 0xFFE   payload (4095 bytes) — meaning set by the ID     │
  ├──────────────────────────────────────────────────────────────────┤
  │ 0xFFF            LOGICAL BLOCK ID (the "metadata byte")           │
  └──────────────────────────────────────────────────────────────────┘
```

### A.1 The metadata byte is a logical block ID, not a block *type*

**This is the single most important correction in this document.**

In the read capture all 200 pages were probed with a 1-byte read at `+0xFFF`
([`serial_capture_example.txt`](serial_capture_example.txt), 200 probes of the form
`52 <page+0xFFF:3 LE> 01 00`). **Every non-empty value occurs exactly once. There are no
duplicates.**

The write capture is conclusive: the OEM CPS walks the ID space in strictly ascending order and
emits a 4 KB write for **every** slot it knows about, including slots that have no page on this
radio — those it addresses with the sentinel `0xFFF001`
([`serial_capture_write_example.txt`](serial_capture_write_example.txt), 36 sentinel writes).

Consequences:

1. `0x12` is not "channel block type"; it is **channel bank slot 1**. `0x13` is slot 2, and so on.
   The same holds for the zone (`0x5C-0x64`) and talk-group (`0x44-0x48`) ranges.
2. Address assignment behaves like a **flash translation layer**. There is no relationship between
   a logical ID and its physical page address.
3. `0xFFF001` is the "no physical page" sentinel address.

> **Previously documented as**: "metadata byte identifies block type". Not wrong for the
> single-instance blocks, but it misled readers into expecting duplicates within the channel range.
> The correct model is one-page-per-ID.

### A.2 Placement is dynamic — never hardcode addresses

The logical-ID → physical-page mapping differs per radio, per firmware, and changes as the codeplug
is edited. `04-MEMORY-LAYOUT.md` already says "never hardcode addresses"; the FTL behaviour above is
*why*.

The map below is **this one radio at this one moment**. It is published only so that readers can
reproduce the capture analysis. **Do not use it as a lookup table.**

<details>
<summary>Observed logical ID → physical page (read capture, DP570UV, DM32.01.01.046) — reference only</summary>

| ID | Page | ID | Page | ID | Page | ID | Page |
|---|---|---|---|---|---|---|---|
| 0x01 | 0x097000 | 0x1A | 0x072000 | 0x51 | 0x0AC000 | 0x60 | 0x082000 |
| 0x02 | 0x06A000 | 0x1C | 0x02A000 | 0x52 | 0x01B000 | 0x64 | 0x069000 |
| 0x03 | 0x011000 | 0x1D | 0x0AD000 | 0x53 | 0x01F000 | 0x65 | 0x018000 |
| 0x04 | 0x026000 | 0x1F | 0x020000 | 0x54 | 0x006000 | 0x66 | 0x01D000 |
| 0x05 | 0x039000 | 0x20 | 0x002000 | 0x55 | 0x016000 | 0x67 | 0x009000 |
| 0x06 | 0x007000 | 0x22 | 0x041000 | 0x56 | 0x081000 | 0x69 | 0x0BC000 |
| 0x07 | 0x001000 | 0x23 | 0x0A2000 | 0x57 | 0x019000 | 0x6A | 0x01E000 |
| 0x08 | 0x0C3000 | 0x30 | 0x00D000 | 0x58 | 0x024000 | 0x6B | 0x096000 |
| 0x09 | 0x004000 | 0x31 | 0x049000 | 0x59 | 0x012000 | 0x6C | 0x051000 |
| 0x0A | 0x005000 | 0x32 | 0x03F000 | 0x5A | 0x04D000 | 0x6D | 0x025000 |
| 0x0B | 0x08D000 | 0x34 | 0x003000 | 0x5B | 0x01C000 | 0x6E | 0x07F000 |
| 0x0C | 0x0C8000 | 0x37 | 0x031000 | 0x5C | 0x035000 | 0x74 | 0x065000 |
| 0x0D | 0x00E000 | 0x3B | 0x028000 | 0x5D | 0x013000 | 0x75 | 0x059000 |
| 0x0E | 0x058000 | 0x3D | 0x084000 | 0x5E | 0x07E000 | 0x7C | 0x098000 |
| 0x0F | 0x00F000 | 0x41 | 0x010000 | 0x5F | 0x022000 | | |
| 0x10 | 0x00C000 | 0x42 | 0x008000 | | | | |
| 0x11 | 0x06B000 | 0x43 | 0x00A000 | | | | |
| 0x12 | 0x0AA000 | 0x44 | 0x00B000 | | | | |
| 0x13 | 0x015000 | 0x4B | 0x0C5000 | | | | |
| 0x14 | 0x017000 | 0x4F | 0x021000 | | | | |
| 0x18 | 0x073000 | 0x50 | 0x01A000 | | | | |

71 entries. Source: machine parse of [`serial_capture_example.txt`](serial_capture_example.txt).

</details>

### A.3 `0x00` vs `0xFF` are not the same thing

`⚠️ DERIVED.` Both are treated as "empty" by the reference implementation
(`src/radios/dm32uv/memory.ts:45-46`, `:69-70`; `src/radios/dm32uv/constants.ts:26-27`), but the
capture shows a structural difference:

| Tag | Count | Distribution | Interpretation |
|---|---|---|---|
| `0xFF` | 114 | one contiguous high region | erased / never-allocated flash page |
| `0x00` | 15 | scattered *among* live pages | allocated-then-invalidated (superseded copy) |

That pattern is consistent with a wear-levelled copy-on-write FTL. It is an inference, not a
measurement — see open question **Q2** in section C.

### A.4 Where the metadata byte is read

**The OEM CPS reads the metadata byte at `0xFFF`, in all 200 probes.**

> **Previously documented as**: "*The OEM CPS software reads metadata at offset `0x00A`… observed in
> serial captures (e.g. `0xFFAF0A` reading metadata `0x12` from block `0xFFA000`)*"
> (`04-MEMORY-LAYOUT.md`). **This is incorrect.** There is not a single `0x00A` metadata read in
> either capture. The `0xFFAF0A` example is a mis-transcription of a `+0xFFF` read. Do not implement
> the "alternative method"; there is no evidence it exists.

Confirmed read form: `52 <page+0xFFF:3 LE> 01 00` → `57 <addr:3 LE> 01 00 <byte>`
(`src/radios/dm32uv/memory.ts:36-42`, `src/radios/dm32uv/constants.ts:52`;
[`serial_capture_write_example.txt`](serial_capture_write_example.txt) lines 117-119).

On write, the ID is **inside** the 4096-byte payload at `data[0xFFF]`, not appended
(`src/radios/dm32uv/connection.ts:262-278`, verbatim comment at `:270-271`:
*"The metadata byte is INSIDE the data block at offset 0xFFF, not sent separately"*).

---

## B. Complete metadata value inventory

Every byte value `0x00`-`0xFF`, grouped into contiguous ranges only where the range is a genuine
bank (the write capture walks it contiguously).

- **Blocks** = pages carrying this ID **on the captured factory radio**. `0` means the ID is a real
  slot the CPS declares but no page was allocated. Remember section A's sample-bias warning.
- **Entry size** = measured stride, in bytes.
- **Status** = CONFIRMED / DERIVED / UNKNOWN / NEVER OBSERVED.
- **In neonplug** = Yes / Partial / No / n-a.
- Evidence citations are repo-relative to [neonplug](https://neonplug.app) unless they name a
  capture file.

| Value | Dec | Name / Purpose | Blocks | Entry size | Status | Implemented in neonplug | Evidence |
|---|---|---|---|---|---|---|---|
| `0x00` | 0 | Page tag — allocated then invalidated. **Not a logical ID.** | 15 | n-a | `⚠️ DERIVED` | Yes (classified `'empty'`) | Read capture: 15/200 pages, scattered among live pages; `src/radios/dm32uv/constants.ts:26`, `src/radios/dm32uv/memory.ts:45-46` |
| `0x01` | 1 | ❓ UNKNOWN — allocated, never touched by the OEM CPS | 1 | ❓ | `❓ UNKNOWN` | No | Read capture: metadata probe only, contents never read by anyone |
| `0x02` | 2 | Frequency adjustment / calibration data | 1 | ❓ | `⚠️ DERIVED` | Partial — block is read and cached, never parsed | Name comes from `src/radios/dm32uv/constants.ts:20` only; read at `src/radios/dm32uv/protocol.ts:599`, `:835`. Capture shows dense binary, read-only, **never written** by the CPS. No field has ever been decoded. `READPLAN.md:507`: *"Cannot verify calibration values without potentially damaging radio (intentionally not verified)"* |
| `0x03` | 3 | **"Call" list** — record layout CONFIRMED, purpose `❓ UNKNOWN`. CPS reads **and** writes it. Entries `"Call 1"`…`"Call 5"`, **UTF-16LE names** (the only block that uses UTF-16) | 1 | **40** (`0x28`) from base `0x218` | `❓ UNKNOWN` (purpose only) | No — `src/radios/dm32uv/memory.ts:53-54` mislabels it `'digitalemergency'` (stale; nothing in the read path uses it) | Read + write capture; 6 entries decoded. Previously documented as Digital Emergency Systems — **incorrect**, that is block `0x10` |
| `0x04` | 4 | Radio Settings — power-on display, colours, key assignments, GPS, digital timings | 1 | single record | CONFIRMED | Partial — ~85 of 4096 bytes annotated | ASCII `"Welcome"` @`0x001`, `"DM-32UV"` @`0x00F` in read capture, matching `src/radios/dm32uv/blockLayouts.ts:52-54` exactly; full offset set `src/radios/dm32uv/blockLayouts.ts:48-99`, `src/radios/dm32uv/structures.ts:1454-1497` |
| `0x05` | 5 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only. **Previously documented as "Not Found / Unused" — incorrect: the page exists and is allocated** |
| `0x06` | 6 | Multi-section analog config: DTMF code table @`0x000`; analog contact names @`0x200`; a second `"BDC"` contact table @`0xAA0`; count byte @`0x1FF` | 1 | 16 (DTMF) / 32 (analog contacts) | CONFIRMED | Partial — only the `0x1FF` counter is consumed | Read capture: DTMF sequences @`0x000` stride 16; ASCII `"AContact 1"` @`0x200` stride 32; `"BDC Cotnacts 1"` @`0xAA0`; byte @`0x1FF` = 7. `src/radios/dm32uv/constants.ts:22`, `:58`. **Previously documented as "DTMF Encode Data" only — incomplete, it is a multi-section block** |
| `0x07` | 7 | ❓ UNKNOWN — a neonplug comment calls it "Config header"; never touched by the CPS | 1 | ❓ | `❓ UNKNOWN` | No — explicitly listed as "known but unhandled" | `src/radios/dm32uv/memory.ts:73-74`; `READPLAN.md:546-563`: *"⛔ Intentionally Not Implemented (Factory/Calibration Data) … Structure is unknown and not actively parsed by OEM CPS … Must never be read or written to avoid bricking the radio"* |
| `0x08` | 8 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only. **Previously documented as "Reserved" — that claim has no evidence behind it (see §B.1)** |
| `0x09` | 9 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only. Previously "Reserved" — unsupported |
| `0x0A` | 10 | Quick text messages | 1 | **129** (`0x81`) | CONFIRMED | Yes | Read capture: count @`0x00` = 5, 16-byte header, `"How are you?"` @`0x011`, stride 129 measured. Matches `src/radios/dm32uv/constants.ts:40`, `:53-55` |
| `0x0B` | 11 | Quick Access Contact List — contact counts, 128-slot usage bitmask, two contact index tables | 1 | 2 B per index entry | CONFIRMED | Partial — fully mapped and rendered; the talk-group parser consumes Index Table 1 at runtime; no encoder | Field map: `src/components/diagnostics/QuickContactsBlockDetails.tsx`. Read-capture header bytes `0a 00 05 00 01 ff…` decode cleanly against it (total=10, group=5). Previously documented as "RX Group List" then as "Unknown" — both understate what is known |
| `0x0C` | 12 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only. Previously "Reserved" — unsupported |
| `0x0D` | 13 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only. Previously "Reserved" — unsupported |
| `0x0E` | 14 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only. Previously "Reserved" — unsupported. (Unrelated to V-frame `0x0E`, a different namespace) |
| `0x0F` | 15 | DMR RX Groups (receive group lists) | 1 | **109** (`0x6D`) | CONFIRMED | Yes | Read capture: count @`0x00` = 31, entries from `0x010`, stride 109 measured. `src/radios/dm32uv/constants.ts:19`, `:41`; `src/radios/dm32uv/memory.ts:63-64`. **Not** the same namespace as V-frame `0x0F` |
| `0x10` | 16 | Multi-section: Digital Emergency @`0x000` / Analog Emergency @`0x0AC` / Encryption keys @`0x300` | 1 | 20 / 36 / 44 | CONFIRMED (block + digital-emergency section); sub-sections 2 and 3 `⚠️ DERIVED` | Partial — encryption keys parse; **both emergency parsers are stubs returning empty** | Write capture page `0x00C000` begins ASCII `"DEmer 1"`…`"DEmer 8"`, 8 × 20 B from `0x000` — matches `src/radios/dm32uv/constants.ts:117` (*"8 entries × 20 bytes = 0x000–0x09F (confirmed by hardware hexdump)"*). Analog/key bases are arithmetic only: `src/radios/dm32uv/constants.ts:118`. Stubs: `src/radios/dm32uv/protocol.ts:3084-3086`, `:3234-3236` |
| `0x11` | 17 | Scan lists | 1 | **57** (`0x39`) | CONFIRMED | Yes | Read capture: count @`0x00` = 2, `"Scan List 1"` @`0x001`, stride 57 measured. `src/radios/dm32uv/constants.ts:35`, `:51` |
| `0x12` | 18 | **Channel bank slot 1** — additionally carries the global channel count | 1 | **48** (`0x30`) | CONFIRMED | Yes | Read capture: count @`0x00` = `19 00` (uint16 LE = 25), `"Channel 1"` @`0x010`, stride 48; write capture: `80 00 ff ff` = 128. `src/radios/dm32uv/constants.ts:8`, `:33`, `:48-49` |
| `0x13`-`0x40` | 19-64 | **Channel bank slots 2-47** | 17 allocated of 46 | 48 | CONFIRMED | Yes | Write capture walks `0x12`…`0x41` contiguously, 48 slots, no gaps; unallocated slots written to sentinel `0xFFF001`. `src/radios/dm32uv/memory.ts:47-48` |
| `0x41` | 65 | **Channel bank slot 48** *and* the **VFO A/B** record page (VFO A @`0x0F9F`, VFO B @`0x0FCF`) | 1 | 48 | CONFIRMED | Partial — VFO records read/written; **VFO TX-contact write deliberately disabled** | Read capture VFO B: RX `00 25 20 43` @`0x0FDF` → 432.02500 MHz, exactly matching `src/radios/dm32uv/blockLayouts.ts:109-112`. Dual role: `src/radios/dm32uv/constants.ts:9` vs `:16` and `src/radios/dm32uv/blockLayouts.ts:148`. Disabled write: `src/radios/dm32uv/protocol.ts:3037-3041` |
| `0x42` | 66 | TX contact index, channels 1-2047 | 1 | 2 | CONFIRMED | Yes | Read capture stride 2 from `0x000`; `src/radios/dm32uv/blockLayouts.ts:116-129`; bit layout `src/components/diagnostics/TxContactStructureReference.tsx:107-110` |
| `0x43` | 67 | TX contact index, channels 2048+ **and** VFO A (`0x0FFA`) / VFO B (`0x0FFC`) | 1 | 2 | CONFIRMED | Partial — read/parsed; VFO slots not written | Both captures, anchored on the metadata byte at `0x0FFF`: read capture `0x0FFA` = `00 01`, `0x0FFC` = `01 01`; write capture `0e 01 0e 01` — the only non-`0xFF` payload bytes in the whole block, pinning the two slots exactly. `src/radios/dm32uv/constants.ts`, `src/radios/dm32uv/blockLayouts.ts` |
| `0x44` | 68 | **Talk group bank slot 1** | 1 | **24** (entry 1 is 25: it carries a leading header byte) | CONFIRMED | Yes | Read capture: `"Contacts 1"` @`0x002`, stride 24 measured. `src/components/diagnostics/TalkGroupsBlockDetails.tsx:231-232`, `:238-242`; write-back stamps the ID at `src/radios/dm32uv/structures.ts:3466` |
| `0x45`-`0x48` | 69-72 | **Talk group bank slots 2-5** | 0 | 24 | CONFIRMED (range membership) | Yes (range handled) | Write capture walks `0x44`-`0x48` contiguously; all four written to sentinel `0xFFF001` on this radio, so **no content has ever been observed for these four IDs** |
| `0x49`-`0x4A` | 73-74 | — | 0 | — | `NEVER OBSERVED` | n-a | none |
| `0x4B` | 75 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only |
| `0x4C`-`0x4E` | 76-78 | — | 0 | — | `NEVER OBSERVED` | n-a | none |
| `0x4F` | 79 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only |
| `0x50`-`0x5B` | 80-91 | ❓ UNKNOWN — **12 consecutive allocated IDs**, none ever touched by the CPS. The largest single unexplained region in the radio | 12 | ❓ | `❓ UNKNOWN` | No | Probe only. `0x5A` was **previously documented as "Reserved"** — unsupported. The contiguity strongly suggests a bank of something, but nothing is known about what |
| `0x5C` | 92 | **Zone bank slot 1** | 1 | **145** (`0x91`) | CONFIRMED | Yes | Read capture: 16-byte header, `"Zone 1"` @`0x010`, stride 145 measured. `src/radios/dm32uv/constants.ts:10`, `:34`, `:50` |
| `0x5D`-`0x64` | 93-100 | **Zone bank slots 2-9** | 5 allocated of 8 | 145 | CONFIRMED | Yes — read and write both cover the full range via `writeAllData()` | Write capture walks `0x5C`…`0x64` contiguously, and a 250-zone hardware round-trip through the reference implementation returned byte-perfect. Together these **retire** its own verbatim caveat *"unverified, pending hardware confirmation"* on `ZONE_LAST` (`src/radios/dm32uv/constants.ts:11`) — flagged for correction in its TODO. Read: `src/radios/dm32uv/protocol.ts` |
| `0x65` | 101 | Roaming zones | 1 | **33** (`0x21`) | CONFIRMED (layout) | **No** — not implemented at all | Read capture: 16-byte header, `"Roam Zone 1"` @`0x010`, stride 33 measured. `READPLAN.md:567-598` marks it *"❌ Not Implemented"* |
| `0x66` | 102 | Roaming channels | 1 | **26** (`0x1A`) | CONFIRMED (layout) | **No** — not implemented at all | Read capture: `"Roam CH 1"` @`0x000`, stride 26 measured; write capture page `0x01D000` begins `"Roam CH 1"`. `READPLAN.md:601-631` *"❌ Not Implemented"* |
| `0x67` | 103 | DMR Radio ID list | 1 | **16** (`0x10`) | CONFIRMED | Yes | Read capture: count @`0x00` = 5, entries from `0x010`, `"Radio 1"` @`0x013`, stride 16. `src/radios/dm32uv/constants.ts:25`, `:42`, `:56-57` |
| `0x68` | 104 | — | 0 | — | `NEVER OBSERVED` | n-a | none |
| `0x69`-`0x6E` | 105-110 | ❓ UNKNOWN — 6 consecutive allocated IDs, never touched | 6 | ❓ | `❓ UNKNOWN` | No | Probe only. `0x6A`, `0x6C`, `0x6D` were **previously documented as "Reserved"** — unsupported |
| `0x6F`-`0x73` | 111-115 | — | 0 | — | `NEVER OBSERVED` | n-a | none |
| `0x74` | 116 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only |
| `0x75` | 117 | ❓ UNKNOWN — allocated, never touched | 1 | ❓ | `❓ UNKNOWN` | No | Probe only. **Previously documented as "Reserved"** — unsupported |
| `0x76`-`0x7B` | 118-123 | — | 0 | — | `NEVER OBSERVED` | n-a | none |
| `0x7C` | 124 | ❓ UNKNOWN — allocated, never touched. Highest live ID observed | 1 | ❓ | `❓ UNKNOWN` | No | Probe only |
| `0x7D`-`0xFE` | 125-254 | — | 0 | — | `NEVER OBSERVED` | n-a | none. 130 consecutive values, no evidence of any of them |
| `0xFF` | 255 | Page tag — free / erased flash page. **Not a logical ID.** | 114 | n-a | `⚠️ DERIVED` | Yes (classified `'empty'`) | Read capture: 114/200 pages, one contiguous high region; `src/radios/dm32uv/constants.ts:27`, `src/radios/dm32uv/memory.ts:69-70` |

Page arithmetic check: `71 live + 15 (0x00) + 114 (0xFF) = 200`. ✔

### B.1 "Reserved" was invention — it is retired

Ten values were previously listed in `04-MEMORY-LAYOUT.md` as **Reserved**:
`0x08 0x09 0x0C 0x0D 0x0E 0x5A 0x6A 0x6C 0x6D 0x75`.

The captures show that **all ten pages exist and are allocated** — they carry a live logical ID —
but the OEM CPS never reads or writes any of them. "Reserved" implies the manufacturer has set them
aside for future use, which no evidence supports. Their correct status is **`❓ UNKNOWN`
(allocated, purpose unestablished)**.

Do not reintroduce the word "Reserved" for a block without evidence that the radio actually reserves
it.

---

## C. Scoreboard

### C.1 Distinct metadata values by status

| Status | Distinct values | Notes |
|---|---|---|
| **CONFIRMED** | **74** | `0x04 0x06 0x0A 0x0B 0x0F 0x10 0x11`, `0x12`-`0x41` (48), `0x42 0x43`, `0x44`-`0x48` (5), `0x5C`-`0x64` (9), `0x65 0x66 0x67` |
| **DERIVED** | **3** | `0x00`, `0x02`, `0xFF` |
| **UNKNOWN** | **32** | `0x01 0x03 0x05 0x07 0x08 0x09 0x0C 0x0D 0x0E`, `0x4B 0x4F`, `0x50`-`0x5B` (12), `0x69`-`0x6E` (6), `0x74 0x75 0x7C` |
| **Subtotal — values ever observed or named** | **109** | |
| **NEVER OBSERVED** | **147** | `0x49-0x4A 0x4C-0x4E 0x68 0x6F-0x73 0x76-0x7B 0x7D-0xFE` |
| **Total byte values** | **256** | |

Of the 32 UNKNOWN values, **31 are allocated on the radio but never touched by the OEM CPS**
(`0x01 0x05 0x07 0x08 0x09 0x0C 0x0D 0x0E 0x4B 0x4F 0x50 0x51 0x52 0x53 0x54 0x55 0x56 0x57 0x58
0x59 0x5A 0x5B 0x69 0x6A 0x6B 0x6C 0x6D 0x6E 0x74 0x75 0x7C`). The 32nd is `0x03`, which the CPS
*does* read and write but whose purpose is still unestablished.

### C.2 Page census on the captured radio

| Category | Pages | Share |
|---|---|---|
| Live logical IDs | 71 | 35.5% |
| Tagged `0x00` (invalidated) | 15 | 7.5% |
| Tagged `0xFF` (free) | 114 | 57.0% |
| **Total** | **200** | 100% |

| Logical-ID category | Count |
|---|---|
| Live IDs the OEM CPS reads and/or writes → **this is the codeplug** | **40** |
| Live IDs the OEM CPS never touches → **the genuine unknowns** | **31** |
| IDs the CPS write pass declares | 75 |
| …of those, IDs with **no page** on this radio → unallocated capacity slots, **not unknowns** | 36 |

The 36 sentinel-written IDs are empty channel / zone / talk-group bank slots
(`0x15 0x16 0x17 0x19 0x1B 0x1E 0x21 0x24-0x2F 0x33 0x35 0x36 0x38-0x3A 0x3C 0x3E 0x3F 0x40 0x45-0x48 0x61-0x63`).
On a fully-loaded codeplug they would all be allocated.

### C.3 Parse completeness for the block types neonplug reads

| Block | Parses completely | Notes |
|---|---|---|
| `0x0A` Quick messages | ✅ complete | `src/radios/dm32uv/protocol.ts:596` |
| `0x0F` RX groups | ✅ complete | `src/radios/dm32uv/protocol.ts:600` |
| `0x11` Scan lists | ✅ complete | 57-byte model, `src/radios/dm32uv/constants.ts:35`, `:51` |
| `0x12`-`0x41` Channels | ⚠️ partial | ~7 byte positions inside the 48-byte record still unknown (section E) |
| `0x42` / `0x43` TX contacts | ⚠️ partial | byte 0 bits 3-1 undecoded (`src/radios/dm32uv/blockLayouts.ts:43-46`); VFO TX write disabled (`src/radios/dm32uv/protocol.ts:3037-3041`) |
| `0x44` Talk groups | ⚠️ partial | parser bails on the first empty entry, so a hole truncates the list (`src/components/diagnostics/TalkGroupsBlockDetails.tsx:117-131`) |
| `0x5C`-`0x64` Zones | ⚠️ partial | reads and writes all 9 slots (`writeAllData()`); a legacy `writeZones()` entry point targets only `0x5C` but is not the path the app uses. Remaining gap: the 16-byte block header is undecoded |
| `0x67` DMR radio IDs | ⚠️ partial | count-field width disputed inside neonplug: 4-byte DWORD (`src/radios/dm32uv/constants.ts:56`) vs 1 byte (`READPLAN.md:639-642`) |
| `0x04` Radio settings | ⚠️ partial | ~85 of 4096 bytes annotated |
| `0x0B` Quick access contacts | ⚠️ partial | mapped and displayed; no encoder |
| `0x10` Emergency / keys | ❌ stub | *"TODO: Structure parsing needs verification - return empty for now"* — `src/radios/dm32uv/protocol.ts:3084-3086`, `:3234-3236`. **The layout is hardware-confirmed; only the parser is missing.** |
| `0x02` Calibration | ❌ not parsed | read and cached only |
| `0x06` Analog config | ❌ not parsed | only the `0x1FF` counter is consumed |
| `0x03` | ❌ not parsed | purpose unknown |

**Summary: 3 of 14 read block types parse completely; 7 parse partially; 4 are not parsed at all.**

### C.4 What is still unknown — prioritised, with an experiment for each

| # | Open question | Suggested experiment | Cost / risk |
|---|---|---|---|
| **Q1** | **What are the 31 CPS-untouched logical IDs?** Especially the contiguous runs `0x50`-`0x5B` (12) and `0x69`-`0x6E` (6). This is by far the largest gap: 31 of 71 live pages, ~124 KB, with *zero* observed content. | Read all 31 pages via a diagnostics hexdump and inspect for ASCII, repeated strides, and all-`0xFF` tails. Reading is non-destructive. | Zero cost, no risk |
| **Q2** | Is `0x00` an invalidated page and `0xFF` a free page? | Change one setting in the OEM CPS, write, then re-probe all 200 tags and diff against the baseline. If a live ID moves to a new page and its old page flips to `0x00`, confirmed. | Cheap, one write cycle |
| **Q3** | What is block `0x03`? The CPS writes it, so it is codeplug data. Its **record layout is now mapped** (entries `"Call 1"`…`"Call 5"` at base `0x218`, stride `0x28`, UTF-16LE names, two uint16 reference fields) — only the *meaning* is open. What do the two references point at? The four observed values (`0x0C91 0x2441 0x17CA 0x4FD6`) match no other list in the codeplug, and the six entries enumerate every pair of them. | Rename one entry in the OEM CPS and diff to find which dialog owns it; then change one reference and see which list the new value comes from. `"Disable"`/`"Enable"` at `0x8A0`/`0x8D0` are further anchors. | Cheap |
| **Q4** | Is byte `0x1FF` of block `0x06` the **talk-group** counter (as neonplug asserts, `src/radios/dm32uv/constants.ts:58`) or the **analog-contact** counter? It reads 7, and exactly 7 analog contact names follow at `0x200`; the radio has 8 talk groups. | Add one analog contact → re-read `0x06`. Then add one talk group → re-read. Whichever action moves the byte wins. | Cheap, decisive |
| **Q5** | Field layout of roaming zones (33 B) and roaming channels (26 B). Strides are hardware-confirmed, contents are not mapped at all, and neonplug does not implement the feature. | Populate roam zones/channels via the OEM CPS one field at a time and diff `0x65` / `0x66`. | Moderate |
| **Q6** | What occupies block `0x10` beyond `0x460`? 2975 bytes of a hardware-confirmed block are entirely unaccounted for. | Populate all 8 digital emergency systems, all 16 analog emergency systems and all 8 encryption keys, then diff. | Moderate |
| **Q7** | The `0x04` radio-settings block is ~98% unmapped. | Drive the OEM CPS settings UI field by field and diff `0x04` after each change. Highest yield per hour of any experiment here. | Moderate, mechanical |
| **Q8** | Content of talk-group slots `0x45`-`0x48` and zone slots `0x61`-`0x63` — the ranges are confirmed but no page has ever been observed. | Load a codeplug with >170 talk groups and >28 zones, then re-read. | Moderate |
| **Q9** | V-frames `0x02` (12 B, two repeated `0xA415` words) and `0x0D` (64 B, queried with an undocumented `0x40`-length request form). | Compare across two radios on different firmware. | Cheap |
| **Q10** | Why does a 25-channel factory radio already have **19** of the 48 channel-slot pages allocated, when one slot holds 84? Page allocation is clearly not purely on demand. *(This is a question about page **lifecycle**, not about the channel layout — that is fully settled: 84 + 47 × 85 = 4079 slots, see § D.)* Likely the same mechanism as Q2. | Fold into the Q2 diff: after a write cycle, check whether the channel-slot page count changes. | Cheap |

---

## D. Per-block detail index

Only blocks whose purpose is established. Offsets are block-relative. Every block ends with its
logical ID at `0xFFF`.

**Implementation reference**: `src/radios/dm32uv/constants.ts`, `src/radios/dm32uv/blockLayouts.ts`,
`src/radios/dm32uv/structures.ts`

### `0x02` — Calibration `⚠️ DERIVED`
Dense binary. No count field, no entry stride, no decoded field. Read-only: the OEM CPS never writes
it. Do not write this block.
→ [05-DATA-STRUCTURES.md § 0x02](05-DATA-STRUCTURES.md#0x02---frequency-adjustmentcalibration-data)

### `0x04` — Radio Settings
Single flat record, not an array. No count field. Notable anchors: Power-On Interface `0x00`;
Power-On Display line 1 `0x01` (14 B ASCII), line 2 `0x0F` (14 B ASCII); backlight `0x30`; colours
`0x34/0x35/0x38/0x39/0x3A/0x3B` (low nibble); GPS byte `0x40`; UTC zone `0x41`; digital timings
`0x60`-`0x67`; key lock `0x85`; side/programmable keys `0x87`-`0x90`; analog call table `0x120`
(4 × stride 2); One Touch Call 1 `0x200`-`0x204`; Fun+0 `0x230`-`0x236`.
→ [05-DATA-STRUCTURES.md § 0x04](05-DATA-STRUCTURES.md#0x04---embedded-information--radio-names)

### `0x06` — Analog config (DTMF + analog contacts)
DTMF code table from `0x000`, stride 16. Count byte at `0x1FF` (= 7 on the captured radio) —
`⚠️ DERIVED` which list it counts, see **Q4**. Analog contact names from `0x200`, stride 32. A second
`"BDC"` contact table at `0xAA0`, stride unknown.
→ [05-DATA-STRUCTURES.md § 0x06](05-DATA-STRUCTURES.md#0x06---dtmf-encode-data)

### `0x0A` — Quick text messages
Count at `0x00`; 16-byte header; entries from `0x010`; stride **129** (`0x81`); max 20.
→ [05-DATA-STRUCTURES.md § 0x0A](05-DATA-STRUCTURES.md#0x0a---quick-text-messages)

### `0x0B` — Quick Access Contact List
Total contact count `0x00`-`0x01` (LE u16); group-call count `0x02`-`0x03` (LE u16); private-call
count `0x04`; `0x05`-`0x0F` observed all `0xFF`; slot usage bitmask `0x10`-`0x1F` (128 slots,
**0 = used, 1 = free**); Index Table 1 (name-sorted) `0x100`-`0x6FF`, 2 B/entry `[contact_index]
[type_byte]`, 768 max; Index Table 2 (alphabetical) `0x740`-`0xCFF`, same format, 704 max.
`type_byte`: `0x30` Private, `0x40` Group, `0x50` All Call, `0xFF 0xFF` empty/end.
No section exists in 05-DATA-STRUCTURES.md yet — source is
`src/components/diagnostics/QuickContactsBlockDetails.tsx`.

### `0x0F` — RX Groups
Count at `0x00` (= 31 on the captured radio); entries from `0x010`; stride **109** (`0x6D`);
max 32 `⚠️ DERIVED` (inferred from a 32-bit bitmask in the header,
`src/radios/dm32uv/constants.ts:120`).
→ [05-DATA-STRUCTURES.md § RX Group List](05-DATA-STRUCTURES.md#rx-group-list-structure)

### `0x10` — Digital Emergency / Analog Emergency / Encryption keys
`0x000`-`0x09F`: 8 digital emergency systems × 20 B (name 10 B @`+0x00`, index @`+0x0A`) —
hardware-confirmed. `0x0AC`-`0x2FF`: up to 16 analog emergency systems × 36 B `⚠️ DERIVED`.
`0x300`-`0x45F`: 8 encryption keys × 44 B `⚠️ DERIVED`. No count field known for any section.
→ [05-DATA-STRUCTURES.md § 0x10](05-DATA-STRUCTURES.md#0x10---multi-section-digital-emergency--analog-emergency--encryption-keys)

> Note the split verdict: the **layout** is hardware-confirmed (write capture page `0x00C000` opens
> with `"DEmer 1"`), while the **reference parser is a stub returning empty**
> (`src/radios/dm32uv/protocol.ts:3084-3086`). Both facts are true and neither cancels the other.

### `0x11` — Scan lists
Count at `0x00` (= 2); entry N at `(57 × N) − 56`, i.e. first entry at `0x001`; stride **57**;
max 32 lists, 15 channels each.
→ [05-DATA-STRUCTURES.md § Scan List](05-DATA-STRUCTURES.md#scan-list-structure-57-bytes)

### `0x12`-`0x41` — Channel banks (48 slots)
Slot 1 (`0x12`) only: channel count at `0x00`-`0x01`, **2-byte uint16 LE**; bytes `0x02`-`0x0F` are
fill `❓ UNKNOWN` (the OEM CPS writes `0xFF` there, the factory image `0x00`). Entries from `0x010`
in slot 1, from `0x000` in slots 2-48. Stride **48** (`0x30`).
**Capacity — settled, fully implemented:**

| | Records | Offsets |
|---|---|---|
| Slot 1 (`0x12`) | **84** (16-byte header displaces one) | `0x010` … `0x0FA0` |
| Slots 2-48 (`0x13`-`0x41`) | **85** each | `0x000` … `0x0FC0` |
| **Total** | **84 + 47 × 85 = 4079** | ≥ the radio's 4000-channel maximum |

Plus two pseudo-channels: **VFO A = channel 4001**, **VFO B = channel 4002**, stored in slot 48
(`0x41`) as ordinary 48-byte channel records and parsed with the same decoder. They are
**end-aligned** to the page rather than sitting on the grid that starts at `0x000`:

```
VFO B = 0xFFF − 48      = 0x0FCF … 0x0FFE     (0xFFF = metadata byte)
VFO A = 0xFFF − 2 × 48  = 0x0F9F … 0x0FCE
```

This does not collide with real channels: slot 48 covers channels 3995-4079, and the radio caps at
4000, so only records 0-5 (`0x000`-`0x11F`) are ever occupied — the VFOs live in the unused tail.
Their TX-contact indices live separately in block `0x43` at `0x0FFA` / `0x0FFC`.

The reference implementation reads **and** writes this whole range correctly — the 4000-channel bank
via `readChannels()` / `generateChannelBlocks()` (which re-stamps the `0xFFF` ID and the uint16 count
per block), and both VFOs via the radio-settings path. The one deliberate gap is VFO **TX-contact**
write, which is disabled pending verification.

Field width settled by the write capture: it writes `80 00 ff ff ff ff …` and then lays down
exactly 128 channel records. Two bytes decode to `0x0080` = 128 ✅; four bytes decode to
`0xFFFF0080` ✗. The read capture's `19 00 00 00` = 25 is ambiguous only because that radio
`0x00`-fills its header. The reference implementation's 2-byte read
(`src/radios/dm32uv/memory.ts`) is **correct**.
→ [05-DATA-STRUCTURES.md § Channel Structure](05-DATA-STRUCTURES.md#channel-structure-48-bytes)

### `0x42` / `0x43` — TX contact index
2 bytes per channel. `0x42` covers channels 1-2047 at `(channel − 1) × 2`. `0x43` continues from
channel 2048 and holds VFO A at `0x0FFA`, VFO B at `0x0FFC`. Record: byte 0 bits 7-4 = talk-group
index bits 11-8; byte 0 bits 3-1 `❓ UNKNOWN`; byte 0 bit 0 = digital flag; byte 1 = index bits 7-0.
No section exists in 05-DATA-STRUCTURES.md yet — source is
`src/components/diagnostics/TxContactStructureReference.tsx`.

### `0x44`-`0x48` — Talk group banks (5 slots)
Entry 1 at `0x000` is **25 bytes** (leading `0x00` header byte, *"First entry MUST have header byte
for radio recognition"*); entry N≥2 at `25 + (N−2) × 24`, **24 bytes**. Fields: flag (`0x00` =
PC-created, `0x01` = radio-created), name 16 B ASCII, null, contact number 3 B LE, call type
(`0x03` Private / `0x04` Group / `0x05` All Call), 2 B pad. No count field in the block — the live
set and display order come from Index Table 1 of block `0x0B`. Capacity 4096/24 = 170 per slot.
No section exists in 05-DATA-STRUCTURES.md yet — source is
`src/components/diagnostics/TalkGroupsBlockDetails.tsx`.

### `0x5C`-`0x64` — Zone banks (9 slots)
16-byte block header (contents `❓ UNKNOWN`); zone N at `16 + (N−1) × 145`; stride **145** (`0x91`);
zone name 11 B; 64 channels per zone; 250 zones max; ~28 zones per slot `⚠️ DERIVED`
(`(4096−16)/145 = 28.1`, comment says *"Approximate"*).
→ [05-DATA-STRUCTURES.md § Zone Structure](05-DATA-STRUCTURES.md#zone-structure-145-bytes)

### `0x65` — Roaming zones
16-byte header; entries from `0x010`; stride **33** (`0x21`) measured from hardware. Field layout
`❓ UNKNOWN`. Not implemented in neonplug.
→ [05-DATA-STRUCTURES.md § 0x65](05-DATA-STRUCTURES.md#0x65---roaming-zones)

### `0x66` — Roaming channels
Entries from `0x000`; stride **26** (`0x1A`) measured from hardware. Field layout `❓ UNKNOWN`. Not
implemented in neonplug.
→ [05-DATA-STRUCTURES.md § 0x66](05-DATA-STRUCTURES.md#0x66---roaming-channels)

### `0x67` — DMR Radio ID list
Count at `0x00` (= 5 on the captured radio); entries from `0x010`; stride **16** (`0x10`); 250 max.
`⚠️` Count-field width is disputed inside the reference implementation: 4-byte DWORD
(`src/radios/dm32uv/constants.ts:56`) vs 1 byte (`READPLAN.md:639-642`). Entry base is likewise
disputed: `0x00` (`src/radios/dm32uv/constants.ts:57`) vs `0x10` (`READPLAN.md:642`). The capture
supports **entries from `0x010`** (`"Radio 1"` observed at `0x013`).
→ [05-DATA-STRUCTURES.md § 0x67](05-DATA-STRUCTURES.md#0x67---dmr-radio-id-list)

---

## E. Unknown and partially-mapped regions *within* known blocks

Unmapped block types are only half the problem. These are the byte ranges inside blocks we otherwise
understand.

| Block | Byte range | Size | What we know | Status |
|---|---|---|---|---|
| `0x04` Radio settings | `0x1D`-`0x2F` | 19 B | nothing | `❓ UNKNOWN` |
| `0x04` | `0x32` | 1 B | sits between backlight duration (`0x31`) and date format (`0x33`) | `❓ UNKNOWN` |
| `0x04` | `0x3C`-`0x3F` | 4 B | between the colour bytes and the GPS byte | `❓ UNKNOWN` |
| `0x04` | `0x43`-`0x5F` | 29 B | after GPS report interval, before the digital block | `❓ UNKNOWN` |
| `0x04` | `0x68`-`0x7F` | 24 B | after the digital-settings group | `❓ UNKNOWN` |
| `0x04` | `0x82`-`0x84` | 3 B | between TX dwell time and key-lock flags | `❓ UNKNOWN` |
| `0x04` | `0x8B`-`0x8C` | 2 B | between SK2 Long and P1 Short — likely two more key slots | `❓ UNKNOWN` |
| `0x04` | `0x91`-`0x92` | 2 B | between P2 Long and long-press time | `❓ UNKNOWN` |
| `0x04` | `0x94`-`0x11F` | 140 B | nothing | `❓ UNKNOWN` |
| `0x04` | `0x128`-`0x1FF` | 216 B | after the 4-entry analog call table | `❓ UNKNOWN` |
| `0x04` | `0x205`-`0x22F` | 43 B | only One Touch Call **1** is mapped; slots 2-5 exist as button functions (`src/radios/dm32uv/displayOptions.ts:40-43`) and almost certainly live here | `❓ UNKNOWN` |
| `0x04` | `0x237`-`0xFFE` | 3528 B | only `Fun+0` is mapped; other Fun+N slots presumably follow | `❓ UNKNOWN` |
| `0x04` | high nibble of `0x34/0x35/0x38/0x39/0x3A/0x3B` | 6 nibbles | colour uses the low nibble only (`src/radios/dm32uv/blockLayouts.ts:26`) | `❓ UNKNOWN` |
| `0x06` Analog config | `0x2E0`-`0xA9F` | 1984 B | 7 analog contacts × 32 B end at `0x2DF`; the `"BDC"` table starts at `0xAA0` | `❓ UNKNOWN` |
| `0x06` | `0xAB0`-`0xFFE` | 1359 B | `"BDC Cotnacts 1"` seen at `0xAA0`; count and stride unknown | `❓ UNKNOWN` |
| `0x0B` Quick contacts | `0x20`-`0xFF` | 224 B | between the slot bitmask and Index Table 1 | `❓ UNKNOWN` |
| `0x0B` | `0x700`-`0x73F` | 64 B | gap between Index Table 1 and Index Table 2 | `❓ UNKNOWN` |
| `0x0B` | `0xD00`-`0xFFE` | 767 B | after Index Table 2 | `❓ UNKNOWN` |
| `0x10` Emergency/keys | `0x0A0`-`0x0AB` | 12 B | between the last digital emergency entry and the analog emergency base | `❓ UNKNOWN` |
| `0x10` | `0x0AC`-`0x2FF` | 596 B | analog emergency, 16 × 36 B — bounds are **arithmetic only** (`src/radios/dm32uv/constants.ts:118`); 16 × 36 = 576, leaving 20 B unexplained | `⚠️ DERIVED` |
| `0x10` | `0x300`-`0x45F` | 352 B | encryption keys, 8 × 44 B; base corroborated by a superseded CPS-RE note (`0x2D5 + entry × 0x2C`) | `⚠️ DERIVED` |
| `0x10` | `0x460`-`0xFFE` | 2975 B | nothing — **73% of a hardware-confirmed block is unaccounted for** | `❓ UNKNOWN` |
| `0x12` Channel bank 1 | `0x02`-`0x0F` | 14 B | header fill after the uint16 count; OEM CPS writes `0xFF`, factory image `0x00` — documented elsewhere as "reserved, all zeros", which is an assumption the write capture contradicts | `❓ UNKNOWN` |
| `0x41` VFO page | `0x0FB8`-`0x0FCE` | 23 B | tail of the VFO A channel record | `❓ UNKNOWN` |
| `0x41` | `0x0FE8`-`0x0FFE` | 23 B | tail of the VFO B channel record | `❓ UNKNOWN` |
| `0x41` | `0x0FB7` / `0x0FE7` mode-flag bits | 2 B | field located, bit meanings not annotated | `❓ UNKNOWN` |
| `0x42` TX contact low | `0xFFE` | 1 B | falls after 2047 × 2 = 4094 bytes of records | `❓ UNKNOWN` |
| `0x42`/`0x43` record | byte 0 bits 3-1 | 3 bits | labelled "Reserved" in the diagnostics reference (`src/components/diagnostics/TxContactStructureReference.tsx:108`) but never verified to be zero | `❓ UNKNOWN` |
| `0x5C`-`0x64` Zone bank | `0x000`-`0x00F` | 16 B | a 16-byte block header precedes zone 1; contents never decoded | `❓ UNKNOWN` |
| `0x65` Roam zone | whole 33-byte entry | 33 B | stride confirmed, no field mapped | `❓ UNKNOWN` |
| `0x66` Roam channel | whole 26-byte entry | 26 B | stride confirmed, no field mapped. `READPLAN.md:601-631` speculates a count at `0xFF0` — unverified | `❓ UNKNOWN` |
| Channel record | `0x1E` | 1 B | **previously documented as Squelch Level 0-255 — incorrect.** Squelch is `0x1C` bits 7-4 (`src/radios/dm32uv/structures.ts:334`). Purpose of `0x1E` now unknown; may hold the digital emergency system ID (`src/radios/dm32uv/structures.ts:332-336`, `:644-645`) | `❓ UNKNOWN` |
| Channel record | `0x20` | 1 B | **previously documented as Color Code — incorrect.** Colour code is `0x1D` bits 3-0. Labelled `Reserved (0x20)` in diagnostics (`src/components/diagnostics/ChannelParserPanel.tsx:606`) | `❓ UNKNOWN` |
| Channel record | `0x28` | 1 B | read is commented out: *"Reserved (0x28) - Unknown purpose, possibly padding or reserved for future use"* (`src/radios/dm32uv/structures.ts:442-443`) | `❓ UNKNOWN` |
| Channel record | `0x2C`-`0x2F` | 4 B | labelled Reserved; `TODO.md:259-260`: *"dump a raw channel block and verify they're truly 0x00, or identify what they control"* | `❓ UNKNOWN` |
| Channel record | `0x19` bits 1-0, `0x1A` bits 6-4/3/1-0, `0x1C` bits 1-0, `0x1D` bits 3-0 (analog), `0x25` bits 7-6/3-0, `0x26` bits 3-1/0, `0x29` bits 3-2/1-0, `0x2A` (analog) | bitfields | each carries a verbatim `Unknown`/`Reserved` comment in `src/radios/dm32uv/structures.ts` and is round-tripped unchanged | `❓ UNKNOWN` |
| Contact record (V-frame `0x0F` region) | sub-fields of the 92-byte entry | 92 B | entry size and the 16-byte region header are confirmed; the internal field order (name → ID 4 B LE → city → province → country → remark) comes from an older note with no code comment confirming the offsets | `⚠️ DERIVED` |

> **Note on the word "Reserved" inside records.** Several channel bytes above are labelled
> "Reserved" by the reference implementation. Unlike the block-level "Reserved" claims retired in
> §B.1, these at least reflect an observation that the bytes read as zero on real radios — but none
> has been verified across a populated codeplug. Treat them as `❓ UNKNOWN`, preserve them on write,
> and do not assume they are free for your own use.

---

## Corrections summary

Claims in earlier revisions of this spec that the hardware captures overturn:

| Previously documented | Correct | Where |
|---|---|---|
| The OEM CPS reads metadata at offset `0x00A` | It reads `0xFFF`, in all 200 probes. No `0x00A` read exists in either capture | §A.4 |
| `0x08 0x09 0x0C 0x0D 0x0E 0x5A 0x6A 0x6C 0x6D 0x75` are "Reserved" | All ten are allocated pages the CPS never touches → `❓ UNKNOWN` | §B.1 |
| `0x05` is "Not Found / Unused" | It is allocated (page `0x039000` on the captured radio) → `❓ UNKNOWN` | §B |
| `0x0B` is "Unknown" | Quick Access Contact List, fully mapped | §B.2, §D |
| `0x06` is "DTMF Encode Data" | Multi-section: DTMF **and** analog contacts **and** a `"BDC"` table **and** a counter at `0x1FF` | §D |
| Zone range `0x5C`-`0x64` is unverified extrapolation | Confirmed — the write pass walks all 9 slots | §B |
| Channel count is 4-byte uint32 LE | 2-byte uint16 LE — the write capture's `80 00 ff ff` = 128 matches the 128 records it then writes | §D |
| Metadata byte identifies a block *type* | It is a unique logical block **ID**; every value occurs at most once | §A.1 |

Corrections to the V-frame region table and to the `0x180000-0x200FFF` "no V-frame points here"
claim live in [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md) and
[01-OVERVIEW.md](01-OVERVIEW.md).

---

## How to reproduce this census

1. Connect and enter programming mode (see [02-CONNECTION-SEQUENCE.md](02-CONNECTION-SEQUENCE.md)).
2. Query V-frame `0x0A` for the main config range.
3. For each 4 KB-aligned page in that range, send `52 <page+0xFFF:3 LE> 01 00` and record the
   returned byte. Allow ~5 ms between probes (`src/radios/dm32uv/memory.ts:87-90`).
4. Tabulate. Every value except `0x00` and `0xFF` must appear at most once — if it appears twice,
   the FTL model in §A.1 is wrong for your firmware and that is a finding worth reporting.
5. Read the full 4 KB of any page whose ID is listed `❓ UNKNOWN` above and post the hexdump.

Reading is non-destructive. **Writing is not** — the radio reboots if it dislikes any part of a
write and there is no protocol-level retry (`IMPROVEMENTS.md:107-109`).
