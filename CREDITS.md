# Credits and Acknowledgments

This documentation is the result of collaborative reverse engineering efforts by the radio programming community. We are grateful to everyone who contributed their time, expertise, and data.

## Contributors

### Reference Implementation

- **[neonplug](https://neonplug.app)** — the working DM-32UV CPS this specification is validated against
  - Live app: https://neonplug.app
  - Source: https://github.com/infamy/NeonPlug
  - A browser-based (WebSerial) programming tool that talks to real DM-32UV hardware
  - Contributed the byte-level structure work that corrected large parts of this specification:
    channel field mappings validated against an official CPS CSV export, the 145-byte zone record,
    the 57-byte scan list record, the 92-byte contact record with its 16-byte region header and
    per-4KB-block packing, the TX contact blocks `0x42`/`0x43`, the talk group block `0x44`, the
    Quick Access Contact List block `0x0B`, the VFO records in block `0x41`, the radio-settings
    offsets in block `0x04`, and the corrected `0x57` write frame (4102 bytes, logical block ID
    inside the payload at `0xFFF`)
  - Its Diagnostics tooling — metadata analysis, offset inspector, CPS comparison panel, debug
    export — is what made the block-by-block census in
    [07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md) reproducible
  - Equally valuable: its **honest markers**. Comments reading "unverified", "pending hardware
    confirmation", "TODO: Structure parsing needs verification — return empty for now" are the
    direct source of most `⚠️ DERIVED` and `❓ UNKNOWN` markers in these documents. Several claims
    this spec once published as fact are now correctly marked as unproven *because* the
    implementation was candid about not having proven them.

### Serial Capture Contributors

These individuals provided serial port captures that were essential for understanding the protocol:

- **OEM CPS read + write captures (DP570UV, firmware `DM32.01.01.046`, DMR CPS V1.41)** — the two
  captures in this repository, [`serial_capture_example.txt`](serial_capture_example.txt) and
  [`serial_capture_write_example.txt`](serial_capture_write_example.txt)
  - Highest-authority source in the project: what the official software and a real radio actually
    did on the wire
  - Machine-parsing these overturned several long-standing claims — that the `+0xFFF` byte is a
    block *type* (it is a unique logical block **ID**), that the OEM CPS reads metadata at `0x00A`
    (it never does), that several allocated blocks were "Reserved" (they are simply
    unidentified), and the decoded start addresses of five memory-pointer V-frames
  - The **write** capture is what proves the complete logical-ID space, because the CPS emits a
    write for every slot it knows about — including unpopulated ones, addressed to the `0xFFF001`
    sentinel
- **emuehlstein/dmrconfig_dm32** - Primary source for serial captures
  - GitHub: https://github.com/emuehlstein/dmrconfig_dm32
  - Provided comprehensive serial port captures from official CPS software
  - Used as reference for protocol verification and sanity checking
  - Serial capture files formed the foundation for protocol analysis
- **Factory Radio Captures** - Initial protocol documentation
- **DMRVA** - Serial captures and protocol verification
- **GBFMcCall** - Additional radio testing and captures
- **EricPlug** - Multiple firmware version captures (2024-2025)
- **St Pete** - L01 firmware variant captures

### Documentation Contributors

- **Protocol Analysis** - Community members who analyzed CPS software behavior
- **Memory Layout** - Contributors who mapped memory structures
- **Testing & Verification** - Radio owners who tested implementations
- **Code Examples** - Developers who provided working implementations
- **Implementation-Driven Corrections** - The neonplug driver (`src/radios/dm32uv/`), whose
  hardware-exercised code is the tiebreaker whenever this specification and working software
  disagree — except where the OEM CPS captures disagree with both, in which case the captures win

### Community Support

- Amateur radio communities who shared knowledge and experience
- DMR user groups who provided feedback and testing
- Open-source radio programming tool developers
- Forum members who answered questions and provided guidance

## Information Sources

### Primary Sources

1. **Serial Port Captures**
   - **emuehlstein/dmrconfig_dm32** - Primary source repository
     - https://github.com/emuehlstein/dmrconfig_dm32
     - Comprehensive serial captures from official CPS software
     - Multiple radio configurations and firmware versions
     - Used for protocol analysis and verification
   - Official CPS software communication logs
   - Multiple radio models and firmware versions
   - Read and write operation examples

2. **CPS Software Analysis**
   - Reverse engineering of official Customer Programming Software
   - Memory structure analysis
   - Command sequence documentation

3. **Radio Testing**
   - Direct testing with multiple DM-32UV radios
   - Firmware version comparison
   - Edge case and error condition testing

4. **Reference Implementation** — [neonplug](https://neonplug.app)
   - Source: https://github.com/infamy/NeonPlug
   - Working code exercised against real hardware, used as the tiebreaker for structure layouts,
     field mappings, wire-frame shapes and timing constants
   - Its diagnostics panels double as executable documentation of block `0x0B`, block `0x44`, the
     TX contact blocks, the VFO records and the radio-settings offsets
   - Its own unverified/TODO markers are carried forward here as `⚠️ DERIVED` / `❓ UNKNOWN` rather
     than being quietly promoted to fact

### Secondary Sources

1. **Community Knowledge**
   - Forum discussions and shared experiences
   - Prior reverse engineering efforts
   - Amateur radio technical documentation

2. **Related Projects**
   - Open-source DMR tools
   - Radio programming software projects
   - Protocol analysis tools

## Firmware Versions

Firmware versions this documentation has been informed by. **Only the `2025-09-28` row is
reproducible from material in this repository** — it is the pair of OEM CPS captures. The other rows
record community contributions whose underlying data is not held here, so claims resting on them
cannot be checked and should not be read as hardware-confirmed. (For a worked example, the L01 boot
image address in [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md) is marked `⚠️ DERIVED` precisely
because no L01 capture is available here.)

| Firmware Version | Date Tested | Source | Notes |
|-----------------|-------------|--------|-------|
| DM32.01.01.046 | 2024-2025 | Factory/DMRVA/GBF | Standard firmware |
| DM32.01.01.046 | 2025-09-28 | OEM CPS read + write captures | Radio reported model `DP570UV` (DM-32UV rebadge); DSP `D1.01.01.004`, radio `R1.00.01.001`, codeplug `C1.00.01.001`, build date `2022-06-27`, DMR CPS V1.41. **Radio was at factory defaults** — 25 channels, 2 zones, 2 scan lists, 5 quick messages, 5 RX groups, 5 DMR radio IDs, 10 talk groups, 3 roaming zones, 3 roaming channels (counted from the block payloads in the read capture; an earlier revision of this row said "1 zone, 8 talk groups" — miscounted) |
| DM32.01.L01.048 | 2025-10-26 | St Pete | L01 variant |
| Various | 2024-2025 | EricPlug | Multiple versions |

## Tools and Software

### Analysis Tools

- **[neonplug](https://neonplug.app) Diagnostics tab** - Live block hexdumps, logical block ID
  census, offset inspector, OEM CPS CSV comparison, and debug export. The fastest way to reproduce
  or extend the census in [07-BLOCK-INVENTORY.md](07-BLOCK-INVENTORY.md), and the tool of choice for
  the open experiments listed there

- **Serial Port Monitors** - For capturing protocol communication
  - Serial Port Monitor (Windows)
  - Wireshark with serial capture
  - Custom logging tools

- **Hex Editors** - For analyzing memory dumps
  - HxD
  - 010 Editor
  - ImHex

- **Disassemblers** - For CPS software analysis
  - IDA Pro
  - Ghidra
  - Binary Ninja

### Development Tools

- **Programming Languages** - Example implementations
  - Python (primary examples)
  - Go (alternative examples)
  - C/C++ (structure definitions)

- **Documentation Tools**
  - Markdown editors
  - Diagram tools (Mermaid)
  - Version control (Git)

## Related Projects and Resources

### Open Source Projects

- **[neonplug](https://neonplug.app)** - Browser-based CPS for the Baofeng DM-32UV
  - Source: https://github.com/infamy/NeonPlug
  - **The reference implementation for this specification.** If a claim here is marked CONFIRMED
    because it is "implemented and exercised against hardware", this is the implementation meant
  - Radio driver lives at `src/radios/dm32uv/`; diagnostics tooling at
    `src/components/diagnostics/`
- **[emuehlstein/dmrconfig_dm32](https://github.com/emuehlstein/dmrconfig_dm32)** - Original serial capture repository
  - Primary source for serial port captures used in this documentation
  - Contains comprehensive protocol analysis data
  - Recommended for additional reference material
- **DMR Libraries** - Open-source DMR protocol implementations
- **Radio Programming Tools** - Community-developed CPS alternatives
- **Protocol Analyzers** - Tools for analyzing radio protocols

### Communities

- **Amateur Radio Forums** - Technical discussions and support
- **DMR User Groups** - User experiences and best practices
- **GitHub Projects** - Open-source radio software development

### Standards and References

- **DMR Standards** - ETSI TS 102 361 (DMR protocol)
- **Amateur Radio Regulations** - FCC Part 97, CEPT, etc.
- **Serial Communication** - RS-232, USB-Serial standards

## Contributions Most Wanted

The specification is explicit about what it does not know. If you own a DM-32UV, these are the gaps
that would benefit most from your radio, ranked by value per unit of effort. Full list with
suggested experiments: [07-BLOCK-INVENTORY.md § C.4](07-BLOCK-INVENTORY.md#c4-what-is-still-unknown--prioritised-with-an-experiment-for-each).

1. **Hexdump the 31 allocated-but-unread pages.** 31 of the 71 live pages on the captured radio were
   never touched by the OEM CPS, so we have literally no content for them — only their existence.
   That is the single largest gap in this document. Reading is non-destructive and zero-risk.
2. **Dump a *populated* codeplug.** Both captures in this repository are from the same
   factory-default radio, so all allocation counts here are lower bounds. A radio with many
   channels, zones and talk groups would settle several open questions at once.
3. **Diff the `0x04` radio-settings block** field by field while driving the OEM CPS UI. Roughly 98%
   of that block is unmapped, and this is the most mechanical, highest-yield experiment available.
4. **Map the roaming zone / roaming channel blocks** (`0x65`, `0x66`). Strides are known; nothing
   else is, and no implementation supports the feature.

## How to Get Credit

If you contributed to this documentation and would like to be credited:

1. **Open an Issue** - Describe your contribution
2. **Submit a Pull Request** - Add yourself to this file
3. **Provide Details** - Include:
   - Your name or handle (as you'd like to be credited)
   - Nature of contribution (serial captures, testing, code, etc.)
   - Date of contribution
   - Any relevant links or references

### Credit Format

Please use this format when adding yourself:

```markdown
- **[Your Name/Handle]** - [Contribution type] - [Date]
  - Brief description of contribution
  - Link to relevant data/code (if applicable)
```

## Disclaimer

While we strive to credit all contributors, some contributions were made anonymously or through indirect channels. If you believe you should be credited and are not listed, please contact us.

## License

All contributions to this documentation are licensed under the MIT License. By contributing, you agree to license your contributions under the same terms.

---

**Thank you to everyone who made this documentation possible!** 🙏

Your contributions help the community create better tools and understand radio protocols more deeply.
