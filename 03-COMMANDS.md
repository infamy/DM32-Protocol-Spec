# DM-32UV Command Reference

Complete reference for all DM-32UV protocol commands with examples.

## Confidence markers used in this document

| Marker | Meaning |
|--------|---------|
| *(no marker)* | CONFIRMED — seen on the wire in an OEM CPS serial capture against real hardware, and/or implemented in the working reference implementation |
| `⚠️ DERIVED` | Implemented or inferred, but not verified against hardware |
| `❓ UNKNOWN` | Purpose not established |

## Evidence base

Everything in this document was re-verified in 2026 against three sources, in this order of authority:

1. **`serial_capture_example.txt`** — full OEM CPS *read* session, DP570UV / firmware `DM32.01.01.046` / DMR CPS V1.41. 602 logged transfers.
2. **`serial_capture_write_example.txt`** — full OEM CPS *write* session, same radio and CPS. 598 logged transfers, including 76 real 4 KB writes.
3. **neonplug** — a working open-source implementation that talks to real DM-32UV hardware.

Where a claim below is marked as corrected, the correction came from machine-parsing the two
captures. Capture timestamps have **1-second resolution**, so the captures cannot confirm any
host-side inter-command delay; all timing figures in this document therefore come from neonplug
and are labelled as such.

**Implementation reference:** `src/radios/dm32uv/connection.ts` (all framing),
`src/radios/dm32uv/constants.ts` (timing/timeouts), `src/radios/dm32uv/protocol.ts` (V-frame decode,
block loops).

## Command Categories

1. **Handshake Commands** (ASCII) - Initial connection
2. **V-Frame Commands** (Binary) - System information queries
3. **Programming Mode Commands** (Binary) - Enter programming mode
4. **Memory Read Commands** (Binary) - Read data from radio
5. **Memory Write Commands** (Binary) - Write data to radio

Both captures use **exactly the same** command set and ordering up to the end of programming-mode
entry; they diverge only afterwards (read session issues `0x52` block reads, write session issues
`0x57` block writes).

---

## 1. Handshake Commands

> **💡 Real Examples**: See `serial_capture_example.txt` (read session) and
> `serial_capture_write_example.txt` (write session) for actual command/response sequences from
> the official CPS software. The handshake is byte-identical in both.

**Implementation reference:** `src/radios/dm32uv/connection.ts`

### PSEARCH - Identify Radio

**Format**: ASCII string  
**Length**: 7 bytes

```
Command:  50 53 45 41 52 43 48
          P  S  E  A  R  C  H

Response: 06 44 50 35 37 30 55 56
          ACK D  P  5  7  0  U  V
```

**Purpose**: Verify radio model (DP570UV = DM-32UV)

**Response is exactly 8 bytes**: `0x06` ACK followed by 7 ASCII model bytes. There is no
terminator and no length prefix on either direction.

**Model validation** — the reference implementation accepts the response if the ASCII part
(NULs stripped, trimmed) contains `DP570`, `DM32`, or `DM-32`.

**Timing (from the reference implementation, not from the capture):** wait **150 ms** after
sending `PSEARCH` before reading, then read 8 bytes with a **5000 ms** timeout.

- No retry logic exists anywhere; the captures show `PSEARCH` sent exactly once per session.

**Serial Capture Reference:** first `Written data` block in `serial_capture_example.txt`
(and identically in `serial_capture_write_example.txt`)

---

### PASSSTA - Get Status

**Format**: ASCII string  
**Length**: 7 bytes

```
Command:  50 41 53 53 53 54 41
          P  A  S  S  S  T  A

Response: 50 XX XX
          P  (2 further bytes)
```

**Status Values**: `50 00 00` is the only value observed, in both captures. (`50 FF FF` was
previously documented as a valid variant — unevidenced.)

**Bytes 1 and 2 are ❓ UNKNOWN.** The reference implementation reads all 3 bytes but validates
only byte 0 (`must be 0x50`, ASCII `'P'`).

---

### SYSINFO - Request System Info

**Format**: ASCII string  
**Length**: 7 bytes

```
Command:  53 59 53 49 4E 46 4F
          S  Y  S  I  N  F  O

Response: 06
          ACK
```

---

## 2. V-Frame Commands

**Implementation reference:** `src/radios/dm32uv/connection.ts` (framing),
`src/radios/dm32uv/constants.ts` (frame ID constants), `src/radios/dm32uv/protocol.ts` (payload
decode), `src/utils/firmware.ts` (firmware-string interpretation),
`src/utils/bootImage.ts` (V-frame `0x0E` consumer)

### Command Format

**Always 5 bytes.**

```
Byte 0:   0x56 ('V')
Byte 1:   0x00        ❓ UNKNOWN (always 0x00 in both captures and in neonplug)
Byte 2:   0x00        ❓ UNKNOWN (always 0x00 in both captures and in neonplug)
Byte 3:   Requested payload length   ⚠️ DERIVED — 0x00 in every request except V-frame 0x0D
Byte 4:   Frame ID
```

Byte 3 is `0x00` for 15 of the 16 V-frame requests the OEM CPS issues. The one exception is the
CPS's **first** V-frame request of the session, `56 00 00 40 0D`, which returns a **64-byte**
(`0x40`) payload. The same frame ID queried later in the same session as `56 00 00 00 0D` returns
**0 bytes**. Because `0x40` in byte 3 correlates exactly with the 64-byte reply, byte 3 is
**⚠️ DERIVED** to be a requested/maximum payload length. It is also consistent with byte 3 being
the high byte of a 16-bit field at bytes 2-3; the captures cannot distinguish these.

The reference implementation always hardcodes `0x00` in bytes 1-3, so it never retrieves the
64-byte `0x0D` payload.

### Response Format

```
Byte 0:   0x56 ('V')            — validated
Byte 1:   Frame ID (echoed)     — validated, must equal the requested ID
Byte 2:   Payload length (N)    — single byte, so a payload can never exceed 255 bytes
Bytes 3+: Payload (N bytes)
```

`N == 0` is legal and means "no payload" (observed for `56 00 00 00 0D`).

### Query order used by the OEM CPS

Both captures issue exactly 16 V-frame requests, in this order, immediately after `SYSINFO` and
**before** entering programming mode:

```
0x0D(len 0x40)  0x01  0x02  0x03  0x04  0x05  0x06  0x07
0x08  0x09  0x0A  0x0B  0x0D(len 0x00)  0x0E  0x0F  0x10
```

**`0x0C` is never queried — not by the OEM CPS, and not by neonplug.** Its purpose is
❓ UNKNOWN. Note that `0x0D` is queried **twice**, with different byte-3 values.

neonplug queries the same set minus `0x0C` and minus the `0x40`-length form, in plain ascending
order: `0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0A 0x0B 0x0D 0x0E 0x0F 0x10`. A failing
frame is logged and skipped rather than aborting the connection.

### V-Frame Reference Table

The "Payload from hardware" column is **verbatim from the serial captures** — both captures
returned byte-identical V-frame payloads on this radio and firmware. "Len" is the payload length
this radio reported, not a fixed field width.

| ID | Len | Purpose | Format | Payload from hardware | Confidence |
|----|-----|---------|--------|-----------------------|------------|
| 0x01 | 14 | Firmware version | ASCII | `"DM32.01.01.046"` | |
| 0x02 | 12 | ❓ UNKNOWN | Binary | `00 00 00 00 00 00 15 A4 00 00 15 A4` | ❓ UNKNOWN — two repeated `0xA415` words; never consumed by neonplug |
| 0x03 | 10 | Build date | ASCII | `"2022-06-27"` | |
| 0x04 | 12 | DSP version | ASCII | `"D1.01.01.004"` | |
| 0x05 | 12 | Radio version | ASCII | `"R1.00.01.001"` | |
| 0x06 | 8 | ❓ UNKNOWN range (previously labelled "Audio resource index") | Memory range | `00 10 20 00 FF 4F 26 00` = **0x201000–0x264FFF** | Range CONFIRMED; label ❓ UNKNOWN |
| 0x07 | 8 | ❓ UNKNOWN range (previously labelled "Compact item table") | Memory range | `00 90 0C 00 FF 9F 14 00` = **0x0C9000–0x149FFF** | Range CONFIRMED; label ❓ UNKNOWN |
| 0x08 | 8 | ❓ UNKNOWN range (previously labelled "Zones") | Memory range | `00 00 18 00 FF 0F 20 00` = **0x180000–0x200FFF** | Range CONFIRMED; label ❓ UNKNOWN |
| 0x09 | 8 | ❓ UNKNOWN range (previously labelled "Emergency/recording") | Memory range | `00 C0 6D 00 FF FF FF 00` = **0x6DC000–0xFFFFFF** | Range CONFIRMED; label ❓ UNKNOWN |
| 0x0A | 8 | **Main config block** (the codeplug) | Memory range | `00 10 00 00 FF 8F 0C 00` = **0x001000–0x0C8FFF** | |
| 0x0B | 12 | Codeplug version | ASCII | `"C1.00.01.001"` | |
| 0x0C | — | ❓ UNKNOWN | — | *never queried by anything* | ❓ UNKNOWN |
| 0x0D | 64 or 0 | ❓ UNKNOWN (previously labelled "Capabilities") | Binary | see below | ❓ UNKNOWN |
| 0x0E | 8 | Boot/startup image region `⚠️ DERIVED` | Memory range | `00 00 15 00 FF 5F 17 00` = **0x150000–0x175FFF** | Range CONFIRMED; *purpose* `⚠️ DERIVED` — see note |
| 0x0F | 8 | Contacts region | Memory range | `00 80 27 00 FF BF 6D 00` = **0x278000–0x6DBFFF** | |
| 0x10 | 3 | Max contact count | uint24 LE | `50 C3 00` = **50000** | |

#### ❗ Correction: five memory-range start addresses were previously decoded wrong

The previous version of this table applied a byte-shift when decoding the little-endian start
addresses. Decoded correctly (`start = uint32 LE at payload[0..3]`):

| ID | Previously documented start | Correct start | Correct range | Size |
|----|-----------------------------|---------------|---------------|------|
| 0x06 | `0x001020` ❌ | `0x201000` | 0x201000–0x264FFF | 409,600 B (400 KiB, 100 × 4 KB) |
| 0x07 | `0x000C9000` ✅ | `0x0C9000` | 0x0C9000–0x149FFF | 528,384 B (516 KiB, 129 × 4 KB) |
| 0x08 | `0x00001800` ❌ | `0x180000` | 0x180000–0x200FFF | 528,384 B (516 KiB, 129 × 4 KB) |
| 0x09 | `0x00006DC0` ❌ | `0x6DC000` | 0x6DC000–0xFFFFFF | 9,584,640 B (9360 KiB) |
| 0x0A | `0x001000` ✅ | `0x001000` | 0x001000–0x0C8FFF | 819,200 B (800 KiB, 200 × 4 KB) |
| 0x0E | `0x00001500` ❌ | `0x150000` | 0x150000–0x175FFF | 155,648 B (152 KiB, 38 × 4 KB) |
| 0x0F | `0x00002780` ❌ | `0x278000` | 0x278000–0x6DBFFF | 4,603,904 B (4496 KiB) |

Two label corrections follow from this:

- **0x08 is not "Zones".** Zones live in the main config block (`0x0A`) as logical block IDs
  `0x5C`–`0x64`. The only code reference to `0x08` as a zone range in neonplug is a
  **commented-out line** that was never implemented. `0x08` points at `0x180000–0x200FFF`, whose
  contents are ❓ UNKNOWN. *`01-OVERVIEW.md` previously stated that no V-frame points at
  `0x180000–0x200FFF` — incorrect; V-frame `0x08` points at exactly that range.*
- **0x0E is the boot/startup image region `⚠️ DERIVED`, not "Memberships/lists".** `payload[0..3]`
  is consumed as the boot-image base address, and 38 × 4 KB holds the 153,600-byte image. (The
  "Memberships" label was a misnomer — RX groups live in the main config block as logical ID
  `0x0F`.) The range is CONFIRMED; the *purpose* stays `⚠️ DERIVED` because neither capture ever
  accesses `0x150000–0x175FFF`.

Also note V-frame `0x09`'s end address is `0x00FFFFFF`, i.e. the top of the 24-bit address space.
*Previously documented as "disabled if 0xFFFFFF" — unsupported.* Nothing reads this frame.

### V-Frame Examples

#### Get Firmware Version (0x01)

**Command**: `56 00 00 00 01`  
**Response**: `56 01 0E 44 4D 33 32 2E 30 31 2E 30 31 2E 30 34 36` (17 bytes: 3 header + 14 payload)  
**Parse**: `"DM32.01.01.046"`

Version strings are decoded as: ASCII/UTF-8 decode → strip **all** NUL bytes → trim whitespace.

#### Get Main Config Block (0x0A - CRITICAL)

**Command**: `56 00 00 00 0A`  
**Response**: `56 0A 08 00 10 00 00 FF 8F 0C 00`  
**Parse**:
- Payload bytes 0-3: start address, uint32 **little-endian** → `0x001000`
- Payload bytes 4-7: end address, uint32 **little-endian** → `0x0C8FFF`

`end_addr` is the **last byte of the last block**, not one-past-the-end. Block enumeration is
therefore:

```
alignedEnd = floor(end / 0x1000) * 0x1000        # 0x0C8000
blockCount = (alignedEnd - start) / 0x1000 + 1   # 200
```

This is the only V-frame neonplug treats as mandatory — missing or shorter than 8 bytes throws
`Failed to get memory layout`.

#### Memory-range V-frames (0x06, 0x07, 0x08, 0x09, 0x0A, 0x0E, 0x0F)

All seven return 8 bytes in the identical format: `start_addr` uint32 LE, then `end_addr`
uint32 LE. Canonical decoder:

```
value = b[i] | (b[i+1] << 8) | (b[i+2] << 16) | (b[i+3] << 24)
```

**V-Frame 0x06**
- Command: `56 00 00 00 06`
- Response: `56 06 08 00 10 20 00 FF 4F 26 00`
- Parse: Start = `0x201000`, End = `0x264FFF`

**V-Frame 0x08**
- Command: `56 00 00 00 08`
- Response: `56 08 08 00 00 18 00 FF 0F 20 00`
- Parse: Start = `0x180000`, End = `0x200FFF`

**V-Frame 0x0F: Contacts**
- Command: `56 00 00 00 0F`
- Response: `56 0F 08 00 80 27 00 FF BF 6D 00`
- Parse: Start = `0x278000`, End = `0x6DBFFF`
- A pair of `0x00000000 / 0x00000000` is treated as "contacts disabled". `⚠️ DERIVED` — handled
  in code, never observed on hardware.
- `⚠️ DERIVED`: the range is expected to be larger on L01 firmware. Only one firmware
  (`DM32.01.01.046`) has been captured, so this is inference, not observation.
  ❓ UNKNOWN: *previously documented as "End address varies by firmware (may be `0x00FFFFFF` for
  extended range)".* Retained here so the claim is not lost, but nothing corroborates the
  `0x00FFFFFF` figure — no L01 radio has been captured and no implementation contains it. (Note
  `0xFFFFFF` **is** the observed end address of V-frame `0x09`, so the older claim may simply be
  a transcription of the wrong frame.)
- Contact capacity in practice keys off the firmware string (`L01` → 150,000, otherwise 50,000)
  and V-frame `0x10`, not this range.

#### V-Frame 0x10: Max Contact Count

**Command**: `56 00 00 00 10`  
**Response**: `56 10 03 50 C3 00`  
**Parse**: the payload is **3 bytes**, i.e. a 24-bit little-endian integer:
`0x50 | (0xC3 << 8) | (0x00 << 16)` = `0x00C350` = **50000**.

Read the count as a little-endian integer over however many bytes the length byte reports —
do not require a 4-byte payload.

#### Get Version Strings (0x01, 0x03, 0x04, 0x05, 0x0B)

All version V-frames return ASCII strings. Lengths below are what this radio returned; they are
string lengths, not fixed field widths.

- **0x01**: Firmware version (14 bytes here: `"DM32.01.01.046"`. `⚠️ DERIVED`: the L01 variant is reported elsewhere as e.g. `"DM32.01.L01.048"` / `"DM32.01.L01.050"`, 15 bytes — no L01 radio has been captured)
- **0x03**: Build date (10 bytes, `"2022-06-27"`)
- **0x04**: DSP version (12 bytes, `"D1.01.01.004"`)
- **0x05**: Radio version (12 bytes, `"R1.00.01.001"`)
- **0x0B**: Codeplug version (12 bytes, `"C1.00.01.001"`)

The firmware string is load-bearing: an `L01` substring selects the 150,000-contact capacity
instead of 50,000, and a trailing numeric segment `>= 49` (e.g. `DM32.01.01.049`,
`DM32.01.L01.050`) gates firmware-dependent behaviour.

#### V-Frame 0x0D — ❓ UNKNOWN (previously labelled "Capabilities")

**Two request forms are observed, in the same session:**

```
56 00 00 40 0D   →  56 0D 40 <64 payload bytes>     (first V-frame request of the session)
56 00 00 00 0D   →  56 0D 00                        (0 payload bytes)
```

The 64-byte payload, verbatim from hardware, with corrected offsets:

```
+0x00: 03 4E 2D 00 00 00 00 00 00 00 00 00 00 00 00 00
+0x10: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
+0x20: 3F 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
+0x30: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

Only **four** bytes are non-zero: `+0x00 = 0x03`, `+0x01 = 0x4E`, `+0x02 = 0x2D`, `+0x20 = 0x3F`.

**Status**: ❓ UNKNOWN. No consumer exists in any implementation. Note the response length is a
function of the **request** (the same firmware returned 64 bytes and 0 bytes to the two forms), so
it does not signal firmware capability.

**Recommendation**: harmless to query; do not rely on it. To retrieve the payload you must send
the `0x40` byte-3 form.

---

## 3. Programming Mode Commands

**Implementation reference:** `src/radios/dm32uv/connection.ts`

Programming mode is entered **after** all V-frames have been queried. The OEM CPS additionally
issues one `0x47` command (see §4.3) immediately before the PROGRAM sequence.

### Enter Programming Mode Sequence

This is a **3-step sequence** that must complete successfully. Observed identically in both
captures.

#### Step 1: PROGRAM Command

```
Command:  FF FF FF FF 0C 50 52 4F 47 52 41 4D      (12 bytes)
          (header)    P  R  O  G  R  A  M

Response: 06 (ACK)
```

The `0x0C` byte is ❓ UNKNOWN. It appears here and in the boot-image mode sequence; nothing
documents or derives its meaning. It is *not* a length byte for `PROGRAM` (which is 7 bytes).

#### Step 2: Mode 02

```
Command:  02                                        (1 byte)

Response: FF FF FF FF FF FF FF FF                   (8 bytes)
```

The reference implementation requires **all 8 bytes to be `0xFF`**, else `Mode 02 failed`.
The meaning of the 8-byte reply is ❓ UNKNOWN.

#### Step 3: Confirm with ACK

```
Command:  06                                        (1 byte)

Response: 06 (ACK)
```

**Timing (reference implementation):** 10 ms between each step. Each response is read with the
default 5000 ms timeout.

### Exit Programming Mode — there is no such command

**No exit command exists.** In both captures the last wire event is an ordinary data transfer,
immediately followed by `Close port` — the OEM CPS simply closes the port, and neonplug sends
nothing on disconnect. (*An `FF FF FF FF 0C "END"` exit frame was previously documented here —
never observed in any capture; do not implement it.*)

**How the radio actually leaves programming mode:** closing the serial port DTR-resets the radio,
which starts exiting programming mode. This matters operationally — if the port is left open the
radio stays in programming mode and **will not answer `PSEARCH` on the next connect attempt**;
that consequence is stated as fact in the reference implementation's own disconnect comment.

`⚠️ DERIVED`: after closing, wait **400 ms** (`REOPEN_DELAY`) before reopening. The reference
implementation applies this delay on every reopen; its port-selection code states the failure mode
verbatim: *"Without this delay, PSEARCH silently times out on retry even though the first attempt
worked fine (radio not yet ready after coming out of programming mode)."* No capture covers a
reconnect, so the symptom rests on that implementation experience alone.

---

## 4. Memory Read Commands

**Implementation reference:** `src/radios/dm32uv/connection.ts` (framing),
`src/radios/dm32uv/memory.ts` (metadata scan and block helpers),
`src/radios/dm32uv/protocol.ts` (bulk read loops)

### 4.1 Single Read Command (0x52)

**Request — always exactly 6 bytes:**
```
Byte 0:    0x52 ('R')
Bytes 1-3: Address (24-bit, little-endian)
Bytes 4-5: Length (16-bit, little-endian)
```

**Response — 6-byte header, then payload:**
```
Byte 0:    0x57 ('W')                      — validated
Bytes 1-3: Address (echoed, little-endian) — NOT validated by neonplug
Bytes 4-5: Length (little-endian)          — validated: must be > 0 and <= requested
Bytes 6+:  Data (length bytes)
```

**There is no host-side ACK in the read path.** The host sends 6 bytes and reads
header + payload; nothing is written back.

**Implementation Notes:**
- Address encoding: 24-bit little-endian (LSB first).
  Address `0x001000` → `00 10 00` (not `00 00 10`).
- Length encoding: 16-bit little-endian.
  Length 4096 (`0x1000`) → `00 10` (not `10 00`).
- A **short** response (`responseLength < requested`) is accepted by the reference
  implementation; only `0` or over-length is rejected.
- Payload is read with a **15000 ms** timeout (4 KB blocks can be slow); the 6-byte header uses
  the default 5000 ms.

**Real Examples from Serial Capture:**

Example 1 — metadata probe, 1 byte at `0x001FFF`. This is the very first metadata probe of the
read session:
```
Command:  52 FF 1F 00 01 00
          R  [addr=0x001FFF] [len=1]

Response: 57 FF 1F 00 01 00 07
          W  [addr=0x001FFF] [len=1] [data=0x07]
```

The byte at `blockBase + 0xFFF` is the block's **logical block ID**. The OEM CPS probes
**all 200 blocks** of the main config region this way — from `0x001FFF` through `0x0C8FFF`,
stepping by `0x1000` — in **both** captures. See `04-MEMORY-LAYOUT.md` for what the IDs mean.

> ❗ **The metadata byte is read at `+0xFFF`, never at `+0x00A`.** There is not a single
> 1-byte read at a `...00A` address in either capture. *A previous version of
> `04-MEMORY-LAYOUT.md` claimed the OEM CPS reads metadata at offset `0x00A` — that claim is
> unsupported and has been retired.*

Example 2 — read a full 4 KB block. This is the **first** 4 KB read of the read session, at
`0x0AA000` (the page whose ID byte probed as `0x12`, the first channel block):
```
Command:  52 00 A0 0A 00 10
          R  [addr=0x0AA000] [len=4096]

Response: 57 00 A0 0A 00 10 19 00 00 00 00 00 ... [4096 bytes of data]
          W  [addr=0x0AA000] [len=4096] [data...]
                            ^^^^^^^^^^^
                            block +0x00 = 0x00000019 = 25 channels (uint32 LE)
```

> Do not assume `0x001000` holds anything in particular — like every other page, its content is
> determined by its ID byte, not by its address.

Example 3 — the CPS's **first** data read after entering programming mode is a 4-byte read at the
contacts base reported by V-frame `0x0F`:
```
Command:  52 00 80 27 04 00
          R  [addr=0x278000] [len=4]

Response: 57 00 80 27 04 00 01 00 00 00
          W  [addr=0x278000] [len=4] [count=1, uint32 LE]
```

### 4.2 Read Command Variations

Lengths actually seen or implemented. The length field is 16-bit, but **no request larger than
4096 bytes occurs in either capture** and neonplug never issues one.

| Length | Hex | Use case | Evidence |
|--------|-----|----------|----------|
| 1 | `01 00` | Logical-block-ID probe at `blockBase + 0xFFF` | 200 occurrences in each capture |
| 2 | `02 00` | Channel count (uint16 LE) at first channel block | neonplug only — see note |
| 4 | `04 00` | Contact count (uint32 LE) at contacts base | 1 occurrence, read capture |
| 4096 | `00 10` | Full 4 KB block (standard) | 77 occurrences, read capture |
| 2048 | `00 08` | Boot-image tail chunk | neonplug only |

> The channel count field is a **uint16 LE** at block `0x12` offset `+0x00` — 2 bytes, not 4.
> Read capture `19 00 …` = 25; write capture `80 00 ff ff …` = 128, followed by exactly 128 channel
> records. *Previously documented as uint32 — incorrect.*
>
> (*A 16-byte "block header" read was also previously documented — never observed; block headers
> are read as part of the surrounding 4 KB block, not as a separate request.*)

**Read session shape (read capture, 278 × `0x52` total):** 1 × 4-byte contact count →
200 × 1-byte ID probes → 77 × 4096-byte block reads.
**Write session shape (write capture, 200 × `0x52` total):** 200 × 1-byte ID probes only, then
straight to writing. The CPS re-probes the whole ID space before writing.

#### The read path uses the `0xFFF001` sentinel too

Of the **77** 4 KB reads in the read session, only **41** target real page addresses; the other
**36** are addressed to `0xFFF001` — the same "no physical page" sentinel the write path uses
(§5.2). *This was previously documented as a write-path-only convention — incomplete.*

The read walk is ordered by **logical ID**, exactly like the write walk, and it reads the sentinel
in place of any ID that has no page:

```
0x0AA000:12 0x015000:13 0x017000:14 FFF001 FFF001 FFF001 0x073000:18 FFF001 0x072000:1A …
… 0x00B000:44 FFF001 FFF001 FFF001 FFF001            (IDs 0x45–0x48, no page)
0x035000:5C … 0x082000:60 FFF001 FFF001 FFF001 0x069000:64     (IDs 0x61–0x63, no page)
0x06A000:02 0x011000:03 0x026000:04 0x007000:06 0x005000:0A 0x08D000:0B
0x00F000:0F 0x00C000:10 0x06B000:11 0x018000:65 0x01D000:66 0x009000:67 0x278000:FF
```

All **36 sentinel reads returned a byte-identical 4096-byte buffer**: every byte `0x00` except
`[0xFFF] = 0x48`. Treat the sentinel's contents as meaningless — in particular **do not treat the
`0x48` as that page's logical ID**; it is a property of the sentinel, not of any block. Why it is
`0x48` specifically is ❓ UNKNOWN.

Note the read walk covers one page the write walk does not: `0x06A000`, ID `0x02` (calibration).
The OEM CPS reads calibration data and never writes it back.

### 4.3 The `0x47` command (`'G'`) — ⚠️ DERIVED

Both captures issue exactly one `0x47` command, immediately before the PROGRAM sequence:

```
Command:  47 00 00 00 00 01                        (6 bytes)

Response: 53 00 00 00 00 01 <256 bytes>            (262 bytes total)
          S  [echo of request bytes 1-5] [data]
```

The frame is **structurally identical to the `0x52` read**: `47 <addr:3 LE> <len:2 LE>` →
`53 <addr:3 LE> <len:2 LE> <data>`, with `addr = 0x000000` and `len = 0x0100` = 256 (and
6 + 256 = 262, matching exactly). `⚠️ DERIVED` — the structural match is strong but the address
space it reads is ❓ UNKNOWN, and nothing in any implementation interprets the 256-byte payload.

The 256 bytes returned by this radio are 165 × `0xFF` plus scattered non-`0xFF` runs. Their
meaning is ❓ UNKNOWN — no implementation interprets anything beyond the leading `0x53`.

**Implementation gap:** neonplug has an `enterBootImageReadMode()` that sends a **5-byte**
`47 00 01 00 00` and then reads 262 bytes. That request does **not** match the capture
(`47 00 00 00 00 01`, 6 bytes), and the function has **no caller anywhere in the codebase** —
`readBootImage()` deliberately avoids it, with the comment *"No extra OEM read sequence (avoids
radio reboot)"*. Treat neonplug's 5-byte form as ❓ UNKNOWN/unverified and prefer the 6-byte
form from the capture.

---

## 5. Memory Write Commands

**Implementation reference:** `src/radios/dm32uv/connection.ts` (framing, ACK handling),
`src/radios/dm32uv/protocol.ts` (block write loops, metadata conventions)

### 5.1 Write Command (0x57) — 4 KB block

**Request — exactly 4102 bytes:**
```
Byte 0:       0x57 ('W')
Bytes 1-3:    Address (24-bit, little-endian)
Bytes 4-5:    Length (16-bit, little-endian) — 00 10 = 4096
Bytes 6-4101: Data (4096 bytes)
```

**Response:**
```
Byte 0: 0x06 (ACK)
```

> ❗ **The frame is 4102 bytes and there is no trailing metadata byte.** All 76 write frames in the
> write capture are 4102 bytes, all ACKed with `0x06`. The metadata byte lives **inside the payload
> at data offset `0xFFF`** (frame offset 4101). *Previously documented as
> `… <data:4096> <metadata:1>`, 4103 bytes — incorrect.*
>
> Bytes 4-5 are an ordinary **16-bit little-endian length** (`00 10` = 4096, `00 08` = 2048) — the
> same encoding the `0x52` read uses. *Previously documented as "reserved" + "size indicator" —
> a mislabel.*

**Preconditions:** the payload must be exactly 4096 bytes. The address is expected to be
4 KB-aligned; that is documented but **not enforced** in the reference implementation. The sole
exception observed on hardware is the sentinel address (see below), which is not aligned.

**Timeout**: 5000 ms for the 1-byte ACK.

### 5.2 The metadata byte (`data[0xFFF]`) — logical block ID

The last byte of the 4096-byte payload is the block's **logical block ID**. It is the same byte
the `0x52` 1-byte probe reads back from `blockBase + 0xFFF`.

> ❗ **Correction: this is a logical block *ID*, not a block *type*.** Across the 200-block config
> region every non-empty ID value occurs **exactly once**. The write capture is conclusive: the
> CPS walks the ID space and emits one write per ID. Physical addresses are assigned by what
> behaves like a flash-translation layer, so an ID's address differs between radios and moves over
> time. **Never hardcode block addresses — always discover them by probing `+0xFFF`.**

**The sentinel address `0xFFF001`.** When the CPS wants to write a logical ID that currently has
**no physical page**, it addresses the write to `0xFFF001` — encoded as `01 F0 FF` in bytes 1-3.
Of the 76 writes in the capture, **40 go to real page addresses and 36 go to the sentinel**. The
radio ACKs all 76 with `0x06`.

The sentinel interpretation is not merely correlational — it was checked exhaustively against the
write capture's own ID probe scan: **every** one of the 40 real-address writes went to a page whose
probed `+0xFFF` byte already equalled the ID being written, and **not one** of the 36 sentinel IDs
appears as a live page ID anywhere in the 200-page scan. (The single write that has no probed page
is the first one, `0x278000`, which lies outside the config region — it is the contacts base.)

The sentinel is **not write-only**: the read session addresses 36 of its 77 4 KB reads to the same
`0xFFF001` (see §4.2), receiving a fixed all-zero buffer with `[0xFFF] = 0x48`.

> **Sample bias:** the captured radio is at factory defaults (25 channels, 1 zone, 2 scan lists,
> 8 talk groups). That is *why* 36 slots were empty. On a fully populated codeplug those slots
> would carry real addresses. The set of allocated pages is a lower bound; the **logical-ID space
> is not**, because the CPS walks every slot it knows about regardless of population.

Excerpt of the observed write walk (`address:metadataID`, in wire order):

```
0x278000:FF  0x011000:03  0x026000:04  0x007000:06  0x005000:0A  0x08D000:0B
0x00F000:0F  0x00C000:10  0x06B000:11  0x018000:65  0x01D000:66  0x009000:67
0x035000:5C  0x013000:5D  0x07E000:5E  0x022000:5F  0x082000:60  0xFFF001:61
0xFFF001:62  0xFFF001:63  0x069000:64  0x0AA000:12  0x015000:13  ...
0x010000:41  0x008000:42  0x00A000:43  0x00B000:44  0xFFF001:45  0xFFF001:46
0xFFF001:47  0xFFF001:48
```

Two things this proves:

1. The write walk is ordered by **logical ID**, not by address — addresses jump around freely.
2. Contiguous ID ranges are real capacity ranges. `0x12`–`0x41` (48 channel slots), `0x44`–`0x48`
   (5 talk-group slots) and `0x5C`–`0x64` (9 zone slots) are each walked **completely**, including
   the empty ones via the sentinel. This retires neonplug's own
   *"unverified, pending hardware confirmation"* comment on its `ZONE_LAST` constant.

The first write of the session, `0x278000` with metadata `0xFF`, is the **contacts region**
(V-frame `0x0F` base). That region has no structured logical ID; preserve whatever byte is at
`+0xFFF` (the OEM CPS writes back `0xFF`).

### 5.3 Example: Write a 4 KB Block

```python
# Write a 4KB channel block. The address MUST come from a prior +0xFFF probe scan --
# it is assigned by the radio's flash-translation layer and is not stable.
address  = 0x0AA000   # discovered address of the block whose ID byte read back as 0x12
block_id = 0x12       # logical block ID (first channel block)

data = bytearray(block_data)      # exactly 4096 bytes
assert len(data) == 4096
data[0xFFF] = block_id            # the ID byte lives INSIDE the payload

# Prepare command: 4102 bytes total (6 header + 4096 payload)
cmd = bytearray(6 + 4096)
cmd[0]    = 0x57                                # 'W'
cmd[1:4]  = address.to_bytes(3, 'little')       # 24-bit LE address
cmd[4:6]  = (4096).to_bytes(2, 'little')        # 16-bit LE length -> 00 10
cmd[6:]   = data                                # payload; data[0xFFF] -> cmd[4101]

port.write(cmd)

# Wait for ACK
response = port.read(1)
if response != b'\x06':
    raise IOError(f"Write not acknowledged: got {response!r}")

time.sleep(0.15)  # BLOCK_READ_DELAY between consecutive block writes
```

To write a logical ID that has no page allocated, use the sentinel address:

```python
cmd[1:4] = (0xFFF001).to_bytes(3, 'little')     # -> 01 F0 FF
```

### 5.4 Variable-size Write (boot image tail)

The same opcode with a different length is used for the final 2048-byte boot-image chunk.
`⚠️ DERIVED` — implemented and exercised by the reference implementation's boot-image writer, not
present in either capture.

```
Command:  57 <addr:3 LE> 00 08 <2048 bytes>      (2054 bytes total)
Response: 06 (ACK)
```

Only 2048- and 4096-byte payloads are known to be used with this form.

### 5.5 Write Timing and Retry

- **Between consecutive block writes**: 150 ms (`BLOCK_READ_DELAY`) — the value the working
  implementation uses. One neonplug path (`writeChannels()`) writes back-to-back with no delay and
  works, so 150 ms is conservative rather than mandatory. `⚠️ DERIVED`.
- **Write timeout**: 5000 ms for the ACK.
- **Retry**: **none exists.** A non-ACK aborts the entire write operation. Neither the OEM CPS
  (which got 76/76 ACKs) nor neonplug retries a failed block.
- **Verification**: re-reading a block after writing is *not* done by the OEM CPS — the write
  session contains zero 4 KB reads.

---

## 6. Integrity — there is no checksum

**The DM-32UV wire protocol has no checksum and no CRC, in either direction.** No request frame
carries one and no response is checked for one. Integrity rests entirely on:

- the fixed magic bytes (`0x50`, `0x53`, `0x56`, `0x57`, `0x47`, `0x06`),
- the echoed frame ID on V-frame responses,
- the response length field on reads,
- the 1-byte `0x06` ACK on writes.

*The words "checksum error" appear in neonplug's speculative error-code comments (see below);
that is a guess about what the radio might be doing internally, not evidence of a wire checksum.*

---

## Command Timing Reference

**All delay values below come from the working reference implementation.** Serial-capture
timestamps have 1-second resolution and cannot corroborate sub-second host delays; the captures
confirm command *order*, not command *spacing*.

| Operation | Delay | Timeout |
|-----------|-------|---------|
| After port open, before handshake | 400 ms (`INIT_DELAY`) + 200 ms buffer clear + 200 ms settle | — |
| After `PSEARCH`, before reading the 8-byte reply | 150 ms (`PSEARCH_READ_DELAY`) | 5000 ms |
| Between handshake commands (`PASSSTA`, `SYSINFO`) | 50 ms | 5000 ms |
| After each ASCII command write | 10 ms | — |
| Around each V-frame request and response | 50 ms before read, 50 ms after payload (the trailing 50 ms is skipped when the payload length is 0) | 5000 ms |
| Programming mode sequence steps | 10 ms | 5000 ms |
| Between 1-byte metadata probes | 5 ms | 5000 ms |
| After a read request, before reading the header | 25 ms | 5000 ms (header) |
| After a read payload (settling) | 30 ms | 15000 ms (payload) |
| Between 4 KB block reads | 150 ms (`BLOCK_READ_DELAY`) | — |
| Between 4 KB block writes | 150 ms (`BLOCK_READ_DELAY`) | 5000 ms (ACK) |
| After closing the port, before reopening | 400 ms (`REOPEN_DELAY`) | 5000 ms (port open) |

*Previously documented as a uniform "500ms" timeout with 10 ms/25 ms delays — unsupported.*
The real per-request timeout is **5000 ms** (15000 ms for a memory-read payload).

**Port parameters:** 115200 baud. Data bits, parity, stop bits and flow control are never
specified by the reference implementation, so platform defaults (8N1, no flow control) apply.

## Error Codes

Write ACK byte, from `writeMemory()`. Only `0x06` was ever observed on hardware (76/76 writes).

| Byte | Meaning | Confidence |
|------|---------|------------|
| `0x06` | ACK — write accepted | CONFIRMED (76/76 writes in the write capture) |
| `0xC0` | *"may indicate: write rejected, invalid address, or radio not in programming mode"* | `⚠️ DERIVED` — verbatim speculation from neonplug; never observed |
| `0xC8` | *"may indicate: invalid block data format, checksum error, or block structure issue"* | `⚠️ DERIVED` — verbatim speculation from neonplug; never observed |
| `0x48` | *"may indicate: write timeout, radio busy processing previous write, or need for longer delay between writes"* | `⚠️ DERIVED` — verbatim speculation from neonplug; never observed |
| anything else | Generic `Write not acknowledged. Expected 0x06 (ACK), got 0x..` | |

Other failure modes:

| Error | Signal | Description |
|-------|--------|-------------|
| No response | Read timeout with 0 bytes received | Radio not connected, powered off, or still in programming mode from a previous session (close the port, wait 400 ms, retry) |
| Bad read header | Response byte 0 != `0x57` | Framing desync — a stale reply from a previous command is still in the buffer |
| Bad read length | Length field is `0` or greater than requested | Malformed response |
| Bad V-frame header | Response byte 0 != `0x56` or byte 1 != requested frame ID | Framing desync |
| Stream ended | Reader reports `done` | Port closed underneath the transfer |
| Timeout on write | No ACK byte within 5000 ms | Never observed (76/76 writes ACKed); the only sourced mitigation is the 150 ms inter-block delay |

> ❓ There is no NAK byte. (*`0x15` was previously documented as a NAK — it appears nowhere in
> either capture or in any implementation.*)

---

## Complete Session Command Order (from the captures)

Both OEM CPS sessions issue exactly this prologue, in this order:

```
PSEARCH → PASSSTA → SYSINFO
56 00 00 40 0D                                   (V-frame 0x0D, 64-byte form)
56 00 00 00 01 … 0B                              (V-frames 0x01–0x0B, ascending)
56 00 00 00 0D                                   (V-frame 0x0D again, 0-byte form)
56 00 00 00 0E, 0F, 10                           (V-frames 0x0E–0x10)
47 00 00 00 00 01                                (⚠️ DERIVED, purpose unknown)
FF FF FF FF 0C PROGRAM → 02 → 06                 (enter programming mode)
```

Then they diverge:

| | Read session | Write session |
|---|---|---|
| next | `52 <contactsBase> 04 00` — contact count | *(skipped)* |
| then | 200 × `52 <blk+0xFFF> 01 00` — ID probe | 200 × `52 <blk+0xFFF> 01 00` — ID probe |
| then | 77 × `52 <addr> 00 10` — 4 KB block reads<br>(41 real pages + **36 to the `0xFFF001` sentinel**) | 76 × `57 <addr> 00 10` + 4096 B — block writes<br>(40 real pages + **36 to the `0xFFF001` sentinel**) |
| end | *(no command — `Close port`)* | *(no command — `Close port`)* |

Both the read walk and the write walk are ordered by **logical block ID** and visit every slot the
CPS knows about, substituting the sentinel address for slots with no page. The read walk visits one
extra page the write walk skips: ID `0x02` (calibration), which is read but never written.

Total wall-clock: 38 s for the read session, 50 s for the write session.

---

## Condensed Byte-Sequence Cheat Sheet

Address bytes are 24-bit little-endian (`a0` = LSB); length bytes are 16-bit little-endian
(`l0` = LSB).

| Purpose | TX (hex) | RX |
|---------|----------|-----|
| Probe / identify | `50 53 45 41 52 43 48` (`PSEARCH`) | `06` + 7 ASCII model bytes |
| Password/status | `50 41 53 53 53 54 41` (`PASSSTA`) | `50` + 2 bytes ❓ UNKNOWN |
| System info | `53 59 53 49 4E 46 4F` (`SYSINFO`) | `06` |
| V-frame query | `56 00 00 00 <id>` | `56 <id> <len>` + `<len>` bytes |
| V-frame query, 64-byte form | `56 00 00 40 0D` | `56 0D 40` + 64 bytes |
| `0x47` query (⚠️ DERIVED) | `47 00 00 00 00 01` | `53 00 00 00 00 01` + 256 bytes |
| Enter programming mode | `FF FF FF FF 0C 50 52 4F 47 52 41 4D` | `06` |
| … step 2 | `02` | `FF FF FF FF FF FF FF FF` |
| … step 3 | `06` | `06` |
| Read memory | `52 <a0> <a1> <a2> <l0> <l1>` | `57 <a0> <a1> <a2> <l0> <l1>` + `<len>` bytes |
| Logical-block-ID probe | `52 <a0> <a1> <a2> 01 00` where addr = `blockBase + 0xFFF` | `57 …` + 1 byte |
| Read 4 KB block, no page allocated | `52 01 F0 FF 00 10` | `57 01 F0 FF 00 10` + 4096 bytes (all `0x00` except `[0xFFF] = 0x48`) |
| Write 4 KB block | `57 <a0> <a1> <a2> 00 10` + 4096 bytes (ID at payload `+0xFFF`) | `06` |
| Write 4 KB block, no page allocated | `57 01 F0 FF 00 10` + 4096 bytes | `06` |
| Write 2 KB block (⚠️ DERIVED) | `57 <a0> <a1> <a2> 00 08` + 2048 bytes | `06` |
| Disconnect | *(no bytes — close the port, DTR reset)* | — |
