# DM-32UV Data Structures

This document provides **code-ready specifications** for all DM-32UV data structures.

## Document Organization

This document is organized as follows:

1. **Primary Structures** - Detailed specifications for the most commonly used structures:
   - Channel Structure (48 bytes)
   - Zone Structure (145 bytes)
   - Scan List Structure (57 bytes)
   - RX Group List Structure (metadata 0x0F, 109 bytes per entry)

2. **Additional Metadata Blocks** - Byte-level parsing for other metadata block types (0x02, 0x03, 0x04, 0x06, 0x07, 0x0A, 0x0F, 0x10, 0x65, 0x66, 0x67)

3. **Data Encoding Reference** - Common encoding schemes used across structures

**Note**: All metadata block addresses are dynamically allocated and must be discovered via metadata discovery. See [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md) for details on metadata discovery.

---

## Channel Structure (48 bytes)

### Memory Layout

```
Total Size: 48 bytes (0x30)
Encoding: Little-endian for multi-byte values
Padding: 0xFF for unused/empty fields
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
│        │ Mode, forbid TX, power level, bandwidth, scan   │
├────────┼─────────────────────────────────────────────────┤
│ 0x20   │ Color Code (1 byte)                             │
├────────┼─────────────────────────────────────────────────┤
│ 0x21   │ RX CTCSS/DCS (2 bytes)                          │
│        │ 73 12 = 127.3 Hz CTCSS                          │
├────────┼─────────────────────────────────────────────────┤
│ 0x23   │ TX CTCSS/DCS (2 bytes)                          │
├────────┼─────────────────────────────────────────────────┤
│ 0x25   │ Additional Flags (7 bytes)                      │
├────────┼─────────────────────────────────────────────────┤
│ 0x2C   │ Reserved (4 bytes)                              │
└────────┴─────────────────────────────────────────────────┘
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
│ 0x1A    │   1  │ talkaround_aprs    │ Talkaround forbid, APRS, reverse     │
│ 0x1B    │   1  │ emergency_settings │ Emergency indicator, ack, system ID  │
│ 0x1C    │   1  │ squelch_aprs       │ Squelch level (bits 7-4), APRS mode  │
│ 0x1D    │   1  │ analog_features    │ VOX, scramble, compander, talkback   │
│ 0x1E    │   1  │ reserved_1e        │ Unknown/Reserved (not used by CPS)   │
│ 0x1F    │   1  │ ptt_id_settings    │ PTT ID display, PTT ID value         │
│ 0x20    │   1  │ color_code         │ DMR color code (0-15)                │
│ 0x21-22 │   2  │ rx_ctcss_dcs       │ RX CTCSS tone or DCS code            │
│ 0x23-24 │   2  │ tx_ctcss_dcs       │ TX CTCSS tone or DCS code            │
│ 0x25    │   1  │ additional_flags   │ VOX-related, compander duplicate     │
│ 0x26    │   1  │ rx_squelch_ptt     │ RX squelch mode, PTT ID display      │
│ 0x27    │   1  │ signaling_settings │ Step frequency, signaling type       │
│ 0x28    │   1  │ reserved_1         │ Not accessed by CPS                  │
│ 0x29    │   1  │ ptt_id_type        │ PTT ID type (OFF/BOT/EOT/BOTH)       │
│ 0x2A    │   1  │ unknown_setting    │ Purpose unknown                      │
│ 0x2B    │   1  │ dmr_radio_id_index │ DMR Radio ID index (0xFF=None)       │
│ 0x2C-2F │   4  │ reserved_2         │ Not accessed by CPS, padding         │
└─────────┴──────┴────────────────────┴──────────────────────────────────────┘
```

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
    // Bits 1-0: Reserved
    
    // 0x1A: Talkaround and APRS
    uint8_t talkaround_aprs;
    // Bit 7: Forbid talkaround (0=Allow, 1=Forbid)
    // Bits 6-4: Unknown setting
    // Bit 3: Unknown
    // Bit 2: APRS receive (0=Off, 1=On)
    // Bits 1-0: Reverse frequency (0-2)
    
    // 0x1B: Emergency settings
    uint8_t emergency_settings;
    // Bit 7: Emergency indicator (0=Off, 1=On)
    // Bit 6: Emergency ack (0=Off, 1=On)
    // Bits 4-0: Emergency system ID (0-31)
    
    // 0x1C: Squelch and APRS
    uint8_t squelch_aprs;
    // Bits 7-4: Squelch level (0-15)
    // Bits 3-2: APRS report mode (0=Off, 1=Digital, 2=Analog)
    // Bits 1-0: Unknown/Reserved
    
    // 0x1D: Analog features
    uint8_t analog_features;
    // Bit 7: VOX function (0=Off, 1=On)
    // Bit 6: Scramble (0=Off, 1=On)
    // Bit 5: Compander (0=Off, 1=On)
    // Bit 4: Talkback (0=Off, 1=On)
    // Bits 3-0: Unknown setting
    
    // 0x1E: Unknown/Reserved
    // NOTE: Squelch level is at byte 0x1C bits 7-4, NOT here.
    // This byte's purpose is unconfirmed; it is not used by CPS software.
    uint8_t reserved_1e;
    
    // 0x1F: PTT ID settings
    uint8_t ptt_id_settings;
    // Bit 7: Reserved
    // Bit 6: PTT ID display (0=Off, 1=On)
    // Bits 5-0: PTT ID (0-63)
    
    // 0x20: Color code
    uint8_t color_code;  // 0-15 for DMR
    
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
    
    // 0x28: Reserved
    uint8_t reserved_1;
    
    // 0x29: PTT ID type
    uint8_t ptt_id_type;
    // Bits 7-4: PTT ID type (0=OFF, 1=BOT, 2=EOT, 3=BOTH)
    // Bits 3-0: Unknown
    
    // 0x2A: Unknown setting
    uint8_t unknown_setting;
    
    // 0x2B: DMR Radio ID index
    // 0x00-0xF9: Index into DMR Radio ID list (0=first entry)
    // 0xFF: None (no DMR Radio ID assigned)
    // NOTE: This is NOT a contact list reference; it selects which of the
    // radio's own DMR IDs to use for this channel's TX. TX contact assignment
    // is stored separately in metadata blocks 0x42 (ch 1-2048) and 0x43 (ch 2049+/VFOs).
    uint8_t dmr_radio_id_index;
    
    // 0x2C-0x2F: Reserved
    uint8_t reserved_2[4];
    
} dm32_channel_t;
#pragma pack(pop)

static_assert(sizeof(dm32_channel_t) == 48, "Channel structure must be 48 bytes");

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
| 6    | 0x40 | Scan Add | 0=Off, 1=On |
| 5-2  | 0x3C | Scan List ID | 0-15 |
| 1-0  | 0x03 | Reserved | - |

### Byte 0x1A (26): Talkaround and APRS

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | Forbid Talkaround | 0=Allow, 1=Forbid |
| 6-4  | 0x70 | Unknown Setting | 0-3 (values ≥4 reset to 0) |
| 3    | 0x08 | Unknown | - |
| 2    | 0x04 | APRS Receive | 0=Off, 1=On |
| 1-0  | 0x03 | Reverse Freq (VFO) | 0-2 |

### Byte 0x1B (27): Emergency Settings

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | Emergency Indicator | 0=Off, 1=On |
| 6    | 0x40 | Emergency Ack | 0=Off, 1=On |
| 4-0  | 0x1F | Emergency System ID | 0-31 |

### Byte 0x1C (28): Squelch and APRS

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | Squelch Level | 0-15 |
| 3-2  | 0x0C | APRS Report Mode | 0=Off, 1=Digital, 2=Analog |
| 1-0  | 0x03 | Unknown/Reserved | - |

### Byte 0x1D (29): Analog Features

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | VOX Function | 0=Off, 1=On |
| 6    | 0x40 | Scramble | 0=Off, 1=On |
| 5    | 0x20 | Compander | 0=Off, 1=On |
| 4    | 0x10 | Talkback | 0=Off, 1=On |
| 3-0  | 0x0F | Unknown Setting | 0-15 |

### Byte 0x1E (30): Unknown/Reserved

**NOTE**: Squelch level is stored at **byte 0x1C bits 7-4**, not here. This byte is not accessed by CPS software; its purpose is unconfirmed.

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | Unknown/Reserved | — |

### Byte 0x1F (31): PTT ID Settings

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | Reserved | - |
| 6    | 0x40 | PTT ID Display | 0=Off, 1=On |
| 5-0  | 0x3F | PTT ID | 0-63 |

### Byte 0x20 (32): Color Code

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | Color Code | 0-15 (DMR) |

### Byte 0x25 (37): Additional Flags

**CPS Functions**: `sub_479C70`, `sub_479CA0`, `sub_479E00`, `sub_479F60`

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-6  | 0xC0 | Unknown | - |
| 5    | 0x20 | Compander (duplicate) | 0=Off, 1=On |
| 4    | 0x10 | VOX-Related Flag | 0=Off, 1=On |
| 3-0  | 0x0F | Unknown Setting | 0-15 (possibly VOX or analog related) |

### Byte 0x26 (38): RX Squelch and PTT ID

**CPS Functions**: `sub_47A0C0`, `sub_47A220`, `sub_47A360`

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | PTT ID Display | 0=Off, 1=On |
| 6-4  | 0x70 | RX Squelch Mode | 0=Carrier/CTC, 1=Optional, 2=CTC&Opt, 3=CTC\|Opt |
| 3-1  | 0x0E | Unknown | 0-7 |
| 0    | 0x01 | Unknown | - |

### Byte 0x27 (39): Signaling Settings

**CPS Functions**: `sub_47A500` (step freq), `sub_47A680` (signaling)

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | Step Frequency | 0=2.5K, 1=5K, 2=6.25K, 3=10K, 4=12.5K, 5=25K, 6=50K, 7=100K |
| 3-0  | 0x0F | Signaling Type | 0=None, 1=DTMF, 2=Two Tone, 3=Five Tone, 4=MDC1200 |

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

### Byte 0x28 (40): Reserved

**Status**: Not accessed by any CPS functions  
**Purpose**: Reserved/unused

### Byte 0x29 (41): PTT ID Type

**CPS Functions**: `sub_47A7E0`, `sub_47A960`

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | PTT ID Type | 0=OFF, 1=BOT, 2=EOT, 3=BOTH |
| 3-2  | 0x0C | Unknown Setting | 0-3 |
| 1-0  | 0x03 | Unknown | - |

**PTT ID Type Values**:
- 0 = OFF
- 1 = BOT (Beginning of Transmission)
- 2 = EOT (End of Transmission)
- 3 = BOTH

### Byte 0x2A (42): Unknown Setting

**CPS Functions**: `sub_47AAE0`, `sub_47AB80`  
**Type**: 8-bit value (0-255)  
**Purpose**: Unknown - possibly DMR or signaling related

### Byte 0x2B (43): DMR Radio ID Index

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | DMR Radio ID Index | 0-249 = index into Radio ID list; 0xFF = None |

**Purpose**: Selects which of the radio's own DMR Radio IDs (metadata block 0x67) to use when transmitting on this channel. This is **not** a reference to the contact/talkgroup list.

**TX Contact Assignment**: The DMR transmit contact for each channel is stored separately — see metadata blocks **0x42** (channels 1–2048) and **0x43** (channels 2049+, VFOs), 2 bytes per channel, little-endian.

### Bytes 0x2C-0x2F (44-47): Reserved

**Status**: Not accessed by any CPS functions  
**Purpose**: Reserved, unused, or padding to 48-byte boundary

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
        'reverse_freq': data[0x1A] & 0x03,
        
        # Byte 0x1B
        'emergency_indicator': bool(data[0x1B] & 0x80),
        'emergency_ack': bool(data[0x1B] & 0x40),
        'emergency_system_id': data[0x1B] & 0x1F,
        
        # Byte 0x1C
        'squelch_level': (data[0x1C] >> 4) & 0x0F,  # 0-15 (NOT power level)
        'aprs_report_mode': (data[0x1C] >> 2) & 0x03,
        
        # Byte 0x1D
        'vox_function': bool(data[0x1D] & 0x80),
        'scramble': bool(data[0x1D] & 0x40),
        'compander': bool(data[0x1D] & 0x20),
        'talkback': bool(data[0x1D] & 0x10),
        
        # Byte 0x1E — Unknown/Reserved (squelch is at 0x1C bits 7-4, NOT here)
        
        # Byte 0x1F
        'ptt_id_display': bool(data[0x1F] & 0x40),
        'ptt_id': data[0x1F] & 0x3F,
        
        # Byte 0x20
        'color_code': data[0x20],
        
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
    reverse_freq: int  # 0-2
    
    # Byte 0x1B: Emergency
    emergency_indicator: bool
    emergency_ack: bool
    emergency_system_id: int  # 0-31
    
    # Byte 0x1C: Squelch and APRS
    squelch_level: int  # 0-15 (bits 7-4)
    aprs_report_mode: int  # 0=Off, 1=Digital, 2=Analog
    
    # Byte 0x1D: Analog features
    vox_function: bool
    scramble: bool
    compander: bool
    talkback: bool
    
    # Byte 0x1E: Unknown/Reserved (NOT squelch)
    
    # Byte 0x1F: PTT ID
    ptt_id_display: bool
    ptt_id: int  # 0-63
    
    # Byte 0x20: Color code
    color_code: int  # 0-15
    
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
    
    # Byte 0x2B: Contact ID
    contact_id: int  # 0-249
    
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
    
    // 0x20: DMR
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
    
    // Squelch (bits 7-4 of byte 0x1C) and color code
    c.SquelchLevel = (data[0x1C] >> 4) & 0x0F  // NOT byte 0x1E
    c.ColorCode = data[0x20]
    
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
    
    // Squelch (bits 7-4 of 0x1C) and color code
    data[0x1C] = c.SquelchLevel << 4  // NOT 0x1E
    data[0x20] = c.ColorCode
    
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

**Metadata**: 0x5c (92)  
**Block Size**: 4 KB (0x1000 bytes)  
**Capacity**: ~28 zones per 4KB block (4080 bytes usable / 145 bytes per zone)  
**Max Channels per Zone**: 64

**Note**: Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

### Offset Calculation

**Block header**: 16 bytes at the start of the 4KB block (purpose unknown/reserved).

**Formula**: `16 + (zone_number - 1) * 145`

Zones are 1-indexed (zone 1, zone 2, etc.)

**Examples**:
- Zone 1: `16 + (1 - 1) * 145 = 16` (offset 0x010)
- Zone 2: `16 + (2 - 1) * 145 = 161` (offset 0x0A1)
- Zone 28: `16 + (28 - 1) * 145 = 3931` (offset 0xF5B)

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

### Zone Name Encoding

- **Length**: 11 bytes maximum (10 chars + null terminator)
- **Format**: ASCII string
- **Termination**: Null-terminated (0x00)
- **Padding**: 0xFF for unused bytes after null
- **Empty zone**: all 0xFF or all 0x00

**Example**:
```
"Zone 1" → 5A 6F 6E 65 20 31 00 FF FF FF FF
           Z  o  n  e     1  \0 (padding)
```

### Channel List Format

- **Count byte** (offset 0x10 within zone): number of channels stored
- **Channel entries** start at offset 0x11 within zone
- **Entry format**: 16-bit little-endian channel number
- **Max slots**: 64 (128 bytes / 2 bytes per channel)
- **Empty slot**: 0x0000 (also terminates parsing if count is unreliable)
- **Valid range**: 0x0001–0x0FA0 (channels 1–4000)

**Example** (3 channels: 1, 15, 100):
```
Offset 0x10: 03           ← channel count = 3
Offset 0x11: 01 00        ← Channel 1
Offset 0x13: 0F 00        ← Channel 15
Offset 0x15: 64 00        ← Channel 100
Offset 0x17: 00 00 ...    ← empty / padding
```

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

```python
def read_all_zones(radio_session):
    """Read all zones from radio"""
    zone_block_addr = find_block_by_metadata(radio_session, 0x5c)
    zone_data = radio_session.commands.read_memory(zone_block_addr, 4096)
    
    zones = []
    for zone_num in range(1, 29):  # Up to ~28 zones per 4KB block
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

---

## Scan List Structure (57 bytes)

**Metadata**: 0x11 (17)  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Scan list definitions for channel scanning

### Memory Organization

**Count Byte**: Offset 0x00 (1 byte) — number of scan lists stored in this block  
**Entry Size**: 0x39 (57) bytes per scan list  
**Max Entries**: 32 scan lists (radio UI limit; formula supports up to ~71 per block)

**Offset Calculation**: Entry N (1-indexed): `(57 * N) - 56`

**Examples**:
- List 1: `(57 * 1) - 56 = 1` (offset 0x001)
- List 2: `(57 * 2) - 56 = 58` (offset 0x03A)
- List 44: `(57 * 44) - 56 = 2452` (offset 0x994)

**Note**: Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

### Field Layout (57 bytes per entry)

```
┌─────────┬──────┬───────────────────────┬──────────────────────────────────┐
│ Offset  │ Size │ Field Name            │ Description                      │
├─────────┼──────┼───────────────────────┼──────────────────────────────────┤
│ 0x00-0A │  11  │ scan_list_name        │ ASCII, null-term, 0xFF padding   │
│ 0x0B    │   1  │ channel_count         │ Number of channels (0-15)        │
│ 0x0C    │   1  │ ctc_tx_mode           │ CTC Scan Mode (bits 0-1) +       │
│         │      │                       │ Scan TX Mode (bits 2-3)          │
│ 0x0D    │   1  │ hang_time             │ Hang time (tenths of seconds)    │
│ 0x0E    │   1  │ priority_types        │ Priority 1 type (bits 0-3) +     │
│         │      │                       │ Priority 2 type (bits 4-7)       │
│ 0x0F-10 │   2  │ priority_channel_1    │ Channel number, 16-bit LE        │
│ 0x11-12 │   2  │ designated_tx_channel │ Channel number, 16-bit LE (+2    │
│         │      │                       │ encoded: stored = actual - 2)    │
│ 0x13-14 │   2  │ priority_channel_2    │ Channel number, 16-bit LE (+2    │
│         │      │                       │ encoded: stored = actual - 2)    │
│ 0x15-19 │   5  │ unknown               │ Unknown/reserved                 │
│ 0x1A-37 │  30  │ channel_list          │ Up to 15 channel numbers, 2 bytes│
│         │      │                       │ LE each; 0x0000 = end of list    │
└─────────┴──────┴───────────────────────┴──────────────────────────────────┘
Total: 57 bytes (0x39)
```

### Field Values

**CTC Scan Mode** (bits 0-1 of byte 0x0C):
- 0 = Not Detection
- 1 = Non Priority
- 2 = Priority
- 3 = Detection

**Scan TX Mode** (bits 2-3 of byte 0x0C):
- 0 = Current Channel
- 1 = Last Active
- 2 = Designated Channel

**Priority Type** (per-priority nibble in byte 0x0E):
- 0 = None
- 1 = Current Channel
- 2 = Designated Channel (value stored in priority_channel field)

**Maximum Channels per Scan List**: 15 channels (30 bytes / 2 bytes per channel)

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x00: Scan list name (11 bytes, null-terminated)
    char scan_list_name[11];
    
    // 0x0B: Channel count (0-15)
    uint8_t channel_count;
    
    // 0x0C: CTC/TX mode byte
    // Bits 0-1: CTC Scan Mode (0=Not Detection, 1=Non Priority, 2=Priority, 3=Detection)
    // Bits 2-3: Scan TX Mode (0=Current, 1=Last Active, 2=Designated)
    uint8_t ctc_tx_mode;
    
    // 0x0D: Hang time (tenths of seconds)
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
    
    // 0x15-0x19: Unknown (5 bytes)
    uint8_t unknown[5];
    
    // 0x1A-0x37: Channel list (30 bytes = 15 × 2-byte channel numbers, LE)
    uint16_t channel_numbers[15];  // 0x0000 = end of list
    
} dm32_scan_list_t;
#pragma pack(pop)

static_assert(sizeof(dm32_scan_list_t) == 57, "Scan list structure must be 57 bytes");
```

---

## RX Group List Structure

**Metadata**: 0x0F (15)  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR receive group lists

**⚠️ CORRECTION NOTE**: Earlier versions of this document incorrectly assigned this structure to metadata 0x0B with 24-byte entries. The correct metadata value is 0x0F, and the actual entry size is 109 bytes (0x6D), matching the memory layout described for V-frame 0x0F (Contacts/Talkgroups).

### Memory Organization

**Entry Size**: 0x6D (109) bytes per RX group  
**Max Entries per Block**: ~37 entries (⌊4096 / 109⌋ = 37)

**Entry Calculation**: `buffer + entry_num * 0x6D` (0-indexed)

**Note**: Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

### Entry Structure (109 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 4 | Bitmask | 32-bit LE bitmask — contact selection/membership flags |
| +0x04 | 1 | Status flag | Entry status |
| +0x05 | 10 | Reserved | Unknown/reserved |
| +0x0F | 1 | Entry flag | Entry validation flag |
| +0x10 | 11 | Name | Null-terminated ASCII, 0xFF padding |
| +0x1B | variable | Contact ID slots | 3 bytes per slot; multiple slots per entry |

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    // +0x00: Bitmask (4 bytes, little-endian)
    uint32_t bitmask;
    
    // +0x04: Status flag
    uint8_t status_flag;
    
    // +0x05: Reserved (10 bytes)
    uint8_t reserved[10];
    
    // +0x0F: Entry validation flag
    uint8_t entry_flag;
    
    // +0x10: Name (11 bytes, null-terminated, 0xFF padding)
    char name[11];
    
    // +0x1B: Contact ID slots (variable, 3 bytes per slot)
    // Remaining bytes to total 109
    uint8_t contact_slots[73];
    
} dm32_rx_group_entry_t;
#pragma pack(pop)

static_assert(sizeof(dm32_rx_group_entry_t) == 109, "RX group entry must be 109 bytes");
```

**⚠️ NOTE on metadata 0x0B**: The purpose of block type 0x0B is currently unconfirmed. It was previously (incorrectly) documented as the RX Group List. Do not rely on 0x0B for RX group data.

---

## Additional Metadata Block Structures

The following sections provide detailed byte-level parsing specifications for other metadata blocks. All metadata blocks are **4096 bytes (0x1000)** when read from the radio (except where noted). Block addresses are dynamically allocated and must be discovered via metadata discovery (see [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)).

---

## 0x02 - Frequency Adjustment/Calibration Data

**Metadata**: 0x02  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Radio frequency calibration and adjustment data

### Structure Overview

The block contains arrays of frequency adjustment values used for radio calibration. Values are indexed by parameter number.

### Field Layout

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| -4 (relative) | 4 | Frequency array 1 | Array of 4-byte frequency values (indexed by `param * 4`) |
| 0x3C (60) | 4 | Frequency array 2 | Array of 4-byte frequency values (indexed by `param * 4`) |
| 0x7E (126) | 2 | Value array 1 | Array of 2-byte values (indexed by `param * 2`) |
| 0x9E (158) | 2 | Value array 2 | Array of 2-byte values (indexed by `param * 2`) |
| 0xB0 (176) | 2 | Value array 3 | Array of 2-byte values (indexed by `param * 2`) |

### Frequency Encoding

Frequencies are stored as 4-byte BCD values, formatted as "XXX.XXXXXX" MHz strings.

### Calibration Parameters

- RX Frequency Adjust
- TX Frequency Adjust
- U/V 4FSK 1-5
- U Low/Mid/High Power 1-5
- V Low/Mid/High Power 1-5

---

## 0x03 - Digital Emergency Systems

**Metadata**: 0x03  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR digital emergency system configurations

### Structure Overview

**Entry Size**: 0x28 (40) bytes per emergency system  
**Entry Base Offset**: 0x218 (536 bytes from buffer start)  
**Max Entries**: ~37 entries ((4096 - 0x218) / 40 ≈ 37)

**Entry Calculation**: `buffer + 0x218 + entry_num * 0x28`

### Entry Structure (40 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 1 | Flag | Bit 0: enabled/disabled |
| +0x01 | 2 | Unknown | Reserved |
| +0x03 | 2 | Value 1 | 16-bit value (little-endian) |
| +0x05 | 2 | Value 2 | 16-bit value (little-endian) |
| +0x1F8 | 16 | Name | Unicode WCHAR string (8 DWORDs) |

### Global Configuration Fields

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x01 | 1 | Count/Index | Validates 1-32 |
| 0x30 | 1 | Unknown | Validates 0-50 |
| 0x31-0x33 | 3 | Numeric fields | Stored as `actual_value + 5` |
| 0x34, 0x36 | 2 | Byte fields | Max 0x14 and 0x32 |
| 0x37-0x3E | 8 | 16-bit values | Four 16-bit values (little-endian) |
| 0x3F | 1 | Bit flags | Bit 0 and bit 1 |
| 0x40 | 1 | Index/count | Stored as `actual_value + 1` |
| 0x41-0x7F | 64 | Entry array | 16 entries × 4 bytes each |
| 0x730-0x7F0 | 192 | Additional config | Extended configuration fields |

### Name Encoding

Unicode (WCHAR), 16 bytes (8 DWORDs) per name. Convert using `WideCharToMultiByte` / `MultiByteToWideChar`.

### Field Values

**Alarm Type**:
- 0 = None
- 1 = Only Whistle
- 2 = Normal
- 3 = Secret
- 4 = Secret With Voice
- 5 = Alarm Whistle

**Alarm Mode**:
- 0 = Emergency Alarm
- 1 = Alarm Call
- 2 = Emergency Call

---

## 0x04 - Embedded Information / Radio Names

**Metadata**: 0x04  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Radio identification and embedded information

### Structure Overview

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x00 | 1 | Byte value | Configuration byte |
| 0x01 | 14 | String field 1 | Stored as 4+4+4+2 byte chunks at offsets +1, +5, +9, +0xD |
| 0x0F (15) | 14 | String field 2 | Stored as 4+4+4+2 byte chunks at offsets +0xF, +0x13, +0x17, +0x1B |
| 0x1D (29) | 1 | Bit flags | Bit 0 flag |
| 0x1E (30) | 1 | Byte value | Range 0-5, max 5 |
| 0x20 (32) | 1 | Bit flags | Multiple flags: bits 0x80, 0x40, 0x20, 0x10, 0x08, 0x04, 0x02 |

### String Encoding

14-byte fields, null-terminated. Value `0xFFFFFFFF` used as "empty" marker.

---

## 0x06 - DTMF Encode Data

**Metadata**: 0x06  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DTMF encoding configuration and code sequences

### Structure Overview

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
| 0x1FF | 1 | Final byte | Configuration byte |

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

## 0x07 - Configuration Header

**Metadata**: 0x07  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Configuration region header/metadata

### Structure Overview

**Status**: Block is discovered via metadata but not actively read/parsed by the CPS software. Structure is unknown.

**Possible Contents**:
- Codeplug version information
- Configuration structure metadata
- Validation/checksum data
- Region size/limits
- Configuration flags

**Note**: This appears to be a header/metadata block for the configuration region itself.

---

## 0x0A - Quick Text Messages

**Metadata**: 0x0A  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Canned/predefined text messages

### Structure Overview

**Count Field**: Offset 0 (1 byte) - number of messages  
**Entry Size**: 0x81 (129) bytes per message  
**Entry Base**: Offset 0x80 (128) for entry 0  
**Max Entries**: ~30 messages (floor((4096 - 128) / 129) = 30)

**Entry Calculation**: `buffer + 0x80 + entry_num * 0x80` = `buffer + 0x80 * (entry_num + 1)`

### Entry Structure (129 bytes)

| Offset within entry | Size | Field | Description |
|---------------------|------|-------|-------------|
| +0x70 (112) | 2 | Check value | Compared against validation constant |
| +0x70+2 (114) | 0x81 | Message text | Null-terminated, 0xFF indicates end |
| +0xF (15) | 1 | Flag/status | Set to 0 when message is set |

### Message Text Encoding

- **If KM mode**: Uses encoding functions (FUN_004825e0, FUN_00482740, FUN_00482690)
- **Otherwise**: Direct string copy (null-terminated, 0xFF indicates end)

---


## 0x0F - TX Contact Assignment

**Metadata**: 0x0F  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR transmit contact assignments per channel

### Structure Overview

**Entry Size**: 0x6D (109) bytes per channel assignment  
**Max Entries**: ~37 entries (floor(4096 / 109) = 37)

**Entry Calculation**: `buffer + entry_num * 0x6D`

### Entry Structure (109 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 4 | Bitmask | 32 bits (little-endian) - contact selection/priority flags |
| +0x04 | 1 | Status flag | Entry status |
| +0x05 | 10 | Reserved | Unknown/reserved |
| +0x0F | 1 | Entry flag | Entry validation flag |

### Additional Entry Fields

These fields are stored **before** the entry base:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| -0x5D (entry_base - 0x5D) | 1 | Validation flag | Entry validation flag |
| -0x5C (entry_base - 0x5C) | 11 | Contact name | Null-terminated, 0xFF indicates end |
| -0x54 (entry_base - 0x54) | Variable | Contact ID slots | 3 bytes per slot, multiple slots per entry |

### Contact ID Slot Format

Each slot is 3 bytes, stored as `[byte2][byte1][byte0]` (little-endian DMR ID).

---

## 0x10 - Analog Emergency Systems

**Metadata**: 0x10  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Analog emergency system configurations

### Structure Overview

**Entry Size**: 0x24 (36) bytes per entry  
**Entry Base Offset**: 0xAC (172 bytes from buffer start)  
**Max Entries**: ~108 entries ((4096 - 172) / 36 ≈ 108)

**Entry Calculation**: `buffer + 0xAC + entry_num * 0x24`

### Entry Structure (36 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 17 | Name | Null-terminated string, max 17 bytes including terminator |
| +0x11 | 1 | Padding | Reserved |
| +0xBD | 1 | Alarm Type | 0-4: None, Only Whistle, Normal, Secret, Secret With Voice |
| +0xBE | 1 | Alarm Mode | 0-1: Emergency Alarm, Alarm Call, Emergency Call |
| +0xBF | 1 | Signalling | 0-3: BDC1200 1-4 |
| +0xC0 | 2 | Revert Channel | Little-endian, 1-16, stored as `value - 1` |
| +0xC2 | 1 | Squelch Mode | 0-1: Carrier, CTC, stored as `value + 1` |
| +0xC3 | 1 | ID Type | 0-1: None, BDC1200, stored as `value + 1` |
| +0xC4 | 1 | Flags | Status byte |
| +0xC5 | 2 | Frequency/ID | 16-bit value (little-endian) |
| +0xC7 | 1 | Flags | Status byte (bit 0: enabled/disabled) |

**Note**: The offsets shown (0xBD, 0xBE, etc.) appear to be relative to a different base. The actual structure uses the 36-byte entry layout starting at `buffer + 0xAC + entry_num * 0x24`.

### Additional Structures

- **Secondary entry structure**: Offset `-0x14 + entry*0x14` (20 bytes, for contact/ID mapping)
- **Tertiary entry structure**: Offset `0x2D5 + entry*0x2C` (44 bytes, for extended data)

### Field Values

**Alarm Type**:
- 0 = None
- 1 = Only Whistle
- 2 = Normal
- 3 = Secret
- 4 = Secret With Voice

**Alarm Mode**:
- 0 = Emergency Alarm
- 1 = Alarm Call
- 2 = Emergency Call

**Signalling**:
- 0-3 = BDC1200 1-4

**Squelch Mode**:
- 0 = Carrier
- 1 = CTC

**ID Type**:
- 0 = None
- 1 = BDC1200

---

## 0x65 - Roaming Zones

**Metadata**: 0x65  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR roaming zone configurations

### Structure Overview

**Entry Size**: 0x21 (33) bytes per entry  
**Entry Base Offset**: 0x00 (entries start at buffer base)  
**Max Entries**: ~124 entries (4096 / 33 ≈ 124)

**Entry Calculation**: `buffer + entry_num * 0x21`

### Entry Structure (33 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 4 | Count/value | DWORD (little-endian) |
| +0x01 | 1 | Bit flag | Bit 0: enabled/disabled |
| +0x02 | 1 | Value | Stored as `value + 1` |
| +0x03 | 1 | Value | Stored as `value + 1`, max 0x40 |
| +0x10 | 16 | Name | Null-terminated string, max 16 bytes including terminator |
| +0x20 | 1 | Channel count | Channel count or index |
| +0x21 | 16 | Channel list | Channel list or additional data |

### Maximum Channels per Zone

16 channels maximum per roaming zone.

---

## 0x66 - Roaming Channels

**Metadata**: 0x66  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: DMR roaming channel configurations

### Structure Overview

**Entry Size**: 0x1A (26) bytes per entry  
**Entry Base Offset**: 0x00 (entries start at buffer base)  
**Max Entries**: ~157 entries (4096 / 26 ≈ 157)

**Count Storage**: Offset 0xFF0 (1 byte, stores count, max 0x96 = 150)

**Entry Calculation**: `buffer + entry_num * 0x1A`

### Entry Structure (26 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 16 | Name | Null-terminated string, max 16 bytes including terminator |
| +0x10 | 4 | RX Frequency | BCD encoded, 4 bytes |
| +0x14 | 4 | TX Frequency | BCD encoded, 4 bytes |
| +0x18 | 1 | Color Code | DMR color code (0-15) |
| +0x19 | 1 | Time Slot | DMR time slot (1 or 2) |

### Frequency Encoding

Frequencies stored in BCD (Binary-Coded Decimal) format, 4 bytes per frequency. The encoding converts frequency strings (e.g., "145.500000") to BCD representation. See [06-ENCODING.md](06-ENCODING.md) for BCD encoding details.

---

## 0x67 - DMR Radio ID List

**Metadata**: 0x67  
**Block Size**: 4 KB (0x1000 bytes)  
**Purpose**: Radio's own DMR Radio IDs

### Structure Overview

**Entry Size**: 0x10 (16) bytes per entry  
**Entry Base Offset**: 0x00 (entries start at buffer base)  
**Max Entries**: 256 entries (4096 / 16 = 256)

**Count Storage**: Offset 0 (4 bytes, DWORD, little-endian, stores count)

**Entry Calculation**: `buffer + entry_num * 0x10`

### Entry Structure (16 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| +0x00 | 3 | DMR Radio ID | 3 bytes, stored as BCD or binary |
| +0x03 | 12 | Name | Null-terminated string, max 12 bytes including terminator |

### ID Encoding

DMR Radio IDs are stored as 3 bytes. The format function uses `"%02X %02X %02X"` to display them as hex (e.g., "01 23 45" for ID 0x012345). The set function converts a 6-digit string to 3 bytes.

**Important Note**: This stores the **radio's own DMR Radio IDs** - the IDs that identify this specific radio when transmitting on digital channels. This is different from the main contact/ID database (V-frame 0x0F) which contains all known contacts/talkgroups.

---

## Data Encoding Reference

### String Encodings

- **ASCII**: Null-terminated, 0xFF padding, empty = 0xFF 0xFF
- **Unicode (WCHAR)**: 16 bytes (8 DWORDs) per name, converted via `WideCharToMultiByte` / `MultiByteToWideChar`

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
3. Some entries have negative offsets (e.g., `entry_base - 0x5D`) which means they're stored before the entry base
4. Multi-byte values are little-endian unless otherwise specified
5. String fields are typically null-terminated with 0xFF padding
6. Maximum entry counts are calculated based on buffer size and entry size, but actual usage may be less
7. Block addresses must be discovered via metadata discovery - never hardcode addresses
