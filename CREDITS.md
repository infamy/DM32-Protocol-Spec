# Credits and Acknowledgments

This documentation is the result of collaborative reverse engineering efforts by the radio programming community. We are grateful to everyone who contributed their time, expertise, and data.

## Contributors

### Serial Capture Contributors

These individuals provided serial port captures that were essential for understanding the protocol:

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

This documentation has been verified against the following firmware versions:

| Firmware Version | Date Tested | Source | Notes |
|-----------------|-------------|--------|-------|
| DM32.01.01.046 | 2024-2025 | Factory/DMRVA/GBF | Standard firmware |
| DM32.01.L01.048 | 2025-10-26 | St Pete | L01 variant |
| Various | 2024-2025 | EricPlug | Multiple versions |

## Tools and Software

### Analysis Tools

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
