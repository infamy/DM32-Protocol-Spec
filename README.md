# DM-32UV Radio Protocol Reference

Protocol specification and memory map reference for the Baofeng DM-32UV radio.

Enough of the protocol is documented here to read and write a codeplug. It is **not** a complete
map of the radio's memory, and it does not pretend to be — every claim carries a confidence marker,
and [07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md) keeps a running count of what is still unknown.

> **⚠️ Documentation Notice**  
> The reverse engineering and protocol analysis was performed by [Infamy](https://github.com/infamy). AI was used to organize and format the raw research notes into structured, digestible documentation, with manual review and corrections throughout.

## 🔗 Reference Implementation

These documents are validated against **[neonplug](https://neonplug.app)** — a working,
browser-based CPS that talks to real DM-32UV hardware over WebSerial.

- **Live app**: [https://neonplug.app](https://neonplug.app)
- **Source**: [github.com/infamy/NeonPlug](https://github.com/infamy/NeonPlug)
- **Radio driver**: `src/radios/dm32uv/` (`connection.ts`, `memory.ts`, `constants.ts`,
  `blockLayouts.ts`, `structures.ts`, `protocol.ts`)

neonplug is the authority for anything these docs describe as implemented. Sections throughout the
spec carry an **Implementation reference** line pointing at the file that implements them. Where an
older revision of this spec was corrected, a short *"previously documented as … — incorrect"* note
is kept so readers holding a cached copy are not silently misled.

Two things outrank even the implementation: the **OEM CPS serial captures** in this repo
(`serial_capture_example.txt`, `serial_capture_write_example.txt`). Those are what the official
software and a real radio actually did on the wire, and several long-standing claims in these docs
were overturned by machine-parsing them — see the corrections summary in
[07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md#corrections-summary).

## 🎯 Confidence Markers

Every non-obvious claim in these documents is marked. The markers are greppable — use them to decide
how much to trust a field before you write it to a radio.

| Marker | Meaning |
|--------|---------|
| *(no marker)* | **CONFIRMED** — observed in a hardware capture, or implemented and exercised against a real radio |
| `⚠️ DERIVED` | Implemented or inferred, but **never verified against hardware**. Plausible, unproven. |
| `❓ UNKNOWN` | Purpose not established. Preserve these bytes on write; do not assume they are free. |

Note that "CONFIRMED" applies to a *claim*, not to a *feature*. A block layout can be
hardware-CONFIRMED while the reference implementation's parser for it is still a stub — both facts
are documented where they apply (block `0x10` is the standing example).

## 📁 Documentation Structure

### Protocol Specification
- **[01-OVERVIEW.md](01-OVERVIEW.md)** - Protocol architecture and layers
- **[02-CONNECTION-SEQUENCE.md](02-CONNECTION-SEQUENCE.md)** - Connection handshake sequence
- **[03-COMMANDS.md](03-COMMANDS.md)** - Complete command reference

### Memory Map
- **[04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)** - Address space, V-frame region pointers, and the metadata-page discovery mechanism
- **[07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md)** - The census: every 4KB metadata page, every logical block ID, and an honest count of what is still unknown

### Data Structures
- **[05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md)** - Byte-level record layouts: channel, zone, scan list, talk group, TX contact, settings
- **[06-ENCODING.md](06-ENCODING.md)** - BCD frequency and CTCSS/DCS encoding schemes

### Additional Resources
- **[GLOSSARY.md](GLOSSARY.md)** - Glossary of technical terms
- **[LICENSE](LICENSE)** - MIT License and disclaimers
- **[CONTRIBUTING.md](CONTRIBUTING.md)** - Contribution guidelines
- **[CREDITS.md](CREDITS.md)** - Acknowledgments and credits
- **[serial_capture_example.txt](serial_capture_example.txt)** - Real **read** session capture from the official CPS
- **[serial_capture_write_example.txt](serial_capture_write_example.txt)** - Real **write** session capture from the official CPS

## 📖 Documentation

This documentation provides a protocol specification and memory map reference for the DM-32UV radio.
It is not complete, and it says where it is not — see [Documentation Status](#-documentation-status).

### 📊 Serial Capture Examples

Two real protocol captures from the official CPS software are included. Both were taken against the
same radio (a DP570UV — a DM-32UV rebadge) running firmware `DM32.01.01.046`, with DMR CPS V1.41.

| File | Session | Demonstrates |
|------|---------|--------------|
| `serial_capture_example.txt` | Full read | Handshake (PSEARCH, PASSSTA, SYSINFO), all V-frame queries, programming mode entry, metadata discovery across all 200 pages, full 4KB block reads |
| `serial_capture_write_example.txt` | Full write | The complete write pass — the OEM CPS walking the entire logical-ID space in ascending order, including the `0xFFF001` sentinel writes for unallocated slots |

These captures are the **highest-authority source in this repository**. They establish, among other
things, that the byte at `+0xFFF` is a unique *logical block ID* rather than a block *type*, and
that the metadata probe reads `0xFFF` (not `0x00A`, as earlier revisions claimed).

> ⚠️ **Sample bias.** The captured radio was at factory defaults. Counted directly out of the read
> capture's block payloads: **25 channels** (block `0x12`, `19 00 00 00`), **2 zones**
> (`Zone 1`, `Func Demo`), **2 scan lists**, **5 quick messages**, **5 RX groups** (block `0x0F`
> bitmask `1F 00 00 00`), **5 DMR radio IDs**, **10 talk groups** (block `0x44`, corroborated by
> block `0x0B`'s total-count field reading `0A 00`), **3 roaming zones**, **3 roaming channels**.
> Allocation counts derived from this radio are a **lower bound**, not a steady state. Entry
> *strides* generalise; entry *counts* do not.

Use these captures to verify your implementation against real-world communication.

## 📈 Documentation Status

The honest scoreboard, from
[07-BLOCK-INVENTORY.md § C](07-BLOCK-INVENTORY.md#c-scoreboard). All figures are from the two
hardware captures above.

**Metadata / logical block ID space** — of the 256 possible byte values:

| Status | Distinct values |
|--------|-----------------|
| CONFIRMED | **74** |
| `⚠️ DERIVED` | **3** |
| `❓ UNKNOWN` | **32** |
| Never observed in any capture, never named in any implementation | **147** |

**Pages on the captured radio** — 200 pages of 4 KB in the main config region:

| Category | Pages |
|----------|-------|
| Carrying a live logical block ID | 71 (35.5%) |
| Tagged `0x00` — `⚠️ DERIVED` invalidated / superseded | 15 (7.5%) |
| Tagged `0xFF` — `⚠️ DERIVED` free / erased | 114 (57.0%) |

Of those 71 live pages, **40 are read and/or written by the OEM CPS** — that set is the codeplug.
The other **31 are allocated but never touched by the CPS in either capture**, and we have no
content for them at all. Those 31 are the single largest gap in this specification.

**Parse completeness in the reference implementation** — of the 14 block types neonplug reads,
**3 parse completely, 7 parse partially, and 4 are not parsed at all**
([§ C.3](07-BLOCK-INVENTORY.md#c3-parse-completeness-for-the-block-types-neonplug-reads)).
One of the four unparsed blocks is `0x10`, whose *layout* is hardware-confirmed while its parser is
a stub — the standing example of a CONFIRMED claim about an unimplemented feature.

Ten prioritised open questions, each with a concrete and mostly cheap experiment, are listed in
[§ C.4](07-BLOCK-INVENTORY.md#c4-what-is-still-unknown--prioritised-with-an-experiment-for-each).
If you own a DM-32UV, Q1 (hexdump the 31 untouched pages) is zero-cost, zero-risk, and would be the
most valuable contribution this project could receive.

## 🔧 Radio Specifications

- **Model**: Baofeng DM-32UV (also sold rebadged — the captures here are from a DP570UV)
- **Type**: 10W Multi-Band DMR Radio
- **Channels**: 4,000 max, plus VFO A / VFO B as pseudo-channels 4001 / 4002
- **Contacts**: 50,000 max — reported by the radio itself via V-frame `0x10` (`56 10 03 50 C3 00` → `0x00C350`)
- **Zones**: 250 max `⚠️ DERIVED`
- **Serial**: 115200 baud. 8N1 / no flow control are the *host defaults* — the reference
  implementation never sets data bits, parity, stop bits or flow control at all, so this is a
  platform default that works, not a documented radio requirement
- **Protocol**: Proprietary binary protocol, 24-bit little-endian addressing over a 16 MB space

## ⚠️ Critical Protocol Notes

1. **Command Order**: Commands must be executed in exact sequence
2. **Byte Order**: Little-endian for all multi-byte values (addresses, lengths, data). Addresses are **3 bytes**, little-endian
3. **BCD Encoding**: Frequencies use reversed BCD format
4. **Timing**: 10-50 ms between ordinary commands, **150 ms between 4 KB block reads** and ~5 ms
   between metadata probes. Timeouts are **5000 ms** per request/response and **15000 ms** for a
   4 KB read payload. See
   [02-CONNECTION-SEQUENCE.md](02-CONNECTION-SEQUENCE.md#timing-and-timeout-reference)
5. **Never Hardcode Addresses**: A page's address is assigned by what behaves like a **flash translation layer** — the same logical block ID lands at different physical addresses on different radios and after different edits. Always discover pages by reading the logical block ID at `+0xFFF`. See [04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)
6. **The `+0xFFF` byte is an ID, not a type**: Every value occurs at most once across the whole main config region. `0x12`-`0x41` are 48 *distinct* channel bank slots, not 48 pages that all mean "channel". Previously documented as a "block type" byte — **incorrect**

## 🚀 Quick Start Example

Here's a minimal example of reading channel data from the radio:

```python
import serial
import time

# 1. Open serial port
# timeout is per read: the protocol allows 5s per request/response cycle.
port = serial.Serial('/dev/ttyUSB0', 115200, timeout=5.0)
time.sleep(0.6)   # reference impl waits 400ms after open, then 200ms after flushing init bytes

# 2. Handshake
port.write(b'PSEARCH')
# 8 bytes: ACK + 7-char model string. The captured radio answers 'DP570UV';
# a DM-32UV answers with its own model string, so match on ACK + prefix, not equality.
psearch = port.read(8)
assert psearch[0] == 0x06

port.write(b'PASSSTA')
port.read(3)  # Status (varies)

port.write(b'SYSINFO')
assert port.read(1) == b'\x06'

# 3. Get memory layout
port.write(b'\x56\x00\x00\x00\x0A')  # V-frame 0x0A
response = port.read(11)
start_addr = int.from_bytes(response[3:7], 'little')  # 0x001000
end_addr = int.from_bytes(response[7:11], 'little')   # 0x0C8FFF

# 4. Enter programming mode
port.write(b'\xFF\xFF\xFF\xFF\x0CPROGRAM')
assert port.read(1) == b'\x06'
port.write(b'\x02')
assert port.read(8) == b'\xFF' * 8
port.write(b'\x06')
assert port.read(1) == b'\x06'

# 5. Find channel bank pages (logical block IDs 0x12-0x41)
for addr in range(start_addr, end_addr, 0x1000):
    # Read the logical block ID: 1 byte at page + 0xFFF
    cmd = b'\x52' + (addr + 0xFFF).to_bytes(3, 'little') + b'\x01\x00'
    port.write(cmd)
    response = port.read(7)   # 57 <addr:3 LE> <len:2 LE> <byte>
    block_id = response[6]    # payload starts at offset 6

    if 0x12 <= block_id <= 0x41:
        print(f"Channel bank slot 0x{block_id:02X} lives at 0x{addr:06X}")
        # Read full 4KB page here...

    time.sleep(0.005)  # ~5ms between probes

port.close()
```

Notes on step 5:

- The read command is 6 bytes: `52 <addr:3 LE> <len:2 LE>`. The probe address is the page base
  **plus `0xFFF`**.
- The radio answers a `0x52` read with a `0x57`-headed frame that echoes the request header, so the
  payload begins at offset 6.
- `0x00` and `0xFF` both mean "no live block here", but they are `⚠️ DERIVED` to mean different
  things — see [07-BLOCK-INVENTORY.md § A.3](07-BLOCK-INVENTORY.md).

See the documentation for complete details on each step.

## 🔍 Troubleshooting

### Radio Not Responding to PSEARCH

**Symptoms:** No response or timeout on initial PSEARCH command

**Solutions:**
- Verify radio is powered ON
- Check USB cable connection (try a different cable)
- Confirm correct serial port (check `/dev/ttyUSB*` on Linux, `COM*` on Windows)
- Ensure no other software is using the serial port (close CPS software)
- Wait after opening the port before sending anything: the reference implementation waits **400 ms**,
  flushes whatever the radio emitted, waits a further **200 ms**, and only then sends `PSEARCH` —
  with a further **150 ms** before reading the reply
  ([02-CONNECTION-SEQUENCE.md](02-CONNECTION-SEQUENCE.md))
- Close and reopen the port for a clean handshake (the reference implementation waits 400 ms between
  close and reopen)

### Connection Drops During Programming

**Symptoms:** Radio stops responding mid-operation, timeouts occur

**Solutions:**
- Add 10-50ms delays between commands (`time.sleep(0.01)`)
- Verify power supply is stable (use powered USB hub if needed)
- Check for loose cable connections
- Re-enter programming mode if connection is lost
- Reduce read/write speed by increasing delays

### Write Operations Fail

**Symptoms:** Write commands time out or return a non-ACK byte

**Solutions:**
- Ensure addresses are 4KB-aligned (0x001000, 0x002000, etc.). The one observed exception is the
  `0xFFF001` sentinel the OEM CPS uses to declare a logical block ID that has no page — see
  [07-BLOCK-INVENTORY.md § A.2](07-BLOCK-INVENTORY.md)
- Use 5000ms timeout for write operations (not 500ms)
- Add 50ms delay between consecutive writes
- Verify the payload is **exactly 4096 bytes** and that the logical block ID sits at payload offset
  `0xFFF`. It is **not** appended after the payload — the whole write frame is 4102 bytes.
  *(Previously documented as `57 <addr:3> 00 10 <data:4096> <metadata:1>`, 4103 bytes — **incorrect**)*
- Check that the radio is in programming mode

**Response bytes**: `0x06` = ACK — that one is CONFIRMED; every 4 KB write in the write capture is
acknowledged with a single `0x06`. The failure codes are **`⚠️ DERIVED`** and are *never observed* —
neither capture contains a failed write, and the reference implementation's own comments say each
"may indicate":

| Byte | Reference implementation's verbatim guess |
|------|-------------------------------------------|
| `0xC0` | write rejected, invalid address, or radio not in programming mode |
| `0xC8` | invalid block data format, checksum error, or block structure issue |
| `0x48` | write timeout, radio busy processing previous write, or need for longer delay between writes |

`❓ UNKNOWN` — a `0x15` NAK was documented in earlier revisions. `0x15` appears as a response byte
nowhere in either capture and nowhere in the reference implementation; treat it as unsupported.

> ⚠️ Reading is non-destructive. **Writing is not.** The radio reboots if it dislikes any part of a
> write, and there is no protocol-level retry.

### Wrong Data Read from Radio

**Symptoms:** Garbage data, incorrect channel information

**Solutions:**
- Verify byte order is little-endian for all multi-byte values
- Use logical-block-ID discovery instead of hardcoded addresses — addresses move
- Check BCD frequency decoding (reversed byte order)
- Ensure you are reading the right page (verify the ID at `+0xFFF` first)
- Confirm firmware version matches expected memory layout

### PASSSTA Returns Different Values

**Symptoms:** The 3-byte response is not `50 00 00`

**Solution:** Validate only the first byte (`0x50`, `'P'`) and ignore bytes 1-2 — that is all any
implementation checks. `50 00 00` is the only value ever captured; `50 FF FF` has been reported
anecdotally but never evidenced.

### V-Frame 0x0D Returns No Data

**Symptoms:** V-frame 0x0D returns `56 0D 00` (0 bytes)

**Solution:** This is normal. In both captures the OEM CPS queries `0x0D` **twice**: once in an
undocumented form carrying `0x40` in the fourth byte (`56 00 00 40 0D` — the very first V-frame of
the session), which returns 64 bytes (`56 0D 40 03 4E 2D 00 …`), and once plainly (`56 00 00 00 0D`),
which returns 0 bytes. Treat a 0-byte plain reply as expected.

`❓ UNKNOWN` — the meaning of the 64-byte payload is not established, and neither is the role of the
`0x40` byte: it sits where every other V-frame carries `0x00`, and calling it a "length" or a "high
byte" is a guess, not an observation. It was previously documented as "capabilities"; that label has
no supporting evidence in either capture and should not be relied on.

## 📝 License

This documentation is released under the **MIT License**. See [LICENSE](LICENSE) file for full terms.

### ⚠️ Important Disclaimers

**Use at Your Own Risk**: This documentation is provided "as is" without warranty of any kind. The authors are not responsible for:
- Equipment damage or data loss
- Regulatory violations or legal issues
- Compatibility issues with specific radio models or firmware versions

**Compliance**: Users must ensure their use complies with:
- Local radio regulations (FCC Part 97, CE, etc.)
- Manufacturer warranties and terms of service
- Applicable laws regarding radio equipment modification

**Purpose**: This documentation is intended for:
- Educational and research purposes
- Interoperability and software development
- Creating compatible tools and applications

**Not Affiliated**: This is an independent reverse engineering effort, not affiliated with or endorsed by Baofeng or any manufacturer.

### 🤝 Contributing

Contributions are welcome! If you find errors or have improvements:
- Open an issue to discuss changes
- Submit pull requests with clear descriptions
- Include test data or serial captures when relevant
- Follow the existing documentation style
- **Mark your confidence.** A new claim needs `⚠️ DERIVED` unless you saw it on hardware. Promoting
  something to CONFIRMED requires a capture, a hexdump, or working code that a real radio accepted
- When you correct an existing claim, leave a short *"previously documented as X — incorrect"* note
  so readers holding a cached copy of the old doc are not silently misled

The most useful contributions right now are hexdumps, not prose — see
[Contributions Most Wanted](CREDITS.md#contributions-most-wanted).

### 📧 Contact

For questions, corrections, or contributions, please open an issue on the repository.

---

## 🙏 Credits and Acknowledgments

This documentation would not be possible without the contributions and prior work of many individuals and projects.

### Primary Contributors

- **Reference Implementation**: [neonplug](https://neonplug.app) ([source](https://github.com/infamy/NeonPlug)) — the working DM-32UV CPS these documents are validated against
- **Serial Capture Data**: Factory, DMRVA, GBFMcCall, EricPlug, St Pete contributors
- **Protocol Analysis**: Community reverse engineering efforts
- **Documentation**: Multiple contributors who tested, verified, and documented the protocol

### Information Sources

- **Serial Port Captures**: Real-world communication logs from official CPS software
  - Primary source: [emuehlstein/dmrconfig_dm32](https://github.com/emuehlstein/dmrconfig_dm32)
- **CPS Software Analysis**: Reverse engineering of official Customer Programming Software
- **Community Testing**: Radio owners who tested and verified protocol behavior across different firmware versions
- **Prior Documentation**: Earlier reverse engineering efforts and community knowledge

### Special Thanks

- Radio owners who provided serial captures from different firmware versions
- Developers who tested implementations and reported findings
- Community members who verified and corrected documentation
- Everyone who contributed bug reports, corrections, and improvements

### Firmware Versions Documented

This documentation covers behavior observed across multiple firmware versions:
- DM32.01.01.046 (Standard firmware)
- DM32.01.L01.048 (L01 variant)
- Various other versions as noted in specific sections

### Tools Used

- **Serial Port Monitors**: For capturing protocol communication
- **Hex Editors**: For analyzing memory dumps and data structures
- **Disassemblers**: For analyzing CPS software behavior
- **Community Tools**: Various open-source radio programming tools

### Related Projects

If you're working with DMR radios or similar protocols, check out:
- **[neonplug](https://neonplug.app)** ([source](https://github.com/infamy/NeonPlug)) - Browser-based CPS for the DM-32UV; the reference implementation for this spec
- **[emuehlstein/dmrconfig_dm32](https://github.com/emuehlstein/dmrconfig_dm32)** - Original serial capture source
- Open-source DMR tools and libraries
- Radio programming software projects
- Amateur radio software communities

---

**Source**: Independent reverse engineering documentation  
**Purpose**: Protocol specification and memory map reference  
**Status**: Community-maintained. Incomplete by 31 unread pages and 32 unidentified block IDs — see [Documentation Status](#-documentation-status)  
**Reference implementation**: [neonplug](https://neonplug.app)  
**Last Updated**: 2026-08-03

**If you contributed to this documentation and would like to be credited by name, please open an issue or pull request.**
