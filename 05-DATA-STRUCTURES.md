# DM-32UV Data Structures

This document provides **code-ready specifications** for all DM-32UV data structures.

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
│        │ Power, bandwidth, scan, etc.                    │
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
│ 0x18    │   1  │ mode_flags         │ Mode, TX forbid, busy lock, lone work│
│ 0x19    │   1  │ scan_bandwidth     │ Bandwidth, scan add, scan list ID    │
│ 0x1A    │   1  │ talkaround_aprs    │ Talkaround forbid, APRS, reverse     │
│ 0x1B    │   1  │ emergency_settings │ Emergency indicator, ack, system ID  │
│ 0x1C    │   1  │ power_aprs         │ Power level, APRS report mode        │
│ 0x1D    │   1  │ analog_features    │ VOX, scramble, compander, talkback   │
│ 0x1E    │   1  │ squelch_level      │ Squelch level (0-255)                │
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
│ 0x2B    │   1  │ contact_id         │ Contact list reference (1-250)       │
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
    // Bits 2-1: Busy lock (0=Off, 1=Carrier, 2=Repeater)
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
    
    // 0x1C: Power and APRS
    uint8_t power_aprs;
    // Bits 7-4: Power level (0=Low, 1=Middle, 2=High)
    // Bits 3-2: APRS report mode (0=Off, 1=Digital, 2=Analog)
    // Bits 1-0: Unknown
    
    // 0x1D: Analog features
    uint8_t analog_features;
    // Bit 7: VOX function (0=Off, 1=On)
    // Bit 6: Scramble (0=Off, 1=On)
    // Bit 5: Compander (0=Off, 1=On)
    // Bit 4: Talkback (0=Off, 1=On)
    // Bits 3-0: Unknown setting
    
    // 0x1E: Squelch level
    uint8_t squelch_level;  // 0-255
    
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
    
    // 0x2B: Contact/name ID
    uint8_t contact_id;  // 0-249 (displayed as 1-250)
    
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
| 2-1  | 0x06 | Busy Lock | 0=Off, 1=Carrier, 2=Repeater |
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

### Byte 0x1C (28): Power and APRS

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-4  | 0xF0 | Power Level | 0=Low, 1=Middle, 2=High |
| 3-2  | 0x0C | APRS Report Mode | 0=Off, 1=Digital, 2=Analog |
| 1-0  | 0x03 | Unknown | - |

### Byte 0x1D (29): Analog Features

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7    | 0x80 | VOX Function | 0=Off, 1=On |
| 6    | 0x40 | Scramble | 0=Off, 1=On |
| 5    | 0x20 | Compander | 0=Off, 1=On |
| 4    | 0x10 | Talkback | 0=Off, 1=On |
| 3-0  | 0x0F | Unknown Setting | 0-15 |

### Byte 0x1E (30): Squelch Level

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | Squelch Level | 0-255 |

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

### Byte 0x2B (43): Contact/Name ID

**CPS Functions**: `sub_47AC20`, `sub_47ACC0`

| Bits | Mask | Field | Values |
|------|------|-------|--------|
| 7-0  | 0xFF | Contact ID | 0-249 (displayed as 1-250 in UI) |

**Purpose**: References a contact from the contact list (250 contacts max)

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
        'busy_lock': (data[0x18] >> 1) & 0x03,
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
        'power_level': (data[0x1C] >> 4) & 0x0F,
        'aprs_report_mode': (data[0x1C] >> 2) & 0x03,
        
        # Byte 0x1D
        'vox_function': bool(data[0x1D] & 0x80),
        'scramble': bool(data[0x1D] & 0x40),
        'compander': bool(data[0x1D] & 0x20),
        'talkback': bool(data[0x1D] & 0x10),
        
        # Byte 0x1E
        'squelch_level': data[0x1E],
        
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
        'contact_id': data[0x2B],
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
    busy_lock: int  # 0=Off, 1=Carrier, 2=Repeater
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
    
    # Byte 0x1C: Power and APRS
    power_level: int  # 0=Low, 1=Medium, 2=High
    aprs_report_mode: int  # 0=Off, 1=Digital, 2=Analog
    
    # Byte 0x1D: Analog features
    vox_function: bool
    scramble: bool
    compander: bool
    talkback: bool
    
    # Byte 0x1E: Squelch
    squelch_level: int  # 0-255
    
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
        busy_lock = (mode_flags >> 1) & 0x03
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
            ((self.busy_lock & 0x03) << 1) |
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
    BusyLock     uint8 // 0=Off, 1=Carrier, 2=Repeater
    LoneWorker   bool
    
    // 0x19: Scan and bandwidth
    Bandwidth    uint8 // 0=12.5KHz narrow, 1=25KHz wide
    ScanAdd      bool
    ScanListID   uint8 // 0-15
    
    // 0x1C: Power
    PowerLevel   uint8 // 0=Low, 1=Medium, 2=High
    
    // 0x1E: Squelch
    SquelchLevel uint8 // 0-255
    
    // 0x20: DMR
    ColorCode    uint8 // 0-15
    
    // 0x21-0x24: CTCSS/DCS
    RXCTCSSHz    float64  // 0 if not used
    RXDCSCode    string   // "" if not used
    TXCTCSSHz    float64
    TXDCSCode    string
    
    // 0x2B: Contact ID
    ContactID    uint8 // 0-249
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
    c.BusyLock = (modeFlags >> 1) & 0x03
    c.LoneWorker = (modeFlags & 0x01) != 0
    
    scanBW := data[0x19]
    c.Bandwidth = (scanBW >> 7) & 0x01
    c.ScanAdd = (scanBW & 0x40) != 0
    c.ScanListID = (scanBW >> 2) & 0x0F
    
    // Power and squelch
    c.PowerLevel = (data[0x1C] >> 4) & 0x0F
    c.SquelchLevel = data[0x1E]
    c.ColorCode = data[0x20]
    
    // CTCSS/DCS
    c.RXCTCSSHz, c.RXDCSCode = decodeCTCSSDCS(data[0x21:0x23])
    c.TXCTCSSHz, c.TXDCSCode = decodeCTCSSDCS(data[0x23:0x25])
    
    // Contact ID
    c.ContactID = data[0x2B]
    
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
                 (c.BusyLock << 1) |
                 boolToByte(c.LoneWorker)
    
    data[0x19] = (c.Bandwidth << 7) |
                 (boolToByte(c.ScanAdd) << 6) |
                 (c.ScanListID << 2)
    
    // Power, squelch, color code
    data[0x1C] = c.PowerLevel << 4
    data[0x1E] = c.SquelchLevel
    data[0x20] = c.ColorCode
    
    // CTCSS/DCS
    copy(data[0x21:0x23], encodeCTCSSDCS(c.RXCTCSSHz, c.RXDCSCode))
    copy(data[0x23:0x25], encodeCTCSSDCS(c.TXCTCSSHz, c.TXDCSCode))
    
    // Contact ID
    data[0x2B] = c.ContactID
    
    return data
}

func boolToByte(b bool) uint8 {
    if b {
        return 1
    }
    return 0
}
```

## Zone Structure (57 bytes)

Zones group channels together for organizational purposes.

### Memory Organization

**Buffer**: `dword_4E80E8`  
**Metadata**: 0x11 (17)  
**Block Size**: 4 KB (0x1000 bytes)  
**Capacity**: ~71 zones maximum (4096 / 57 ≈ 71)  
**Practical Limit**: 64 zones (from CPS UI limit)

### Offset Calculation

**Formula**: `57 * zone_number - 56`

Zones are 1-indexed (zone 1, zone 2, etc.)

**Examples**:
- Zone 1: `57 * 1 - 56 = 1` (offset 0x001)
- Zone 2: `57 * 2 - 56 = 58` (offset 0x03A)
- Zone 64: `57 * 64 - 56 = 3592` (offset 0xE08)

**CPS Code Reference**:
```c
// Get zone name (sub_47B810)  
v2 = (char *)dword_4E80E8 + 38 * a2 + 19 * a2 - 56;  // = 57 * a2 - 56
// Reads first 11 bytes as name
```

### Field Layout

```
┌─────────┬──────┬──────────────┬─────────────────────────────────────┐
│ Offset  │ Size │ Field        │ Description                         │
├─────────┼──────┼──────────────┼─────────────────────────────────────┤
│ 0x00-0A │  11  │ zone_name    │ ASCII, null-term, 0xFF padding      │
│ 0x0B-38 │  46  │ channel_list │ Up to 64 channel numbers (see below)│
└─────────┴──────┴──────────────┴─────────────────────────────────────┘
```

### Zone Name Encoding

- **Length**: 11 bytes maximum
- **Format**: ASCII string
- **Termination**: Null-terminated (0x00)
- **Padding**: 0xFF for unused bytes after null
- **Empty zone**: 0xFF 0xFF... (all 0xFF)

**Example**:
```
"Zone 1" → 5A 6F 6E 65 20 31 00 FF FF FF FF
           Z  o  n  e     1  \0 (padding)
```

### Channel List Format

**Confirmed Format**: 2 bytes per channel × 23 slots = 46 bytes

The 46-byte channel list stores up to **23 channel numbers** (not 64 as the UI might suggest).

**Storage Details:**
- Each channel number: 16-bit little-endian value
- Empty slot: 0x0000
- Valid range: 0x0001-0x0FA0 (channels 1-4000)
- Total slots: 23 (46 bytes ÷ 2 bytes per channel)

**Note on UI Limit:** The CPS software UI may show a limit of 64 channels per zone, but the actual storage format only supports 23 channels. This is a limitation of the zone structure size (57 bytes total, with 11 bytes for name and 46 bytes for channels).

**Channel Number Encoding:**
- 0x0000 = Empty slot
- 0x0001-0x0FA0 = Channel 1-4000
- Little-endian format

**Example** (3 channels: 1, 15, 100):
```
Offset 0x0B: 01 00  15 00  64 00  00 00  00 00 ...
             Ch1    Ch15   Ch100  Empty  Empty
             
Remaining 20 slots: 00 00 00 00 ... (all empty)
```

### Code Structure (C/C++)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x00-0x0A: Zone name (11 bytes)
    char zone_name[11];
    
    // 0x0B-0x38: Channel list (46 bytes)
    // Format: Up to 23 channel numbers (2 bytes each, little-endian)
    uint16_t channel_numbers[23];  // 0 = empty slot
    
} dm32_zone_t;
#pragma pack(pop)

static_assert(sizeof(dm32_zone_t) == 57, "Zone structure must be 57 bytes");
```

### Code Structure (Python)

```python
from dataclasses import dataclass
from typing import List
import struct

@dataclass
class DM32Zone:
    """DM32 Zone Structure (57 bytes)"""
    
    zone_name: str
    channel_numbers: List[int]  # Up to 64 channels (1-4000)
    
    @classmethod
    def from_bytes(cls, data: bytes, zone_number: int) -> 'DM32Zone':
        """Parse zone from 4KB block"""
        if len(data) < 4096:
            raise ValueError("Zone data block must be 4096 bytes")
        
        # Calculate offset for this zone
        offset = 57 * zone_number - 56
        if offset + 57 > 4096:
            raise ValueError(f"Zone {zone_number} offset out of range")
        
        zone_data = data[offset:offset+57]
        
        # Parse name (11 bytes)
        name_bytes = zone_data[0:11]
        name = name_bytes.split(b'\x00')[0].replace(b'\xFF', b'').decode('ascii', errors='ignore')
        
        # Parse channel list (46 bytes = 23 × 2-byte channel numbers)
        channels = []
        for i in range(23):
            ch_offset = 11 + (i * 2)
            ch_num = struct.unpack('<H', zone_data[ch_offset:ch_offset+2])[0]
            if ch_num != 0 and ch_num <= 4000:
                channels.append(ch_num)
        
        return cls(zone_name=name, channel_numbers=channels)
    
    def to_bytes(self) -> bytes:
        """Convert zone to 57-byte record"""
        data = bytearray(57)
        
        # Name (11 bytes)
        name_bytes = self.zone_name.encode('ascii')[:10]  # Max 10 chars + null
        data[0:len(name_bytes)] = name_bytes
        if len(name_bytes) < 11:
            data[len(name_bytes)] = 0x00  # Null terminator
            for i in range(len(name_bytes) + 1, 11):
                data[i] = 0xFF  # Padding
        
        # Channel list (46 bytes = 23 slots)
        for i in range(23):
            ch_offset = 11 + (i * 2)
            if i < len(self.channel_numbers):
                ch_num = self.channel_numbers[i]
                data[ch_offset:ch_offset+2] = struct.pack('<H', ch_num)
            else:
                data[ch_offset:ch_offset+2] = b'\x00\x00'  # Empty slot
        
        return bytes(data)
```

### Code Structure (Go)

```go
package dm32

import (
    "encoding/binary"
    "fmt"
)

// Helper functions
// Note: decodeBCDFrequency, encodeBCDFrequency, decodeCTCSSDCS, encodeCTCSSDCS
// are defined in 06-ENCODING.md

func bytesUntilNull(data []byte) int {
    for i, b := range data {
        if b == 0x00 || b == 0xFF {
            return i
        }
    }
    return len(data)
}

// Zone represents a DM32 zone (57 bytes)
type Zone struct {
    Name            string
    ChannelNumbers  []uint16  // Up to 64 channels
}

// FromBytes parses a zone from the 4KB zone block
func (z *Zone) FromBytes(data []byte, zoneNumber int) error {
    if len(data) < 4096 {
        return fmt.Errorf("zone block must be 4096 bytes")
    }
    
    // Calculate offset: 57 * zoneNumber - 56
    offset := 57*zoneNumber - 56
    if offset+57 > 4096 {
        return fmt.Errorf("zone %d offset out of range", zoneNumber)
    }
    
    zoneData := data[offset : offset+57]
    
    // Parse name (11 bytes)
    nameBytes := zoneData[0:11]
    z.Name = string(nameBytes[:bytesUntilNull(nameBytes)])
    
    // Parse channel list (23 slots × 2 bytes)
    z.ChannelNumbers = make([]uint16, 0, 64)
    for i := 0; i < 23; i++ {
        chOffset := 11 + (i * 2)
        chNum := binary.LittleEndian.Uint16(zoneData[chOffset : chOffset+2])
        if chNum != 0 && chNum <= 4000 {
            z.ChannelNumbers = append(z.ChannelNumbers, chNum)
        }
    }
    
    return nil
}

// ToBytes converts zone to 57-byte record
func (z *Zone) ToBytes() []byte {
    data := make([]byte, 57)
    
    // Name (11 bytes, null-terminated, pad with 0xFF)
    copy(data[0:11], z.Name)
    if len(z.Name) < 11 {
        data[len(z.Name)] = 0x00
        for i := len(z.Name) + 1; i < 11; i++ {
            data[i] = 0xFF
        }
    }
    
    // Channel list (23 slots)
    for i := 0; i < 23 && i < len(z.ChannelNumbers); i++ {
        chOffset := 11 + (i * 2)
        binary.LittleEndian.PutUint16(data[chOffset:chOffset+2], z.ChannelNumbers[i])
    }
    
    return data
}
```

### Reading Zones from Radio

```python
def read_all_zones(radio_session):
    """Read all zones from radio"""
    # Read zone block (metadata 0x11)
    zone_block_addr = find_block_by_metadata(radio_session, 0x11)
    zone_data = radio_session.commands.read_memory(zone_block_addr, 4096)
    
    # Parse zones
    zones = []
    for zone_num in range(1, 65):  # Zones 1-64
        try:
            zone = DM32Zone.from_bytes(zone_data, zone_num)
            if zone.zone_name:  # Not empty
                zones.append(zone)
        except ValueError:
            break  # Beyond valid zones
    
    return zones
```

### Usage Example

```python
# Parse zone 1
zone_block = radio.read_memory(zone_address, 4096)
zone1 = DM32Zone.from_bytes(zone_block, zone_number=1)

print(f"Zone: {zone1.zone_name}")
print(f"Channels: {zone1.channel_numbers}")

# Output:
# Zone: VHF
# Channels: [1, 2, 3, 15, 20, 25]
```

---

## Scan List Structure (92 bytes)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x00/0x10: Scan list name (16 bytes)
    // Note: Offset 16 for lists 1-44, 0 for lists 45+
    char scan_list_name[16];
    
    // Settings
    uint8_t ctc_scan_mode;  // 0-3
    uint8_t settings[8];
    
    // Channel list (up to 16 channels)
    uint8_t channel_list[64];
    
} dm32_scan_list_t;
#pragma pack(pop)
```

## RX Group List Structure (109 bytes)

```c
#pragma pack(push, 1)
typedef struct {
    // 0x00-0x0A: RX group name (11 bytes)
    char rx_group_name[11];
    
    // 0x0B-0x0F: Settings/flags (5 bytes)
    uint8_t settings[5];
    
    // 0x10-0x6C: Contact list (93 bytes = 31 contacts × 3 bytes each)
    uint8_t contact_ids[93];  // BCD encoded, 3 bytes per contact
    
} dm32_rx_group_t;
#pragma pack(pop)
```
