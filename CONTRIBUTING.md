# Contributing to DM-32UV Protocol Documentation

Thank you for your interest in improving this documentation! This guide will help you contribute effectively.

## How to Contribute

### Reporting Issues

If you find errors, inconsistencies, or missing information:

1. **Search existing issues** to avoid duplicates
2. **Open a new issue** with:
   - Clear description of the problem
   - Location in documentation (file name, section)
   - Expected vs. actual behavior
   - Radio model and firmware version (if applicable)
   - Serial capture data (if relevant)

### Suggesting Improvements

For documentation improvements:

1. Open an issue describing the improvement
2. Explain why it would be helpful
3. Provide examples or references if available

### Submitting Changes

1. **Fork the repository**
2. **Create a branch** for your changes (`git checkout -b fix-memory-layout`)
3. **Make your changes** following the style guide below
4. **Test your changes** (verify examples, check links)
5. **Commit with clear messages** (`git commit -m "Fix: Correct V-frame 0x0A address encoding"`)
6. **Submit a pull request** with:
   - Description of changes
   - Reason for changes
   - Test results or verification method

## Documentation Style Guide

### Markdown Formatting

- Use ATX-style headers (`#`, `##`, `###`)
- Use fenced code blocks with language tags (` ```python `)
- Use tables for structured data
- Use bullet points for lists
- Keep lines under 100 characters when possible

### Technical Writing

- **Be precise**: Use exact terminology (little-endian, not "reversed")
- **Be clear**: Explain complex concepts with examples
- **Be consistent**: Use the same terms throughout (DM-32UV, not DM32)
- **Be complete**: Include all necessary details
- **Be concise**: Avoid unnecessary words

### Code Examples

- Include complete, working examples
- Add comments explaining non-obvious parts
- Use consistent indentation (4 spaces for Python, tabs for Go/C)
- Include error handling where appropriate
- Test code before submitting

### Hex Values

- Use uppercase for hex digits: `0xFF`, not `0xff`
- Use `0x` prefix for hex values
- Show byte sequences with spaces: `FF FF FF FF`, not `FFFFFFFF`
- Include ASCII representation where helpful

### Cross-References

- Link to related sections: `See [03-COMMANDS.md](03-COMMANDS.md)`
- Reference serial capture lines: `See lines 84-86 in serial_capture_example.txt`
- Point to code examples: `See 06-ENCODING.md for implementation`

## What to Contribute

### High Value Contributions

- **Corrections**: Fix errors in existing documentation
- **Clarifications**: Improve unclear explanations
- **Examples**: Add working code examples
- **Test Data**: Provide serial captures from different radios/firmware
- **Verification**: Confirm or correct undocumented features

### Medium Value Contributions

- **Formatting**: Improve readability and consistency
- **Diagrams**: Add visual explanations of complex concepts
- **Translations**: Translate documentation to other languages
- **Tools**: Create example tools using the protocol

### What NOT to Contribute

- **Speculation**: Don't document unverified behavior
- **Copyrighted Material**: Don't include proprietary code or documentation
- **Malicious Content**: Don't document security exploits or harmful uses
- **Off-Topic**: Keep contributions focused on the DM-32UV protocol

## Verification Process

When contributing technical information:

1. **Test with real hardware** when possible
2. **Provide serial captures** showing the behavior
3. **Document test conditions**:
   - Radio model and firmware version
   - Test procedure
   - Expected vs. actual results
4. **Cross-reference** with existing documentation
5. **Note any discrepancies** or firmware-specific behavior

## Serial Capture Guidelines

When submitting serial captures:

- Use a tool like Serial Port Monitor or Wireshark
- Include timestamps
- Show both sent and received data
- Annotate important sequences
- Specify radio model and firmware version
- Remove any personal information (if applicable)

## Code of Conduct

- Be respectful and professional
- Focus on the technical content
- Accept constructive feedback
- Help others learn
- Follow the project's goals and scope

## Questions?

If you're unsure about anything:

1. Open an issue asking for clarification
2. Check existing issues and pull requests
3. Review the documentation structure and style

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

Thank you for helping improve this documentation! 🙏
