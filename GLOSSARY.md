# Glossary

Quick reference for technical terms used in this documentation.

## Confidence Markers

These three markers appear throughout the specification. They are deliberately greppable. Check the
marker on a field before you write it to a radio.

### CONFIRMED *(no marker)*
The claim was observed in an OEM CPS hardware capture, or it is implemented in the reference
implementation and exercised against a real radio. An unmarked claim is a CONFIRMED claim.

Note that CONFIRMED attaches to a *claim*, not to a *feature*: a block layout can be
hardware-CONFIRMED while the reference implementation's parser for it is still a stub. Where that is
the case, both facts are stated.

### ⚠️ DERIVED
Implemented or inferred, but **never verified against hardware**. Typically this means the reference
implementation carries the value with a comment saying "approximate", "unverified", or "pending
hardware confirmation", or that the value was arrived at by arithmetic rather than observation.
Plausible; unproven.

### ❓ UNKNOWN
Purpose not established. Either nothing implements it, or the implementation explicitly says the
purpose is unknown. Preserve these bytes byte-for-byte on write; do not assume they are free for
your own use.

> "Reserved" is **not** a marker in this specification and no longer appears as a claim. Blocks and
> bytes previously labelled "Reserved" were allocated-but-unexplained, which is `❓ UNKNOWN`.
> "Reserved" implied a manufacturer intent that no capture ever supported.

## Protocol Terms

### ACK (Acknowledgment)
A single byte response (0x06) indicating successful command execution. Used throughout the protocol to confirm operations.

### BCD (Binary Coded Decimal)
A method of encoding decimal numbers where each decimal digit is represented by 4 bits (one nibble). Used for frequency encoding in the DM-32UV.

**Example**: 145.350 MHz → `00 50 53 14` (little-endian BCD)

### Big-Endian
Byte order where the most significant byte comes first. Not commonly used in DM-32UV protocol (which uses little-endian).

### CPS (Customer Programming Software)
Official software from the manufacturer used to program the radio. Also called "code plug software."

### CTCSS (Continuous Tone-Coded Squelch System)
Sub-audible tones (67-254.1 Hz) used to filter unwanted transmissions. Stored as 2 bytes in the radio.

**Example**: 127.3 Hz → `73 12`

### DCS (Digital-Coded Squelch)
Digital codes used for squelch control. Stored as 2 bytes with high byte ≥ 0x80.

**Example**: D023N → `23 80`

### Little-Endian
Byte order where the least significant byte comes first. Used throughout the DM-32UV protocol for all multi-byte values.

**Example**: Address 0x001000 → `00 10 00`

### Metadata Byte
The single byte at the end of each 4KB page (offset `+0xFFF`) that carries the page's **logical block ID**. Used for dynamic memory discovery: probe every 4KB-aligned page with `52 <page+0xFFF:3 LE> 01 00` and record what comes back.

> **Previously documented as** a byte identifying the block *type* — **incorrect**. See [Logical Block ID](#logical-block-id). Earlier revisions also claimed the OEM CPS reads this byte at offset `0x00A`; the captures contain no such read. It is always `0xFFF`.

**Non-ID values**: `0xFF` = `⚠️ DERIVED` free / erased page. `0x00` = `⚠️ DERIVED` allocated-then-invalidated page. Both mean "no live block here", but they are believed to mean different things.

### Logical Block ID
The value carried in a page's metadata byte. It identifies **which** block this is, not what kind of block it is — **every ID occurs at most once** across the entire main config region. `0x12` and `0x13` are two *different* channel bank slots, not two pages that both mean "channel".

Because IDs are unique and addresses are assigned dynamically, the ID is the stable identifier and the address is not. See [Flash Translation Layer](#flash-translation-layer-ftl).

Full census: [07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md).

### NAK (Negative Acknowledgment)
`❓ UNKNOWN` — **this radio has no known NAK byte.** `0x15` does not appear as a response byte anywhere in either capture, nor anywhere in the reference implementation. Failure is signalled by the absence of the expected magic byte for the command, not by a dedicated NAK code.

> **Previously documented as** "a response (0x15) indicating command failure or rejection" — **unsupported**.

Three speculative write-failure codes circulate (`0xC0`, `0xC8`, `0x48`); none has ever been observed — all `⚠️ DERIVED`. See [03-COMMANDS.md](03-COMMANDS.md).

### Programming Mode
Special mode the radio enters after the PROGRAM command sequence. Required before memory read/write operations.

### V-Frame
Binary command format starting with 0x56 ('V') used to query radio information (firmware version, memory layout, region pointers, etc.). Sent before entering programming mode.

**Request** (5 bytes): `56 00 00 <b3> <ID>` — byte 3 is `0x00` for every V-frame in both captures except one: the OEM CPS opens each session by querying `0x0D` as `56 00 00 40 0D`, and that form returns 64 bytes where the plain `56 00 00 00 0D` returns 0. `❓ UNKNOWN` — what byte 3 means is not established; naming it a length or a high byte would be a guess.

**Response**: `56 <ID> <len> <payload…>` — payload begins at offset 3.

Memory-pointer V-frames (`0x06 0x07 0x08 0x09 0x0A 0x0E 0x0F`) return **two little-endian uint32s**: region start followed by region end. Decoding these as anything else produces byte-shifted garbage, which is exactly what earlier revisions of this spec published for `0x06 0x08 0x09 0x0E 0x0F`. Correct values are in [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md).

## Memory Terms

### 4KB Block / Page
The basic unit of memory organization in the DM-32UV. Reads and writes of block data are 4096 bytes at a 4KB-aligned address (0x001000, 0x002000, …). This document uses **page** for the 4 KB physical slot and **block** for the logical contents it carries; the two are separable, which is the whole point of [Logical Block ID](#logical-block-id).

The one observed non-aligned address is the [sentinel](#sentinel-address-0xfff001).

### Metadata Block
A 4 KB page whose last byte (`+0xFFF`) carries a logical block ID — i.e. a live page in the main config region. The main config region holds **200 pages**; on the captured factory radio **71** were live, 15 were tagged `0x00` and 114 were tagged `0xFF`.

### Block Inventory
The census of every logical block ID and every page in the main config region, together with an explicit count of what remains unidentified. Lives in [07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md). It is the document that answers "is this block known, and how well?"; [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md) answers "what are its bytes?".

### Flash Translation Layer (FTL)
The behaviour, inferred from the captures, by which the radio assigns a physical page address to a logical block ID rather than fixing it at manufacture. A block's address changes between radios and after edits; its logical ID does not.

`⚠️ DERIVED` — the FTL is a model that explains the observations (unique IDs, scattered `0x00` pages among live ones, a contiguous high run of `0xFF` pages, wear-levelled copy-on-write). It is not a documented manufacturer mechanism.

**Practical consequence**: never hardcode an address. Discover it.

### Sentinel Address (0xFFF001)
The address `0xFFF001` — the only non-4KB-aligned address either capture ever uses. The OEM CPS addresses it whenever a logical block ID has **no physical page** on this radio, and it does so in *both* directions, 36 times each:

| Session | Frame | Payload |
|---|---|---|
| Write | `57 01 F0 FF 00 10` + 4096 B | all `0xFF`, with the logical block ID in the last byte |
| Read | `52 01 F0 FF 00 10` → `57 01 F0 FF 00 10` + 4096 B | all `0x00` except `+0xFFF`, which is `0x48` — byte-for-byte identical on all 36 reads |

36 is exactly the number of logical IDs the CPS declares but that have no page on this factory radio, which is how the write capture proves the full logical-ID space: the CPS emits a transfer for *every* slot it knows about, populated or not.

`❓ UNKNOWN` — the exact semantics (free? deallocate? declare-capacity? "null page" handle?) are not established. The constant `0x48` in the read reply is also `❓ UNKNOWN`; it coincides with the highest talk-group slot ID, which may be meaningful or may be a stale buffer. No implementation performs either transfer.

> An earlier revision described this as a write-session-only behaviour. The read capture performs the same 36 transfers.

### Channel Bank
The 48 logical block IDs `0x12`-`0x41`, which together hold up to 4,000 channel definitions. **48 distinct slots**, each a separate logical block — not 48 pages sharing one type tag, and not necessarily at consecutive addresses.

Confirmed by the write capture, which walks all 48 slots contiguously.

### TX Contact Block
Logical blocks `0x42` and `0x43`, holding the per-channel TX contact (talk group) assignment as a **2-byte record per channel**. `0x42` covers the low channel range, `0x43` covers the high range plus the two VFO entries.

This data is *not* in the 48-byte channel record — the channel record's own contact field always parses as zero. Layout in [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md).

### Talk Group Block
Logical blocks `0x44`-`0x48` (5 slots, confirmed by the write capture), holding the talk group / DMR contact definitions that channels point at via their [TX contact block](#tx-contact-block) entry.

Display order and liveness are driven by an index table in block `0x0B`, the [Quick Access Contact List](#quick-access-contact-list) — the talk group block alone does not tell you which entries are active.

### Quick Access Contact List
Logical block `0x0B`: counts, a 128-slot usage bitmask, and two index tables (one sorted by name, one by DMR ID) over the talk group entries. Adding a talk group requires updating this block as well as `0x44`.

> **Previously documented as** "RX Group List", then as "Unknown" — both **incorrect**. RX Groups are block `0x0F`.

### Zone Bank
The 9 logical block IDs `0x5C`-`0x64`. Confirmed by the write capture, which walks all 9 slots. This retires the reference implementation's own comment that the zone range end was *"unverified, pending hardware confirmation"*.

### Main Config Region
Memory region `0x001000-0x0C8FFF` (819,200 bytes = 200 pages of 4 KB) containing channels, zones, scan lists, settings and other configuration data. Discovered at runtime via V-frame `0x0A`. Previously called the "Main Config Block", which invited confusion with a 4 KB block.

### Memory Map
Organization of data in the radio's address space. The DM-32UV uses a 16MB address space (0x000000-0xFFFFFF), addressed with **3-byte little-endian** addresses.

### Offset
Position within a block or structure. Often specified in hexadecimal (e.g., offset 0x10 = 16 bytes from start). Unless stated otherwise, offsets in this documentation are **block-relative** — measured from the start of the 4 KB page, not from an absolute address.

### Stride
The byte distance between the start of one entry and the start of the next within a block. Strides measured from hardware captures generalise across radios; the *number* of entries observed in a capture does not, because the captured radio was at factory defaults.

## Data Structure Terms

### Channel
48-byte structure containing the settings for one radio channel (name, frequencies, mode, power, etc.). Lives in the [channel bank](#channel-bank), blocks `0x12`-`0x41`, stride 48 bytes.

The channel's TX contact is **not** in this record — see [TX Contact Block](#tx-contact-block). About 7 byte positions inside the 48-byte record remain `❓ UNKNOWN`.

### Code Plug
Complete radio configuration including channels, zones, contacts, and settings. Stored in radio memory. In this documentation, the codeplug is specifically the 40 logical blocks the OEM CPS reads and/or writes — the remaining live blocks appear to be firmware/factory/bootloader-owned.

### Contact
DMR contact entry (call sign, DMR ID, type). **92 bytes** (`0x5C`). Stored in its own memory region, discovered via V-frame `0x0F` — separate from both the channel banks and the [talk group blocks](#talk-group-block). The region opens with a 16-byte header (4-byte LE count + padding); contacts do not straddle 4KB boundaries.

Maximum count is reported by the radio via V-frame `0x10` (50,000 on the captured radio).

`⚠️ DERIVED`: whether the 92 bytes are **fixed-offset slots** or **packed variable-length fields**. The fixed-offset reading is what the reference implementation reads and writes and it round-trips against hardware, so implement that. The DMR ID at `+0x10` is **uint24 LE** with an `❓ UNKNOWN` byte at `+0x13` — see [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md).

> **Previously documented as** 109 bytes — **incorrect**. 109 is the RX group size; it was copied into the contacts section by mistake.

### Talk Group
A DMR group call identifier plus its name, stored in the [talk group blocks](#talk-group-block) `0x44`-`0x48`. Channels select one via their [TX contact block](#tx-contact-block) entry. Distinct from a [Contact](#contact), which lives in a different memory region entirely.

### DMR Radio ID
An entry in the radio's own list of DMR IDs it may transmit as. 16 bytes per entry, block `0x67`; the count at offset `0x00` is **1 byte** (settled by the OEM write capture's `01 FF FF FF` header). Channels reference one by index.

### RX Group (Receive Group)
List of DMR contacts the radio will receive on a channel. Also called "RX Group List". **109 bytes** per entry, block `0x0F`; a 17-byte block header (uint32 LE active-group bitmask + 12 unknown bytes + a `0x01` flag) precedes the first entry, so entry N sits at `0x11 + N * 0x6D` (N 0-based). Maximum 32 groups `⚠️ DERIVED` — inferred from the width of that 32-bit bitmask, never observed; the captured factory radio's mask was `1F 00 00 00`, i.e. **5** groups.

> ⚠️ Do not read the byte at `0x000` as a plain count. It is the low byte of the bitmask: `0x1F` means "slots 0-4 in use" (5 groups), not "31 groups". An intermediate analysis of the read capture made exactly that mistake.

> **Previously documented as** block `0x0B` — **incorrect**; `0x0B` is the [Quick Access Contact List](#quick-access-contact-list). A separate stale section documenting `0x0F` as "TX Contact Assignment" was also incorrect.

### Scan List
List of channels to scan. **57 bytes** per entry, block `0x11`; count byte at block offset `0x00`, entry N at `(57 * N) - 56`. The channel list is **15 slots of 2 bytes at `+0x1A`…`+0x37`**; the word at `+0x18`-`+0x19` is a separate `❓ UNKNOWN` field to preserve verbatim (the OEM write capture's nine scan lists all store `00 00` there followed by exactly the channels their names imply). Maximum 32 lists `⚠️ DERIVED` — a reference-implementation/UI limit; 71 entries would physically fit in one 4 KB page.

> Whether `+0x18` is instead the first channel slot (16 slots total) is **unresolved** — full evidence table in [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md#scan-list-structure-57-bytes). *Previously documented as 92-byte entries — incorrect; the 57-byte record is hardware-confirmed.*

### Zone
Collection of channels grouped together for easy access. **145 bytes** per zone, in the [zone banks](#zone-bank) `0x5C`-`0x64`. Up to 64 channels per zone, ~28 zones per page, 250 zones maximum `⚠️ DERIVED` (9 pages × 28 = 252 slots; the 250 figure comes from the reference implementation, not from a capture). Zone name is 11 bytes.

In the **first** zone page zones start at block offset 16, after a 16-byte header whose first byte is the *global* zone count across all zone pages (hardware-confirmed) and whose remaining 15 bytes are `❓ UNKNOWN` — copy them through verbatim on write. Later zone pages have **no header**; records start at offset `0x000`.

> **Previously documented as** a 57-byte structure holding up to 23 channels — **incorrect**. An intermediate revision of this glossary also described the whole 16-byte header as unknown; byte `0x000` of it is known.

### Quick Message
Pre-canned SMS text. **129 bytes** (`0x81`) per entry, block `0x0A`; count at block offset `0x00`, 16-byte header, entries start at `0x10`. 20 maximum.

### Roaming Zone / Roaming Channel
Blocks `0x65` (stride 33 bytes) and `0x66` (stride 26 bytes). Strides and name offsets are hardware-confirmed from the read capture; the remaining field layouts are `⚠️ DERIVED` from CPS decompilation, and the reference implementation does not implement the feature at all.

### VFO
The radio's two variable-frequency-oscillator entries, stored as ordinary 48-byte channel records inside block `0x41` (VFO A at block offset `0x0F9F`, VFO B at `0x0FCF`). Treated as pseudo-channels 4001 and 4002. Their TX contact entries live at block-relative `0x0FFA` / `0x0FFC` in block `0x43`.

### Boot Image
The 240 × 320 startup splash screen. 153,600 bytes (2 bytes per pixel), no header. Transferred as 37 full 4KB blocks plus one 2048-byte chunk. Its base address comes from V-frame `0x0E`, which returned `0x150000-0x175FFF` on the captured radio.

`⚠️ DERIVED`: the pixel channel order (described as both BGR565 and RGB565 in different places; unverified). `⚠️ DERIVED`: the *purpose* of V-frame `0x0E` — the OEM CPS never touches that region in either capture.

## Radio Terms

### Color Code
DMR-specific setting (0-15) used to filter transmissions. Similar to CTCSS for analog.

### DMR (Digital Mobile Radio)
Digital radio standard (ETSI TS 102 361) used by the DM-32UV for digital mode operation.

### Time Slot
DMR feature allowing two conversations on one frequency. Values: 1 or 2.

### Talkgroup
DMR group call identifier. Allows multiple radios to communicate on the same channel. For how the DM-32UV stores them, see [Talk Group](#talk-group) and [Talk Group Block](#talk-group-block).

### TX/RX Frequency
Transmit and receive frequencies. May be the same (simplex) or different (duplex/repeater).

## Encoding Terms

### ASCII
Text encoding where each character is one byte. Used for channel names, firmware versions, etc.

### Hex (Hexadecimal)
Base-16 number system (0-9, A-F). Used throughout documentation to represent binary data.

**Example**: 0xFF = 255 decimal = 11111111 binary

### Nibble
4 bits (half a byte). Used in BCD encoding where each nibble represents one decimal digit.

### Null-Terminated String
Text string ending with 0x00 byte. Used for channel names and zone names.

### Padding
Unused bytes filled with a specific value (usually 0xFF or 0x00) to reach a required size.

## Command Terms

### Handshake
Initial connection sequence (PSEARCH, PASSSTA, SYSINFO) required before other operations.

### Read Command (0x52)
Command to read data from radio memory. Request (6 bytes): `52 <addr:3 LE> <len:2 LE>`.

The radio answers a `0x52` read with a **`0x57`-headed** frame that echoes the request header: `57 <addr:3 LE> <len:2 LE> <data…>`. Payload therefore begins at **offset 6**.

### Write Command (0x57)
Command to write data to radio memory. Frame (4102 bytes): `57 <addr:3 LE> 00 10 <data:4096>`. Length is always `0x1000`. ACK is a single `0x06`.

The logical block ID is **inside the payload** at `data[0xFFF]` — i.e. the last byte of the 4096 — not appended after it.

> **Previously documented as** `57 <addr:3> 00 10 <data:4096> <metadata:1>`, total 4103 bytes — **incorrect**. The reference implementation's own docstring carries the same error; its code does not.

### Timeout
Maximum time to wait for a response. **5000 ms** per request/response cycle — handshake, V-frame query, read header, write ACK — and **15000 ms** for the *payload* of a 4 KB read. Pacing delays are separate: ~150 ms between block reads, ~5 ms between metadata probes, 50 ms between V-frame queries.

> **Previously documented as** "typically 500 ms for reads" — **incorrect**. No 500 ms timeout exists anywhere in the protocol or the reference implementation. See [02-CONNECTION-SEQUENCE.md](02-CONNECTION-SEQUENCE.md#timing-and-timeout-reference).

## Abbreviations

| Term | Meaning |
|------|---------|
| ACK | Acknowledgment |
| BCD | Binary Coded Decimal |
| CPS | Customer Programming Software |
| CTCSS | Continuous Tone-Coded Squelch System |
| DCS | Digital-Coded Squelch |
| DMR | Digital Mobile Radio |
| FTL | Flash Translation Layer |
| KB | Kilobyte (1024 bytes) |
| LE | Little-Endian |
| LSB | Least Significant Byte |
| MB | Megabyte (1024 KB) |
| MSB | Most Significant Byte |
| NAK | Negative Acknowledgment |
| RX | Receive |
| TG | Talk Group |
| TX | Transmit |
| VFO | Variable Frequency Oscillator |

## See Also

- **[01-OVERVIEW.md](01-OVERVIEW.md)** - Protocol architecture overview
- **[04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)** - Memory organization details
- **[05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md)** - Byte-level record layouts
- **[06-ENCODING.md](06-ENCODING.md)** - Encoding algorithms and examples
- **[07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md)** - Logical block ID census and the unknown scoreboard

---

**Note**: This glossary covers terms specific to the DM-32UV protocol. For general radio or DMR terms, consult amateur radio references or DMR standards documentation.

**Reference implementation**: definitions here that describe how a value is stored or parsed are validated against [neonplug](https://neonplug.app) (`src/radios/dm32uv/`). Definitions that describe what a real radio actually did on the wire come from the two OEM CPS captures in this repository, which outrank the implementation where they disagree.
