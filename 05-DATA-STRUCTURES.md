# DM-32UV Data Structures

This document provides **code-ready specifications** for all DM-32UV data structures.

## Document Organization

This document is organized as follows:

1. **Primary Structures** - Detailed specifications for the most commonly used structures:
   - Channel Structure (48 bytes)
   - Zone Structure (145 bytes)
   - Scan List Structure (57 bytes)
   - RX Group List Structure (metadata 0x0F, 109 bytes per entry)
   - DMR Contact Database (V-frame 0x0F raw region, 92 bytes per entry)

2. **Additional Metadata Blocks** - Byte-level parsing for other metadata block types (0x02, 0x03, 0x04, 0x06, 0x07, 0x0A, 0x0B, 0x0F, 0x10, 0x42, 0x43, 0x44, 0x65, 0x66, 0x67)

3. **Data Encoding Reference** - Common encoding schemes used across structures

**Note**: All metadata block addresses are dynamically allocated and must be discovered via metadata discovery. See [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md) for details on metadata discovery.

---

## Channel Structure (48 bytes)

### Block Organization

**Logical block IDs**: `0x12` (first) … `0x41` (last) — **48 channel block slots**. Confirmed by the
OEM CPS write capture, which walks all 48 IDs contiguously (slots that have no physical page on the
radio are still addressed, using the `0xFFF001` "no page" sentinel).

| Item | Value | Notes |
|------|-------|-------|
| Block size | 4096 bytes (0x1000) | last byte (`0xFFF`) carries the logical block ID |
| Channel record size | 48 bytes (0x30) | stride confirmed on hardware (see worked example below) |
| Header (block `0x12` only) | 16 bytes, `0x000`–`0x00F` | later channel blocks have **no** header |
| Channel count | uint16 LE at `0x000`–`0x001` of block `0x12` | read capture `19 00 …` = 25; write capture `80 00 ff ff …` = 128 |
| First channel entry | offset `0x010` of block `0x12` | |
| Channels in block `0x12` | 84 | last one at `0x010 + 83 × 48 = 0xFA0` |
| Channels in blocks `0x13`–`0x41` | 85 each | entries start at offset `0x000` |
| Max channels | 4000 ⚠️ DERIVED | reference-implementation limit; the largest attested codeplug is 128 channels |

Header bytes `0x002`–`0x00F`: ❓ UNKNOWN. `0x00` in the factory read capture, **`0xFF` in the OEM
CPS write capture** — i.e. fill, not part of any field. The reference implementation rebuilds the
block from an all-`0xFF` fill and writes only bytes 0–1, which matches the OEM CPS.

> **Count width — 2 bytes, not 4.** The write capture settles it: the `0x12` header reads
> `80 00 ff ff …` while the same capture writes exactly **128** channel records — two bytes decode
> to 128, four to 4,294,901,888. (*Previously documented as uint32 on the strength of the read
> capture's `19 00 00 00`, where bytes 2–3 are zero only by coincidence of that radio's `0x00`
> fill.*)

**Entry offset formula** (channel `N`, 1-based):

```
if N <= 84:   block = 0x12                              offset = 0x010 + (N - 1) * 48
else:         block = 0x12 + 1 + floor((N - 85) / 85)   offset = ((N - 85) mod 85) * 48
```

**Empty slot**: a 48-byte record whose bytes are all `0xFF` and/or all `0x00`. Slots are positional
on read — an empty slot still consumes a channel number.
⚠️ DERIVED: the reference implementation **re-packs** channels contiguously on write, ignoring the
stored channel number, so a gap in the list renumbers every later channel while zone and scan-list
entries keep pointing at absolute channel numbers.

**VFO channels**: VFO A and VFO B are ordinary 48-byte channel records stored in block `0x41` at
offsets `0x0F9F` and `0x0FCF`. The two records abut and end exactly at the `0x0FFF` metadata byte
(`0x0F9F + 48 = 0x0FCF`, `0x0FCF + 48 = 0x0FFF`). neonplug addresses them as channels 4001/4002.
⚠️ Collision hazard: `0x41` is simultaneously the last channel block and the VFO block. A codeplug
large enough to fill all 48 channel blocks writes channel records over the VFO area.

**Implementation reference**: `src/radios/dm32uv/protocol.ts` (block geometry, read/write loops),
`src/radios/dm32uv/structures.ts` (`parseChannel` / `encodeChannel`).

### Memory Layout

```
Total Size: 48 bytes (0x30)
Encoding: Little-endian for multi-byte values
Padding: 0xFF for unused/empty fields (the OEM CPS write capture uses 0xFF; the factory image as
         read from a virgin radio uses 0x00 in the same places — see 0x28 and 0x2C-0x2F)
```

**Visual Overview:**
```
┌──────────────────────────────────────────────────────────┐
│                  Channel Structure (48 bytes)            │
├────────┬─────────────────────────────────────────────────┤
│ 0x00   │ Channel Name (16 bytes)                         │
│        │ "VHF Repeater\0\xFF\xFF\xFF"                    │
├────────┼─────────────────────────────────────────────────┤
│ 0x10   │ RX Frequency (4 bytes BCD)                      │
│        │ 00 50 53 14 = 145.350 MHz                       │
├────────┼─────────────────────────────────────────────────┤
│ 0x14   │ TX Frequency (4 bytes BCD)                      │
│        │ 00 50 54 14 = 145.450 MHz                       │
├────────┼─────────────────────────────────────────────────┤
│ 0x18   │ Mode & Flags (8 bytes)                          │
│        │ Mode, forbid TX, power, bandwidth, scan, colour │
│        │ 0x1D/0x1F are DUAL-PURPOSE (analog vs digital)  │
├────────┼─────────────────────────────────────────────────┤
│ 0x20   │ ❓ UNKNOWN (1 byte) — previously documented as   │
│        │ "Color Code"; that is INCORRECT. Colour code    │
│        │ lives at 0x1D bits 3-0 on digital channels.     │
├────────┼─────────────────────────────────────────────────┤
│ 0x21   │ RX CTCSS/DCS (2 bytes)                          │
│        │ 73 12 = 127.3 Hz CTCSS                          │
├────────┼─────────────────────────────────────────────────┤
│ 0x23   │ TX CTCSS/DCS (2 bytes)                          │
├────────┼─────────────────────────────────────────────────┤
│ 0x25   │ Additional Flags (7 bytes)                      │
│        │ incl. 0x2A encryption key ID (digital)          │
├────────┼─────────────────────────────────────────────────┤
│ 0x2C   │ ❓ UNKNOWN / padding (4 bytes)                   │
└────────┴─────────────────────────────────────────────────┘
NOTE: the TX contact (talkgroup) is NOT in this record - see blocks 0x42/0x43.
```

### Field-by-Field Breakdown

```
┌─────────┬──────┬────────────────────┬──────────────────────────────────────┐
│ Offset  │ Size │ Field Name         │ Description                          │
├─────────┼──────┼────────────────────┼──────────────────────────────────────┤
│ 0x00-0F │  16  │ channel_name       │ ASCII, null-term, 0xFF padding       │
│ 0x10-13 │   4  │ rx_frequency       │ BCD encoded, little-endian           │
│ 0x14-17 │   4  │ tx_frequency       │ BCD encoded, little-endian           │
│ 0x18    │   1  │ mode_flags         │ Mode, TX forbid, power level, lone   │
│ 0x19    │   1  │ scan_bandwidth     │ Bandwidth, scan add, scan list ID    │
│ 0x1A    │   1  │ talkaround_aprs    │ Talkaround forbid, APRS RX, + ❓      │
│ 0x1B    │   1  │ emergency_settings │ Emergency indicator, ack, system ID  │
│ 0x1C    │   1  │ squelch_aprs       │ Squelch level (bits 7-4), APRS mode  │
│ 0x1D    │   1  │ mode_features      │ DUAL: analog = VOX/scramble/compander│
│         │      │                    │ /talkback; digital = flags + COLOR   │
│         │      │                    │ CODE (bits 3-0) + timeslot (bit 4)   │
│ 0x1E    │   1  │ unknown_1e         │ ❓ UNKNOWN (never read/written)       │
│ 0x1F    │   1  │ mode_settings      │ DUAL: analog = PTT ID display + PTT  │
│         │      │                    │ ID; digital = private confirm + RX   │
│         │      │                    │ group list ID                        │
│ 0x20    │   1  │ unknown_20         │ ❓ UNKNOWN — NOT the color code       │
│ 0x21-22 │   2  │ rx_ctcss_dcs       │ RX CTCSS tone or DCS code            │
│ 0x23-24 │   2  │ tx_ctcss_dcs       │ TX CTCSS tone or DCS code            │
│ 0x25    │   1  │ additional_flags   │ VOX-related, compander duplicate     │
│ 0x26    │   1  │ rx_squelch_ptt     │ RX squelch mode, PTT ID display      │
│ 0x27    │   1  │ signaling_settings │ Step frequency, signaling type       │
│ 0x28    │   1  │ unknown_28         │ ❓UNKNOWN. Factory 0x00, OEM CPS 0xFF │
│ 0x29    │   1  │ ptt_id_type        │ PTT ID type (OFF/BOT/EOT/BOTH)       │
│ 0x2A    │   1  │ encryption_id      │ Digital: encryption key 0-8 ⚠️DERIVED │
│         │      │                    │ Analog: ❓UNKNOWN opaque byte         │
│ 0x2B    │   1  │ dmr_radio_id_index │ DMR Radio ID index (0xFF=None).      │
│         │      │                    │ Base 0 vs 1 ⚠️DERIVED, see below      │
│ 0x2C-2F │   4  │ unknown_2c         │ ❓UNKNOWN. Factory 0x00, OEM CPS 0xFF │
└─────────┴──────┴────────────────────┴──────────────────────────────────────┘
```

**The TX contact (talkgroup) is NOT in this record.** It lives in metadata blocks `0x42`/`0x43`,
2 bytes per channel — see [TX Contact Assignment](#tx-contact-assignment-blocks-0x42--0x43) below.

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x00-0x0F: Channel name (16 bytes)
    char channel_name[16];
    
    // 0x10-0x13: RX frequency (4 bytes, BCD)
    uint8_t rx_frequency[4];
    
    // 0x14-0x17: TX frequency (4 bytes, BCD)
    uint8_t tx_frequency[4];
    
    // 0x18: Mode and basic flags
    uint8_t mode_flags;
    // Bits 7-4: Channel mode (0=Analog, 1=Digital, 2=Fixed Analog, 3=Fixed Digital)
    // Bit 3: Forbid TX (0=Allow, 1=Forbid)
    // Bits 2-1: Power level (0=Low, 1=Medium, 2=High)
    // Bit 0: Lone worker (0=Off, 1=On)
    
    // 0x19: Scan and bandwidth
    uint8_t scan_bandwidth;
    // Bit 7: Bandwidth (0=12.5KHz narrow, 1=25KHz wide)
    // Bit 6: Scan add (0=Off, 1=On)
    // Bits 5-2: Scan list ID (0-15)
    // Bits 1-0: UNKNOWN (0 in both captures; "Reserved" is an unsupported label)
    
    // 0x1A: Talkaround and APRS
    uint8_t talkaround_aprs;
    // Bit 7: Forbid talkaround (0=Allow, 1=Forbid)
    // Bits 6-4: Unknown setting
    // Bit 3: Unknown
    // Bit 2: APRS receive (0=Off, 1=On)
    // Bits 1-0: UNKNOWN (previously documented as "Reverse frequency (0-2)" - unsupported;
    //           the OEM CPS write capture stores the value 3 here on every channel)
    
    // 0x1B: Emergency settings
    uint8_t emergency_settings;
    // Bit 7: Emergency indicator (0=Off, 1=On)
    // Bit 6: Emergency ack (0=Off, 1=On)
    // Bits 4-0: Emergency system ID (0-31)
    
    // 0x1C: Squelch and APRS
    uint8_t squelch_aprs;
    // Bits 7-4: Squelch level (0-15)
    // Bits 3-2: APRS report mode (0=Off, 1=Digital, 2=Analog)
    // Bits 1-0: UNKNOWN (0 in both captures; parsed but hard-written as 0 by the
    //           reference implementation, so radio-side content is lost on write)
    
    // 0x1D: DUAL-PURPOSE — meaning depends on channel mode (byte 0x18 bits 7-4)
    uint8_t mode_features;
    // ANALOG (mode 0 / 2):
    //   Bit 7: VOX function (0=Off, 1=On)
    //   Bit 6: Scramble (0=Off, 1=On)
    //   Bit 5: Compander (0=Off, 1=On)      (duplicated at 0x25 bit 5)
    //   Bit 4: Talkback (0=Off, 1=On)
    //   Bits 3-0: Unknown setting
    // DIGITAL (mode 1 / 3):
    //   Bit 7: Encryption enable
    //   Bit 6: Short data confirm
    //   Bit 5: TDMA direct mode
    //   Bit 4: Timeslot (0=TS1, 1=TS2)
    //   Bits 3-0: COLOR CODE (0-15)   <-- color code lives here, NOT at 0x20
    
    // 0x1E: Unknown
    // NOTE: Squelch level is at byte 0x1C bits 7-4, NOT here.
    // Neither parsed nor written by the reference implementation; purpose unestablished.
    uint8_t unknown_1e;
    
    // 0x1F: DUAL-PURPOSE — meaning depends on channel mode
    uint8_t mode_settings;
    // ANALOG:  Bit 7: unknown/unused; Bit 6: PTT ID display; Bits 5-0: PTT ID (0-63)
    // DIGITAL: Bit 7: unknown/unused; Bit 6: Private call confirm; Bits 5-0: RX group list ID (0-63)
    
    // 0x20: Unknown — previously documented as "Color Code", which is INCORRECT.
    // Hardware read capture: 0x00 on a digital channel whose color code is 1 (stored at 0x1D).
    uint8_t unknown_20;
    
    // 0x21-0x22: RX CTCSS/DCS
    uint8_t rx_ctcss_dcs[2];
    
    // 0x23-0x24: TX CTCSS/DCS
    uint8_t tx_ctcss_dcs[2];
    
    // 0x25: Additional flags
    uint8_t additional_flags;
    // Bits 7-6: Unknown
    // Bit 5: Compander duplicate (0=Off, 1=On)
    // Bit 4: VOX-related (0=Off, 1=On)
    // Bits 3-0: Unknown setting
    
    // 0x26: RX squelch and PTT ID
    uint8_t rx_squelch_ptt;
    // Bit 7: PTT ID display (0=Off, 1=On)
    // Bits 6-4: RX squelch mode (0=Carrier/CTC, 1=Optional, 2=CTC&Opt, 3=CTC|Opt)
    // Bits 3-1: Unknown
    // Bit 0: Unknown
    
    // 0x27: Signaling settings
    uint8_t signaling_settings;
    // Bits 7-4: Step frequency (0=2.5K, 1=5K, 2=6.25K, 3=10K, 4=12.5K, 5=25K, 6=50K, 7=100K)
    // Bits 3-0: Signaling type (0=None, 1=DTMF, 2=Two Tone, 3=Five Tone, 4=MDC1200)
    
    // 0x28: Unknown. Factory image 0x00; OEM CPS writes 0xFF on every channel.
    // The reference implementation writes 0x00 (matching neither reliably).
    uint8_t unknown_28;
    
    // 0x29: PTT ID type
    uint8_t ptt_id_type;
    // Bits 7-4: PTT ID type (0=OFF, 1=BOT, 2=EOT, 3=BOTH)
    // Bits 3-0: Unknown
    
    // 0x2A: Encryption key ID on DIGITAL channels (0=None, 1-8 = key index) - DERIVED.
    // Opaque/unknown on ANALOG channels.
    uint8_t encryption_id;
    
    // 0x2B: DMR Radio ID index into the list in metadata block 0x67.
    // 0xFF: None (no DMR Radio ID assigned).
    // WARNING: whether the index is 0-based or 1-based is UNRESOLVED. The reference
    // implementation treats it as 0-based; the OEM CPS write capture assigns 0x01 on a
    // radio whose Radio ID list has exactly one entry, which implies 1-based.
    // NOTE: This is NOT a contact list reference; it selects which of the
    // radio's own DMR IDs to use for this channel's TX. TX contact assignment
    // is stored separately in metadata blocks 0x42 (ch 1-2048) and 0x43 (ch 2049+/VFOs).
    uint8_t dmr_radio_id_index;
    
    // 0x2C-0x2F: Unknown / padding to the 48-byte boundary.
    // Factory read capture shows 0x00 here; the OEM CPS write capture shows 0xFF.
    uint8_t unknown_2c[4];
    
} dm32_channel_t;
#pragma pack(pop)

static_assert(sizeof(dm32_channel_t) == 48, "Channel structure must be 48 bytes");
```

---

## Channel Flags - Complete Reference

### Byte 0x18 (24): Mode and Basic Flags

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | Channel Mode | 0=Analog, 1=Digital, 2=Fixed Analog, 3=Fixed Digital |
| 3    | 0x08 | Forbid TX | 0=Allow, 1=Forbid |
| 2-1  | 0x06 | Power Level | 0=Low, 1=Medium, 2=High |
| 0    | 0x01 | Lone Worker | 0=Off, 1=On |

### Byte 0x19 (25): Scan and Bandwidth

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | Bandwidth | 0=12.5KHz narrow, 1=25KHz wide |
| 6    | 0x40 | Scan Add | 0=Off, 1=On (CPS calls this "Auto Scan") |
| 5-2  | 0x3C | Scan List ID | 0-15 (0 = None) |
| 1-0  | 0x03 | ❓ Unknown | 0 in both captures — preserve |

⚠️ **DERIVED — bandwidth polarity is strongly indicated but not directly attested.** Both captures
show the same pattern, independently:

| Capture | Digital channels | Analog channels |
|---|---|---|
| Factory read (25 ch) | `0x19` bit 7 **clear** | the analog FM channels carry bit 7 **set** (`0x19 = 0x80`) |
| OEM CPS write (128 ch) | bit 7 clear on all 127 | the single analog channel (`RATS UHF`, `0x18 = 0x04`) carries `0x19 = 0x80` |

Since DMR is always 12.5 kHz, bit 7 clear on digital channels is consistent with `0 = 12.5 kHz`, and
a wide (25 kHz) analog default matches bit 7 set. What is still missing is an independent statement
of the analog channel's bandwidth — the field could equally be ignored on digital channels, in which
case the analog sample alone does not fix the polarity. The reference implementation implements the
polarity documented here but flags it verbatim: *"NOTE: Spec appears inverted!"* — **that hedge is
not yet retired.** Settle it by toggling Band Width in the OEM CPS and diffing the block.

### Byte 0x1A (26): Talkaround and APRS

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | Forbid Talkaround | 0=Allow, 1=Forbid |
| 6-4  | 0x70 | ❓ Unknown Setting | 0-3 (values ≥4 reset to 0). Round-tripped verbatim |
| 3    | 0x08 | ❓ Unknown | boolean; round-tripped verbatim |
| 2    | 0x04 | APRS Receive | 0=Off, 1=On |
| 1-0  | 0x03 | ❓ Unknown | The OEM CPS write capture stores **`3`** here on every channel — carries data, preserve. (*Previously documented as "Reverse Freq (VFO) 0-2" — unsupported, and 3 is outside that range*) |

**Hardware values for byte 0x1A**: factory read capture `0x00` on every channel; OEM CPS write
capture `0x1B` on 123 of 128 channels and `0x0B` on the other 5 — i.e. bits 6-4 = 1 vs 0. ❓ No
correlation with any decoded field was found (in particular it does *not* track Forbid TX), so the
meaning of bits 6-4 remains unestablished.

### Byte 0x1B (27): Emergency Settings

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | Emergency Indicator | 0=Off, 1=On |
| 6    | 0x40 | Emergency Ack | 0=Off, 1=On |
| 5    | 0x20 | ❓ Unknown | **Set on every channel in the OEM CPS write capture** (`0x1B = 0x20`) — carries data, preserve |
| 4-0  | 0x1F | Emergency System ID | 0-31 ⚠️ DERIVED |

⚠️ DERIVED — the reference implementation's own comment claims "bits 0-5 (mask 0x1F)", which is
self-contradictory (mask `0x1F` is bits 4-0). Bit 5 is therefore never read and always cleared.
Whether the field is really 5 or 6 bits wide is unresolved, and the capture does not settle it:
the factory image has `0x1B = 0x00` everywhere while the OEM CPS writes `0x1B = 0x20` on every
channel. Under a 5-bit reading that is "system ID 0 plus an unknown flag"; under a 6-bit reading it
is "system ID 32". Preserve the byte rather than reconstructing it.

### Byte 0x1C (28): Squelch and APRS

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | Squelch Level | 0-15 (hardware: `0x30` = squelch 3, the factory default) |
| 3-2  | 0x0C | APRS Report Mode | 0=Off, 1=Digital, 2=Analog |
| 1-0  | 0x03 | ❓ Unknown | 0 in both captures — preserve |

### Bytes 0x1D (29) and 0x1F (31): DUAL-PURPOSE (analog vs digital)

Bytes `0x1D` and `0x1F` are interpreted differently depending on the channel mode in byte `0x18`
bits 7-4: modes 0/2 (Analog, Fixed Analog) vs modes 1/3 (Digital, Fixed Digital).

> **Previously documented as** a fixed "0x1D = Analog Features / 0x1F = PTT ID Settings" pair with
> the color code at `0x20` — **incorrect**. On digital channels `0x1D` carries the color code and
> timeslot, and `0x1F` carries the RX group list.

#### Byte 0x1D — ANALOG channels

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | VOX Function | 0=Off, 1=On ⚠️ DERIVED — a diagnostics mapping in the reference implementation associates the CPS "VOX Function" control with `0x25` bit 4 instead; the two disagree |
| 6    | 0x40 | Scramble | 0=Off, 1=On |
| 5    | 0x20 | Compander | 0=Off, 1=On — **duplicated at `0x25` bit 5**; the two copies are not kept in sync by the reference implementation |
| 4    | 0x10 | Talkback | 0=Off, 1=On ⚠️ DERIVED |
| 3-0  | 0x0F | ❓ Unknown Setting | 0-15. Hardware: the analog channel in the read capture stores `0x01` here |

#### Byte 0x1D — DIGITAL channels

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | Encryption enable | 0=Off, 1=On ⚠️ DERIVED |
| 6    | 0x40 | Short Data Confirm | 0=Off, 1=On ⚠️ DERIVED |
| 5    | 0x20 | TDMA Direct Mode | 0=Off, 1=On ⚠️ DERIVED |
| 4    | 0x10 | Timeslot | **0=TS1, 1=TS2** |
| 3-0  | 0x0F | **Color Code** | 0-15 |

Color code is hardware-confirmed here: the read capture's digital channels store `0x1D = 0x01`
(color code 1, timeslot 1) while byte `0x20` of the same records is `0x00`.

The **timeslot bit is independently confirmed by the OEM CPS write capture**, where the user named
their channels after the slot they use: `RIC Monitor TS1` stores `0x1D = 0x01` (bit 4 clear) and
`RIC Monitor TS2` stores `0x1D = 0x11` (bit 4 set). Same for the `BEA`/`PBG`/… site groups.

### Byte 0x1E (30): ❓ Unknown

**NOTE**: Squelch level is stored at **byte 0x1C bits 7-4**, not here. On hardware this byte
reads `0x00`. It has been conjectured to hold the digital-emergency system ID, but that has never
been located — the channel-side storage of that ID is **UNKNOWN**. Preserve the byte.

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | ❓ Unknown | — |

#### Byte 0x1F — ANALOG channels

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | ❓ Unknown | **Set** on the analog channel in the OEM CPS write capture (`0x1F = 0x80`), same as on digital channels — carries data, preserve |
| 6    | 0x40 | PTT ID Display | 0=Off, 1=On — **duplicated at `0x26` bit 7** ⚠️ DERIVED |
| 5-0  | 0x3F | PTT ID | 0-63 ⚠️ DERIVED |

#### Byte 0x1F — DIGITAL channels

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | ❓ Unknown | **Set on every digital channel in the OEM CPS write capture** (`0x1F = 0x80` or `0x81`) — carries data, preserve |
| 6    | 0x40 | Private Call Confirm | 0=Off, 1=On ⚠️ DERIVED |
| 5-0  | 0x3F | RX Group List ID | 0-63, 0 = None. Read capture: digital channels store `0x01` (RX group 1). Write capture: `0x00` (None) or `0x01` |

**Mode-switch caution**: `0x1D`/`0x1F` carry a different field family per mode — changing a
channel between analog and digital redefines both bytes.

### Byte 0x20 (32): ❓ Unknown

> **Previously documented as "Color Code" — this is INCORRECT.** The color code is byte `0x1D`
> bits 3-0 on digital channels; analog channels have no color code. In the hardware read capture,
> digital channels with color code 1 have `0x1D = 0x01` and `0x20 = 0x00`.

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | ❓ Unknown | Not parsed and not written by the reference implementation. `0x00` on every channel in **both** captures, including a write capture in which every other trailing byte is `0xFF` — so `0x00` here is deliberate, not fill |

### Byte 0x25 (37): Additional Flags

**CPS Functions**: `sub_479C70`, `sub_479CA0`, `sub_479E00`, `sub_479F60`

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-6  | 0xC0 | ❓ Unknown | Hardware: `0` in the factory read capture, **`3` on every channel** in the OEM CPS write capture |
| 5    | 0x20 | Compander (duplicate of `0x1D` bit 5) | 0=Off, 1=On ⚠️ DERIVED |
| 4    | 0x10 | VOX-Related Flag | 0=Off, 1=On ⚠️ DERIVED — the CPS "VOX Function" control has been mapped to **both** this bit and `0x1D` bit 7 by different analyses; which copy the firmware honours is unresolved |
| 3-0  | 0x0F | ❓ Unknown Setting | 0-15 (possibly VOX or analog related) |

**Duplicate flags**: `pttIdDisplay` and `compander` each have two copies in the channel byte
layout. Which copy the firmware reads is unverified — write both, or preserve whichever you do
not model.

### Byte 0x26 (38): RX Squelch and PTT ID

**CPS Functions**: `sub_47A0C0`, `sub_47A220`, `sub_47A360`

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | PTT ID Display (duplicate of `0x1F` bit 6, analog) | 0=Off, 1=On |
| 6-4  | 0x70 | RX Squelch Mode | 0=Carrier/CTC, 1=Optional, 2=CTC&Opt, 3=CTC\|Opt (values 4-7 fall back to Carrier/CTC) |
| 3-1  | 0x0E | ❓ Unknown | 0-7 |
| 0    | 0x01 | ❓ Unknown | Hardware: `0` in the factory read capture, **`1` on every channel** in the OEM CPS write capture |

### Byte 0x27 (39): Signaling Settings

**CPS Functions**: `sub_47A500` (step freq), `sub_47A680` (signaling)

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | Step Frequency | 0=2.5K, 1=5K, 2=6.25K, 3=10K, 4=12.5K, 5=25K, 6=50K, 7=100K ⚠️ DERIVED |
| 3-0  | 0x0F | Signaling Type | 0=None, 1=DTMF, 2=Two Tone, 3=Five Tone, 4=MDC1200 ⚠️ DERIVED |

⚠️ Both enumerations are ⚠️ DERIVED, and the OEM CPS write capture stores **`0x27 = 0xFF`** on
some channels — `15` in both nibbles, outside both documented ranges. Treat out-of-range values as
"unset / not applicable" and round-trip the byte rather than clamping it.

**Step Frequency Values**:
- 0 = 2.5 KHz
- 1 = 5 KHz
- 2 = 6.25 KHz
- 3 = 10 KHz
- 4 = 12.5 KHz
- 5 = 25 KHz
- 6 = 50 KHz
- 7 = 100 KHz

**Signaling Type Values**:
- 0 = None
- 1 = DTMF
- 2 = Two Tone
- 3 = Five Tone
- 4 = MDC1200

### Byte 0x28 (40): ❓ Unknown

**Status**: Not parsed by the reference implementation, which writes `0x00` here. Hardware values
differ by source: **`0x00` in the factory read capture, `0xFF` on every channel in the OEM CPS write
capture.** So neither value is a fixed constant, and the reference implementation's `0x00` does not
match what the OEM CPS writes.  
**Purpose**: ❓ UNKNOWN — *previously documented as "Reserved" / "not accessed by any known CPS
function"; both are unsupported claims.* Verbatim implementation comment: *"Unknown purpose,
possibly padding or reserved for future use."*

### Byte 0x29 (41): PTT ID Type

**CPS Functions**: `sub_47A7E0`, `sub_47A960`

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | PTT ID Type | 0=OFF, 1=BOT, 2=EOT, 3=BOTH ⚠️ DERIVED |
| 3-2  | 0x0C | ❓ Unknown Setting | 0-3 (round-tripped verbatim) |
| 1-0  | 0x03 | ❓ Unknown | round-tripped verbatim. Hardware: `0` in the factory read capture, **`3` on every channel** in the OEM CPS write capture |

**PTT ID Type Values** ⚠️ DERIVED:
- 0 = OFF
- 1 = BOT (Beginning of Transmission)
- 2 = EOT (End of Transmission)
- 3 = BOTH

⚠️ The OEM CPS write capture stores `0x29 = 0xF3` on most channels, i.e. a PTT-ID-Type nibble of
**15** — outside the documented 0-3 range. Either the nibble means something else, or `0xF` is an
"unset" encoding. Unresolved; do not clamp this nibble on write.

### Byte 0x2A (42): Encryption Key ID (digital) / ❓ Unknown (analog)

**CPS Functions**: `sub_47AAE0`, `sub_47AB80`  
**Type**: 8-bit value

| Channel mode | Meaning | Confidence |
|---|---|---|
| Digital / Fixed Digital | Encryption key ID: `0` = None, `1`–`8` = key index. Keys themselves live in metadata block `0x10` | ⚠️ DERIVED |
| Analog / Fixed Analog | Opaque byte, round-tripped unchanged | ❓ UNKNOWN |

Hardware: `0x00` on every channel of the factory read capture (no encryption configured), so the
non-zero encodings are not directly attested. The OEM CPS write capture stores **`0xFF`** on most
channels and `0x00` on the rest — `0xFF` is outside the claimed 0-8 range, which further weakens
the encryption-key-ID reading. Round-trip the byte; do not clamp it.

### Byte 0x2B (43): DMR Radio ID Index

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | DMR Radio ID Index | Index into the Radio ID list (block `0x67`); `0xFF` = None. **Base of the index is ⚠️ DERIVED — see below** |

**Purpose**: Selects which of the radio's own DMR Radio IDs (metadata block 0x67) to use when transmitting on this channel. This is **not** a reference to the contact/talkgroup list.

⚠️ **DERIVED / contested — 0-based vs 1-based is unresolved.**

| Source | Evidence | Implies |
|---|---|---|
| Reference implementation | *"Radio uses 0-based indexing: 0=first entry … 0xFF (255) = None"* | 0-based, `0` valid |
| Factory read capture | Block `0x67` holds **5** Radio IDs (`Radio 1`…`Radio 5`); every channel stores `0x2B = 0x00` | either "Radio 1" (0-based) or "None" (1-based) — ambiguous |
| **OEM CPS write capture** | Block `0x67` holds exactly **one** Radio ID (`CHANGEME`, count byte = 1); of 128 channels, **113 store `0x01`**, 13 store `0x00`, 2 store `0xFF` | **1-based** — under 0-based indexing the 113 channels holding `1` would point at a second Radio ID that does not exist |

*Previously documented here as "`0` is a valid index (first Radio ID); only `0xFF` means None" —
that is not supported.* The write capture is the stronger evidence and points the other way. Until
this is settled by experiment (add a second Radio ID in the OEM CPS, assign it, re-read), treat the
base as unknown and preserve the byte rather than recomputing it.

### Bytes 0x2C-0x2F (44-47): ❓ Unknown / padding

**Status**: Not parsed and not deliberately written by the reference implementation (they keep the
`0xFF` initial fill after a neonplug write). Hardware values differ by source: `00 00 00 00` in the
factory read capture, `ff ff ff ff` on every channel in the OEM CPS write capture — so the `0xFF`
fill matches OEM behaviour.  
**Purpose**: ❓ UNKNOWN. Most likely padding to the 48-byte boundary, but that is not established.

---

## TX Contact Assignment (blocks 0x42 / 0x43)

**This is the single most important channel quirk.** The channel record contains **no** talkgroup /
TX-contact field. Byte `0x2B` is the DMR *Radio ID* index, not a contact. The TX contact for every
channel lives in two dedicated metadata blocks, **2 bytes per channel**:

| Channel | Block | Offset within block |
|---|---|---|
| 1 … 2047 | `0x42` | `(channel - 1) * 2` |
| 2048 … 4000 | `0x43` | `(channel & 0x7FF) * 2` |
| 4001 (VFO A) | `0x43` | `0x0FFA` (fixed) |
| 4002 (VFO B) | `0x43` | `0x0FFC` (fixed) |

> ⚠️ The split is **2047 / 2048**, not 2048 / 2049 as sometimes stated — channel 2048's entry is
> at offset 0 of block `0x43`.

### TX contact record (2 bytes)

| Byte | Bits | Meaning |
|---|---|---|
| byte0 | 7-4 | Contact index, high nibble (index bits 11-8) |
| byte0 | 3-1 | ❓ Unknown (previously labelled "Reserved" — unsupported) |
| byte0 | 0 | Digital flag (1 = digital, 0 = analog/mixed) |
| byte1 | 7-0 | Contact index, low byte (index bits 7-0) |

```
contactId = ((byte0 >> 4) << 8) | byte1     // 12-bit, 0-4095; 0 = None
byte0     = (contactId >> 8) << 4 | (isDigital ? 0x01 : 0x00)
byte1     =  contactId & 0xFF
```

The index refers to the **Talk Groups list in metadata block `0x44`**, 1-based (`0` = None).

### Behaviour notes

- The channel record itself carries **no** contact field — resolve every channel's TX contact from
  these blocks after parsing.
- The TX contact is meaningful on digital channels; what an analog channel's entry means is
  unestablished — preserve it rather than regenerating it.
- Never generate these blocks from scratch: read-modify-write, and write them **after** the channel
  blocks.

Hardware, block `0x43` tail (both captures, anchored on the block's own metadata byte at 0x0FFF):

```
factory read    0FF8  00 00 00 01 01 01 00 43     → 0x0FFA-0x0FFB = 00 01, 0x0FFC-0x0FFD = 01 01
OEM CPS write   0FF8  ff ff 0e 01 0e 01 ff 43     → 0x0FFA-0x0FFB = 0e 01, 0x0FFC-0x0FFD = 0e 01
```

In the OEM write capture those **four bytes are the only non-0xFF payload bytes in the entire block**
besides the metadata byte, which pins the two VFO slots to 0x0FFA and 0x0FFC exactly.

> **Previously documented as** `0xFFA = 01 01`, `0xFFC = 01 00` — **incorrect**, that is the same run
> of bytes quoted one position early.

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseTxContactForChannel`,
`encodeTxContactForChannel`, `getTxContactOffset`).

---

## Channel CTCSS / DCS Encoding (bytes 0x21-0x24)

Two bytes per tone, stored **`[low, high]`**. RX ("CTC/DCS Decode" in the CPS) at `0x21-0x22`,
TX ("CTC/DCS Encode") at `0x23-0x24`.

| Case | Bytes | Rule |
|---|---|---|
| None (as written by the radio/OEM CPS) | `FF FF` | hardware read capture shows `FF FF` on tone-less channels |
| None (alternate) | `00 00` | decodes to a computed frequency of 0 → None. **The neonplug reference implementation emits `00 00` for None, not `FF FF`** — a write asymmetry versus the OEM CPS |
| DCS | high ≥ `0x80` | `code = (high & 0x0F)*100 + ((low >> 4) & 0x0F)*10 + (low & 0x0F)` — BCD digit nibbles |
| DCS polarity | high `0x80`–`0xBF` = normal (`N`), high ≥ `0xC0` = inverted (`I`) | encode: `high = (inverted ? 0xC0 : 0x80) \| hundreds`, `low = (tens << 4) \| ones` |
| CTCSS | high < `0x80` | `freq = ((high >> 4)*100 + (high & 0x0F)*10 + ((low >> 4) & 0x0F)) + (low & 0x0F)/10` |
| CTCSS encode | | `low = (ones << 4) \| tenths`, `high = (hundreds << 4) \| tens` |

Worked values (round-trip tested in the reference implementation, all 104 DCS codes × both
polarities):

| Tone | `[low, high]` |
|---|---|
| CTCSS 67.0 Hz | `70 06` |
| CTCSS 100.0 Hz | `00 10` |
| CTCSS 127.3 Hz | `73 12` |
| CTCSS 203.5 Hz | `35 20` |
| DCS D023N | `23 80` |
| DCS D023I | `23 C0` |
| DCS D754N | `54 87` |
| DCS D754I | `54 C7` |

**Hardware-attested example** (OEM CPS write capture, the codeplug's one analog channel
`RATS UHF`): bytes `0x21-0x22` and `0x23-0x24` both read `44 07`, which decodes as
`(0×100 + 7×10 + 4) + 4/10` = **74.4 Hz** — a standard CTCSS tone. This is the only tone value in
either capture that is not `FF FF`, and it confirms the `[low, high]` byte order and the nibble
arithmetic against real CPS output rather than against a round trip.

Naming note: the reference implementation labels the inverted polarity `'P'` internally where the
CPS and this spec use `I`. Historical warning from the project's notes: decode *and* encode were
both wrong and mutually consistent for `D0xx` codes for a long time, so round-trip tests did not
catch it — validate against a radio, not against a round trip.

See [06-ENCODING.md](06-ENCODING.md) for the full tone tables.

---

## Channel Worked Example — real hardware (read capture)

Block `0x12` (first channel block), read as `52 00 a0 0a 00 10` → response payload:

```
offset 0x000  19 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   count = 25 (uint16 LE) + header fill
offset 0x010  43 68 61 6e 6e 65 6c 20 31 00 00 00 00 00 00 00   "Channel 1\0" + padding
offset 0x020  50 12 00 43 50 12 00 43 14 00 00 00 34 01 00 01   RX/TX/flags
offset 0x030  00 ff ff ff ff 00 00 00 00 00 00 00 00 00 00 00   0x20..0x2F of the record
offset 0x040  43 68 61 6e 6e 65 6c 20 32 00 ...                 "Channel 2" — stride 48 confirmed
```

Decoding record 1 (record base = block offset `0x010`):

| Record offset | Bytes | Decoded |
|---|---|---|
| 0x00-0x0F | `43 68 …` | name `Channel 1` |
| 0x10-0x13 | `50 12 00 43` | RX 430.01250 MHz (BCD, byte order reversed) |
| 0x14-0x17 | `50 12 00 43` | TX 430.01250 MHz (simplex) |
| 0x18 | `14` | mode 1 = Digital, forbid TX 0, power (bits 2-1) = 2 = High, lone worker 0 |
| 0x19 | `00` | bandwidth 12.5 kHz, scan add off, scan list 0 |
| 0x1A | `00` | — |
| 0x1B | `00` | — |
| 0x1C | `34` | squelch level 3, APRS report mode 1 |
| 0x1D | `01` | digital: color code **1**, timeslot TS1 |
| 0x1E | `00` | ❓ unknown |
| 0x1F | `01` | digital: RX group list 1 |
| 0x20 | `00` | ❓ unknown — **note it is 0 while the color code is 1** |
| 0x21-0x24 | `ff ff ff ff` | RX and TX tone = None |
| 0x25-0x2A | `00 …` | — |
| 0x2B | `00` | DMR Radio ID index 0 (first ID) |
| 0x2C-0x2F | `00 00 00 00` | ❓ unknown/padding |

The analog channel in the same block (`Channel 10`) has `0x18 = 04` (mode 0 = Analog, power High)
and `0x19 = 80` (bandwidth 25 kHz), which is what pins the bandwidth polarity.

### Implementation guidance — preserving the channel record

The OEM CPS write capture proves several nominally-unknown positions carry real data on *every*
channel: `0x1A` bits 1-0 = `3`, `0x1B` bit 5 = `1`, `0x1F` bit 7 = `1`, `0x28` = `0xFF`,
`0x2C`–`0x2F` = `0xFF`. An implementation that regenerates records from defaults destroys them.

- **Preserve** `0x1E`, `0x20`, `0x28`, `0x2C`–`0x2F` and every bit marked `❓ UNKNOWN` above —
  read-modify-write, do not rebuild.
- **Tone "None"** has two spellings on the wire (`FF FF` and `00 00`); accept both.
- **Names**: restrict to ASCII `0x20`–`0x7E` before encoding (a multi-byte character can overrun
  the 16-byte field into the RX frequency BCD), and remember a 16-character name carries **no**
  terminator.

---

## Parsing Example

```python
def parse_channel_flags(data: bytes) -> dict:
    """Parse all channel flags from 48-byte channel data"""
    
    return {
        # Byte 0x18
        'channel_mode': (data[0x18] >> 4) & 0x0F,
        'forbid_tx': bool(data[0x18] & 0x08),
        'power_level': (data[0x18] >> 1) & 0x03,  # 0=Low, 1=Medium, 2=High
        'lone_worker': bool(data[0x18] & 0x01),
        
        # Byte 0x19
        'bandwidth': 1 if (data[0x19] & 0x80) else 0,  # 0=12.5KHz narrow, 1=25KHz wide
        'scan_add': bool(data[0x19] & 0x40),
        'scan_list_id': (data[0x19] >> 2) & 0x0F,
        
        # Byte 0x1A
        'forbid_talkaround': bool(data[0x1A] & 0x80),
        'aprs_receive': bool(data[0x1A] & 0x04),
        'unknown_1a_1_0': data[0x1A] & 0x03,  # UNKNOWN - was mis-documented as "reverse_freq"
        
        # Byte 0x1B
        'emergency_indicator': bool(data[0x1B] & 0x80),
        'emergency_ack': bool(data[0x1B] & 0x40),
        'emergency_system_id': data[0x1B] & 0x1F,
        
        # Byte 0x1C
        'squelch_level': (data[0x1C] >> 4) & 0x0F,  # 0-15 (NOT power level)
        'aprs_report_mode': (data[0x1C] >> 2) & 0x03,
        
        # Bytes 0x1D / 0x1F are DUAL-PURPOSE - decode by mode (see below)
        # ANALOG (channel_mode 0 or 2):
        'vox_function': bool(data[0x1D] & 0x80),
        'scramble': bool(data[0x1D] & 0x40),
        'compander': bool(data[0x1D] & 0x20),
        'talkback': bool(data[0x1D] & 0x10),
        'ptt_id_display': bool(data[0x1F] & 0x40),
        'ptt_id': data[0x1F] & 0x3F,
        # DIGITAL (channel_mode 1 or 3):
        'encryption': bool(data[0x1D] & 0x80),
        'short_data_confirm': bool(data[0x1D] & 0x40),
        'tdma_direct_mode': bool(data[0x1D] & 0x20),
        'timeslot': 2 if (data[0x1D] & 0x10) else 1,   # bit 4: 0=TS1, 1=TS2
        'color_code': data[0x1D] & 0x0F,               # NOT byte 0x20!
        'private_confirm': bool(data[0x1F] & 0x40),
        'rx_group_list_id': data[0x1F] & 0x3F,
        
        # Byte 0x1E - Unknown (squelch is at 0x1C bits 7-4, NOT here)
        # Byte 0x20 - Unknown (previously mis-documented as the color code)
        
        # Byte 0x25
        'compander_dup': bool(data[0x25] & 0x20),
        'vox_related': bool(data[0x25] & 0x10),
        
        # Byte 0x26
        'ptt_id_display_dup': bool(data[0x26] & 0x80),
        'rx_squelch_mode': (data[0x26] >> 4) & 0x07,
        
        # Byte 0x27
        'step_frequency': (data[0x27] >> 4) & 0x0F,
        'signaling_type': data[0x27] & 0x0F,
        
        # Byte 0x29
        'ptt_id_type': (data[0x29] >> 4) & 0x0F,
        
        # Byte 0x2B
        'dmr_radio_id_index': data[0x2B],  # 0xFF = None; NOT a contact list reference
    }
```

---

### Code Structure (Python)

```python
from dataclasses import dataclass
from typing import Optional
import struct

@dataclass
class DM32Channel:
    """DM32 Radio Channel Structure (48 bytes)"""
    
    # Name (16 bytes)
    channel_name: str
    
    # Frequencies (4 bytes each, BCD encoded)
    rx_frequency: float  # MHz
    tx_frequency: float  # MHz
    
    # Byte 0x18: Mode flags
    channel_mode: int  # 0=Analog, 1=Digital, 2=FixedAnalog, 3=FixedDigital
    forbid_tx: bool
    power_level: int  # 0=Low, 1=Medium, 2=High
    lone_worker: bool
    
    # Byte 0x19: Scan and bandwidth
    bandwidth: int  # 0=12.5KHz narrow, 1=25KHz wide
    scan_add: bool
    scan_list_id: int  # 0-15
    
    # Byte 0x1A: Talkaround and APRS
    forbid_talkaround: bool
    aprs_receive: bool
    unknown_1a_1_0: int  # UNKNOWN (was mis-documented as "reverse_freq", 0-2; hardware stores 3)
    
    # Byte 0x1B: Emergency
    emergency_indicator: bool
    emergency_ack: bool
    emergency_system_id: int  # 0-31
    
    # Byte 0x1C: Squelch and APRS
    squelch_level: int  # 0-15 (bits 7-4)
    aprs_report_mode: int  # 0=Off, 1=Digital, 2=Analog
    
    # Byte 0x1D: DUAL-PURPOSE (analog vs digital)
    # ANALOG:
    vox_function: bool
    scramble: bool
    compander: bool
    talkback: bool
    # DIGITAL:
    color_code: int   # bits 3-0  (NOT byte 0x20)
    timeslot: int     # bit 4: 0=TS1, 1=TS2
    
    # Byte 0x1E: Unknown (NOT squelch)
    
    # Byte 0x1F: DUAL-PURPOSE
    ptt_id_display: bool   # analog, bit 6
    ptt_id: int            # analog, bits 5-0 (0-63)
    rx_group_list_id: int  # digital, bits 5-0 (0-63, 0=None)
    
    # Byte 0x20: Unknown
    
    # Bytes 0x21-0x24: CTCSS/DCS
    rx_ctcss_tone: Optional[float] = None  # Hz
    rx_dcs_code: Optional[str] = None  # e.g., "D023N"
    tx_ctcss_tone: Optional[float] = None  # Hz
    tx_dcs_code: Optional[str] = None  # e.g., "D023N"
    
    # Byte 0x27: Signaling
    step_frequency: int  # 0=2.5K, 1=5K, 2=6.25K, 3=10K, 4=12.5K, 5=25K, 6=50K, 7=100K
    signaling_type: int  # 0=None, 1=DTMF, 2=TwoTone, 3=FiveTone, 4=MDC1200
    
    # Byte 0x29: PTT ID type
    ptt_id_type: int  # 0=OFF, 1=BOT, 2=EOT, 3=BOTH
    
    # Byte 0x2B: DMR Radio ID index (0xFF = None). NOT the TX contact -
    # the TX contact/talkgroup comes from blocks 0x42/0x43.
    dmr_radio_id_index: int
    
    @classmethod
    def from_bytes(cls, data: bytes) -> 'DM32Channel':
        """Parse 48-byte channel record"""
        if len(data) != 48:
            raise ValueError(f"Channel data must be 48 bytes, got {len(data)}")
        
        # Parse name (16 bytes, null-terminated, 0xFF padded)
        name_bytes = data[0:16]
        name = name_bytes.split(b'\x00')[0].replace(b'\xFF', b'').decode('ascii', errors='ignore')
        
        # Parse frequencies (see 06-ENCODING.md for details)
        rx_freq = decode_bcd_frequency(data[0x10:0x14])
        tx_freq = decode_bcd_frequency(data[0x14:0x18])
        
        # Parse flags
        mode_flags = data[0x18]
        channel_mode = (mode_flags >> 4) & 0x0F
        forbid_tx = bool(mode_flags & 0x08)
        power_level = (mode_flags >> 1) & 0x03  # 0=Low, 1=Medium, 2=High
        lone_worker = bool(mode_flags & 0x01)
        
        scan_bw = data[0x19]
        bandwidth = 1 if (scan_bw & 0x80) else 0  # 0=12.5KHz narrow, 1=25KHz wide
        scan_add = bool(scan_bw & 0x40)
        scan_list_id = (scan_bw >> 2) & 0x0F
        
        # ... (continue parsing all fields)
        
        return cls(
            channel_name=name,
            rx_frequency=rx_freq,
            tx_frequency=tx_freq,
            channel_mode=channel_mode,
            forbid_tx=forbid_tx,
            # ... (set all fields)
        )
    
    def to_bytes(self) -> bytes:
        """Convert to 48-byte channel record"""
        # Build 48-byte structure
        data = bytearray(48)
        
        # Name (16 bytes, null-terminated, padding with 0xFF)
        name_bytes = self.channel_name.encode('ascii')[:15]  # Max 15 chars + null
        data[0:len(name_bytes)] = name_bytes
        data[len(name_bytes)] = 0x00  # Null terminator
        for i in range(len(name_bytes) + 1, 16):
            data[i] = 0xFF  # Padding
        
        # Frequencies
        data[0x10:0x14] = encode_bcd_frequency(self.rx_frequency)
        data[0x14:0x18] = encode_bcd_frequency(self.tx_frequency)
        
        # Flags
        data[0x18] = (
            ((self.channel_mode & 0x0F) << 4) |
            (0x08 if self.forbid_tx else 0) |
            ((self.power_level & 0x03) << 1) |  # bits 2-1: power level
            (0x01 if self.lone_worker else 0)
        )
        
        data[0x19] = (
            (0x80 if self.bandwidth else 0) |
            (0x40 if self.scan_add else 0) |
            ((self.scan_list_id & 0x0F) << 2)
        )
        
        # ... (continue encoding all fields)
        
        return bytes(data)
```

### Code Structure (Go)

```go
package dm32

import (
    "encoding/binary"
    "fmt"
)

// Channel represents a DM32 radio channel (48 bytes)
type Channel struct {
    // 0x00-0x0F: Name
    Name string
    
    // 0x10-0x17: Frequencies (MHz)
    RXFrequency float64
    TXFrequency float64
    
    // 0x18: Mode flags
    ChannelMode  uint8 // 0=Analog, 1=Digital
    ForbidTX     bool
    PowerLevel   uint8 // 0=Low, 1=Medium, 2=High  (bits 2-1; NOT busy lock)
    LoneWorker   bool
    
    // 0x19: Scan and bandwidth
    Bandwidth    uint8 // 0=12.5KHz narrow, 1=25KHz wide
    ScanAdd      bool
    ScanListID   uint8 // 0-15
    
    // 0x1C: Squelch (bits 7-4) and APRS (bits 3-2)
    SquelchLevel uint8 // 0-15 (NOT byte 0x1E)
    
    // 0x1D bits 3-0 (digital channels only): DMR color code. NOT byte 0x20.
    ColorCode    uint8 // 0-15
    
    // 0x21-0x24: CTCSS/DCS
    RXCTCSSHz    float64  // 0 if not used
    RXDCSCode    string   // "" if not used
    TXCTCSSHz    float64
    TXDCSCode    string
    
    // 0x2B: DMR Radio ID index
    // 0xFF = None; selects which Radio ID (block 0x67) to use for TX.
    // TX contact is in blocks 0x42/0x43, not here.
    DMRRadioIDIndex uint8
}

// FromBytes parses a 48-byte channel record
func (c *Channel) FromBytes(data []byte) error {
    if len(data) != 48 {
        return fmt.Errorf("channel data must be 48 bytes, got %d", len(data))
    }
    
    // Parse name
    nameBytes := data[0:16]
    c.Name = string(nameBytes[:bytesUntilNull(nameBytes)])
    
    // Parse frequencies
    c.RXFrequency = decodeBCDFrequency(data[0x10:0x14])
    c.TXFrequency = decodeBCDFrequency(data[0x14:0x18])
    
    // Parse flags
    modeFlags := data[0x18]
    c.ChannelMode = (modeFlags >> 4) & 0x0F
    c.ForbidTX = (modeFlags & 0x08) != 0
    c.PowerLevel = (modeFlags >> 1) & 0x03  // bits 2-1: 0=Low, 1=Medium, 2=High
    c.LoneWorker = (modeFlags & 0x01) != 0
    
    scanBW := data[0x19]
    c.Bandwidth = (scanBW >> 7) & 0x01
    c.ScanAdd = (scanBW & 0x40) != 0
    c.ScanListID = (scanBW >> 2) & 0x0F
    
    // Squelch (bits 7-4 of byte 0x1C) and color code (bits 3-0 of byte 0x1D, digital only)
    c.SquelchLevel = (data[0x1C] >> 4) & 0x0F  // NOT byte 0x1E
    if c.ChannelMode == 1 || c.ChannelMode == 3 {  // Digital / Fixed Digital
        c.ColorCode = data[0x1D] & 0x0F        // NOT byte 0x20
    }
    
    // CTCSS/DCS
    c.RXCTCSSHz, c.RXDCSCode = decodeCTCSSDCS(data[0x21:0x23])
    c.TXCTCSSHz, c.TXDCSCode = decodeCTCSSDCS(data[0x23:0x25])
    
    // DMR Radio ID index
    c.DMRRadioIDIndex = data[0x2B]
    
    return nil
}

// ToBytes converts channel to 48-byte record
func (c *Channel) ToBytes() []byte {
    data := make([]byte, 48)
    
    // Name (16 bytes, pad with 0xFF)
    copy(data[0:16], c.Name)
    if len(c.Name) < 16 {
        data[len(c.Name)] = 0x00 // Null terminator
        for i := len(c.Name) + 1; i < 16; i++ {
            data[i] = 0xFF
        }
    }
    
    // Frequencies
    copy(data[0x10:0x14], encodeBCDFrequency(c.RXFrequency))
    copy(data[0x14:0x18], encodeBCDFrequency(c.TXFrequency))
    
    // Flags
    data[0x18] = (c.ChannelMode << 4) | 
                 (boolToByte(c.ForbidTX) << 3) |
                 (c.PowerLevel << 1) |   // bits 2-1: power level
                 boolToByte(c.LoneWorker)
    
    data[0x19] = (c.Bandwidth << 7) |
                 (boolToByte(c.ScanAdd) << 6) |
                 (c.ScanListID << 2)
    
    // Squelch (bits 7-4 of 0x1C); color code goes in 0x1D bits 3-0 on digital channels
    data[0x1C] = c.SquelchLevel << 4  // NOT 0x1E
    if c.ChannelMode == 1 || c.ChannelMode == 3 {
        data[0x1D] = (data[0x1D] &^ 0x0F) | (c.ColorCode & 0x0F)  // NOT 0x20
    }
    
    // CTCSS/DCS
    copy(data[0x21:0x23], encodeCTCSSDCS(c.RXCTCSSHz, c.RXDCSCode))
    copy(data[0x23:0x25], encodeCTCSSDCS(c.TXCTCSSHz, c.TXDCSCode))
    
    // DMR Radio ID index
    data[0x2B] = c.DMRRadioIDIndex
    
    return data
}

func boolToByte(b bool) uint8 {
    if b {
        return 1
    }
    return 0
}
```

## Zone Structure (145 bytes)

Zones group channels together for organizational purposes.

### Memory Organization

**Logical block IDs**: `0x5C` (first) … `0x64` (last) — **9 zone block slots**. Confirmed by the
OEM CPS write capture, which walks all 9 IDs contiguously. *This retires the reference
implementation's own hedge on the range end (`ZONE_LAST: 0x64 … unverified, pending hardware
confirmation`) — it is now hardware-confirmed.*

**Block Size**: 4 KB (0x1000 bytes)  
**Zone record size**: 145 bytes  
**Capacity**: 28 zones per 4KB block ⚠️ DERIVED (first block: `(4096 − 16) / 145`; later blocks:
`4096 / 145`). Verbatim from the reference implementation: *"`ZONES_PER_BLOCK: 28, // Approximate
(4096 - 16) / 145`"*. **Not attested** — the read capture has 2 zones and the write capture 9, both
inside the first block, so nothing in either capture exercises a block boundary.  
**Max Zones**: 250 ⚠️ DERIVED (9 blocks × 28 = 252 slots; the 250 limit comes from the reference
implementation, not from a capture)  
**Max Channels per Zone**: 64 — `(145 − 17) / 2`

**Note**: Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseZones` / `encodeZone`),
`src/radios/dm32uv/protocol.ts` (multi-block zone write).

#### Zone block header (first zone block only) — 16 bytes

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0x000 | 1 | **Global** zone count | Total zones across *all* zone blocks, not this block's share. Hardware-confirmed three times: a radio with 29 zones stored `0x1D`; the factory read capture with 2 zones stores `0x02`; the OEM CPS write capture with 9 zones stores `0x09` |
| 0x001–0x00F | 15 | ❓ Unknown | **Not a constant.** Factory read capture: `01 00 0C 00 01 00 01 00 00 00 00 00 00 00 00`. OEM CPS write capture: `01 FF 01 FF 01 FF FF FF FF FF FF FF FF FF FF`. Not parsed by anyone |

⚠️ The reference implementation copies bytes `0x001`–`0x00F` verbatim from the block it read
(*"Preserved original bytes 1-15 for first block"*). That is the conservative choice, but note the
**OEM CPS does not do this** — it writes its own pattern, which differs from the factory one. Until
the field's meaning is known, copying what you read is still the safer behaviour for a third-party
tool.

Zone blocks after the first have **no header** — zone records start at offset `0x000`.
⚠️ DERIVED: no capture contains a second zone block, so the "no header" rule for blocks
`0x5D`–`0x64` comes from the reference implementation only.

### Offset Calculation

Zones are 1-indexed. The 145-byte stride and the 16-byte first-block header are hardware-confirmed
(both captures). ⚠️ DERIVED: the claim that the packing **restarts at every 4 KB block boundary** —
rather than running as a flat 145-byte stride across the concatenated image — comes from the
reference implementation alone; no capture contains more than one zone block.

```
zoneIdx      = zone_number - 1
blockIdx     = zoneIdx / 28                      # integer division; 0 = block 0x5C
indexInBlock = zoneIdx % 28

offset_in_block = (blockIdx == 0) ? 16 + indexInBlock * 145
                                  :      indexInBlock * 145
```

**Examples**:
- Zone 1: block `0x5C`, offset `16` (0x010)
- Zone 2: block `0x5C`, offset `161` (0x0A1) — confirmed in **both** captures: the factory read
  capture's second zone name (`Func Demo`) and the OEM CPS write capture's second zone name
  (`Beaverdam`) each begin exactly at 0x0A1
- Zone 9: block `0x5C`, offset `1176` (0x498) — confirmed in the write capture (`Powhatan`)
- Zone 28: block `0x5C`, offset `3931` (0xF5B); the first block ends at `16 + 28 × 145 = 4076`,
  leaving 19 spare bytes (`0xFEC`–`0xFFE`) plus the `0xFFF` metadata byte
- Zone 29: block `0x5D`, offset `0` (no header) ⚠️ DERIVED — not exercised by any capture
- Zone 57: block `0x5E`, offset `0` ⚠️ DERIVED

### Field Layout

```
┌─────────┬──────┬────────────────────┬──────────────────────────────────────┐
│ Offset  │ Size │ Field              │ Description                          │
├─────────┼──────┼────────────────────┼──────────────────────────────────────┤
│ 0x00-0A │  11  │ zone_name          │ ASCII, null-term, 0xFF padding       │
│ 0x0B-0F │   5  │ padding            │ 0xFF padding                         │
│ 0x10    │   1  │ channel_count      │ Number of channels in this zone      │
│ 0x11-90 │ 128  │ channel_list       │ Up to 64 channel numbers, 2 bytes LE │
└─────────┴──────┴────────────────────┴──────────────────────────────────────┘
Total: 145 bytes
```

Offsets are relative to the start of the 145-byte zone record, **not** to the block. Every field in
this table is confirmed against the hardware read capture (see the hex example below).

### Zone Name Encoding

- **Length**: 11 bytes maximum (10 chars + null terminator)
- **Format**: ASCII string
- **Termination**: Null-terminated (0x00)
- **Padding**: 0xFF for unused bytes after null
- **Empty zone**: all 0xFF or all 0x00

**Example** (exactly as read from hardware):
```
"Zone 1" → 5A 6F 6E 65 20 31 00 FF FF FF FF
           Z  o  n  e     1  \0 (padding)
```

An 11-character name fills the field with no terminator; the reference implementation writes at most
10 characters plus an explicit `0x00`.

### Channel List Format

- **Count byte** (offset 0x10 within zone): number of channels stored
- **Channel entries** start at offset 0x11 within zone
- **Entry format**: 16-bit little-endian channel number, **absolute channel numbers**, 1-based,
  stored as-is (no offset/bias)
- **Max slots**: 64 (128 bytes / 2 bytes per channel)
- **Empty slot / fill**: `0x0000` in the factory image, **`0xFFFF` in blocks written by the OEM
  CPS**. A parser must treat both as end-of-list
- **Valid range**: 0x0001–0x0FA0 (channels 1–4000)

Channel numbers here are 1-based and absolute — confirmed by the OEM CPS write capture, where zone
`Richmond` stores `01 00 02 00 03 00 04 00 05 00 07 00 … 06 00 80 00` (14 channels, note the
out-of-order `6` and the reference to channel `128` = `0x80`) against a 128-channel codeplug, and
zone `Beaverdam` stores channels 16–27.

**Example** (3 channels: 1, 15, 100):
```
Offset 0x10: 03           ← channel count = 3
Offset 0x11: 01 00        ← Channel 1
Offset 0x13: 0F 00        ← Channel 15
Offset 0x15: 64 00        ← Channel 100
Offset 0x17: 00 00 ...    ← empty / padding
```

**Hardware example** (read capture, first zone block, block offset 0x010):
```
0x010  5A 6F 6E 65 20 31 00 FF FF FF FF FF FF FF FF FF   name (11 B) + padding (5 B)
0x020  10 01 00 02 00 03 00 04 00 05 00 06 00 07 00 08   count = 16, then channels 1,2,3,…
0x030  00 09 00 0A 00 0B 00 0C 00 0D 00 0E 00 0F 00 10   … up to channel 16 (0x0010)
0x040  00 00 00 00 00 ...                                zero fill to the end of the record
0x0A1  46 75 6E 63 20 44 65 6D 6F 00 FF FF FF FF FF FF   zone 2 record starts at 0x0A1
0x0B1  09 11 00 12 00 13 00 14 00 15 00 16 00 17 00 18   count = 9, then channels 17,18,…
0x0C1  00 19 00 00 00 ...                                … up to channel 25, then zero fill
```

**Hardware example** (OEM CPS **write** capture, same block — note the `0xFF` fill):
```
0x000  09 01 FF 01 FF 01 FF FF FF FF FF FF FF FF FF FF   count = 9 zones + ❓ header bytes 1-15
0x010  52 69 63 68 6D 6F 6E 64 00 FF FF FF FF FF FF FF   "Richmond\0" + 0xFF padding
0x020  0E 01 00 02 00 03 00 04 00 05 00 07 00 08 00 09   count = 14, channels 1,2,3,4,5,7,8,9,…
0x030  00 0A 00 0B 00 0C 00 0D 00 06 00 80 00 FF FF FF   … 13, 6, 128, then 0xFF fill
0x0A1  42 65 61 76 65 72 64 61 6D 00 FF FF FF FF FF FF   zone 2 "Beaverdam" also at 0x0A1
```

### Terminator semantics (hardware-observed)

| Rule | Detail |
|---|---|
| Within a zone record | **Do not write a `0x0000` terminator after the channel list.** The reference implementation records a hardware regression: *"No 0x0000 terminator - pad with 0xFF only. Radio uses channel count (byte 16) to know how many channels to read. Writing 0x0000 caused the radio to show null slots / lose channels."* |
| What the OEM CPS writes | **`0xFF` fill** after the last channel, in 8 of the 9 zones it writes. This *agrees* with the reference implementation's rule above. *Previously documented here as "the OEM-written blocks read back with `0x00` fill, so `0x00` is what the radio holds" — incorrect: the `0x00` fill appears only in the **factory** image read off a virgin radio, which the OEM CPS never wrote.* **Pad with `0xFF`.** |
| Exception worth recording | In the write capture, zone 2 (`Beaverdam`, count 12) has a single `00 00` slot immediately after its 12 channels and `0xFF` thereafter, while zone 1 (`Richmond`, count 14) goes straight to `0xFF`. ❓ Unexplained; do not derive a rule from it |
| Between zone records | ⚠️ DERIVED **and contradicted by hardware**: the reference implementation writes a `0x0000` at `16 + zoneCount × 145` as an end-of-zones marker. The OEM CPS does **not** — after its 9th zone record (ending at byte 1320) the write capture holds `0xFF` all the way to the metadata byte. Emitting the marker is at best unnecessary |
| Parse stop | A record whose name starts `0x00`/`0xFF` **and** whose whole 145 bytes are `0x00`/`0xFF` ends the scan (zones are contiguous). A populated record with an empty name is skipped, not treated as the end |
| Channel-list stop | A channel number of `0` ends the channel list |

⚠️ Treat the count byte as authoritative but not perfectly trustworthy — captured records exist
whose populated slots disagree with it by one.

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x00-0x0A: Zone name (11 bytes)
    char zone_name[11];
    
    // 0x0B-0x0F: Padding (5 bytes, 0xFF)
    uint8_t padding[5];
    
    // 0x10: Channel count
    uint8_t channel_count;  // 0-64
    
    // 0x11-0x90: Channel list (128 bytes = 64 × 2-byte channel numbers)
    uint16_t channel_numbers[64];  // 0 = empty slot, little-endian
    
} dm32_zone_t;
#pragma pack(pop)

static_assert(sizeof(dm32_zone_t) == 145, "Zone structure must be 145 bytes");
```

### Code Structure (Python)

```python
from dataclasses import dataclass
from typing import List
import struct

@dataclass
class DM32Zone:
    """DM32 Zone Structure (145 bytes)"""
    
    zone_name: str
    channel_numbers: List[int]  # Up to 64 channels (1-4000)
    
    @classmethod
    def from_bytes(cls, data: bytes, zone_number: int) -> 'DM32Zone':
        """Parse zone from 4KB block"""
        if len(data) < 4096:
            raise ValueError("Zone data block must be 4096 bytes")
        
        # 16-byte block header, then 145-byte zone entries
        offset = 16 + (zone_number - 1) * 145
        if offset + 145 > 4096:
            raise ValueError(f"Zone {zone_number} offset out of range")
        
        zone_data = data[offset:offset+145]
        
        # Parse name (11 bytes, null-terminated)
        name_bytes = zone_data[0:11]
        name = name_bytes.split(b'\x00')[0].replace(b'\xFF', b'').decode('ascii', errors='ignore')
        
        # Channel count at byte 16 within zone entry
        channel_count = min(zone_data[16], 64)
        
        # Channel list starts at byte 17 within zone entry
        channels = []
        for i in range(channel_count):
            ch_offset = 17 + (i * 2)
            ch_num = struct.unpack('<H', zone_data[ch_offset:ch_offset+2])[0]
            if ch_num == 0:
                break
            if 1 <= ch_num <= 4000:
                channels.append(ch_num)
        
        return cls(zone_name=name, channel_numbers=channels)
    
    def to_bytes(self) -> bytes:
        """Convert zone to 145-byte record"""
        data = bytearray(145)
        data[:] = b'\xff' * 145  # Initialize to 0xFF
        
        # Name (11 bytes, null-terminated)
        name_bytes = self.zone_name.encode('ascii')[:10]
        data[0:len(name_bytes)] = name_bytes
        data[len(name_bytes)] = 0x00  # Null terminator
        # Bytes 11-15: 0xFF padding (already set)
        
        # Channel count at byte 16
        channel_count = min(len(self.channel_numbers), 64)
        data[16] = channel_count
        
        # Channel list starting at byte 17
        for i in range(channel_count):
            ch_offset = 17 + (i * 2)
            data[ch_offset:ch_offset+2] = struct.pack('<H', self.channel_numbers[i])
        
        return bytes(data)
```

### Code Structure (Go)

```go
package dm32

import (
    "encoding/binary"
    "fmt"
)

func bytesUntilNull(data []byte) int {
    for i, b := range data {
        if b == 0x00 || b == 0xFF {
            return i
        }
    }
    return len(data)
}

// Zone represents a DM32 zone (145 bytes)
type Zone struct {
    Name           string
    ChannelNumbers []uint16  // Up to 64 channels (1-4000)
}

// FromBytes parses a zone from the 4KB zone block
func (z *Zone) FromBytes(data []byte, zoneNumber int) error {
    if len(data) < 4096 {
        return fmt.Errorf("zone block must be 4096 bytes")
    }
    
    // 16-byte block header, then 145-byte zone entries
    offset := 16 + (zoneNumber-1)*145
    if offset+145 > 4096 {
        return fmt.Errorf("zone %d offset out of range", zoneNumber)
    }
    
    zoneData := data[offset : offset+145]
    
    // Parse name (11 bytes)
    z.Name = string(zoneData[:bytesUntilNull(zoneData[0:11])])
    
    // Channel count at byte 16
    channelCount := int(zoneData[16])
    if channelCount > 64 {
        channelCount = 64
    }
    
    // Channel list starting at byte 17
    z.ChannelNumbers = make([]uint16, 0, channelCount)
    for i := 0; i < channelCount; i++ {
        chOffset := 17 + i*2
        chNum := binary.LittleEndian.Uint16(zoneData[chOffset : chOffset+2])
        if chNum == 0 {
            break
        }
        if chNum >= 1 && chNum <= 4000 {
            z.ChannelNumbers = append(z.ChannelNumbers, chNum)
        }
    }
    
    return nil
}

// ToBytes converts zone to 145-byte record
func (z *Zone) ToBytes() []byte {
    data := make([]byte, 145)
    for i := range data {
        data[i] = 0xFF  // Initialize to 0xFF
    }
    
    // Name (11 bytes, null-terminated)
    copy(data[0:10], z.Name)
    if len(z.Name) < 10 {
        data[len(z.Name)] = 0x00
    }
    // Bytes 11-15: 0xFF padding (already set)
    
    // Channel count at byte 16
    channelCount := len(z.ChannelNumbers)
    if channelCount > 64 {
        channelCount = 64
    }
    data[16] = byte(channelCount)
    
    // Channel list starting at byte 17
    for i := 0; i < channelCount; i++ {
        chOffset := 17 + i*2
        binary.LittleEndian.PutUint16(data[chOffset:chOffset+2], z.ChannelNumbers[i])
    }
    
    return data
}
```

### Reading Zones from Radio

> **Single-block example.** The code below only covers zones 1–28 in block `0x5C`. A complete
> implementation must walk all nine zone blocks `0x5C`–`0x64` and restart the 145-byte packing at
> each block boundary (see [Offset Calculation](#offset-calculation) above). The reference
> implementation's legacy `writeZones()` has the same limitation and warns about it verbatim:
> *"this legacy single-block method only targets the first zone block and intentionally does not
> recognize the full 0x5c-0x64 zone range — its flat write-offset math below isn't block-boundary
> aware, so writing to multiple blocks here would silently misplace zone data."*

```python
def read_all_zones(radio_session):
    """Read all zones from radio"""
    zone_block_addr = find_block_by_metadata(radio_session, 0x5c)
    zone_data = radio_session.commands.read_memory(zone_block_addr, 4096)
    
    zones = []
    for zone_num in range(1, 29):  # 28 zones per 4KB block; blocks 0x5D-0x64 hold the rest
        offset = 16 + (zone_num - 1) * 145
        if offset + 145 > len(zone_data):
            break
        zone = DM32Zone.from_bytes(zone_data, zone_num)
        if zone.zone_name:
            zones.append(zone)
        elif zone_data[offset] == 0xFF:
            break  # Empty zone signals end
    
    return zones
```

⚠️ Multi-block caveat: when concatenating cached blocks before parsing, a missing block must still
consume its 4096 bytes of address space, or every later block's data lands 4096 bytes early.

---

## Scan List Structure (57 bytes)

**Metadata (logical block ID)**: 0x11 (17)  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Scan list definitions for channel scanning

Block `0x11` is confirmed as the scan-list block by the hardware read capture (`"Scan List 1"` at
block offset `0x001`, count `2` at `0x000`, 57-byte stride).

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseScanLists` /
`encodeScanList`), `src/radios/dm32uv/protocol.ts` (block write).

### Memory Organization

**Count Byte**: Offset 0x00 (1 byte) — number of scan lists (read capture `0x02`; write capture `0x09`, and the CPS does write 9 entries)  
**Entry Size**: 0x39 (57) bytes per scan list  
**Max Entries**: 32 scan lists ⚠️ DERIVED — a reference-implementation/UI limit, not a captured
fact; `(4096 − 1) / 57 = 71` entries would physically fit in one 4 KB block

**Offset Calculation**: Entry N (1-indexed): `(57 * N) - 56`

**Examples**:
- List 1: `(57 * 1) - 56 = 1` (offset 0x001) — hardware-confirmed
- List 2: `(57 * 2) - 56 = 58` (offset 0x03A) — hardware-confirmed
- List 44: `(57 * 44) - 56 = 2452` (offset 0x994)

> ❓ A "44 lists per block, first 44 start at offset 16" figure circulates in older notes; it
> contradicts the `(57 × N) − 56` formula and nothing in the captures supports it — disregard it.

**Note**: Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

### Field Layout (57 bytes per entry)

```
┌─────────┬──────┬───────────────────────┬──────────────────────────────────┐
│ Offset  │ Size │ Field Name            │ Description                      │
├─────────┼──────┼───────────────────────┼──────────────────────────────────┤
│ 0x00-0A │  11  │ scan_list_name        │ ASCII; NO terminator when the    │
│         │      │                       │ name is exactly 11 chars         │
│ 0x0B    │   1  │ channel_count         │ ⚠️ Does NOT match the number of   │
│         │      │                       │ channels present - see the boxed │
│         │      │                       │ note below                       │
│ 0x0C    │   1  │ ctc_tx_mode           │ CTC Scan Mode (bits 0-1) +       │
│         │      │                       │ Scan TX Mode (bits 2-3)          │
│ 0x0D    │   1  │ hang_time             │ ⚠️ DERIVED (see Field Values)     │
│ 0x0E    │   1  │ priority_types        │ Priority 1 type (bits 0-3) +     │
│         │      │                       │ Priority 2 type (bits 4-7)       │
│ 0x0F-10 │   2  │ priority_channel_1    │ Channel number, 16-bit LE        │
│ 0x11-12 │   2  │ designated_tx_channel │ Channel number, 16-bit LE (−2    │
│         │      │                       │ encoded: stored = actual - 2)    │
│ 0x13-14 │   2  │ priority_channel_2    │ Channel number, 16-bit LE (−2    │
│         │      │                       │ encoded: stored = actual - 2)    │
│ 0x15-16 │   2  │ unknown               │ ❓ UNKNOWN. Read cap: 0A 00 / 00  │
│         │      │                       │ A3. Write cap: 00 FE (all 9)     │
│ 0x17    │   1  │ unknown_flags         │ ❓ UNKNOWN. Read cap: 0x7F (both) │
│         │      │                       │ Write cap: 0xFF (all 9)          │
│ 0x18-19 │   2  │ ❓ first slot OR a     │ ⚠️ AMBIGUOUS - see the boxed note │
│         │      │ separate uint16 field │ below. Read cap: 0001. Write     │
│         │      │                       │ cap: 0000 (all 9 entries)        │
│ 0x1A-37 │  30  │ channel_list          │ 15 channel numbers, 2 bytes LE   │
│         │      │                       │ each; fill is 0x0000 (factory)   │
│         │      │                       │ or 0xFFFF (OEM CPS write)        │
│ 0x38    │   1  │ padding               │ ❓ UNKNOWN. Read cap 0x00, write  │
│         │      │                       │ cap 0xFF                         │
└─────────┴──────┴───────────────────────┴──────────────────────────────────┘
Total: 57 bytes (0x39)
```

> ### ⚠️ Where the channel list starts is **UNRESOLVED** (`+0x18` vs `+0x1A`)
>
> Two readings fit the read capture; the write capture strongly favours the second.
>
> | | Reading A — list at `+0x18`, `count` slots | Reading B — list at `+0x1A`, `count − 1` slots |
> |---|---|---|
> | `+0x18-0x19` | first channel slot | a separate ❓ UNKNOWN uint16 |
> | Max channels | 16 | 15 (matches `SCAN_LIST_CHANNELS_MAX = 15`) |
> | Read capture, list 1 (`count`=16) | 1,2,…,15,**15** | 2,3,…,15,**15** |
> | Read capture, list 2 (`count`=9) | 1,2,…,9 | 2,3,…,9 |
> | Write capture, all 9 lists (`count`=3) | **0**,14,15 — includes a channel number `0`, which is not a valid 1-based channel | 14,15 — two channels for a declared count of 3 |
>
> **The write capture is what breaks reading A.** All nine written lists are named `<SITE> Mon All`
> and all nine store `00 00` at `+0x18` followed by exactly the two channels that the same capture
> names `<SITE> Monitor TS1` and `<SITE> Monitor TS2`:
>
> | Scan list | `+0x18` … | Channels 1-based | Channel names in the same capture |
> |---|---|---|---|
> | `RIC Mon All` | `00 00 0e 00 0f 00` | 14, 15 | `RIC Monitor TS1`, `RIC Monitor TS2` |
> | `BEA Mon All` | `00 00 1c 00 1d 00` | 28, 29 | `BEA Monitor TS1`, `BEA Monitor TS2` |
> | `GCH Mon All` | `00 00 2a 00 2b 00` | 42, 43 | `GCH Monitor TS1`, `GCH Monitor TS2` |
>
> Under reading B the list is *semantically exactly right* for all nine — which is strong evidence
> that `+0x18-0x19` is **not** a channel slot and that the reference implementation's `+0x1A` is
> correct. Under reading A every list would contain a bogus "channel 0".
>
> What neither reading explains: `channel_count` at `+0x0B` is **3** while only two channels are
> present. Reading A absorbs the discrepancy by counting an empty slot; reading B requires the count
> to be `channels + 1`. Both are unattested.
>
> **Recommendation for implementers**: read the channel list at `+0x1A` (15 slots, stop on `0x0000`
> or `0xFFFF`), treat `+0x18-0x19` as an opaque uint16 to be preserved verbatim, and do **not** trust
> `+0x0B` as a channel count. Settle this by creating a scan list with a known channel set in the OEM
> CPS and diffing the block.

### Scan List Worked Example — real hardware (read capture)

Scan-list block, payload from block offset `0x000`:

```
block   entry
offset  offset  bytes                                 meaning
0x000     —     02                                    count = 2 scan lists
0x001   +0x00   53 63 61 6E 20 4C 69 73 74 20 31      "Scan List 1"  (11 B, no NUL)
0x00C   +0x0B   10                                    channel_count = 16
0x00D   +0x0C   03                                    ctc_tx_mode (CTC mode 3, TX mode 0)
0x00E   +0x0D   06                                    hang_time raw value
0x00F   +0x0E   00                                    priority_types (both = None)
0x010   +0x0F   01 00                                 priority_channel_1 raw = 1
0x012   +0x11   00 00                                 designated_tx_channel raw = 0
0x014   +0x13   00 00                                 priority_channel_2 raw = 0
0x016   +0x15   0A 00 7F                              ❓ unknown (note the 0x7F)
0x019   +0x18   01 00                                 ❓ ambiguous: first channel, or a field
0x01B   +0x1A   02 00 03 00 … 0F 00 0F 00             channel list (15 slots, ends at +0x37)
0x039   +0x38   00                                    padding
0x03A   +0x00   53 63 61 6E 20 4C 69 73 74 20 32      entry 2 at 0x03A = (57×2)−56
0x045   +0x0B   09                                    channel_count = 9
0x046   +0x0C   03 06 00 01 00 00 00 00 00 00 A3 7F   same fields; unknown = 00 A3 7F
0x052   +0x18   01 00                                 ❓ ambiguous (same as above)
0x054   +0x1A   02 00 … 09 00 00 00                   channels 2…9, then 0x0000 fill
```

And the same region of the **write** capture (9 scan lists, OEM CPS → radio), entry 1:

```
block   entry
offset  offset  bytes                                 meaning
0x000     —     09                                    count = 9 scan lists
0x001   +0x00   52 49 43 20 4D 6F 6E 20 41 6C 6C      "RIC Mon All"  (11 B, no NUL)
0x00C   +0x0B   03                                    channel_count = 3
0x00D   +0x0C   03 06 00                              ctc_tx_mode / hang_time / priority_types
0x010   +0x0F   01 00  00 00  00 00                   priority_1 / designated_tx / priority_2
0x016   +0x15   00 FE FF                              ❓ unknown (0xFF here, not 0x7F)
0x019   +0x18   00 00                                 ❓ ambiguous — 0x0000 in ALL NINE entries
0x01B   +0x1A   0E 00 0F 00 FF FF FF …                channels 14, 15 then 0xFFFF fill
0x038   +0x38   FF                                    padding is 0xFF here, not 0x00
```

❓ Anomalies, recorded verbatim rather than turned into rules:

- Read-capture entry 1 declares `channel_count = 16` but its last two slots both hold channel 15
  (`0F 00 0F 00`).
- Every write-capture entry declares `channel_count = 3` while carrying two channels plus the
  ambiguous `+0x18` word.
- `+0x15`…`+0x17` and `+0x38` hold different values in the two captures, so none of them is a
  constant — preserve them verbatim.

### Field Values

All of the enumerations below are ⚠️ **DERIVED** — the labels come from the reference
implementation's UI, not from a capture or from CPS disassembly. **Both** captures store
`0x0C = 0x03` and `0x0D = 0x06` in *every* scan list — the factory radio's 2 and the OEM CPS's 9 —
so neither capture disambiguates these enumerations at all.

**CTC Scan Mode** (bits 0-1 of byte 0x0C) ⚠️ DERIVED:
- 0 = Not Detection CTC
- 1 = Detection CTC Non Priority
- 2 = Detection CTC Priority
- 3 = Detection CTC

**Scan TX Mode** (bits 2-3 of byte 0x0C) ⚠️ DERIVED:
- 0 = Current Channel
- 1 = Last Active Channel
- 2 = Designated Channel

> ❓ Open contradiction: the reference implementation's UI labels Scan TX Mode 0/1/2 as
> Current / Last Active / Designated, while its *parser* treats 0/1/2 as None / Current / Specific
> when decoding `designated_tx_channel`. One of the two is wrong; unresolved.

**Priority Type** (per-priority nibble in byte 0x0E) ⚠️ DERIVED:
- 0 = None
- 1 = Current Channel
- 2 = Specific/Designated Channel (number stored in the matching priority_channel field)

Note the raw `priority_channel_1` in the capture is `1` while `priority_types` is `0` (None) — the
channel field holds a stale/default value when the type says None, so **always gate the channel
number on the type nibble**.

**Hang Time** (byte 0x0D) ⚠️ DERIVED: documented as tenths of a second (1–255 → 0.1 s … 25.5 s),
which is what the reference implementation assumes. The captured radio stores `0x06`, i.e. 0.6 s
under that reading — implausibly short for a factory default, so the unit is **not** established.
A stored `0` decodes as "unset" in the reference implementation.

**Maximum Channels per Scan List**: **15** under the recommended reading (30 bytes at
`+0x1A`…`+0x37`), or 16 if `+0x18-0x19` turns out to be the first slot — see the boxed note above.
The reference implementation uses 15 (`SCAN_LIST_CHANNELS_MAX`). *An earlier revision of this
document asserted 16 as confirmed; that is retracted — the write capture does not support it.*

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x00: Scan list name (11 bytes). NOTE: a name of exactly 11 characters has NO
    // null terminator - do not assume one when decoding.
    char scan_list_name[11];
    
    // 0x0B: Channel count. WARNING: in both captures this byte is one MORE than the number of
    // channels actually present at 0x1A. Do not size the list from it.
    uint8_t channel_count;
    
    // 0x0C: CTC/TX mode byte
    // Bits 0-1: CTC Scan Mode (0=Not Detection, 1=Non Priority, 2=Priority, 3=Detection)
    // Bits 2-3: Scan TX Mode (0=Current, 1=Last Active, 2=Designated)
    uint8_t ctc_tx_mode;
    
    // 0x0D: Hang time - unit DERIVED (assumed tenths of a second); hardware value 0x06
    uint8_t hang_time;
    
    // 0x0E: Priority types
    // Bits 0-3: Priority 1 type (0=None, 1=Current, 2=Designated)
    // Bits 4-7: Priority 2 type (0=None, 1=Current, 2=Designated)
    uint8_t priority_types;
    
    // 0x0F-0x10: Priority channel 1 (16-bit LE)
    uint16_t priority_channel_1;
    
    // 0x11-0x12: Designated TX channel (16-bit LE, stored as actual - 2)
    uint16_t designated_tx_channel;
    
    // 0x13-0x14: Priority channel 2 (16-bit LE, stored as actual - 2)
    uint16_t priority_channel_2;
    
    // 0x15-0x16: Unknown (2 bytes). Read capture: "0A 00" / "00 A3". Write capture: "00 FE".
    uint8_t unknown[2];
    
    // 0x17: Unknown flags byte. Read capture: 0x7F (both lists). Write capture: 0xFF (all 9).
    uint8_t unknown_flags;
    
    // 0x18-0x19: AMBIGUOUS - either the first channel slot or a separate uint16 field.
    // Read capture: 0x0001 in both lists. Write capture: 0x0000 in all nine lists.
    // See the boxed note above; preserve this word verbatim.
    uint16_t unknown_or_first_channel;
    
    // 0x1A-0x37: Channel list (30 bytes = 15 x 2-byte channel numbers, LE)
    uint16_t channel_numbers[15];  // fill is 0x0000 (factory image) or 0xFFFF (OEM CPS write)
    
    // 0x38: Padding / unknown. Read capture 0x00, write capture 0xFF.
    uint8_t padding;
    
} dm32_scan_list_t;
#pragma pack(pop)

static_assert(sizeof(dm32_scan_list_t) == 57, "Scan list structure must be 57 bytes");
```

### Implementation guidance — preserving the scan-list record

- **Preserve `+0x15`…`+0x19` verbatim.** They carry data on hardware (`0A 00 7F` / `00 A3 7F` in
  the factory image, `00 FE FF` in every OEM-written entry) and `+0x18-0x19`'s meaning is
  unresolved — writing zeros there may be destructive.
- **Do not trust `+0x0B` as a channel count** (it disagrees with the populated slots in both
  captures); terminate the list on `0x0000`/`0xFFFF`.
- **Accept an unterminated 11-character name** — that is what the radio itself writes.

---

## RX Group List Structure

**Metadata**: 0x0F (15)  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR receive group lists (the set of talk-group DMR IDs a channel will receive)

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseRXGroups` / `encodeRXGroups`), `src/radios/dm32uv/protocol.ts` (`readRXGroups` / `writeRXGroups`)

**Do not confuse RX groups with contacts.** An RX-group entry is **109 bytes (0x6D)**. A DMR
**contact** entry is **92 bytes (0x5C)** and lives in a completely different, raw (non-metadata)
region located via V-frame 0x0F — see the *DMR Contact Database* section near the end of this
document.

### Memory Organization

Both hardware captures agree on this layout.

- **Factory read capture**, block 0x0F: `0x000 = 1F 00 00 00`, `0x004-0x00F = 00`, `0x010 = 01`,
  `"RX Group 1"` at 0x011 and `"RX Group 2"` at 0x07E — a **109-byte stride** — with five populated
  entries and entries 5+ all zero. Byte 0x000 = 0x1F is therefore a **bitmask with bits 0-4 set
  (5 groups)**, not a plain count of 31.
- **OEM CPS write capture**, same block: `0x000 = 01 00 00 00` (one group), `0x004 = 01`,
  `0x005-0x00F = FF`, `0x010 = 01`, one populated entry at 0x011.

**Block header (17 bytes, 0x000-0x010)**

| Offset | Size | Field | Encoding |
|--------|------|-------|----------|
| 0x000 | 4 | Active-group bitmask | uint32 LE; bit *n* set = group slot *n* is in use. Read capture `0x0000001F` = 5 groups; write capture `0x00000001` = 1 group |
| 0x004 | 1 | ❓ UNKNOWN — **carries data** | Factory read = `0x00` (5 groups); OEM write = `0x01` (1 group). Not a plain group count — preserve it |
| 0x005 | 11 | ❓ UNKNOWN | Factory read = 0x00, OEM write = 0xFF — preserve |
| 0x010 | 1 | Header flag | 0x01 in both captures |
| 0x011 | — | First entry | entries follow contiguously |

> **Previously documented as** "0x004 | 12 | Reserved — the encoder writes 0x00". **Incorrect**:
> 0x004 is not reserved, the OEM CPS writes 0x01 there. "Reserved" is an unsupported claim.

**Entry Calculation**: `offset = 0x11 + entry_num * 0x6D` (entry_num 0-based)

**Max Entries**: **32**, limited by the 32-bit bitmask. Entry 31 occupies 0x0D44-0x0DB0.
(A doc comment in the reference implementation estimates "~37 entries = ⌊4096 / 109⌋"; that ignores
both the 17-byte header and the bitmask cap. 32 is the enforced limit.)

**Count derivation is asymmetric in the reference implementation**: the parser uses
`(highest set bit index) + 1`, so a sparse mask such as `0b1001` yields 4 groups; the encoder always
sets bits `0..N-1` contiguously, so gaps cannot survive a write.

**Note**: Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

### Entry Structure (109 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 11 | Name | ASCII, null-terminated, 0x00-padded. Encoder writes at most **10** chars + terminator. (The OEM CPS does not scrub the tail: the write capture holds `"DMRVA\0up 1\0"` in this field — bytes after the terminator are stale.) |
| +0x0B | 96 | DMR ID slots | 32 slots × 3 bytes, uint24 LE. `0x000000` or `0xFFFFFF` = empty; parse stops at the first empty slot |
| +0x6B | 1 | ❓ UNKNOWN — **carries data** | OEM write capture holds **0x02** here on a group with 14 IDs; factory read holds 0x00. Purpose unestablished — preserve it |
| +0x6C | 1 | ❓ UNKNOWN | 0xFF in the OEM write, 0x00 in the factory read |

> **Previously documented as** "+0x6B | 2 | Padding — zero-filled by the encoder, never observed
> carrying data". **Incorrect**: the OEM CPS wrote `02 FF` there. Do not treat it as padding.

**The slots hold raw 24-bit DMR IDs, not talk-group indices — CONFIRMED.** In the OEM write capture
the single RX group `"DMRVA"` carries the slot values
`43277, 27500, 9999, 9998, 31511, 27501, 27502, 27503, 31516, 31514, 31513, 3151, 31515, 3154`,
which are exactly the 24-bit **contact numbers** of 14 of the 16 talk groups written to block 0x44 in
the same session (`HEARS Link` = 43277, `Local` = 27500, `Audio Test` = 9999, `Parrot` = 9998,
`RVA Metro` = 31511, …). They are not 1-based slot indices — the largest is far beyond the 16 talk
groups that exist.

(The factory read capture cannot distinguish the two readings on its own: its RX groups hold
`1,2,3,4,5` and its talk groups happen to have contact numbers 1-10.)

**Fill bytes**: an entry being written is zero-filled first; entry slots beyond the active group
count are filled with **0xFF**.

**Empty-entry detection**: all-0xFF ⇒ skip; empty name **and** zero IDs ⇒ skip. A missing name is
substituted with `RX Group <n>` by the parser.

**Round-trip guidance**: preserve the unknown header bytes (`0x004`-`0x00F`) and entry tail bytes
(`+0x6B`/`+0x6C`) — both carry data on hardware.

> (*An entry layout with "negative offsets" (`entry_base − 0x5C` etc.) was previously documented
> here. It was the **block
> header** at 0x000-0x010 mistaken for a per-entry header — nothing in the format uses negative
> offsets. The hardware dump shows the group name at the start of each entry.*)

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x000: Active-group bitmask (uint32 LE), bit n = slot n in use
    uint32_t bitmask;

    // 0x004: UNKNOWN - carries data (0x01 in the OEM CPS write capture)
    uint8_t  unknown_004;

    // 0x005: UNKNOWN (0x00 factory / 0xFF OEM write)
    uint8_t  unknown_005[11];

    // 0x010: Header flag, always 0x01
    uint8_t  header_flag;
} dm32_rx_group_header_t;   // 17 bytes, entries start at 0x011

typedef struct {
    // +0x00: Name (11 bytes, null-terminated, 0x00 padding)
    char     name[11];

    // +0x0B: 32 talk-group DMR IDs, 3 bytes each, little-endian
    //        0x000000 / 0xFFFFFF = empty slot
    uint8_t  dmr_ids[32][3];

    // +0x6B: UNKNOWN - carries data (0x02 in the OEM CPS write capture), NOT padding
    // +0x6C: UNKNOWN
    uint8_t  unknown_6b[2];
} dm32_rx_group_entry_t;
#pragma pack(pop)

static_assert(sizeof(dm32_rx_group_header_t) == 17, "RX group header must be 17 bytes");
static_assert(sizeof(dm32_rx_group_entry_t) == 109, "RX group entry must be 109 bytes");

// offset of entry n = 0x11 + n * 0x6D
```

---

## Additional Metadata Block Structures

The following sections provide detailed byte-level parsing specifications for other metadata blocks. All metadata blocks are **4096 bytes (0x1000)** when read from the radio (except where noted). Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

---

## 0x02 - Frequency Adjustment/Calibration Data

**Metadata / logical block ID**: 0x02  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Factory RF calibration / frequency-adjustment data  
**Access**: **read-only** — nothing writes this block

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseCalibration`), `src/radios/dm32uv/protocol.ts` (`readCalibration`), `src/models/Calibration.ts`

### Read-only status (hardware evidence)

The OEM CPS reads this block in full (one 4 KB read in the factory read capture) and **never writes
it**: block 0x02 does not appear among the 75 logical block IDs the CPS walks in the write capture.
The reference implementation matches — it has a parser and **no encoder**, and its UI labels the
calibration view `READ-ONLY` / *"Display Only: This is factory calibration data"*.

The reference implementation selects this block by *discovered block type* (`type === 'calibration'`,
i.e. metadata byte 0x02) rather than by metadata value directly — it is the only region handled that
way.

### Structure Overview

Dense binary. In the factory read capture the populated area runs from 0x001 to **0x34F**
(interleaved with 0xFF holes); everything from 0x350 to 0xFFE is 0x00/0xFF. There are **no strings**
— the only printable-looking runs are coincidental binary (`lIHEA@`, `db^\]QMKJK`) — so, unlike every
other block in this document, it cannot be anchored by name. Every offset in the field table below
comes from the reference implementation alone.

**One region can nevertheless be anchored.** Offsets
0x300-0x34F of the factory read capture are twenty consecutive 4-byte fields that decode cleanly with
the *channel* frequency codec (little-endian byte order, BCD nibble pairs, 8 digits, `XXX.XXXXX`):

| Offset | Raw | Decoded | | Offset | Raw | Decoded |
|--------|-----|---------|---|--------|-----|---------|
| 0x300 | `00 00 00 40` | 400.00000 MHz | | 0x328 | `00 00 60 13` | 136.00000 MHz |
| 0x304 | `00 00 75 41` | 417.50000 MHz | | 0x32C | `00 00 55 14` | 145.50000 MHz |
| 0x308 | `00 00 50 43` | 435.00000 MHz | | 0x330 | `00 00 50 15` | 155.00000 MHz |
| 0x30C | `00 00 25 45` | 452.50000 MHz | | 0x334 | `00 00 45 16` | 164.50000 MHz |
| 0x310 | `00 00 00 47` | 470.00000 MHz | | 0x338 | `00 00 40 17` | 174.00000 MHz |

0x314-0x327 repeats the five UHF values byte-for-byte and 0x33C-0x34F repeats the five VHF values, so
the region is **2 × 5 U-band + 2 × 5 V-band** frequencies — the shape the parameter names imply
("U … 1-5", "V … 1-5": five calibration points per band). Smaller frequency-shaped fields occur
elsewhere: 0x030-0x03B decodes as 400.22500 / 435.22500 / 469.98500 MHz.

⚠️ **DERIVED** — the bytes are hardware, the reading is inference (nothing labels these fields). It
matters because the reference implementation's two "frequency arrays" are based at **0x00 and 0x3C**,
spanning at most 0x000-0x133 and 0x03C-0x16F, so **neither reaches 0x300-0x34F** — the only
contiguous run of frequency-shaped values anywhere in the block.

The field table below documents what the reference implementation does. Five parallel arrays, indexed
by parameter number **1-77** (1-based in every loop).

### Field Layout — ⚠️ DERIVED, and internally inconsistent

| Array | Base offset | Element | Stride | Params | Values skipped as "unset" |
|--------|------|-------|----------|------|------|
| Frequency array 1 | 0x00 | uint32 **little-endian** | 4 | 1-77, offset = `(param-1)*4` | `0x00000000`, `0xFFFFFFFF` |
| Frequency array 2 | 0x3C (60) | uint32 LE | 4 | 1-77 | `0x00000000`, `0xFFFFFFFF` |
| Value array 1 | 0x7E (126) | uint16 LE | 2 | 1-77 | `0x0000`, `0xFFFF` |
| Value array 2 | 0x9E (158) | uint16 LE | 2 | 1-77 | `0x0000`, `0xFFFF` |
| Value array 3 | 0xB0 (176) | uint16 LE | 2 | 1-77 | `0x0000`, `0xFFFF` |

**Do not build a tool on this table.** Three separate problems:

1. **The bases and the parameter count cannot all be true.** 77 params × 4 bytes from 0x00 spans
   0x000-0x133, which swallows frequency array 2's base (0x3C) *and* both of the first two
   value-array bases. Either the arrays are shorter than 77 entries, or the bases are wrong, or the
   arrays are interleaved rather than contiguous. Unresolved. ❓
2. **Base 0x00 is a guess.** The original CPS decompilation implies a nonsensical "offset −4 for
   param 0"; base 0x00 is the charitable reinterpretation, not a fact.
3. **BCD vs raw.** The `0x300-0x34F` band table above is unmistakably 4-byte little-endian BCD in
   the standard channel codec (8 digits, `XXX.XXXXX`), so *some* fields in this block carry
   frequencies — but which of the five arrays (if any) is BCD remains unestablished. Store raw
   4-byte values and decode on demand.

### Calibration Parameters (1-77)

The CPS exposes exactly 77 named parameters. The mapping of *name* to *array element* inherits all
the uncertainty above.

| # | Name | # | Name |
|---|------|---|------|
| 1 | RX Freq Adjust | 43-45 | U Analog Mod Low / Mid / High |
| 2 | TX Freq Adjust | 46-48 | V Analog Mod Low / Mid / High |
| 3-7 | U 4FSK 1-5 | 49 | DCS Mod |
| 8-12 | V 4FSK 1-5 | 50 | CTCSS Mod |
| 13-17 | U Low Power 1-5 | 51-55 | U SQL9 1-5 |
| 18-22 | U Mid Power 1-5 | 56-60 | U SQL3 1-5 |
| 23-27 | U High Power 1-5 | 61-65 | V SQL9 1-5 |
| 28-32 | V Low Power 1-5 | 66-70 | V SQL3 1-5 |
| 33-37 | V Mid Power 1-5 | 71 | Battery Adjust |
| 38-42 | V High Power 1-5 | 72-74 | U RX Bit Error Low / Mid / High |
| | | 75-77 | V RX Bit Error Low / Mid / High |

### Suggested experiment

Calibration is the only block in the codeplug that is read but never written, so it can be probed
without risk: dump block 0x02 from two radios of the same model and diff. Per-unit calibration values
will differ; structural constants will not. That alone would separate the arrays.

---

## 0x03 - "Call" list — layout confirmed, purpose `❓ UNKNOWN`

**Metadata**: 0x03  
**Block Size**: 4 KB (0x1000 bytes)  
**Status**: Record layout **CONFIRMED** from the read capture. Semantics `❓ UNKNOWN`.

This block **is** real on-radio codeplug data. The OEM CPS both **reads and writes** it
(read capture: full 4 KB read of page `0x011000`; write capture: full 4 KB write). It is the only
block classified `❓ UNKNOWN` that the CPS actually touches.

> **Previously documented as**: "Digital Emergency Systems". **Incorrect** — digital emergency
> systems live in block `0x10` at offset `0x000` (`"DEmer 1"`…`"DEmer 8"`, confirmed by hardware).
>
> A later revision then dismissed this block entirely as "a CPS-internal UI buffer (`DAT_004e80d4`),
> not a mirror of what the radio stores on-flash", and **deleted the record layout**. That was an
> over-correction: the hardware capture reproduces the deleted numbers *exactly* — entry base
> `0x218`, entry stride `0x28`, UTF-16LE names. Those figures are restored below. Only the
> *meaning* of the records was ever unknown.

### Entry array

**Entry base**: `0x218` **Entry stride**: `0x28` (40 bytes)
**Entry N**: `0x218 + N * 0x28` — capacity to the end of the block is 92 entries `⚠️ DERIVED`
(no capture populates more than 6).

| Offset | Size | Field | Observed |
|--------|------|-------|----------|
| `+0x00` | 2 | Flags / in-use marker | `FE FF` on populated entries, `FF FF` on empty |
| `+0x02` | 2 | Reference A — uint16 LE | `❓ UNKNOWN` referent |
| `+0x04` | 2 | Reference B — uint16 LE | `❓ UNKNOWN` referent |
| `+0x06` | 2 | Padding | always `FF FF` |
| `+0x08` | 32 | **Name — UTF-16LE**, NUL-terminated, max 16 chars | `"Call 1"`…`"Call 5"` |

**This is the only block in the codeplug that stores names as UTF-16LE.** Every other block uses
single-byte ASCII. Decode with `WideCharToMultiByte` / `TextDecoder('utf-16le')`.

### Hardware evidence (read capture, page `0x011000`)

| N | Offset | Flags | Ref A | Ref B | Name |
|---|--------|-------|-------|-------|------|
| 0 | `0x218` | `FF FF` | `0xFFFF` | `0xFFFF` | `"Call 1"` |
| 1 | `0x240` | `FE FF` | `0x0C91` (3217) | `0x2441` (9281) | `"Call 2"` |
| 2 | `0x268` | `FE FF` | `0x0C91` (3217) | `0x17CA` (6090) | `"Call 3"` |
| 3 | `0x290` | `FE FF` | `0x0C91` (3217) | `0x4FD6` (20438) | `"Call 4"` |
| 4 | `0x2B8` | `FE FF` | `0x2441` (9281) | `0x17CA` (6090) | `"Call 5"` |
| 5 | `0x2E0` | `FE FF` | `0x2441` (9281) | `0x4FD6` (20438) | *(empty)* |
| 6 | `0x308` | `FF FF` | `0xFFFF` | `0xFFFF` | *(empty)* |

The two reference fields draw from a **set of exactly four values** — `0x0C91`, `0x2441`, `0x17CA`,
`0x4FD6` — and the six populated entries enumerate **every unordered pair** of that set
(C(4,2) = 6). The same four values appear consecutively at block offset `0x030`:

```
0x030  05 05 05 05 00 ff 00 91 0c 41 24 ca 17 d6 4f ff   |.........A$...O.|
                          └─ 0x0C91 ─┘└0x2441┘└0x17CA┘└0x4FD6┘
```

They match **nothing else in the codeplug** — not the DMR radio IDs in block `0x67`
(123, 2, 3, 4, 5), not the talk-group contact numbers in `0x44` (1, 2, 3 …), not any channel or
zone index. The exhaustive-pairs pattern is consistent with factory demo data.

### Other populated regions

| Offset | Content | Status |
|--------|---------|--------|
| `0x001` | `05` — possibly an entry count (5 named entries exist) | `⚠️ DERIVED` |
| `0x030`–`0x04D` | header/config: `05 05 05 05 00 FF 00`, the four reference values, then `0A 01 03 FF FF 03 01` | `❓ UNKNOWN` |
| `0x730`–`0x778` | index-like runs `01 02 03 04 05 …` and `F1 01 02 …` / `F2 02 02 …` | `❓ UNKNOWN` |
| `0x8A0`, `0x8D0` | ASCII `"Disable"` / `"Enable"` — an option-label table | `❓ UNKNOWN` |
| everything else | `0xFF` fill | — |

**To identify this block**: it is written by the CPS, so it is driven by some CPS dialog. Toggle
DMR call-related settings one at a time and diff. See
[07-BLOCK-INVENTORY.md § C.4 Q3](07-BLOCK-INVENTORY.md#c4-what-is-still-unknown--prioritised-with-an-experiment-for-each).

---

## 0x04 - Embedded Information / Radio Names

**Metadata / logical block ID**: 0x04  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: **Radio Settings** — the radio's single global configuration block: power-on display,
display/UI colours, GPS, digital (DMR) settings, key lock and button assignments, one-key operation,
APRS, and the menu enable/disable flags

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseRadioSettings` / `encodeRadioSettings`), `src/radios/dm32uv/protocol.ts` (`readRadioSettings` / `writeRadioSettings`), `src/radios/dm32uv/blockLayouts.ts`, `src/radios/dm32uv/settingsProfile.ts`, `src/models/RadioSettings.ts`

> **The heading is historical** — "Embedded Information / Radio Names" described only the two
> power-on display strings; this block is the whole Radio Settings area. The heading is kept so
> cross-document links do not break.
>
> ⚠️ **VFO A / VFO B are not in this block.** They are 48-byte channel records in block **0x41** —
> see *0x41 - VFO A / VFO B Channel Records*.

### Hardware anchors

Both serial captures contain a full 4 KB read/write of this block, so most of the map below is
anchored directly in hardware. Factory read capture, first 0xB0 bytes:

```
0000  00 57 65 6c 63 6f 6d 65 00 d3 c3 00 00 00 00 44   .Welcome.......D
0010  4d 2d 33 32 55 56 00 fa 00 00 00 00 00 00 00 00   M-32UV..........
0020  7f c0 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0030  04 01 02 11 00 00 00 00 00 00 00 00 00 00 00 00   ................
0040  48 14 05 00 00 01 00 00 00 00 00 00 00 00 00 00   H...............
0050  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0060  03 02 0c 03 03 2b 00 00 00 00 00 00 00 00 00 00   .....+..........
0070  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0080  be 01 00 00 00 00 00 16 18 19 01 00 00 02 0f 07   ................
0090  28 00 00 02 00 00 00 00 00 00 00 00 00 00 00 00   (...............
00A0  16 05 00 0a 36 23 04 03 00 04 00 00 00 00 00 00   ....6#..........
```

Power-on line 1 = `"Welcome"` at 0x001, line 2 = `"DM-32UV"` at 0x00F. The OEM write capture carries a
different codeplug in the same block (line 1 `"DMRVA2.6"`, line 2 `"2025-07-03"`), which independently
confirms both string offsets and bounds line 1's field at 14 bytes.

### Read / write contract

| Step | Behaviour |
|------|-----------|
| Locate | by metadata/logical ID 0x04; if the block is absent the reference implementation returns `null` and continues (*"radio may not support this feature"*) |
| Parse precondition | input must be ≥ **0x508** bytes, else *"Radio Settings data must be at least 1288 bytes (0x508)"* |
| Encode output | always a full 4096-byte block |
| Preservation | the original block is copied first and only changed fields are overwritten, so every ❓ UNKNOWN byte survives a write. If no original is supplied the buffer is filled with **0xFF** |
| Write guard | the reference implementation **refuses to write this block without having read it first**: *"Cannot write Radio Settings without reading from radio first. Original block data is required to preserve unknown fields."* Any independent implementation should adopt the same rule |
| Metadata | the encoder re-stamps `data[0xFFF] = 0x04` |

### Power-on region (0x000-0x02F)

| Offset | Bits | Field | Encoding | Confidence |
|--------|------|-------|----------|-----------|
| 0x000 | 7-0 | Power On Interface | 0 = Power On Picture, 1 = Custom Message, 2 = Battery Volt; clamped to 0-2 on write | |
| 0x001-0x00E | — | Power On Display Line 1 | 14-byte ASCII field, NUL-terminated. Parser reads 14; encoder writes at most **13** chars + NUL | |
| 0x00F-0x01C | — | Power On Display Line 2 | same, 14-byte field | |
| 0x01D | 0 | Allow Reset | bool; read-modify-write on bit 0 only | ⚠️ DERIVED |
| 0x01D | 7-1 | — | preserved | ❓ UNKNOWN |
| 0x01E | 7-0 | Auto Power Off | 0-5 (0=Off, 1=30 min, 2=60, 3=120, 4=240, 5=480) | ⚠️ DERIVED |
| 0x01F | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |
| 0x020 | 7-0 | Alert Tone flags | bitfield, written whole | ⚠️ DERIVED |
| 0x021 | 7-0 | Alert Tone flags (cont.) | bitfield, written whole | ⚠️ DERIVED |
| 0x022-0x02F | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |

Alert-tone bit labels come from the reference implementation's settings UI schema, not from a
capture — treat the whole of both tables as ⚠️ DERIVED.

| 0x020 bit | Label | | 0x021 bit | Label |
|---|---|---|---|---|
| 0 | Key Press Tone | | 0 | Battery Low |
| 1 | Key Release Tone | | 1 | Analog TX End Tone |
| 2 | Menu Exit Tone | | 2 | Analog TX Alert Tone |
| 3 | Call End Tone | | 3-7 | ❓ UNKNOWN — the model claims a 2-bit field here, unlabelled |
| 4 | Talk Permit Tone | | | |
| 5 | StartUp Sound | | | |
| 6 | Voice Prompt | | | |
| 7 | Scan Stop Tone | | | |

Factory dump has 0x020 = 0x7F, 0x021 = 0xC0 — i.e. bits 6-7 of 0x021, which no label covers, carry
data.

### Display / UI region (0x030-0x03B)

| Offset | Bits | Field | Encoding | Confidence |
|--------|------|-------|----------|-----------|
| 0x030 | 7-0 | Backlight Brightness | **stored 0-5, displayed 1-6** (`raw+1`) | |
| 0x031 | 7-0 | Auto Backlight Duration | stored 0-5 → 5-30 s step 5 (`(raw+1)*5`) | ⚠️ DERIVED |
| 0x032 | 7-0 | — | round-tripped verbatim, not exposed in any UI. 0x02 in the factory dump | ❓ UNKNOWN |
| 0x033 | 7-0 | Display flags | bitfield | ⚠️ DERIVED |
| 0x033 | 3 | Data Display Format | 0 = yyy/m/d, 1 = d/m/yyy | ⚠️ DERIVED |
| 0x034 | 3-0 | Callsign Colour | low nibble only; high nibble preserved on write | |
| 0x035 | 3-0 | Standby Text Colour | low nibble | |
| 0x036 | 7-0 | Menu Exit Time | 1-30 | ⚠️ DERIVED |
| 0x037 | 7-0 | Standby Character Colour 1 | 0-30 | ⚠️ DERIVED |
| 0x038 | 3-0 | Channel A Colour | low nibble | |
| 0x039 | 3-0 | Channel B Colour | low nibble | |
| 0x03A | 3-0 | Zone A Colour | low nibble | |
| 0x03B | 3-0 | Zone B Colour | low nibble | |
| 0x03C-0x03F | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |

**Standby Character Colour 2 has no known offset.** ❓ UNKNOWN.

Colour enum (all seven colour fields): 0 White, 1 Black, 2 Orange, 3 Red, 4 Yellow, 5 Green,
6 Cyan, 7 Blue. Values 8-15 are storable (the mask is 0x0F) but ❓ UNKNOWN.

0x033 bit labels: bit 0 Volume Change Prompt, bit 1 Time Display, bit 3 Data Display Format;
bits 2 and 4-7 ❓ UNKNOWN. The factory dump has 0x033 = 0x11, so bit 4 — unlabelled — carries data.

### GPS region (0x040-0x045)

The reference implementation marks this range *"confirmed via CPS RE"*, and the factory dump decodes
consistently (0x040 = 0x48, 0x041 = 0x14 = UTC+8, 0x042 = 0x05 = the 5 s minimum).

| Offset | Bits | Field | Values | Confidence |
|--------|------|-------|--------|-----------|
| 0x040 | 0 | GPS Switch | 0 = Off, 1 = On | |
| 0x040 | 1 | Distance Unit | 0 = Metric, 1 = British | |
| 0x040 | 3-2 | GPS Mode | 0 = GPS, 1 = BDS, 2 = GPS+BDS | |
| 0x040 | 5-4 | Speed Unit | 0 = Kph, 1 = Mph, 2 = Kts | |
| 0x040 | 6 | GPS Display Format | 0 = Degree, 1 = Deg/Min/Sec | |
| 0x040 | 7 | — | explicitly preserved on write (*"preserve unknown bits"*) | ❓ UNKNOWN |
| 0x041 | 7-0 | UTC Zone | 0-25 = UTC−12 … UTC+13; **12 = UTC** | |
| 0x042 | 7-0 | GPS Report Interval | raw **is** the value in seconds; 5-255 | |
| 0x043-0x044 | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |
| 0x045 | 7-0 | — | round-tripped verbatim; model says *"8 bits, partially identified"*. 0x01 in the factory dump | ❓ UNKNOWN |
| 0x046-0x05F | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |

### Digital (DMR) settings region (0x060-0x067)

Marked *"confirmed via CPS RE + en.bf"* in the reference implementation. Factory dump:
`03 02 0c 03 03 2b 00 00`.

| Offset | Field | Encoding | Confidence |
|--------|-------|----------|-----------|
| 0x060 | Digital decode flags | bit 0 = Private Call Match, bit 1 = Group Call Match; bits 2-7 ❓ UNKNOWN | |
| 0x061 | Call Hold Time | raw = seconds, 0-61 | |
| 0x062 | Active Wait Time | raw = `combo_idx + 1`; ms = `(raw−1)*30 + 300`, range 300-4800 | scaling ⚠️ DERIVED |
| 0x063 | Active Retries Time | raw = count 1-8 | |
| 0x064 | Pre-Carrier Time | raw = `combo_idx`; ms = `(raw+1)*120`, range 120-8640 | scaling ⚠️ DERIVED |
| 0x065 | Digital settings flags | see bits below | |
| 0x066 | SMS Format | raw = `combo_idx`; s = `(raw+1)*10`, range 10-120 | scaling ⚠️ DERIVED |
| 0x067 | Name display flags | see bits below | |
| 0x068-0x07F | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |

0x065 bits — 7 Remote Monitor Decode, 6 Radio Disable Decode, 5 Radio Check Decode, 4 Radio Enable
Decode, 3 Call Alert Decode, 2-1 Data Service (2-bit field, values never enumerated ❓), 0 Missed Call
Alert.

0x067 bits — 7-6 Name Data Format (2-bit, values never enumerated ❓), 5-4 ❓ UNKNOWN, 3 Send TX Name,
2 Name Display Priority, 1-0 ❓ UNKNOWN.

The scaling formulas exist only in the reference implementation's *model* comments; the code stores
and loads the raw byte and the UI exposes it labelled "(raw)". ⚠️ DERIVED.

### Miscellaneous (0x080-0x081, 0x0A0-0x0A7)

| Offset | Field | Encoding | Confidence |
|--------|-------|----------|-----------|
| 0x080 | VFO embedded flags | bitfield; only bits 0/1/2 have (placeholder) labels *"VFO Embedded 0/1/2"*. 0xBE in the factory dump | ❓ UNKNOWN |
| 0x081 | TX Dwell Time | direct value 0-255 | ⚠️ DERIVED |
| 0x082-0x084 | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |
| 0x0A0-0x0A7 | Language / other settings | opaque 8-byte blob, copied in and out verbatim, no field decomposition. `16 05 00 0a 36 23 04 03` in the factory dump | ❓ UNKNOWN |

### Key lock and button functions (0x085-0x093)

Every button byte in the factory dump lands inside the 0-42 enum and decodes to a sensible default
(SK1 short = 22 FM Radio, SK1 long = 24 GPS Information, SK2 short = 25 Monitor, SK2 long = 1 Power
Select, P1 short = 2 Volt, P1 long = 15 CSV Contacts, P2 short = 7 V/M, P2 long = 40 One Key Scan
Freq), which is strong corroboration of the offsets.

| Offset | Bits | Field | Encoding | Confidence |
|--------|------|-------|----------|-----------|
| 0x085 | 0 | Lock Key | 0 = Manual, 1 = Auto | |
| 0x085 | 1 | Knob Lock | 0 = Off, 1 = On | |
| 0x085 | 2 | Side Key Lock | 0 = Off, 1 = On | |
| 0x085 | 7-3 | — | preserved (read-modify-write). Real data: the OEM write capture stores **0xF0** here | ❓ UNKNOWN |
| 0x086 | 7-0 | Auto Keypad Lock Delay | seconds, 5-60 | |
| 0x087 | 7-0 | SK1 Short | button-function enum 0-42 | |
| 0x088 | 7-0 | SK1 Long | idem | |
| 0x089 | 7-0 | SK2 Short | idem | |
| 0x08A | 7-0 | SK2 Long | idem | |
| 0x08B-0x08C | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |
| 0x08D | 7-0 | P1 Short | idem | |
| 0x08E | 7-0 | P1 Long | idem | |
| 0x08F | 7-0 | P2 Short | idem | |
| 0x090 | 7-0 | P2 Long | idem | |
| 0x091-0x092 | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |
| 0x093 | 7-0 | Long Press Time | **stored 0-4, displayed 1-5** (`raw+1`) | |
| 0x094-0x09F | — | — | empty in both captures (0x00 in the factory read, 0xFF in the OEM write) | ❓ UNKNOWN |

**Button-function enum (0-42)**

| Val | Label | Val | Label | Val | Label |
|---|---|---|---|---|---|
| 0 | None | 15 | CSV Contacts | 30 | TBST Send |
| 1 | Power Select | 16 | Zone Up | 31 | APRS Send |
| 2 | Volt | 17 | Zone Down | 32 | Channel Type |
| 3 | Talkaround | 18 | Scan | 33 | Display Mode |
| 4 | Digital Encrypt | 19 | Record Switch | 34 | CTC Scan |
| 5 | Call | 20 | Previous Record | 35 | CTC Setting |
| 6 | VOX | 21 | Next Record | 36 | Silent Tone |
| 7 | V/M | 22 | FM Radio | 37 | Roaming |
| 8 | Alarm | 23 | FM Search | 38 | Sub-PTT |
| 9 | One Touch Call 1 | 24 | GPS Information | 39 | Analog Scramble Switch |
| 10 | One Touch Call 2 | 25 | Monitor | 40 | One Key Scan Freq |
| 11 | One Touch Call 3 | 26 | Switch Main Channel | 41 | Flashlight |
| 12 | One Touch Call 4 | 27 | Lone Work | 42 | Man Down Alarm |
| 13 | One Touch Call 5 | 28 | Keypad Lock | | |
| 14 | SMS | 29 | Nuisance Channel Delete | | |

### One-Key Operation

**Analog Call — base 0x120, 4 entries × 2 bytes** (0x120, 0x122, 0x124, 0x126)

| Entry offset | Field | Values | Confidence |
|--------------|-------|--------|-----------|
| +0x00 | Call Type | 0 = No., 1 = Call Type, 2 = Call ID | ⚠️ DERIVED — see below |
| +0x01 | Call ID | raw byte | ⚠️ DERIVED |

> ⚠️ **The factory dump contradicts the enum.** Bytes 0x120-0x127 read
> `01 01 02 01 03 01 00 00`, i.e. the *first* byte of each entry runs 1, 2, 3 while the second is
> constant 1. That looks like `[number][call type]`, not `[call type][number]`, and the value 3 is
> outside the documented 0-2 range. Unresolved — do not trust the field order, and do not clamp.

**One Touch Call — base 0x200, 5 entries × 5 bytes** (0x200, 0x205, 0x20A, 0x20F, 0x214)

| Entry offset | Size | Field | Values |
|--------------|------|-------|--------|
| +0x00 | 1 | Call Type | 0 = Off, 1 = Analog, 2 = Digital |
| +0x01 | 2 | Call Object | uint16 **little-endian** |
| +0x03 | 1 | Digital Call Type | 0 Off, 1 Private, 2 Group, 3 Message, 4 Call Alert, 5 Radio Check, 6 Remote Monitor, 7 Active, 8 Kill |
| +0x04 | 1 | SMS | SMS number/index |

Factory dump 0x200-0x218 = `02 01 00 02 00 | 02 02 00 03 04 | 02 03 00 02 00 | 01 01 00 00 00 |
01 03 00 00 00` — five entries that all decode inside their enums. Layout CONFIRMED.

> **Previously documented** (in the reference implementation's model comments) as base 0x1FB —
> incorrect; the working code and the hardware dump both say **0x200**. Likewise the analog-call base
> is **0x120**, not 0x11E.

**Fun+ — base 0x230, 10 entries × 7 bytes** (0x230, 0x237, 0x23E, 0x245, 0x24C, 0x253, 0x25A, 0x261,
0x268, 0x26F)

| Entry offset | Field | Values | Confidence |
|--------------|-------|--------|-----------|
| +0x00 | Operate Mode | 0 = Call, 1 = Menu | |
| +0x01 | Menu Select | 0 Off, 1 SMS, 2 New SMS, 3 Shortcut Text, 4 Inbox, 5 Outbox, 6 Contact List, 7 Manual Dial, 8 Call Log, 9 Sent Call, 10 Answered Call, 11 Missed Call, 12 Zone, 13 Radio Setting | |
| +0x02 | Padding | parser skips it, encoder writes 0x00 | ❓ UNKNOWN |
| +0x03 | Call Way | 0 = Off, 1 = Analog, 2 = Digital | ⚠️ DERIVED |
| +0x04 | Call Object | contact/ID, **single byte** (unlike One Touch Call's uint16) | ⚠️ DERIVED |
| +0x05 | Digital Call Type | 0-8, same enum as One Touch Call | ⚠️ DERIVED |
| +0x06 | SMS | raw byte | ⚠️ DERIVED |

The Fun+ number is the entry index (0-9); it is **not** stored in the entry.

> **Intra-entry alignment: resolved by hardware.** The two sources inside the reference implementation
> disagreed — its parser puts Operate Mode at +0x00, its Diagnostics layout table puts a "Number Key"
> byte at +0x00 and shifts everything by one. The factory dump settles it: entries 2-10 read
> `01 01`, `01 02`, `01 03`, `01 04`, `01 05`, `01 06`, `01 07`, `01 0C`, `01 0D` — Operate Mode = 1
> ("Menu") with Menu Select running 1…7, 12, 13, every value inside the 0-13 enum. Under the shifted
> reading the second byte would be Operate Mode holding 1…13, far outside its 0-1 range. **The parser
> layout above is correct; the Diagnostics "Number Key" row is wrong.**

### APRS and fixed GPS position (0x301-0x334)

| Offset | Size | Field | Encoding | Confidence |
|--------|------|-------|----------|-----------|
| 0x301 | 1 | APRS Scheduled Send Time | 0 = Off, otherwise `n*30` s up to 7200 s | scaling ⚠️ DERIVED |
| 0x302 | 1 (bit 0) | APRS Fixed Beacon | 0 = Off, 1 = On; read-modify-write | ⚠️ DERIVED |
| 0x303-0x305 | 3 | — | 0x00 / 0xFF in the captures | ❓ UNKNOWN |
| 0x306-0x30E | **9** | Latitude | ASCII, NUL-terminated, e.g. `"23.000000"` | |
| 0x30F | 1 | Latitude Direction | ASCII `'N'` (0x4E) / `'S'` (0x53) | |
| 0x310-0x318 | **9** | Longitude | ASCII, e.g. `"118.00000"` | |
| 0x319 | 1 | Longitude Direction | ASCII `'E'` (0x45) / `'W'` (0x57) | |
| 0x31A-0x31F | 6 | — | | ❓ UNKNOWN |
| 0x320 | 2 | APRS Report Channel 1 | uint16 LE; 0 = current channel | ⚠️ DERIVED |
| 0x322 | 2 | APRS Report Channel 2 | uint16 LE | ⚠️ DERIVED |
| 0x324 | 2 | APRS Report Channel 3 | uint16 LE | ⚠️ DERIVED |
| 0x326 | 2 | APRS Report Channel 4 | uint16 LE | ⚠️ DERIVED |
| 0x328 | 2 | APRS Report Channel 5 | uint16 LE | ⚠️ DERIVED |
| 0x32A | 2 | APRS Report Channel 6 | uint16 LE | ⚠️ DERIVED |
| 0x32C | 2 | APRS Report Channel 7 | uint16 LE | ⚠️ DERIVED |
| 0x32E | 2 | APRS Report Channel 8 | uint16 LE | ⚠️ DERIVED |
| 0x330 | 1 | APRS Repeater Active Delay | 0 = Off, 1 = 100 ms … 10 = 1000 ms | ⚠️ DERIVED |
| 0x331 | 1 (bit 0) | APRS Call Type | 0 = Private, 1 = Group | ⚠️ DERIVED |
| 0x332-0x334 | 3 | APRS Upload ID | 24-bit DMR ID — **byte order disputed, see below** | ⚠️ DERIVED |
| 0x335-0x4FF | — | — | | ❓ UNKNOWN |

**Position fields — CONFIRMED by the OEM write capture.** It contains
`… 30 30 2e 30 30 30 30 30 30 4e | 30 30 2e 30 30 30 30 30 30 45` at 0x306: `"00.000000"` + `'N'` at
0x30F, `"00.000000"` + `'E'` at 0x319. Both string fields are exactly **9 bytes** and the direction
bytes are ASCII. (The factory read capture holds `"23.000000"` / `"118.00000"` with direction bytes
0x00 — an unset factory default, not a second encoding.)

> ⚠️ **9 bytes is the field width.** Writing a wider field at 0x306 clobbers the direction byte at
> 0x30F and runs into the longitude field; at 0x310 it overruns into `0x31A-0x31D`, which no field
> claims.

> ⚠️ **APRS Upload ID byte order is unresolved.** Both captured values decode as plausible DMR IDs
> only under **little-endian** (factory `C8 01 00` → 456 LE vs 13,107,456 BE; OEM write `01 00 00`
> → 1 LE), but a big-endian reading is also in circulation. One CPS round-trip with a known APRS ID
> settles it.

### Menu enable/disable flags (0x500-0x507)

Eight bytes of per-menu-item enable bits: **bit set = item enabled**. Confidence **⚠️ DERIVED**:
no capture pins the polarity (an inverted-bit reading also circulates).

| Offset | Bit | Item | | Offset | Bit | Item |
|--------|-----|------|---|--------|-----|------|
| 0x500 | 0 | Zone List | | 0x504 | 0 | Add Contact |
| 0x500 | 1 | New Zone | | 0x504 | 1 | Del Contact |
| 0x500 | 2-7 | ❓ UNKNOWN | | 0x504 | 2 | Edit Contact |
| 0x501 | 0 | Call Alert | | 0x504 | 3 | Send Message |
| 0x501 | 1 | Radio Check | | 0x504 | 4 | Functionality |
| 0x501 | 2 | Remote Monitor | | 0x504 | 5 | Manual Dial |
| 0x501 | 3 | Radio Enable | | 0x504 | 6 | CSV Contacts |
| 0x501 | 4 | Radio Disable | | 0x504 | 7 | ❓ UNKNOWN |
| 0x501 | 5 | Measure Period | | 0x505 | 0 | Missed Call |
| 0x501 | 6-7 | ❓ UNKNOWN | | 0x505 | 1 | Answered Call |
| 0x502 | 0 | Talkaround | | 0x505 | 2 | Sent Call |
| 0x502 | 1 | Alert Tone | | 0x505 | 3 | Del Log |
| 0x502 | 2 | TX Power | | 0x505 | 4-7 | ❓ UNKNOWN |
| 0x502 | 3 | Start Display | | 0x506 | 0 | RX Frequency |
| 0x502 | 4 | Lang Select | | 0x506 | 1 | TX Frequency |
| 0x502 | 5 | Match Private | | 0x506 | 2 | CTC/DCS |
| 0x502 | 6 | Match Group | | 0x506 | 3 | TX Contact |
| 0x502 | 7 | Display Mode | | 0x506 | 4 | Color Code |
| 0x503 | 0 | SMS Format | | 0x506 | 5 | Time Slot |
| 0x503 | 1 | Sub Channel Mode | | 0x506 | 6 | Radio ID |
| 0x503 | 2 | Power Save | | 0x506 | 7 | Radio Name |
| 0x503 | 3 | FM Radio | | 0x507 | 0 | Channel Type |
| 0x503 | 4 | GPS | | 0x507 | 1 | TDMA Direct Mode |
| 0x503 | 5 | APRS | | 0x507 | 2 | RX Group List |
| 0x503 | 6 | Record | | 0x507 | 3 | Add Channel |
| 0x503 | 7 | ❓ UNKNOWN | | 0x507 | 4 | Channel Name |
| | | | | 0x507 | 5-7 | ❓ UNKNOWN |

The OEM write capture has 0x500-0x507 = `fd c3 07 80 ff ff 00 e0`, which proves the ❓ UNKNOWN bits are
not padding: 0x500 bits 2-7, 0x503 bit 7, 0x505 bits 4-7 and 0x507 bits 5-7 all carry values there.

> ⚠️ Read-modify-write only the bits you understand — rebuilding these eight bytes from defaults
> zeroes every unlabelled bit above, all of which carry data on hardware.

### Unmapped byte ranges

Of the 4096 bytes in this block, **232 carry a documented field**; adding the metadata byte at 0xFFF
makes 233 accounted for and **3863 unaccounted**. Nothing in this document or in the reference
implementation reads or writes the following, and they are preserved verbatim on write:

```
0x01F        0x022-0x02F  0x03C-0x03F  0x043-0x044  0x046-0x05F
0x068-0x07F  0x082-0x084  0x08B-0x08C  0x091-0x092  0x094-0x09F
0x0A8-0x11F  0x128-0x1FF  0x219-0x22F  0x276-0x300  0x303-0x305
0x31A-0x31F  0x335-0x4FF  0x508-0xFFE
```

(The Fun+ table ends at 0x275 — its tenth entry is 0x26F-0x275 — so **0x276 is unmapped**. It was
previously listed as the last mapped byte, which is where the earlier "234 mapped" count came from.)

Almost all of it really is empty. Across both captures the **only** unmapped bytes holding anything
other than 0x00/0xFF are:

| Offset | Factory read | OEM write | Note |
|--------|--------------|-----------|------|
| 0x0A9 | 0x04 | 0xFF | one byte past the 8-byte language blob ❓ |
| 0x600 | 0x00 | 0xFC | ❓ UNKNOWN — an unmapped sub-region the CPS writes |
| 0x601-0x602 | 0x0A 0x0A | 0x00 0x00 | ❓ UNKNOWN |
| 0x700 | 0x0F | 0xFF | ❓ UNKNOWN |
| 0x702-0x703 | 0x10 0x88 | 0xFF 0xFF | ❓ UNKNOWN |

Note the asymmetry, verified byte-for-byte against both captures: only **0x600-0x602** is written by
the OEM CPS in the write capture (`FC 00 00`). At 0x0A9 and 0x700-0x703 the write capture holds plain
0xFF fill, i.e. the CPS did *not* reproduce the values the factory radio had there — so those three
sites may belong to the radio/firmware rather than to the codeplug the CPS owns. (Earlier revisions
of this table showed 0x00 in the write column for those rows; that was wrong.)

Those five sites are the entire remaining reverse-engineering surface of block 0x04 and are the
highest-value target for the next diff experiment (change one unexplained setting in the OEM CPS,
re-read, diff).

---

## 0x06 - DTMF Encode Data

**Metadata**: 0x06  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Multi-section analog config — DTMF codes, analog (DTMF) contacts, and more.
(*Previously documented as "DTMF Encode Data" only — incomplete.*)

**Hardware anchors** (both captures): DTMF code sequences from `0x000` (stride 16); a 1-byte count
at `0x1FF`; **analog contact names from `0x200`** (stride 32 — `"AContact 1"`, `"AContacts 1"`…);
and a second table at `0xAA0` whose first entry reads `"BDC Cotnacts 1"` (sic) — `❓ UNKNOWN`.
The count at `0x1FF` tracks the analog contacts in both captures and is **not** a talk-group
counter — see
[*Block 0x06 offset 0x1FF*](#block-0x06-offset-0x1ff-is-not-a-talk-groups-counter) under the 0x44
section for the evidence.

### Structure Overview — `⚠️ DERIVED` (CPS decompilation; only the anchors above are hardware-checked)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x00-0xF0 | 256 | DTMF entries | 16 entries × 16 bytes each (indexed by `param * 0x10 + -0x10`, 1-based) |
| 0x100 (256) | 1 | Byte value | Range 0-255, stored as `value + 0x0F` |
| 0x101 | 1 | Byte value | Configuration |
| 0x102 | 1 | Byte value | Configuration |
| 0x103 | 1 | Byte value | Configuration |
| 0x104 | 1 | Byte value | Range 0-255, stored as `value + 1` |
| 0x105 | 1 | Bit flags | Bit 0 flag |
| 0x106-0x108 | 3 | Duration/Interval | Numeric value (likely milliseconds) |
| 0x109 | 1 | DTMF Group Code | Range 0-6, stored as `value + 9` |
| 0x10A | 1 | DTMF Interval Sign | Range 0-5, stored as `value + 10` |
| 0x10B | 1 | Byte value | Configuration |
| 0x10C | 1 | DTMF Auto Ack | Range 0-2, stored as `value + 4` |
| 0x10E | 1 | Byte value | Configuration |
| 0x110, 0x120, 0x130, 0x140 | 64 | Additional DTMF codes | 4 more 16-byte DTMF code sequences |
| 0x1FF | 1 | ❓ UNKNOWN — probably the analog/DTMF contact count | The factory read has 7 here with 7 analog contact names at 0x200; the OEM CPS write has 1 with 1 analog contact name. It tracks the analog contacts in both, and matches the talk-group count in neither. Leave it alone until identified — see *0x44 - Talk Groups → Block 0x06 offset 0x1FF* |

### DTMF Code Encoding

Each DTMF digit stored as single byte:
- **0-9**: 0x00-0x09
- **A-D**: 0x0A-0x0D
- **\***: 0x0E
- **#**: 0x0F

**Note**: Some documentation shows encoding as 0x30-0x39 for digits 0-9, but the actual storage uses 0x00-0x09.

### Field Values

**DTMF Group Code** (stored as `value + 9`):
- 0 = Off
- 1 = A
- 2 = B
- 3 = C
- 4 = D
- 5 = *
- 6 = #

**DTMF Interval Sign** (stored as `value + 10`):
- 0 = A
- 1 = B
- 2 = C
- 3 = D
- 4 = *
- 5 = #

**DTMF Auto Ack** (stored as `value + 4`):
- 0 = Off
- 1 = Alert Tone
- 2 = Alert Tone And Ack

---

## 0x07 - ❓ UNKNOWN (labelled "Configuration Header")

**Metadata**: 0x07  
**Block Size**: 4 KB (0x1000 bytes)  
**Status**: `❓ UNKNOWN` — allocated on the captured radio but never read or written by the OEM CPS
in either capture. The "Config header" label is a single unacted-upon comment in the reference
implementation; no content has ever been observed. Its project notes warn: *"Must never be read or
written to avoid bricking the radio"* — treat that as caution about **writing**; reading any page
is non-destructive.

---

## 0x0A - Quick Text Messages

**Metadata**: 0x0A  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Canned/predefined text messages

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseQuickMessages` / `encodeQuickMessages`), `src/radios/dm32uv/protocol.ts` (`readQuickMessages` / `writeQuickMessages`)

Both hardware captures match the layout below exactly.

- **Factory read**: count byte 0x000 = 5; entry 1 status byte 0x010 = **0x0C**; `"How are you?"`
  (12 characters) at 0x011; entry stride **129**.
- **OEM CPS write**: count byte 0x000 = 1; entry 1 status byte 0x010 = **0x01**; text `31 00`
  (`"1"`, 1 character) at 0x011, then 0xFF to the end of the field.

In both cases the status byte equals the **character count** of the message text, which settles the
contested meaning of that byte (see below).

### Structure Overview

**Count Field**: Offset 0x00 (**1 byte**) - number of messages (read capture 5, write capture 1)  
**Header**: 0x01-0x0F (15 bytes) ❓ UNKNOWN - never read or written by the reference implementation; 0x00 in the factory image, 0xFF in the OEM write  
**Entry Size**: 0x81 (129) bytes per message = 1 status byte + 128 text bytes  
**Entry Base**: Offset 0x10 for entry 1 (entries are numbered **1-based**)  
**Max Entries**: **20**

**Entry Calculation** (N is 1-based):

```
status  offset = (N * 0x81) - 0x71
message offset = (N * 0x81) - 0x70      (128 bytes)
```

| N | Status byte | Message text | Entry ends |
|---|-------------|--------------|-----------|
| 1 | 0x0010 | 0x0011-0x0090 | 0x0090 |
| 2 | 0x0091 | 0x0092-0x0111 | 0x0111 |
| 20 | 0x09A3 | 0x09A4-0x0A23 | 0x0A23 |

Bytes 0x0A24-0x0FFE are ❓ UNKNOWN — never read or written by the reference implementation, and
empty in both captures (0x00 in the factory image, 0xFF in the OEM write).

> (*Previously documented with base 0x80, a `buffer + 0x80 * (entry_num + 1)` formula, ~30
> messages, and a "check value" at +0x70 — all incorrect; the formulas above match both the working
> implementation and the hardware dump.*)

### Entry Structure (129 bytes)

| Offset within entry | Size | Field | Description |
|---------------------|------|-------|-------------|
| +0x00 | 1 | Status = **character count** | Confirmed by both captures (0x0C for a 12-character message, 0x01 for a 1-character message). The reference implementation's parser reads it as-is and its encoder writes the byte length of the text ("Ensure flag is set to character count") |
| +0x01 | 128 | Message text | ASCII, occupying +0x01-+0x80. Terminator/fill differs by writer — see below |
| +0x80 | (1, the last byte of the text field) | ❓ UNKNOWN | The OEM CPS write capture sets the **final byte of the 128-byte field** (block offset 0x090 for entry 1) to **0x00** while the whole tail before it is 0xFF — the only non-0xFF byte in the block besides the count and the message. Purpose unestablished; the reference implementation writes 0xFF there. It may simply be a hard terminator guaranteeing the field can never be un-terminated |

### Message Text Encoding

Three writers, three conventions, all readable with the same rule ("text ends at the first 0xFF or
0x00"):

| Producer | Bytes after the text |
|---|---|
| OEM CPS (write capture) | `0x00` terminator, then 0xFF to the end of the field |
| Factory image (read capture) | `0x00` terminator, then 0x00 to the end of the field |
| Reference implementation | `0xFF` terminator, then 0xFF to the end of the field |

- The reference parser scans for the first **0xFF**, then strips embedded 0x00 bytes and trims
  whitespace — which is why it decodes all three forms. An implementation that only honours 0xFF
  would read trailing NUL padding as part of the string on a factory radio.
- A text of exactly 128 bytes has its **last byte overwritten with 0xFF**, so the usable maximum is
  **127 characters**.
- Writing clears all 20 status bytes and all 20 × 128 text areas to 0xFF first, then writes the count
  byte at 0x00 and the populated entries. The rest of the block is a read-modify-write on the block
  read from the radio.

### ❓ UNKNOWN / implementation notes

- The `checkValue` field described by earlier documentation (2 bytes at +0x70) is **UNKNOWN**: the
  reference implementation hardcodes it to 0 with the comment *"Check value not in this format"*, and
  nothing ever encodes it.
- The `flag` byte is **settled**: the model comment in the reference implementation says "set to 0
  when message is set", but both hardware captures carry the character count (0x0C for
  `"How are you?"`, 0x01 for `"1"`). The comment is wrong; the field is a character count.
- The parser trusts the count byte at 0x00 and never scans past it, so messages stored in slots beyond
  the recorded count are invisible.
- A read→write round trip **compacts** messages into slots 1..N: the parser skips empty-text entries
  while keeping their index, and the encoder then rewrites entries positionally.

---


## 0x0B - Quick Access Contact List (Talk Group index tables)

**Metadata**: 0x0B  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Counts, a slot-usage bitmask, and two sorted index tables over the Talk Groups in block 0x44

**Implementation reference**: `src/radios/dm32uv/protocol.ts` (`writeQuickContacts`), `src/components/diagnostics/QuickContactsBlockDetails.tsx`

> **Previously documented as**: "purpose unconfirmed" / (earlier still) the RX Group List. The RX
> Group List is metadata **0x0F**. 0x0B is written by the OEM CPS and by the reference implementation
> every time Talk Groups are written.
>
> Most of the layout below is corroborated by both hardware captures (see the worked examples);
> the exceptions are marked inline.

### Hardware evidence

Both captures are decoded with the layout below and cross-checked against the talk groups in block
0x44 of the same session:

| | Factory read capture | OEM CPS write capture |
|---|---|---|
| Talk groups in block 0x44 | **10** (`Contacts 1`…`Contacts 10`) | **16** (`WV Statewide`, `Simplex`, …) |
| Call types present | 5 × Group (0x04), 4 × Private (0x03), 1 × All (0x05) | 16 × Group (0x04) |
| Bytes 0x000-0x004 | `0A 00 05 00 01` | `10 00 10 00 00` |
| Total at 0x000 (uint16 LE) | 10 ✅ matches | 16 ✅ matches |
| Group count at 0x002 (uint16 LE) | 5 ✅ matches | 16 ✅ matches |
| Byte 0x004 | 1 — ❌ does **not** match the 4 Private Call entries; it matches the 1 All Call entry | 0 — matches both 0 Private and 0 All |
| Bitmask 0x010… | `00 FC` = 10 bits cleared ✅ | `00 00` = 16 bits cleared ✅ |
| Table 1 at 0x100 | 10 entries | 16 entries, exactly alphabetical by name ✅ |
| Table 2 at 0x740 | 10 entries | 16 entries, exactly ascending by DMR ID ✅ |

So the total count, the Group Call count, the bitmask semantics, both table base offsets, the 2-byte
entry format and both sort orders are **hardware-confirmed**. Only byte 0x004 is contradicted.

### Block Layout

| Offset | Size | Field | Encoding |
|--------|------|-------|----------|
| 0x000-0x001 | 2 | Total talk-group count | uint16 LE — confirmed by both captures |
| 0x002-0x003 | 2 | Group Call count | uint16 LE (call type 0x04) — confirmed by both captures |
| 0x004 | 1 | ❓ UNKNOWN — **not** the Private Call count | Long believed to hold the Private Call count, but the factory radio has 4 Private Call talk groups and this byte reads **1** — its number of **All Call** entries. Both captures are consistent with an All-Call count; the write capture (0 private, 0 all → 0) cannot separate them. Treat as unknown |
| 0x005-0x00F | 11 | ❓ UNKNOWN | left untouched by the reference writer; 0xFF in both captures |
| 0x010-0x01F | 16 | Slot usage bitmask | 128 slots, **0 = used, 1 = free**; initialised to all-0xFF, then one bit cleared per used display slot. Confirmed: 10 talk groups → `00 FC …`, 16 talk groups → `00 00 …` |
| 0x020-0x0FF | 224 | ❓ UNKNOWN | never touched; 0xFF in both captures |
| 0x100-… | — | **Index Table 1** — name-sorted | Base **0x100 confirmed**. 2 bytes/entry: `[contact_index][type_byte]` |
| …-0x6FF | — | Table 1 extent ⚠️ DERIVED | The reference implementation stops at 0x700 (768 entries max). Neither capture populates more than 16 entries, so the end of the table is **not** established by hardware |
| 0x700-0x73F | 64 | ❓ UNKNOWN | gap between the two tables; never touched; 0xFF in both captures |
| 0x740-… | — | **Index Table 2** — DMR-ID-sorted (numeric ascending) | Base **0x740 confirmed**; same 2-byte format |
| …-0xCFF | — | Table 2 extent ⚠️ DERIVED | The reference implementation stops at 0xD00 (**736** entries = ⌊(0xD00 − 0x740) / 2⌋). Not established by hardware |
| 0xD00-0xFFE | 767 | ❓ UNKNOWN | never touched; 0xFF in both captures |
| 0xFFF | 1 | Metadata / logical block ID (0x0B) | |

In the OEM write capture the only non-0xFF bytes in the whole block are `0x000-0x004`, `0x010-0x011`,
`0x100-0x11F`, `0x740-0x75F` and `0xFFF` — i.e. exactly the header, the bitmask, 16 Table 1 entries,
16 Table 2 entries and the logical ID.

### Index entry format (2 bytes)

| Byte | Field | Encoding |
|------|-------|----------|
| 0 | `contact_index` | **1-based physical slot position in block 0x44** — *not* a talk group's own index value |
| 1 | `type_byte` | 0x30 = Private Call (0x03), 0x40 = Group Call (0x04), 0x50 = All Call (0x05); anything else defaults to 0x40 |

Both mappings are hardware-confirmed. Factory read Table 1 = `01 40  0A 50  02 40  03 40 …`: talk
group 1 (`Contacts 1`, Group Call) → `01 40`, talk group 10 (`Contacts 10`, contact number 0xFFFFFF,
call type 0x05) → `0A 50`, and the four Private Call groups appear as `06 30  07 30  08 30  09 30`.

**Index semantics (important):** the index stored in 0x0B must match the *physical* position of the
entry in block 0x44, not any logical/UI index, because deletions can leave gaps in the latter. The
same 1-based physical position is what a channel's TX-Contact index (blocks 0x42/0x43) refers to.

**Sort orders — confirmed.** In the OEM write capture, Table 1 reads
`0D 40  0E 40  0A 40  0F 40  0C 40  05 40  02 40  03 40  10 40  04 40  09 40  07 40  06 40  0B 40  08 40  01 40`,
which is exactly alphabetical by talk-group name (`Audio Test`, `Clear Timeslot`, `HEARS Link`,
`Local`, `Parrot`, `RVA Metro`, `Simplex`, `TAC A`, `TAC B`, `TAC C`, `VA Peninsula`,
`VA Shen Valley`, `VA Southwest`, `VA Statewide`, `VA Tidewater`, `WV Statewide`). Table 2 reads
`02 40  0B 40  01 40  0C 40 …`, exactly ascending by 24-bit contact number (99, 3151, 3154, 9998, …).

> ⚠️ **Collation**: the radio's own convention appears to be plain byte-wise ASCII — the factory
> image orders `Contacts 1`, `Contacts 10`, `Contacts 2`, `Contacts 3`, … (a numeric-aware sort
> would place `Contacts 10` last). Sort Table 1 with plain ASCII ordering.

---

## 0x0F — see *RX Group List Structure*

**Metadata 0x0F is the RX Group List**, documented in the *RX Group List Structure* section above.
(*Previously documented here as "TX Contact Assignment" — incorrect: TX contact assignment lives in
blocks 0x42/0x43 at 2 bytes per channel, and the odd "negative offset" fields were the RX-group
block header misread as a per-entry header.*)

---

## 0x10 - Multi-section: Digital Emergency / Analog Emergency / Encryption Keys

**Metadata / logical block ID**: 0x10  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: three unrelated feature areas packed into one block

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseDigitalEmergencies` / `encodeDigitalEmergencies`, `parseAnalogEmergencies` / `encodeAnalogEmergencies`, `parseEncryptionKeys` / `encodeEncryptionKey`), `src/radios/dm32uv/protocol.ts` (`readDigitalEmergencies` / `writeDigitalEmergencies`, `readAnalogEmergencies` / `writeAnalogEmergencies`, `writeEncryptionKeys`)

Both OEM serial captures contain a full 4 KB transfer of this block, so the section boundaries are
hardware evidence rather than inference: `"DEmer 1"`…`"DEmer 8"` on a 20-byte stride from 0x000,
`"AEMER 1"` inside the analog region of the write capture, and `"Encrypt 1"`…`"Encrypt 4"` on a
44-byte stride from 0x300.

> **Writing rule — read-modify-write the whole block.** Three independent features share these
> 4096 bytes, and each one's writer must preserve the other two. The reference implementation does
> this (its digital-emergency encoder is documented as *"Preserves existingBlockData (analog emergency
> at 0x0AC, encryption keys at 0x300)"*, and its analog encoder clears **only** 0x0AC-0x2FF), and it
> re-stamps `data[0xFFF] = 0x10` before every write. Note that in that implementation all three
> writers run back-to-back from the same cached copy of the block during a codeplug write, so a
> naive port can lose whichever section was written first — seed each write from the result of the
> previous one.

### Block Layout

| Offset | Size | Section | Max entries | Entry size | Confidence |
|--------|------|---------|------------|------------|-----------|
| 0x000 | 0x0A0 | Digital Emergency Systems | **8** | 20 bytes (0x14) | |
| 0x0A0-0x0AB | 0x00C | Gap between sections | — | — | ❓ UNKNOWN — 0x00 in the read capture, 0xFF in the write capture |
| 0x0AC | 0x254 | Analog Emergency Systems | **16** | 36 bytes (0x24) | ⚠️ DERIVED — base disputed, see Section 2 |
| 0x300 | 0x160 | Encryption / Privacy Keys | **8** | 44 bytes (0x2C) | |
| 0x460-0xFFE | — | ❓ UNKNOWN | — | — | mostly empty, but **not** reserved — see below |
| 0xFFF | 1 | Metadata / logical block ID (0x10) | — | — | |

> **Previously documented as** "0x460 | Empty / reserved" — **unsupported**. Nothing establishes
> that the tail is reserved, and it is not entirely empty: the factory read capture has
> `0x0B01-0x0B02 = 0A 0A` and `0x0B07-0x0B09 = 0A 0A 05`, and the OEM write capture has
> `0x0B00-0x0B09 = 00 0A 0A E0 FF 00 01 0A 0A 05`. Repeated `0A 0A`
> pairs in an area no known section covers. Mark it ❓ UNKNOWN, preserve it byte-for-byte on write,
> and do not assume it is free space.
>
> The same `0A 0A` signature also appears at **0x200-0x202 = `0A 0A 04`** in the write capture. That
> offset is *inside* the analog-emergency range (0x0AC-0x2FF) in the table above, not in the tail, so
> it is not evidence about the tail — it is evidence that either the analog-emergency section does not
> really extend to 0x2FF, or that something else is interleaved with it. ❓ UNKNOWN either way, and one
> more reason the analog section's bounds are marked ⚠️ DERIVED. A writer that erases the whole
> `0x0AC-0x2FF` span would destroy these three bytes — preserve them.

---

### Section 1: Digital Emergency Systems (0x000–0x09F)

**Entry size**: 20 bytes (0x14)  
**Entry formula**: `offset = 0x000 + index * 0x14` (index 0–7)  
**Entry bases**: 0x000, 0x014, 0x028, 0x03C, 0x050, 0x064, 0x078, 0x08C (last ends 0x09F)  
**Max entries**: **8**

**The layout is CONFIRMED against hardware.** Factory read capture, block 0x10 at 0x00C000:

```
0000  44 45 6d 65 72 20 31 00 00 00 01 03 00 00 05 05  |DEmer 1.........|
0010  00 00 01 00 44 45 6d 65 72 20 32 00 00 00 02 03  |....DEmer 2.....|
0020  00 00 05 05 00 00 01 00 44 45 6d 65 72 20 33 00  |........DEmer 3.|
0030  ff ff 03 03 00 00 05 05 00 ff 03 00 44 45 6d 65  |............DEme|
```

Eight names on an exact 20-byte stride. The reference implementation's constant carries the same
provenance verbatim: `DIGITAL_EMERGENCY_MAX: 8,  // 8 entries × 20 bytes = 0x000–0x09F (confirmed by
hardware hexdump)`, and its parser doc-block adds *"Field layout confirmed by CPS decompilation
(DMR CPS.exe.c FUN_00470xxx)"*.

| Entry offset | Size | Field | Encoding (stored ↔ displayed) | Confidence |
|-------------|------|-------|-------------------------------|-----------|
| +0x00 | 10 | Name | ASCII, NUL-padded, max 10 chars. Empty ⇒ `DEmer <n>` | |
| +0x0A | 1 | Alarm Type | 0 None, 1 Only Whistle, 2 Normal, 3 Secret, 4 Secret With Voice, 5 Alarm Whistle | |
| +0x0B | 1 | Alarm Mode | **stored = value + 1**: raw 1 = Emergency Alarm, 2 = Alarm Call, 3 = Emergency Call | |
| +0x0C | 2 | Revert Channel | uint16 **little-endian**; 0 = None | |
| +0x0E | 1 | Retransmission | raw = value, 1–15 | |
| +0x0F | 1 | HOT MIC Duration | raw = value, 1–15 seconds | |
| +0x10 | 1 | Emergency Calls Number | **raw 0–11 → `(raw+1)*10`** = 10…120 step 10 | |
| +0x11 | bit 0 | Enabled | bit 0 | |
| +0x11 | bits 7-1 | — | carry data on real hardware (the OEM write capture stores **0xFE** here) | ❓ UNKNOWN |
| +0x12 | 1 | RX Duration Time | raw = value, 1–255 seconds | |
| +0x13 | 1 | Auto Emergency Call Timer | **raw 0–11 → `(raw+1)*10`** = 10…120 step 10 | |

**Empty entry detection**: skip entries where all 20 bytes are 0x00 or 0xFF.

> ⚠️ **Trap for porters: the entry is 20 bytes, not 40.** The reference implementation carries a
> `BLOCK_SIZE.DIGITAL_EMERGENCY: 40  // Bytes per digital emergency entry` constant that contradicts
> its own parser (`entrySize = 0x14`) and its own `DIGITAL_EMERGENCY_MAX` comment. The constant is
> dead — nothing in the codebase references it — and the hardware stride is 20 (`"DEmer 1"` … `"DEmer
> 8"` at 0x000, 0x014, 0x028 …). **20 is correct; ignore the 40.**

> (*`+0x0A` was previously documented as an "entry index" — refuted by the dump: the eight bytes
> read `01 02 03 04 05 02 04 00`, which is not an index sequence but Alarm Type values, all inside
> the 0–5 range.*)

> ⚠️ `+0x11` bits 7-1 carry data on hardware (the OEM CPS writes `0xFE`) — read-modify-write
> **bit 0 only** when toggling the Enabled flag.

---

### Section 2: Analog Emergency Systems (0x0AC–0x2FF)

**Entry size**: 36 bytes (0x24) — **CONFIRMED**  
**Entry formula**: `offset = 0x0AC + index * 0x24` — ⚠️ **base DERIVED**  
**Max entries**: **16** — `(0x300 − 0x0AC) / 36 = 16`, the section ends where encryption keys begin

> **Previously documented as**: *"No analog emergency entries have been captured from a real radio
> dump"*. That is no longer true. The OEM **write** capture contains exactly one entry, named
> `"AEMER 1"`, occupying **0x0D0–0x0F3** — 36 bytes exactly, with 0xFF fill on both sides:
>
> ```
> 00D0  41 45 4d 45 52 20 31 00 ff ff ff ff ff ff ff ff  |AEMER 1.........|
> 00E0  ff 00 00 00 01 00 0f 05 02 00 00 fe 0a 01 fe 00  |................|
> 00F0  01 01 01 fe ff ff ff ff ff ff ff ff ff ff ff ff  |................|
> ```
>
> This **confirms the 36-byte entry size**. It does *not* confirm the base: 0x0D0 = 0x0AC + 1 × 36,
> so either the section starts at 0x0AC and the CPS placed its first system in **slot 1**, or the
> section really starts at **0x0D0**. Unresolved. (The factory read capture has a lone `0x01` at
> 0x0B0 = 0x0AC + 4, the only non-zero byte in the whole region, which weakly favours 0x0AC.)

> ⚠️ **The field table below does not decode the one real entry we have.** Aligning the entry at
> 0x0D0 gives Revert Channel = 3840 and Squelch Mode = 4, both far outside their documented ranges;
> testing every byte alignment from −6 to +6 produces **no** alignment in which Alarm Type, Alarm
> Mode, Signalling, Revert Channel, Squelch Mode and ID Type are all in range. Treat the entire
> intra-entry layout as **⚠️ DERIVED / probably wrong**.

| Entry offset | Size | Field | Encoding | Confidence |
|-------------|------|-------|----------|-----------|
| +0x00 | 17 | Name | ASCII, NUL-terminated; encoder truncates to 16 chars + NUL | ⚠️ DERIVED |
| +0x11 | 1 | Padding | written as 0x00 | ⚠️ DERIVED |
| +0x12 | 1 | Alarm Type | 0 None, 1 Only Whistle, 2 Normal, 3 Secret, 4 Secret With Voice | ⚠️ DERIVED |
| +0x13 | 1 | Alarm Mode | 0 Emergency Alarm, 1 Alarm Call | ⚠️ DERIVED |
| +0x14 | 1 | Signalling | 0–3 = BDC1200-1 … BDC1200-4 | ⚠️ DERIVED |
| +0x15 | 2 | Revert Channel | uint16 LE, **stored = value − 1** | ⚠️ DERIVED |
| +0x17 | 1 | Squelch Mode | 0 Carrier, 1 CTC, **stored = value + 1** | ⚠️ DERIVED |
| +0x18 | 1 | ID Type | 0 None, 1 BDC1200, **stored = value + 1** | ⚠️ DERIVED |
| +0x19 | 1 | Flags | opaque status byte, round-tripped verbatim | ❓ UNKNOWN |
| +0x1A | 2 | Frequency / ID | uint16 LE, semantics unestablished | ❓ UNKNOWN |
| +0x1C | 1 | Enabled | bit 0 | ⚠️ DERIVED |
| +0x1D-+0x23 | 7 | — | never read, never written | ❓ UNKNOWN |

> **Previously documented as** Alarm Type at +0x11 … Enabled at +0x1B with a 20-byte reserved tail.
> Those offsets are each one lower than the implemented ones and the tail size was wrong (the entry
> is 36 bytes, so a tail starting at +0x1C is 20 bytes only if the entry ends at +0x2F). Corrected
> above to match the working implementation: +0x11 is padding and the tail is +0x1D-+0x23 (7 bytes).

**Write with caution.** With the intra-entry layout unverified, writing this section risks
producing entries the radio cannot interpret. Preserve the rest of block `0x10` when touching it.

---

### Section 3: Encryption / Privacy Keys (0x300–0x45F)

**Entry size**: 44 bytes (0x2C)  
**Entry formula**: `offset = 0x300 + (entryNumber − 1) * 0x2C` (**1-based** entry numbering)  
**Entry bases**: 0x300, 0x32C, 0x358, 0x384, 0x3B0, 0x3DC, 0x408, 0x434 (last ends 0x45F)  
**Max entries**: **8**

**CONFIRMED against hardware.** The factory read capture holds four populated keys on exactly this
stride:

```
0300  01 45 6e 63 72 79 70 74 20 31 00 04 00 00 00 00  |.Encrypt 1......|
0310  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  |................|
0320  00 00 00 00 ab cd ef 12 34 56 78 90 02 45 6e 63  |........4Vx..Enc|
0330  72 79 70 74 20 32 00 03 00 00 00 00 00 00 00 00  |rypt 2..........|
```

| Entry offset | Size | Field | Encoding | Confidence |
|-------------|------|-------|----------|-----------|
| +0x00 | 1 | Entry ID | 0x01–0x08, equals the entry number | |
| +0x01 | 10 | Name | ASCII, NUL-terminated / NUL-padded, max 10 chars | |
| +0x0B | 1 | Encryption Type | 0 None, 1 Custom, 2 ARC4, 3 AES128, 4 AES256 | |
| +0x0C | 32 | Key material | see placement note | |

Observed in the factory capture — entry ID, name and type all decode exactly:

| Entry | Base | ID | Name | Type | Key bytes within the 32-byte field |
|---|---|---|---|---|---|
| 1 | 0x300 | 01 | `Encrypt 1` | 4 (AES256) | `AB CD EF 12 34 56 78 90` at **+0x24–+0x2B** (last 8 of 32) |
| 2 | 0x32C | 02 | `Encrypt 2` | 3 (AES128) | `BC DE FA 12 34 56 78 90` at **+0x14–+0x1B** (last 8 of the first 16) |
| 3 | 0x358 | 03 | `Encrypt 3` | 2 (ARC4) | `CD EF AB 12 34` at **+0x0C–+0x10** (first 5) |
| 4 | 0x384 | 04 | `Encrypt 4` | 1 (Custom) | `DE FA BC 12 34 56 78` at **+0x0C–+0x12** (first 7) |
| 5 | 0x3B0 | 00 | — | — | `00` at +0x00 then 43 × `FF` — an erased slot, *not* uniformly 0xFF |
| 6–8 | 0x3DC… | 00 | all 0x00 | 0 | empty slots |

The mixed fill of entry 5 is why the "empty" test has to accept **either** 0x00 or 0xFF per byte
rather than a single fill value.

> ⚠️ **DERIVED — key placement depends on the encryption type.** For the two AES entries the key
> bytes sit at the **end** of the type's key length (32 bytes for AES256, 16 for AES128), i.e. the
> key is right-aligned with leading zero padding; for ARC4 and Custom the bytes start at **+0x0C**,
> left-aligned. Two samples of each is not proof, but any implementation that assumes "key always
> starts at +0x0C, zero-padded on the right" will mis-render AES keys. Preserve all 32 key bytes
> verbatim — trimming trailing zeros is lossy for AES keys.

**Related channel field**: channel byte 0x2A holds the per-channel key reference for digital
channels — 0 = None, 1–8 = key ID. See *Byte 0x2A* in the channel-flags reference.

---

## 0x41 - VFO A / VFO B Channel Records

**Metadata / logical block ID**: 0x41  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: the last channel block — and the home of the two VFO channel records

**Implementation reference**: `src/radios/dm32uv/protocol.ts` (`readRadioSettings` / `writeRadioSettings`), `src/radios/dm32uv/structures.ts` (`parseChannel` / `encodeChannel`), `src/radios/dm32uv/blockLayouts.ts`

VFO A and VFO B are **not** stored in the Radio Settings block. They are two ordinary **48-byte
channel records** living at the very end of block 0x41, addressed by the reference implementation as
pseudo-channels **4001** (VFO A) and **4002** (VFO B) and parsed/encoded with the normal channel
routines.

| Item | Value |
|------|-------|
| VFO A record | block-relative **0x0F9F**, 48 bytes |
| VFO B record | block-relative **0x0FCF**, 48 bytes |
| VFO A TX contact | block **0x43**, fixed offset **0x0FFA** (2 bytes) |
| VFO B TX contact | block **0x43**, fixed offset **0x0FFC** (2 bytes) |

The two records are contiguous (`0x0FCF − 0x0F9F = 0x30 = 48`) and VFO B ends at 0x0FFE, immediately
before the block's metadata byte at 0x0FFF.

Field offsets inside each record (they are just channel records, so the full 48-byte channel layout
applies — see *Channel Structure*):

| Field | VFO A | VFO B |
|-------|-------|-------|
| Name (16 B ASCII) | 0x0F9F | 0x0FCF |
| RX frequency (4 B BCD) | 0x0FAF | 0x0FDF |
| TX frequency (4 B BCD) | 0x0FB3 | 0x0FE3 |
| Mode flags | 0x0FB7 | 0x0FE7 |

**Hardware worked example** (factory read capture, VFO B):

```
0FDF  00 25 20 43      RX
0FE3  00 25 20 43      TX
0FE7  14               mode flags
```

Reverse the 4 bytes → `43 20 25 00` → read as BCD digit pairs → `4 3 2 0 2 5 0 0` → **432.02500 MHz**,
a valid UHF frequency. This confirms both the VFO B offsets and the channel frequency encoding
(little-endian byte order, BCD nibble pairs, 8 digits, `XXX.XXXXX`).

**VFO A is confirmed by the OEM write capture**, which carries a different codeplug and puts a valid
frequency at *both* record bases:

```
read capture   0FAF  33 38 43 43   VFO A RX = 434.33833 MHz    0FDF  00 25 20 43   VFO B RX = 432.02500 MHz
write capture  0FAF  00 00 00 40   VFO A RX = 400.00000 MHz    0FDF  00 00 00 40   VFO B RX = 400.00000 MHz
               0FB3  00 00 00 40   VFO A TX = 400.00000 MHz    0FE3  00 00 00 40   VFO B TX = 400.00000 MHz
```

Two independent codeplugs both decode as valid frequencies at 0x0FAF/0x0FB3 and 0x0FDF/0x0FE3, 48
bytes apart — so both record bases are hardware-confirmed, not implementation-only. (Everything below
0x0F9F is 0x00/0xFF in the factory read: this radio has no channels in block 0x41.)

> ⚠️ **DERIVED — latent collision, not a reachable one.** Block 0x41 is simultaneously the last
> channel block *and* the VFO block, and the two uses do overlap on paper: with 48-byte channel
> records packed from 0x000, the **84th** record spans 0x0F90-0x0FBF (covering VFO A at 0x0F9F) and
> the **85th** spans 0x0FC0-0x0FEF (covering VFO B at 0x0FCF). But it cannot happen at the documented
> capacity. The reference implementation packs **84 channels into the first channel block** (0x12,
> after its 16-byte header) and **85 into each of the other 47**, so block 0x41 holds channels
> **3995-4079**; the enforced 4000-channel maximum fills only its first six records (0x000-0x11F).
> Reaching VFO A would take channel 4078.
> A previous revision stated the collision as fact — it is not, at 4000 channels. Any implementation
> that packs 85 channels into every block *including* 0x41 without the cap, or that raises the channel
> limit, would silently overwrite both VFOs.

> ⚠️ **VFO TX-contact write is not implemented** in the reference implementation — it is read-only:
> *"NOTE: TX Contact for VFOs is stored in block 0x43, but we don't write it here to avoid potential
> corruption. VFO TX Contact write is disabled until properly debugged. … VFO Talk Group changes
> won't persist."* Reading works; the TX contact is only copied into the VFO when the VFO is in a
> digital mode.

Hardware read capture, block 0x43 tail (re-verified byte-by-byte against the capture; the trailing
`43` is the block's own metadata byte at 0x0FFF, which anchors the alignment):

```
0FF8  00 00 00 01 01 01 00 43
```

So VFO A's slot at 0x0FFA-0x0FFB = `00 01` (TG index 1, analog) and VFO B's at 0x0FFC-0x0FFD =
`01 01` (TG index 1, digital — bit 0 of byte 0 is the digital flag).

The OEM **write** capture confirms the same two slots independently — its block 0x43 tail is

```
0FF8  ff ff 0e 01 0e 01 ff 43
```

two identical 2-byte records at exactly 0x0FFA and 0x0FFC, framed by 0xFF fill, with 0x0FFE unused
and the metadata byte at 0x0FFF. Both decode as TG index 1, analog (`0x0E` has a zero high nibble and
bit 0 clear; bits 1-3 of byte 0 are ❓ UNKNOWN and carry data here).

> *Previously documented as `0xFFA = 01 01`, `0xFFC = 01 00` — incorrect: the same run of bytes
> quoted one position early. The values above are anchored by the metadata byte at 0x0FFF and
> corroborated by the write capture.*

---

## 0x42 / 0x43 - TX Contact Assignment (per-channel Talk Group index)

**Metadata**: 0x42 (channels 1-2047), 0x43 (channels 2048-4000 + VFO A/B)  
**Block Size**: 4 KB (0x1000 bytes) each  
**Purpose**: Maps each channel to the Talk Group it transmits to, as a 12-bit index into the Talk Groups list in block 0x44

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseTxContactEntry`, `encodeTxContactEntry`, `getTxContactOffset`, `parseAllTxContacts`, `encodeTxContactForChannel`), `src/radios/dm32uv/protocol.ts` (`writeTxContactBlocks`)

Both blocks are confirmed present by hardware capture: the OEM CPS reads *and writes* 0x42 and 0x43
as ordinary 4 KB blocks, 2 bytes per channel. In the OEM **write** capture the only non-0xFF payload
bytes in block 0x43 are `0x0FFA-0x0FFD` = `0e 01 0e 01` — exactly the VFO A / VFO B slots described
below. (**Previously documented as** `01 01` at 0x0FFA and `01 00` at 0x0FFC, quoted from the read
capture one byte early; the read capture actually holds `00 01` at 0x0FFA and `01 01` at 0x0FFC.)

**This block does not store DMR IDs.** It stores an *index* into the Talk Groups list (block 0x44);
the 24-bit DMR talk group ID lives there.

### Per-channel entry (2 bytes)

```
Byte 0:  bits 7-4 = TG index bits 11-8
         bits 3-1 = Reserved ❓ UNKNOWN
         bit  0   = Digital flag (1 = Digital, 0 = Analog/Mixed)
Byte 1:  bits 7-0 = TG index bits 7-0
```

| Item | Value |
|------|-------|
| TG index width | **12 bits, 0-4095** |
| Decode | `index = ((byte0 >> 4) & 0x0F) << 8 \| byte1` — equivalently `(byte0 >> 4) * 256 + byte1` |
| Encode | `byte0 = (((index >> 8) & 0x0F) << 4) \| digital` ; `byte1 = index & 0xFF` |
| Digital flag | `byte0 & 0x01` |
| Index 0 | **None** (no talk group assigned) |
| Index 1+ | 1-based **physical slot position** in the Talk Groups list (block 0x44) |
| Encode clamp | index is clamped to `0..4095` |

**Worked examples from hardware.**

| Source | Bytes | Decode |
|---|---|---|
| Factory read, 0x43 `0x0FFA` (VFO A) | `00 01` | index **1**, digital flag **clear** |
| Factory read, 0x43 `0x0FFC` (VFO B) | `01 01` | index **1**, digital flag **set** |
| OEM write, 0x43 `0x0FFA` / `0x0FFC` | `0e 01` | index **1**, digital flag **clear**, bits 3-1 = `111` |
| OEM write, 0x42 pair 3 (channel 4, `"RIC TAC A"`) | `0f 03` | index **3** = talk group 3 `"TAC A"` ✅ |
| OEM write, 0x42 pair 13 (channel 14, `"RIC Monitor TS1"`) | `0f 00` | index **0** = None ✅ |

**The channel → offset mapping is confirmed by the write capture.** Walking block 0x42 as
`(channel - 1) * 2` and resolving each index against the 16 talk groups written to block 0x44 in the
same session produces a run of exact name matches: channel 3 `RIC HEARS` → `HEARS Link`, channel 4
`RIC TAC A` → `TAC A`, channel 5 `RIC TAC B` → `TAC B`, channel 6 `RIC TAC C*` → `TAC C`, channel 7
`RIC VA Shen Vall` → `VA Shen Valley`, channels 8-10 likewise, channel 12 `RIC Echo Test` →
`Parrot`, channel 13 `RIC Clear Timesl` → `Clear Timeslot`, channels 14-15 `RIC Monitor TS*` → None.

**The digital flag is confirmed by the factory read.** Its channels 1-9, 17 and 22-25 are digital and
their block-0x42 entries all have bit 0 set; its analog channels 10-16 and 18-21 all have bit 0
clear.

**⚠️ Bits 3-1 of byte 0 are ❓ UNKNOWN and carry data.** They are **not** always zero on hardware:
the factory image holds `000` (byte 0 = 0x00/0x01) while the OEM CPS writes `111`
(byte 0 = 0x0E/0x0F) for every channel and both VFOs. The radio accepts both — preserve them
rather than composing byte 0 from scratch.

### Block split point and offset formulas

Offsets are **block-relative** (within each 4 KB block).

| Channel range | Block | Block-relative offset |
|---------------|-------|-----------------------|
| 1 - 2047 | **0x42** | `(channel - 1) * 2` |
| 2048 - 4000 | **0x43** | `(channel & 0x7FF) * 2` |
| VFO A (4001) | **0x43** | fixed **0x0FFA** |
| VFO B (4002) | **0x43** | fixed **0x0FFC** |

Note that channel **2048** belongs to block **0x43** at offset 0 — the split is 2047 / 2048, not
2048 / 2049 as sometimes stated.

**Combined-buffer notation**: if 0x42 and 0x43 are viewed as one contiguous 8 KB buffer, VFO A is
at 0x1FFA and VFO B at 0x1FFC. Because the blocks are handled separately as 4 KB units, the
block-relative offsets 0x0FFA / 0x0FFC are the ones to implement.

### Resulting byte map

| Block | Last channel entry | Occupied | Leftover |
|-------|--------------------|----------|----------|
| 0x42 | ch 2047 at `(2047-1)*2 = 0x0FFC` (spans 0x0FFC-0x0FFD) | 0x0000-0x0FFD | 0x0FFE ❓ unused; 0x0FFF = metadata byte |
| 0x43 | ch 4000 at `(4000 & 0x7FF)*2 = 0x0F40` (spans 0x0F40-0x0F41) | 0x0000-0x0F41 | 0x0F42-0x0FF9 ❓ UNKNOWN; VFO A 0x0FFA-0x0FFB; VFO B 0x0FFC-0x0FFD; 0x0FFE ❓ unused; 0x0FFF = metadata byte |

**Collision hazard, not an actual collision**: in block 0x42, channel 2047 occupies block-relative
0x0FFC-0x0FFD — the same *block-relative* offset that VFO B uses in block 0x43. Different blocks, no
conflict. Do not conflate the two.

The leftover ranges are ❓ UNKNOWN rather than known-padding, but neither capture puts anything in
them: in the OEM write capture block 0x43 is 0xFF everywhere except the two VFO slots and 0x0FFF, and
block 0x42 is 0xFF from the end of its channel entries to 0x0FFE.

### Read / write behaviour

- On read, the TX contact is applied **only to digital channels** (`Digital` or `Fixed Digital`); an
  analog channel keeps the placeholder `contactId = 0` from the channel parser.
- **Channel byte 0x2B is NOT the TX contact.** It is the DMR Radio ID index used for TX (0xFF = None).
- Writing is a **read-modify-write**: the reference implementation refuses to write unless both 0x42
  and 0x43 are already in cache — *"TX Contact blocks not in cache - skipping TX Contact write to
  avoid data loss. Read from radio first!"*. The blocks are never generated from scratch.
- ⚠️ **Asymmetry**: the writer emits an entry for **every** channel including analog ones (digital
  flag 0, index `contactId ?? 0`), while the reader only consumes entries for digital channels. An
  analog channel's stored TG index is therefore zeroed on write.

### Code Structure (C/C++)

```c
// Decode
static inline uint16_t tx_contact_index(uint8_t b0, uint8_t b1) {
    return (uint16_t)((((b0 >> 4) & 0x0F) << 8) | b1);   // 0 = None
}
static inline bool tx_contact_is_digital(uint8_t b0) { return (b0 & 0x01) != 0; }

// Encode
static inline void tx_contact_encode(uint16_t index, bool digital,
                                     uint8_t *b0, uint8_t *b1) {
    if (index > 4095) index = 4095;
    *b0 = (uint8_t)((((index >> 8) & 0x0F) << 4) | (digital ? 0x01 : 0x00));
    *b1 = (uint8_t)(index & 0xFF);
}

// Offset routing: returns block metadata id, writes block-relative offset to *off
static inline uint8_t tx_contact_offset(uint16_t channel, uint16_t *off) {
    if (channel == 4001) { *off = 0x0FFA; return 0x43; }        // VFO A
    if (channel == 4002) { *off = 0x0FFC; return 0x43; }        // VFO B
    if (channel >= 1 && channel <= 2047) { *off = (uint16_t)((channel - 1) * 2); return 0x42; }
    if (channel >= 2048) { *off = (uint16_t)((channel & 0x7FF) * 2); return 0x43; }
    *off = 0; return 0x42;                                       // fallback
}
```

---

## 0x44 - Talk Groups (the radio's "TX Contacts" list)

**Metadata**: 0x44 (first block); the talk-group block range is **0x44-0x48** (5 logical block slots)  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: The talk-group list — name, 24-bit DMR contact number and call type per entry. This is what the per-channel TX Contact index in blocks 0x42/0x43 points at.

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseQuickContacts` / `encodeQuickContacts` — named "QuickContact" in code, but their own doc comments say "Talk Groups"), `src/radios/dm32uv/protocol.ts` (`readQuickContacts` / `writeQuickContacts`)

**Terminology warning**: the reference implementation overloads the word "contact" three ways —
`Contact` = the big DMR contact *database* (92-byte entries, raw region); `QuickContact` = a **Talk
Group** in this block; `Channel.contactId` = a 12-bit **index** into this block's list. They are three
different things.

### Talk-group block range 0x44-0x48 — block math

The OEM CPS write capture walks logical block IDs **0x44, 0x45, 0x46, 0x47, 0x48** contiguously, so
the talk-group region is **5 block slots** (**CONFIRMED**). On the captured factory radio only 0x44
had a physical page; 0x45-0x48 were written to the "no physical page" sentinel address 0xFFF001.

Capacity arithmetic per block (from the entry formula below):

- Largest fully-contained entry: `25 + (N-2)*24 + 24 ≤ 4095` ⇒ **N ≤ 170** (entry 170 spans
  0x0FD9-0x0FF0); with a trailing all-zero sentinel entry, **169 real talk groups** per block.
- 5 blocks × 169 = 845 ≥ the advertised maximum of **800** talk groups. ⚠️ DERIVED both ways: the
  800 figure is unverified, and how entries are distributed across 0x45-0x48 is **not
  established** — no capture contains a populated 0x45-0x48 page.
- Beware the metadata byte: an entry must end at or before `0x0FFE`, not `0x0FFF`.

### Block layout

| Offset | Size | Field | Notes |
|--------|------|-------|-------|
| 0x000 | 1 | Block header byte ⚠️ DERIVED | 0x00 in the factory image, **0xFF in the OEM CPS write**. The reference implementation writes 0x00 and calls it mandatory — *"Contact 1 ALWAYS has 1-byte header (0x00) - this is critical for the radio to recognize the block"* — but the OEM CPS does not, so "critical" is unsupported |
| 0x001 | — | Entry 1 | 24 bytes of entry data follow the header byte (entry 1 is 25 bytes total) |
| … | 24 | Entries 2+ | 24 bytes each |
| after last entry | 24 | Sentinel ⚠️ DERIVED | The reference implementation appends one all-zero 24-byte entry. The OEM CPS writes **no sentinel** — it simply leaves 0xFF from the end of the last entry onward |
| after last entry → 0x0FFE | — | Tail fill ⚠️ DERIVED | **0x00** as written by the reference implementation and as found in the factory image; **0xFF** as written by the OEM CPS. The radio accepts both |
| 0x0FFF | 1 | Metadata / logical block ID | **0x44** |

**Entry offset formula** (N is 1-based):

```
offset(1) = 0x000                      (1 header byte + 24 data bytes = 25)
offset(N) = 25 + (N - 2) * 24          for N >= 2
```

Both hardware captures agree on the **24-byte stride** and on the field positions. Because entry 1 is
25 bytes and entries 2+ are 24, the *name* of every entry N lands at `24*(N-1) + 2` — which is how the
stride was measured in both dumps:

| Capture | Names | Positions |
|---|---|---|
| Factory read | `Contacts 1` … `Contacts 10` (10 talk groups) | 0x002, 0x01A, 0x032, … |
| OEM CPS write | `WV Statewide`, `Simplex`, `TAC A`, … (16 talk groups) | 0x002, 0x01A, 0x032, … |

⚠️ Note that the byte-level evidence is equally consistent with a simpler model — **every** entry is
24 bytes with **two** leading fill bytes and the name at +0x02 — because both captures show two fill
bytes before every name (`00 00` factory, `FF FF` OEM) and a uniform 24-byte stride. The
"1-byte block header + 1-byte flag" split above is the reference implementation's reading; it
produces identical bytes, so nothing in the captures decides between them.

### Entry field table (24 bytes)

Offsets are relative to the entry start. For **entry 1 add +1** to every offset, because of the
block header byte.

| Rel. offset (entries 2+) | Rel. offset (entry 1) | Size | Type | Field | Encoding |
|--------------------------|-----------------------|------|------|-------|----------|
| — | 0x00 | 1 | u8 | Block header ⚠️ DERIVED | 0x00 in the factory image, 0xFF in the OEM write; entry 1 only |
| 0x00 | 0x01 | 1 | u8 | Flag ⚠️ DERIVED | The reference implementation reads it as `0x00` = created by PC/CPS, `0x01` = created on the radio, and always writes 0x00. **The OEM CPS writes 0xFF**, so the two-value enumeration is not established |
| 0x01 | 0x02 | 16 | ASCII | Name | fixed 16-byte field; NUL-terminated, then 0x00 padding (factory) or 0xFF padding (OEM). The parser stops at the first 0x00 **or** 0xFF |
| 0x11 | 0x12 | 1 | u8 | ❓ UNKNOWN (fill) | The reference implementation writes 0x00 and calls it a "null terminator" for the name; the factory image has 0x00 but **the OEM CPS writes 0xFF**. It is fill, not a terminator — the name's own NUL is inside the 16-byte field |
| 0x12 | 0x13 | 3 | uint24 LE | Contact number (DMR TG/ID) | little-endian — **confirmed**: `WV Statewide` = 3154 → `52 0c 00`; `HEARS Link` = 43277 → `0d a9 00`; `Contacts 10` = 0xFFFFFF → `ff ff ff` |
| 0x15 | 0x16 | 1 | u8 | Call type | `0x03` Private Call, `0x04` Group Call, `0x05` All Call — **confirmed**: the factory image's 5 Group / 4 Private / 1 All Call groups carry exactly those values and match the type bytes in block 0x0B. **No padding before it** |
| 0x16 | 0x17 | 2 | — | Fill ⚠️ DERIVED | 0x00 0x00 in the factory image and from the reference encoder; 0xFF 0xFF in the OEM write |
| **0x18** | **0x19** | — | — | *(next entry starts here)* | 24 / 25 bytes total |

**Name encoding on write**: only printable ASCII (0x20-0x7E) survives; the name is truncated to 16
bytes.

**Empty entry (parse)**: if the first name byte is 0x00, the slot is skipped by advancing 23 bytes
from the name start **and incrementing the index** — so index numbering counts empty slots.
Consequently a talk group's index is its **1-based physical slot position**, which is exactly what
blocks 0x42/0x43 and the 0x0B index tables refer to.

**Empty-list guard (hardware-motivated)**: if zero talk groups are supplied, the encoder injects a
default entry `{ name: "All", contactNumber: 16777215 (0xFFFFFF), callType: 0x05 }` with the comment
*"This prevents the radio from crashing when the block is empty"*.

### Block 0x06 offset 0x1FF is **not** a Talk Groups counter

The 1-byte count at block **0x06** offset **0x1FF** was long believed to be the talk-group counter.
**Hardware falsifies that label** — both captures line the byte up with the **analog/DTMF contact**
names that start at 0x06:0x200 (32-byte stride), not with the talk groups in block 0x44:

| Capture | 0x06:0x1FF | Analog contact names at 0x06:0x200+ | Talk groups in 0x44 |
|---|---|---|---|
| Factory read | **7** | 7 (`AContact 1`, `AContacts 1`…`AContacts 6`) | 10 |
| OEM CPS write | **1** | 1 (`AContacts 1`) | 16 |

The byte tracks the analog-contact count in both, and matches the talk-group count in neither.
The positive identification is not proven (nothing reads it back); **leave the byte alone** until
the field is identified. Confirming experiment: add one analog contact and re-read; then add one
talk group and re-read; whichever moves the byte wins.

> *Previously documented as "Talk Groups counter" and, earlier still, as a "final configuration
> byte" — both incorrect.*

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    uint8_t  flag;          // +0x00: UNKNOWN. 0x00 from the factory image / reference impl,
                            //        0xFF from the OEM CPS
    char     name[16];      // +0x01: NUL-terminated, then 0x00 (factory) or 0xFF (OEM) padding
    uint8_t  name_fill;     // +0x11: UNKNOWN fill - 0x00 (factory) / 0xFF (OEM), NOT a terminator
    uint8_t  contact_no[3]; // +0x12: 24-bit DMR contact number, little-endian
    uint8_t  call_type;     // +0x15: 0x03 Private, 0x04 Group, 0x05 All Call
    uint8_t  fill[2];       // +0x16: 0x00 0x00 (factory) / 0xFF 0xFF (OEM)
} dm32_talk_group_t;
#pragma pack(pop)

static_assert(sizeof(dm32_talk_group_t) == 24, "Talk group entry must be 24 bytes");

// Block layout: [header byte][entry 1][entry 2]…[fill…][0xFFF = 0x44]
// Reference impl: header 0x00, one all-zero 24-byte sentinel entry, then 0x00 fill.
// OEM CPS:        header 0xFF, no sentinel, 0xFF fill. Both are accepted by the radio.
// offset(1) = 0;  offset(N) = 25 + (N - 2) * 24   for N >= 2
```

---

## 0x65 - Roaming Zones

**Metadata**: 0x65  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR roaming zone configurations  
**Status**: stride and name offset hardware-confirmed; **field layout `⚠️ DERIVED`** (from CPS
decompilation). Not implemented in the reference implementation, so nothing has ever exercised the
field table below against a radio.

**Hardware anchors** (read capture): a 16-byte header at `0x000` (first bytes `03 00 01 01` — the
`03` plausibly a count: three zones exist), then `"Roam Zone 1"` at `0x010`, `"Roam Zone 2"` at
`0x031`, `"Roam Zone 3"` at `0x052` — **stride 33 (`0x21`)**, name at entry offset `+0x10`.

### Entry Structure (33 bytes) — `⚠️ DERIVED`

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 1 | Bit flag | Bit 0: enabled/disabled |
| +0x02, +0x03 | 1, 1 | Values | stored as `value + 1`; +0x03 max 0x40 |
| +0x10 | 16 | Name | ASCII, null-terminated — offset hardware-confirmed |
| +0x20 | 1 | Channel count / index | last byte of the entry |

(*A previous revision listed a 4-byte "Count/value DWORD" at +0x00 overlapping the fields at
+0x01–+0x03 — internally impossible; dropped.*) Max 16 channels per roaming zone `⚠️ DERIVED`;
where the channel list actually lives within 33 bytes is unresolved.

---

## 0x66 - Roaming Channels

**Metadata**: 0x66  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR roaming channel configurations  
**Status**: stride and name offset hardware-confirmed; **field layout `⚠️ DERIVED`** (CPS
decompilation). Not implemented in the reference implementation.

**Hardware anchors** (read capture): `"Roam CH 1"` at `0x000`, `"Roam CH 2"` at `0x01A`,
`"Roam CH 3"` at `0x034` — **stride 26 (`0x1A`)**, entries from the buffer base, no header.
Entry 1's bytes `+0x10`–`+0x19` read `00 00 00 40 00 00 00 41 00 00` — `00 00 00 40` /
`00 00 00 41` decode as 400.00000 / 410.00000 MHz under the standard BCD codec, corroborating the
RX/TX offsets below.

### Entry Structure (26 bytes) — `⚠️ DERIVED` except as noted

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 16 | Name | ASCII, null-terminated — hardware-confirmed |
| +0x10 | 4 | RX Frequency | BCD, standard codec — corroborated by capture |
| +0x14 | 4 | TX Frequency | BCD — corroborated by capture |
| +0x18 | 1 | Color Code | 0-15 |
| +0x19 | 1 | Time Slot | 1 or 2 |

**Count Storage**: offset `0xFF0` (1 byte, max 150) `⚠️ DERIVED` — from CPS decompilation only.

---

## 0x67 - DMR Radio ID List

**Metadata**: 0x67  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Radio's own DMR Radio IDs

**Implementation reference**: `src/radios/dm32uv/structures.ts` (`parseDMRRadioIDs` / `encodeDMRRadioID`), `src/radios/dm32uv/protocol.ts` (`readDMRRadioIDs` / `writeDMRRadioIDs`)

Both hardware captures confirm the entry base, the stride, the count width and the ID encoding:

```
factory read   0x000  05 00 00 00 00 ...            count = 5
               0x010  7b 00 00 52 61 64 69 6f 20 31 00 ...   ID 123, "Radio 1"
               0x020  02 00 00 52 61 64 69 6f 20 32 00 ...   ID 2,   "Radio 2"   (stride 16)

OEM CPS write  0x000  01 ff ff ff ff ...            count = 1
               0x010  87 d6 12 43 48 41 4e 47 45 4d 45 00 ff ...  ID 0x12D687, "CHANGEME"
```

`87 d6 12` read as uint24 **little-endian** is 0x12D687 = **1234567** — a plausible DMR ID and a
decisive check against a BCD reading (which would give "87 d6 12" nonsense digits). In the OEM write
the only non-0xFF payload bytes in the whole block are 0x000 and 0x010-0x01B.

### Structure Overview

**Entry Size**: 0x10 (16) bytes per entry  
**Entry Base Offset**: **0x10** — offsets 0x00-0x0F are the block header, not entry 0  
**Max Entries**: **250** (limit enforced by the implementation). Entry 249 sits at `0x10 + 249*16 = 0x0FA0` and ends at 0x0FB0, clear of the metadata byte  
**Count Storage**: Offset 0x00, **1 byte**

**Entry Calculation**: `offset = 0x10 + entry_num * 0x10` (entry_num 0-based)

| Offset | Size | Field | Encoding |
|--------|------|-------|----------|
| 0x000 | 1 | ID count | u8, capped at 250 on read and written as `count & 0xFF`. Note the factory image has `05 00 00 00` here, which a 4-byte reading also satisfies; the OEM CPS write has `01 FF FF FF`, which **rules out** a uint32 and confirms a 1-byte field |
| 0x001-0x00F | 15 | ❓ UNKNOWN | 0x00 in the factory image, 0xFF in the OEM write; never read. ⚠️ The reference implementation zeroes them |
| 0x010 | 16 | Entry 0 | first real entry |
| 0x0FFF | 1 | Metadata / logical block ID | 0x67 |

> **Previously documented as**: count = "4 bytes, DWORD, little-endian", entry base 0x00, max 256
> entries. **Corrections**: the count is read and written as a **single byte** by the working
> implementation (max 250, so one byte always suffices); entries start at **0x10**; the maximum is
> **250**, not 256. The reference implementation's own parser comment hedges on the entry base
> (*"the spec says 'Entry Base Offset: 0x00', so let's try both"*) — the hardware dump settles it at
> 0x10.

### Entry Structure (16 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 3 | DMR Radio ID | **uint24 little-endian binary** (`b0 \| b1<<8 \| b2<<16`), *not* BCD — confirmed by `87 d6 12` = 1234567 in the OEM write |
| +0x03 | 12 | Name | ASCII, null-terminated, then 0x00 (factory) or 0xFF (OEM) padding. The encoder writes at most **11** chars + 0x00 terminator |
| +0x0F | 1 | ❓ UNKNOWN | The parser slices only bytes [3,15); the encoder never writes byte 15 (it stays 0x00 from the entry fill). The OEM CPS leaves 0xFF there |

### ID Encoding

DMR Radio IDs are stored as **3 bytes, little-endian binary**, and are displayed as a decimal number.
Some older CPS-derived documentation shows them formatted with `"%02X %02X %02X"` as hex — that is a
*display* convention in the CPS, not the storage encoding.

**Fill byte**: the factory image fills unused space with 0x00, **the OEM CPS with 0xFF** — both
are accepted; empty-entry detection must treat an all-0xFF **or** all-0x00 entry as empty.

**Index semantics**: a skipped slot still consumes its index, so a radio ID's index is its
**physical slot number**. That is what the channel field at byte 0x2B references (0xFF = None;
whether the index is 0- or 1-based is contested — see *Byte 0x2B* in the channel-flags reference).

⚠️ Header bytes `0x001`-`0x00F` carry the writer's fill convention and their purpose is unknown —
preserve them on write.

**Important Note**: This stores the **radio's own DMR Radio IDs** - the IDs that identify this specific radio when transmitting on digital channels. This is different from the DMR contact database (V-frame 0x0F region, 92-byte entries) which contains all known contacts, and from the Talk Groups list (block 0x44).

---

## DMR Contact Database (V-frame 0x0F region)

**Storage**: raw memory region — **not** a metadata block. There is no metadata/logical-ID tag for it.  
**Entry Size**: **92 bytes (0x5C)**, fixed  
**Purpose**: The large user contact address book (name, DMR ID, callsign, city, province, country, remark)

**Implementation reference**: `src/radios/dm32uv/protocol.ts` (`readContacts` / `writeContacts`), `src/radios/dm32uv/structures.ts` (`parseContactEntry` / `encodeContactEntry`)

**⚠️ SIZE CORRECTION**: earlier revisions of this document described contacts with **109-byte**
entries. That is the **RX-group** entry size and was copied into the contacts material by mistake.
**A DMR contact entry is 92 bytes (0x5C).** The 109-byte figure must not be used for contacts.

### Locating the region

The region is **not** at a fixed address and is **not** discovered by metadata scanning. Query two
V-frames:

| V-frame | Request | Response | Meaning |
|---------|---------|----------|---------|
| 0x0F | `56 00 00 00 0F` | `56 0F 08` + 8 bytes | `start_addr` (uint32 LE) + `end_addr` (uint32 LE) |
| 0x10 | `56 00 00 00 10` | `56 10 <len>` + payload | maximum contact count (little-endian) |

Exact bytes from the read capture:

```
→ 56 00 00 00 0f      ← 56 0f 08 00 80 27 00 ff bf 6d 00     start 0x00278000, end 0x006DBFFF
→ 56 00 00 00 10      ← 56 10 03 50 c3 00                    0x00C350 = 50000
```

Observed values:

| Firmware | V-frame 0x0F range | V-frame 0x10 max contacts |
|----------|--------------------|---------------------------|
| Standard (capture: DM32.01.01.046) | `0x278000` - `0x6DBFFF` (≈ 4,472 KB) | **50000** (`0xC350`) |
| L01 | `0x278000` - `0xFFFFFF` (≈ 13.5 MB) | 150000 (`0x000249F0`) ⚠️ DERIVED — not present in any capture available here |

> **Correction to the V-frame 0x10 response length.** The older `ContactReadWrite.md` note documents
> the response as `[0x56][0x10][0x04][count_4_bytes]`, i.e. a length byte of **0x04**. The hardware
> capture shows a length byte of **0x03** with the payload `50 c3 00` (= 50000). Document **3 bytes
> as observed**; the 4-byte claim is unsupported by capture.
>
> Implementation consequence: the reference implementation only accepts a V-frame 0x10 payload of
> **≥ 4 bytes**; against this radio it therefore falls back to its hardcoded default of 50000 — which
> happens to equal the real value, masking the mismatch. Implementations should accept a 3-byte
> payload.

**Disabled case**: if `start_addr == 0 && end_addr == 0`, contacts are disabled — return an empty
list rather than reading.

**Capacity from the range**: `(end_addr - start_addr + 1) / 0x5C` — the range is inclusive, so the
`+ 1` belongs there. Neither figure is a hard limit; the real cap is the V-frame 0x10 count.

### Region header and block math

**Evidence tier.** The OEM CPS *does* touch this region in both captures — contrary to an earlier
note in this document that said it did not:

- Read capture: a **4-byte read at 0x278000** returning `01 00 00 00`, followed by a full **4 KB read
  at 0x278000**.
- Write capture: a **4 KB write to 0x278000** with the same content.

That one block is the whole hardware sample, and it contains exactly **one** contact — so the region
header and the start of entry 0 are hardware-confirmed, while everything about the *second and later*
blocks (the 44-per-block walk) still rests only on the reference implementation's live read/write
path (`readContacts` / `writeContacts`).

Raw payload of the 4 KB block at 0x278000 (identical in the read and the write capture; everything
not shown is 0xFF):

```
0x000  01 00 00 00 ff ff ff ff ff ff ff ff ff ff ff ff   count = 1 (uint32 LE), then 0xFF
0x010  43 6f 6e 74 61 63 74 73 20 31 00 ff ff ff ff ff   "Contacts 1\0" + 0xFF pad   (entry 0 name)
0x020  01 00 00 f0 ff ff ff ff ff ff ff ff ff ff ff ff   entry 0 +0x10: 01 00 00 f0
```

The region is read and written in 4 KB blocks aligned to `floor(start_addr / 0x1000) * 0x1000`.

| Field | Address | Size | Encoding |
|-------|---------|------|----------|
| Contact count | `start_addr + 0x00` | 4 | uint32 LE — confirmed; the OEM CPS issues an explicit 4-byte read of exactly this field |
| ❓ UNKNOWN | `start_addr + 0x04` | 12 | **0xFF** in both captures. ⚠️ The reference implementation explicitly zero-fills these 12 bytes on write, which is a divergence from the OEM CPS — previously documented here as "Padding, 0x00" |
| Entry 0 | `start_addr + 0x10` | 92 | first contact — confirmed |

Per 4 KB block — ⚠️ **DERIVED** (only one contact block exists in the captures, so the walk beyond
block 0 is unverified):

| Block | Entry offset formula | Entries per block |
|-------|----------------------|-------------------|
| Block 0 (contains the 16-byte header) | `0x10 + i * 0x5C`, `i = 0..43` | 44 |
| Block 1+ | `i * 0x5C`, `i = 0..43` | 44 |

Global index → location: `block = contact_index / 44`, `index_in_block = contact_index % 44`.

⚠️ **The flat formula does not work past the first block.** The original contact note gives
`contact_address = start_addr + contact_index * 0x5C`. That form is wrong twice over: it omits the
16-byte region header, and — more importantly — entries do **not** straddle 4 KB block boundaries.
44 × 92 = 4048 bytes, so each block leaves 48 bytes (block 0: 32 bytes) unused before the next block
restarts entry numbering at offset 0x00. Use the block-walking form above; it is what the working
implementation reads and writes.

**Trailing bytes**: block 0's last entry ends at `0x10 + 44*0x5C = 0x0FE0`; blocks 1+ end at
`44*0x5C = 0x0FD0`. Everything from the end of the last written entry up to (not including) 0x0FFF is
filled with **0xFF**.

**Byte 0x0FFF is preserved verbatim**, never stamped with a metadata value: *"Contacts are in a raw
data region (no metadata blocks), so we just preserve whatever is at 0xFFF"*.

### Contact entry (92 bytes / 0x5C)

| Offset | Size | Type | Field | Encoding / rules |
|--------|------|------|-------|------------------|
| +0x00 | 16 | ASCII | Name | Null-terminated, 0xFF-padded — confirmed (`"Contacts 1\0"` + 0xFF in both captures). Encoder writes max **15** chars + 0x00 terminator |
| +0x10 | 3 | uint24 LE | DMR ID | ⚠️ **See the note below** — often documented as a 4-byte uint32, but the one real entry available reads `01 00 00 f0`, i.e. ID 1 with a non-zero byte at +0x13 |
| +0x13 | 1 | ❓ UNKNOWN | flag / type? | **0xF0** in the only hardware entry. Purpose unestablished. It is *not* plausibly the top byte of the DMR ID: DMR IDs are 24-bit, and `0xF0000001` is out of range |
| +0x14 | 8 | ASCII | Callsign | Null-terminated, max **7** chars + terminator; leading 0xFF bytes skipped on parse. All 0xFF (empty) in the hardware entry |
| +0x1C | 16 | ASCII | City | Null-terminated, max 15 chars + 0x00, 0xFF fill. All 0xFF (empty) in the hardware entry |
| +0x2C | 16 | ASCII | Province / State | as above |
| +0x3C | 16 | ASCII | Country | as above |
| +0x4C | 16 | ASCII | Remark | as above |
| +0x5C | — | — | *(end of entry)* | no bytes past 0x5B; all 92 bytes are accounted for |

> ⚠️ **The 4-byte-uint32-ID reading is contradicted by hardware.** The single contact in both captures
> is `"Contacts 1"` with `01 00 00 f0` at +0x10. Read as a uint32 LE that is 0xF0000001 —
> 4,026,531,841 — far outside the 24-bit DMR ID space. Read as uint24 LE + one flag byte it is
> ID **1** plus an unexplained `0xF0`, which is consistent with every other ID field in this radio
> (talk groups, RX groups and the radio-ID list all store 24-bit little-endian IDs).
>
> Implementations should treat +0x10-+0x12 as the ID and preserve
> +0x13 untouched until it is identified.

**Fill byte**: the encoder initialises the whole 92-byte entry to **0xFF**, then writes 0x00
terminators. An empty string field is written by the reference implementation as a single `0x00`
followed by 0xFF padding; the OEM CPS leaves empty fields as **all 0xFF** with no leading NUL.

**Empty-entry detection (parse)**: first name byte 0xFF or 0x00 ⇒ empty; also empty if the decoded
name is blank.

**Contact numbering**: the parser returns a synthetic 1-based position as the contact's `id`; it is
**not** stored in the entry. Do not confuse it with the DMR ID at +0x10.

### ⚠️ DERIVED alternative reading: "Normal" vs "Big" contacts

The original `ContactReadWrite.md` note describes the same 92 bytes as a *variable-field* layout
rather than fixed slots:

> Name (variable, null-terminated) → padding → ID (4 bytes LE) → padding (typically 8 bytes) →
> City → padding → Province → padding → Country → padding (typically 12 bytes) → Remark → 0xFF fill.
> Fields that are empty are **omitted entirely** for "Normal" contacts; "Big" contacts carry the
> city/province/country/remark fields.

Both descriptions agree on the entry size (92 bytes), the name at +0x00 and a little-endian ID field
at +0x10; they differ on whether the later fields sit at **fixed offsets** or are **packed**. The
fixed-offset layout in the table above is what the working implementation reads *and* writes, and it
round-trips against real hardware, so it is the one to implement.

The single hardware entry available does not discriminate: `"Contacts 1"` has an ID at +0x10 and
**every optional field empty**, which both readings predict. So the packed "Normal vs Big"
description stays ⚠️ DERIVED; if you encounter contacts whose city/province/country do not decode at
the fixed offsets, this is the likely explanation. Capturing an OEM CPS session with a "Big" contact
would settle it.

(*+0x14-0x1B was previously described as "Padding", with contact 0 carrying the count inside its
own entry — superseded: +0x14 is the callsign, and the count lives in the separate 16-byte region
header.*)

### Constraints

- **Call type ⚠️ DERIVED**: `ContactReadWrite.md` states *"Only 'Private Call' is supported. All
  contacts are stored as Private Call type regardless of input."* There is **no call-type field in
  the 92-byte layout above**, so nothing in the record contradicts or corroborates this; it is an
  unverified statement about how the radio interprets the database. (Talk groups, which *do* carry a
  call type, live in block 0x44 — a different structure entirely.)
- **Not stored here ⚠️ DERIVED**: Repeater field, Alert Call field — same source, same status.
- Contacts are stored sequentially; every entry is exactly 92 bytes.

### Read / write framing

- **Read**: `52 <addr:3 LE> 00 10` — a 4 KB read. Response echoes the 6-byte header
  (`57 <addr:3 LE> 00 10`) and then 4096 bytes of data.
- **Write**: `57 <addr:3 LE> 00 10 <4096 bytes>` — **4102 bytes total** (6 header + 4096 payload).
  The radio replies `0x06` (ACK).

> (*A write frame with a **trailing metadata byte** — 4103 bytes — was previously documented. That
> is wrong: all 76 write frames in the OEM CPS write capture are exactly **4102 bytes**, and the
> metadata / logical-block-ID byte is the last byte of the 4096-byte payload. For contact blocks
> that byte is simply preserved from the block that was read back, since the contact region is not
> metadata-tagged.*)

- All contact operations are 4 KB-aligned; a 4 KB block holds up to 44 contacts.

---

## Data Encoding Reference

### String Encodings

- **ASCII**: Null-terminated, padded with 0xFF or 0x00 depending on the writer — used by every
  record type except one
- **UTF-16LE**: used **only** by the entry names in block 0x03 (32-byte fields, NUL-terminated) —
  see the *0x03* section

### Number Encodings

- **BCD (Binary-Coded Decimal)**: Each decimal digit represented by its own binary sequence (used for frequencies). See [06-ENCODING.md](06-ENCODING.md) for details.
- **Little-Endian**: Multi-byte values stored with least significant byte first
- **DWORD**: 32-bit value (4 bytes, little-endian)
- **WORD**: 16-bit value (2 bytes, little-endian)

### Special Values

- **0xFF**: Often used as padding or "empty" marker
- **0x00**: Often used as null terminator or empty value
- **Stored Value Offsets**: Some values are stored as `actual_value + offset` (e.g., `value + 1`, `value + 5`). Always subtract the offset when reading.

---

## Notes

1. All blocks are 4 KB (4096 bytes) when read from the radio
2. Entry offsets are relative to the buffer start unless otherwise specified
3. ~~Some entries have negative offsets (e.g., `entry_base - 0x5D`) which means they're stored before the entry base~~ — **retracted**. No structure in this document uses negative entry offsets. The claim came from misreading the RX-group *block header* (0x000-0x010, read once per block) as a per-entry header; see the *RX Group List Structure* section
4. Multi-byte values are little-endian unless otherwise specified
5. String fields are typically null-terminated with 0xFF padding
6. Maximum entry counts are calculated based on buffer size and entry size, but actual usage may be less
7. Block addresses must be discovered via metadata discovery - never hardcode addresses
