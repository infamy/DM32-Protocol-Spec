# DM-32UV Command Reference

Complete reference for all DM-32UV protocol commands with examples.

## Command Categories

1. **Handshake Commands** (ASCII) - Initial connection
2. **V-Frame Commands** (Binary) - System information queries
3. **Programming Mode Commands** (Binary) - Enter/exit programming mode
4. **Memory Read Commands** (Binary) - Read data from radio
5. **Memory Write Commands** (Binary) - Write data to radio

---

## 1. Handshake Commands

> **💡 Real Examples**: See `serial_capture_example.txt` for actual command/response sequences from the official CPS software.

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
**Timeout**: 500ms  
**Retry**: Up to 3 times

**Serial Capture Reference:** Lines 3-5 in `serial_capture_example.txt`

---

### PASSSTA - Get Status

**Format**: ASCII string  
**Length**: 7 bytes

```
Command:  50 41 53 53 53 54 41
          P  A  S  S  S  T  A

Response: 50 XX XX
          P  (status bytes)
```

**Status Values**:
- `50 00 00` - Normal
- `50 FF FF` - Variant (both valid)

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

### Command Format

```
Byte 0:   0x56 ('V')
Bytes 1-3: 0x00 0x00 0x00
Byte 4:   Frame ID
```

### Response Format

```
Byte 0:   0x56 ('V')
Byte 1:   Frame ID (echoed)
Byte 2:   Data length (N)
Bytes 3+: Data (N bytes)
```

### V-Frame Reference Table

| ID | Length | Purpose | Format | Example Response |
|----|--------|---------|--------|-------------------|
| 0x01 | 14-15 | Firmware version | ASCII string | "DM32.01.01.046" or "DM32.01.L01.048" |
| 0x02 | 12 | Unknown | Binary (format unknown) | `00 00 00 00 00 00 15 a4 00 00 15 a4` |
| 0x03 | 10 | Build date | ASCII string | "2022-06-27" |
| 0x04 | 12 | DSP version | ASCII string | "D1.01.01.004" |
| 0x05 | 12 | Radio version | ASCII string | "R1.00.01.001" |
| 0x06 | 8 | Audio resource index | Memory range (start+end) | `00 10 20 00 ff 4f 26 00` = 0x001020-0x00264FFF |
| 0x07 | 8 | Compact item table | Memory range (start+end) | `00 90 0c 00 ff 9f 14 00` = 0x000C9000-0x00149FFF |
| 0x08 | 8 | Zones | Memory range (start+end) | `00 00 18 00 ff 0f 20 00` = 0x00001800-0x00200FFF |
| 0x09 | 8 | Emergency/recording | Memory range (start+end) | `00 c0 6d 00 ff ff ff 00` = 0x00006DC0-0x00FFFFFF (disabled if 0xFFFFFF) |
| 0x0A | 8 | **Main config block** | Memory range (start+end) | `00 10 00 00 ff 8f 0c 00` = 0x001000-0x0C8FFF |
| 0x0B | 12 | Code plug version | ASCII string | "C1.00.01.001" |
| 0x0D | 0-64 | Capabilities | Binary (special format) | See V-Frame 0x0D below |
| 0x0E | 8 | Memberships/lists | Memory range (start+end) | `00 00 15 00 ff 5f 17 00` = 0x00001500-0x00175FFF |
| 0x0F | 8 | Contacts/Talkgroups | Memory range (start+end) | `00 80 27 00 ff bf 6d 00` = 0x00002780-0x006DBFFF (varies by firmware) |
| 0x10 | 3 | Unknown | Binary (format unknown) | `50 c3 00` |

### V-Frame Examples

#### Get Firmware Version (0x01)

**Command**: `56 00 00 00 01`  
**Response**: `56 01 0E <14 bytes ASCII>`  
**Parse**: Decode ASCII string (e.g., "DM32.01.01.046")

#### Get Main Config Block (0x0A - CRITICAL)

**Command**: `56 00 00 00 0A`  
**Response**: `56 0A 08 <8 bytes>`  
**Parse**: 
- Bytes 0-3: Start address (little-endian uint32)
- Bytes 4-7: End address (little-endian uint32)
- Example: `00 10 00 00 FF 8F 0C 00` = 0x001000 - 0x0C8FFF

#### Get Memory Range V-Frames (0x06, 0x07, 0x08, 0x09, 0x0A, 0x0E, 0x0F)

All memory pointer V-frames return 8 bytes in the same format:
- Bytes 0-3: Start address (little-endian uint32)
- Bytes 4-7: End address (little-endian uint32)

**V-Frame 0x06: Audio Resource Index**
- Command: `56 00 00 00 06`
- Response: `56 06 08 00 10 20 00 ff 4f 26 00`
- Parse: Start = 0x001020, End = 0x00264FFF

**V-Frame 0x08: Zones**
- Command: `56 00 00 00 08`
- Response: `56 08 08 00 00 18 00 ff 0f 20 00`
- Parse: Start = 0x00001800, End = 0x00200FFF

**V-Frame 0x0F: Contacts/Talkgroups**
- Command: `56 00 00 00 0F`
- Response: `56 0F 08 00 80 27 00 ff bf 6d 00`
- Parse: Start = 0x00002780, End = 0x006DBFFF
- Note: End address varies by firmware (may be 0x00FFFFFF for extended range)

#### Get Version Strings (0x01, 0x03, 0x04, 0x05, 0x0B)

All version V-frames return ASCII strings:
- **0x01**: Firmware version (14-15 bytes, e.g., "DM32.01.01.046" or "DM32.01.L01.048")
- **0x03**: Build date (10 bytes, e.g., "2022-06-27")
- **0x04**: DSP version (12 bytes, e.g., "D1.01.01.004")
- **0x05**: Radio version (12 bytes, e.g., "R1.00.01.001")
- **0x0B**: Code plug version (12 bytes, e.g., "C1.00.01.001")

**V-Frame 0x01: Firmware Version**
- Command: `56 00 00 00 01`
- Response: `56 01 0E 44 4D 33 32 2E 30 31 2E 30 31 2E 30 34 36`
- Parse: "DM32.01.01.046" (14 bytes)
- **Serial Capture Reference:** Lines 18-20 in `serial_capture_example.txt`

#### V-Frame 0x0D: Capabilities (Special Format)

**Command Format**: `56 00 00 40 0D` (note: byte 3 is 0x40, not 0x00)

**Response**:
- If supported: `56 0D 40 <64 bytes of capability data>`
- If not supported: `56 0D 00` (0 bytes)

**Purpose**: Radio capabilities and feature flags (64 bytes when supported)

**Known Capability Data (when supported):**
```
Byte 0-2:   03 4E 2D        Unknown identifier
Byte 3-34:  00 00 00...     Reserved/unknown (32 bytes)
Byte 35-38: 00 3F 00 00     Capability flags (exact meaning unknown)
Byte 39-63: 00 00 00...     Reserved/unknown (25 bytes)
```

**Status**: The exact format and meaning of capability flags is **undocumented**. Most firmware versions return 0 bytes (not supported). This V-frame is optional and can be safely skipped.

**Recommendation**: Query this V-frame for completeness, but do not rely on it for critical functionality.

---

## 3. Programming Mode Commands

### Enter Programming Mode Sequence

This is a **3-step sequence** that must complete successfully:

#### Step 1: PROGRAM Command

```
Command:  FF FF FF FF 0C 50 52 4F 47 52 41 4D
          (header)    P  R  O  G  R  A  M

Response: 06 (ACK)
```

#### Step 2: Mode 02

```
Command:  02

Response: FF FF FF FF FF FF FF FF (8 bytes)
```

#### Step 3: Confirm with ACK

```
Command:  06

Response: 06 (ACK)
```

### Exit Programming Mode

```
Command:  FF FF FF FF 0C 45 4E 44 00 00 00 00
          (header)    E  N  D  (padding)

Response: 06 (ACK)
```

---

## 4. Memory Read Commands

### Single Read Command (0x52)

**Format**:
```
Byte 0:   0x52 ('R')
Bytes 1-3: Address (24-bit, little-endian)
Bytes 4-5: Length (16-bit, little-endian)
```

**Response**:
```
Byte 0:   0x57 ('W')
Bytes 1-3: Address (echoed, little-endian)
Bytes 4-5: Length (echoed, little-endian)
Bytes 6+:  Data (length bytes)
```

**Implementation Notes:**
- Address encoding: Little-endian (LSB first)
- Example: Address 0x001000 = `00 10 00` (not `00 00 10`)
- Length encoding: Little-endian
- Example: Length 4096 (0x1000) = `00 10` (not `10 00`)

**Real Examples from Serial Capture:**

Example 1: Read 1 byte from 0x001FFF (metadata probe)
```
Command:  52 FF 1F 00 01 00
          R  [addr=0x001FFF] [len=1]
          
Response: 57 FF 1F 00 01 00 07
          W  [addr=0x001FFF] [len=1] [data=0x07]
          (metadata value = 0x07)
```

Example 2: Read 4KB block from 0x001000
```
Command:  52 00 10 00 00 10
          R  [addr=0x001000] [len=4096]
          
Response: 57 00 10 00 00 10 [4096 bytes of data]
          W  [addr=0x001000] [len=4096] [data...]
```

See lines 84-86 in `serial_capture_example.txt` for metadata discovery sequence

### Read Command Variations

| Length | Hex | Use Case |
|--------|-----|----------|
| 1 | 01 00 | Single byte (metadata probe) |
| 4 | 04 00 | Channel count, small values |
| 16 | 10 00 | Block header |
| 4096 | 00 10 | Full 4KB block (standard) |

---

## 5. Memory Write Commands

### Write Command (0x57)

**Format**:
```
Byte 0:      0x57 ('W')
Bytes 1-3:   Address (24-bit, little-endian)
Byte 4:      0x00 (reserved)
Byte 5:      0x10 (size indicator = 4KB)
Bytes 6-4101: Data (4096 bytes)
Byte 4102:   Metadata byte
```

**Response**:
```
Byte 0: 0x06 (ACK)
```

**Timeout**: 5000ms (5 seconds)

### Example: Write Channel Block

```python
# Write 4KB channel block to address 0x01B000
address = 0x01B000
metadata = 0x12  # Channel block 1

# Prepare command (4102 bytes total)
cmd = bytearray(4102)
cmd[0] = 0x57                                    # Write command
cmd[1:4] = address.to_bytes(3, 'little')         # Address (little-endian)
cmd[4] = 0x00                                    # Reserved
cmd[5] = 0x10                                    # Size (4KB)
cmd[6:4102] = channel_data  # 4096 bytes        # Channel data
cmd[4101] = metadata                             # Metadata

# Send
port.write(cmd)

# Wait for ACK
response = port.read(1)
if response != b'\x06':
    raise IOError("Write not acknowledged")

time.sleep(0.05)  # 50ms delay before next write
```

### Write Timing Requirements

- **Between writes**: 10-50ms delay
- **Write timeout**: 5000ms
- **Verification**: Re-read block after write to verify

---

## Command Timing Reference

| Operation | Delay | Timeout |
|-----------|-------|---------|
| Between handshake commands | 10ms | 500ms |
| Between V-frame queries | 10ms | 500ms |
| Programming mode sequence | 10ms | 500ms |
| Between metadata reads | 5ms | 500ms |
| Between 4KB block reads | 25ms | 500ms |
| Between 4KB block writes | 10-50ms | 5000ms |

## Error Codes

| Error | Code | Description |
|-------|------|-------------|
| No response | Timeout | Radio not connected or powered off |
| Wrong ACK | 0x15 or other | Command rejected |
| NAK | 0x15 | Negative acknowledgment |
| Timeout on write | - | Radio busy or buffer full |
