# DM-32UV Radio Protocol Reference

Complete protocol specification and memory map reference for the Baofeng DM-32UV radio.

> **⚠️ Documentation Notice**  
> The reverse engineering and protocol analysis was performed by [Infamy](https://github.com/infamy). AI was used to organize and format the raw research notes into structured, digestible documentation, with manual review and corrections throughout.

## 📁 Documentation Structure

### Protocol Specification
- **[01-OVERVIEW.md](01-OVERVIEW.md)** - Protocol architecture and layers
- **[02-CONNECTION-SEQUENCE.md](02-CONNECTION-SEQUENCE.md)** - Connection handshake sequence
- **[03-COMMANDS.md](03-COMMANDS.md)** - Complete command reference

### Memory Map
- **[04-MEMORY-LAYOUT.md](04-MEMORY-LAYOUT.md)** - Memory organization and addressing

### Data Structures
- **[05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md)** - Channel, zone, and all metadata block structures
- **[06-ENCODING.md](06-ENCODING.md)** - BCD frequency and CTCSS/DCS encoding schemes

### Additional Resources
- **[GLOSSARY.md](GLOSSARY.md)** - Glossary of technical terms
- **[LICENSE](LICENSE)** - MIT License and disclaimers
- **[CONTRIBUTING.md](CONTRIBUTING.md)** - Contribution guidelines
- **[CREDITS.md](CREDITS.md)** - Acknowledgments and credits
- **[serial_capture_example.txt](serial_capture_example.txt)** - Real protocol capture from official CPS

## 📖 Documentation

This documentation provides a complete protocol specification and memory map reference for the DM-32UV radio.

### 📊 Serial Capture Example

A real protocol capture from the official CPS software is included in `serial_capture_example.txt`. This capture demonstrates:
- Complete connection handshake sequence (PSEARCH, PASSSTA, SYSINFO)
- All V-frame queries (firmware version, memory layout, etc.)
- Programming mode entry
- Metadata discovery (reading block types)
- Full 4KB block reads

Use this capture to verify your implementation against real-world communication.

## 🔧 Radio Specifications

- **Model**: Baofeng DM-32UV
- **Type**: 10W Multi-Band DMR Radio
- **Channels**: 4,000 max
- **Serial**: 115200 baud, 8N1, no flow control
- **Protocol**: Proprietary binary protocol

## ⚠️ Critical Protocol Notes

1. **Command Order**: Commands must be executed in exact sequence
2. **Byte Order**: Little-endian for all multi-byte values (addresses, lengths, data)
3. **BCD Encoding**: Frequencies use reversed BCD format
4. **Timing**: 10-50ms delays required between commands
5. **Memory Discovery**: Channel bank address varies by firmware - use metadata discovery

## 🚀 Quick Start Example

Here's a minimal example of reading channel data from the radio:

```python
import serial
import time

# 1. Open serial port
port = serial.Serial('/dev/ttyUSB0', 115200, timeout=0.5)

# 2. Handshake
port.write(b'PSEARCH')
assert port.read(8) == b'\x06DP570UV'

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

# 5. Find channel blocks (metadata 0x12-0x41)
for addr in range(start_addr, end_addr, 0x1000):
    # Read metadata byte
    cmd = b'\x52' + addr.to_bytes(3, 'little') + b'\xFF\x0F\x01\x00'
    port.write(cmd)
    response = port.read(7)
    metadata = response[6]
    
    if 0x12 <= metadata <= 0x41:
        print(f"Found channel block at 0x{addr:06X}, metadata=0x{metadata:02X}")
        # Read full 4KB block here...

port.close()
```

See the documentation for complete details on each step.

## 🔍 Troubleshooting

### Radio Not Responding to PSEARCH

**Symptoms:** No response or timeout on initial PSEARCH command

**Solutions:**
- Verify radio is powered ON
- Check USB cable connection (try a different cable)
- Confirm correct serial port (check `/dev/ttyUSB*` on Linux, `COM*` on Windows)
- Ensure no other software is using the serial port (close CPS software)
- Try increasing timeout to 1000ms
- Some radios require a brief delay (100ms) after opening the port before sending commands

### Connection Drops During Programming

**Symptoms:** Radio stops responding mid-operation, timeouts occur

**Solutions:**
- Add 10-50ms delays between commands (`time.sleep(0.01)`)
- Verify power supply is stable (use powered USB hub if needed)
- Check for loose cable connections
- Re-enter programming mode if connection is lost
- Reduce read/write speed by increasing delays

### Write Operations Fail

**Symptoms:** Write commands timeout or return NAK (0x15)

**Solutions:**
- Ensure addresses are 4KB-aligned (0x001000, 0x002000, etc.)
- Use 5000ms timeout for write operations (not 500ms)
- Add 50ms delay between consecutive writes
- Verify metadata byte is correct for block type
- Check that radio is in programming mode

### Wrong Data Read from Radio

**Symptoms:** Garbage data, incorrect channel information

**Solutions:**
- Verify byte order is little-endian for all multi-byte values
- Use metadata discovery instead of hardcoded addresses
- Check BCD frequency decoding (reversed byte order)
- Ensure reading from correct block (verify metadata first)
- Confirm firmware version matches expected memory layout

### PASSSTA Returns Different Values

**Symptoms:** Response is `50 FF FF` instead of `50 00 00` (or vice versa)

**Solution:** Both responses are valid - different radios/firmware return different status bytes. Accept either response.

### V-Frame 0x0D Returns No Data

**Symptoms:** V-frame 0x0D returns `56 0D 00` (0 bytes)

**Solution:** This is normal for some firmware versions. V-frame 0x0D (capabilities) is optional and not all radios support it.

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

### 📧 Contact

For questions, corrections, or contributions, please open an issue on the repository.

---

## 🙏 Credits and Acknowledgments

This documentation would not be possible without the contributions and prior work of many individuals and projects.

### Primary Contributors

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
- **[emuehlstein/dmrconfig_dm32](https://github.com/emuehlstein/dmrconfig_dm32)** - Original serial capture source
- Open-source DMR tools and libraries
- Radio programming software projects
- Amateur radio software communities

---

**Source**: Independent reverse engineering documentation  
**Purpose**: Protocol specification and memory map reference  
**Status**: Community-maintained  
**Last Updated**: 2025-12-02

**If you contributed to this documentation and would like to be credited by name, please open an issue or pull request.**
