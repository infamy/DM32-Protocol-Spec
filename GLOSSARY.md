# Glossary

Quick reference for technical terms used in this documentation.

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

### Metadata
A single byte at the end of each 4KB block (offset +0xFFF) that identifies the block type. Used for dynamic memory discovery.

**Values**: 0x12-0x41 (channels), 0x11 (zones), 0x0F (RX groups), etc.

### NAK (Negative Acknowledgment)
A response (0x15) indicating command failure or rejection.

### Programming Mode
Special mode the radio enters after the PROGRAM command sequence. Required before memory read/write operations.

### V-Frame
Binary command format starting with 0x56 ('V') used to query radio information (firmware version, memory layout, etc.).

**Format**: `56 00 00 00 <ID>`

## Memory Terms

### 4KB Block
The basic unit of memory organization in the DM-32UV. All addresses must be 4KB-aligned (0x001000, 0x002000, etc.).

### Channel Bank
Collection of 48 consecutive 4KB blocks (metadata 0x12-0x41) containing up to 4,000 channel definitions.

### Main Config Block
Memory region (0x001000-0x0C8FFF, 800KB) containing channels, zones, scan lists, and other configuration data. Discovered via V-frame 0x0A.

### Memory Map
Organization of data in the radio's address space. The DM-32UV uses a 16MB address space (0x000000-0xFFFFFF).

### Offset
Position within a block or structure. Often specified in hexadecimal (e.g., offset 0x10 = 16 bytes from start).

## Data Structure Terms

### Channel
48-byte structure containing all settings for one radio channel (name, frequencies, mode, power, etc.).

### Code Plug
Complete radio configuration including channels, zones, contacts, and settings. Stored in radio memory.

### Contact
DMR contact entry (call sign, DMR ID, type). Stored in a separate memory region from channels.

### RX Group (Receive Group)
List of DMR contacts that the radio will receive. Also called "RX Group List."

### Scan List
List of channels to scan. Up to 16 channels per scan list.

### Zone
Collection of up to 23 channels grouped together for easy access. 57-byte structure.

## Radio Terms

### Color Code
DMR-specific setting (0-15) used to filter transmissions. Similar to CTCSS for analog.

### DMR (Digital Mobile Radio)
Digital radio standard (ETSI TS 102 361) used by the DM-32UV for digital mode operation.

### Time Slot
DMR feature allowing two conversations on one frequency. Values: 1 or 2.

### Talkgroup
DMR group call identifier. Allows multiple radios to communicate on the same channel.

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
Command to read data from radio memory. Format: `52 <addr:3> <len:2>`

### Write Command (0x57)
Command to write data to radio memory. Format: `57 <addr:3> 00 10 <data:4096> <metadata:1>`

### Timeout
Maximum time to wait for a response. Typically 500ms for reads, 5000ms for writes.

## Abbreviations

| Term | Meaning |
|------|---------|
| ACK | Acknowledgment |
| BCD | Binary Coded Decimal |
| CPS | Customer Programming Software |
| CTCSS | Continuous Tone-Coded Squelch System |
| DCS | Digital-Coded Squelch |
| DMR | Digital Mobile Radio |
| KB | Kilobyte (1024 bytes) |
| LSB | Least Significant Byte |
| MB | Megabyte (1024 KB) |
| MSB | Most Significant Byte |
| NAK | Negative Acknowledgment |
| RX | Receive |
| TX | Transmit |

## See Also

- **[01-OVERVIEW.md](01-OVERVIEW.md)** - Protocol architecture overview
- **[04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)** - Memory organization details
- **[06-ENCODING.md](06-ENCODING.md)** - Encoding algorithms and examples

---

**Note**: This glossary covers terms specific to the DM-32UV protocol. For general radio or DMR terms, consult amateur radio references or DMR standards documentation.
