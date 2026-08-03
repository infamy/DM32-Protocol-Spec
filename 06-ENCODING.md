# DM-32UV Encoding and Decoding

This document provides complete implementations for all encoding/decoding algorithms used in the DM-32UV protocol.

## Confidence markers

Every claim in this document carries a confidence level:

| Marker | Meaning |
|--------|---------|
| *(no marker)* | **CONFIRMED** — matches a hardware capture, or is implemented in the reference implementation (which programs real radios) with no caveat in the source |
| `⚠️ DERIVED` | Implemented in the reference implementation but carrying a source caveat, or not corroborated by hardware |
| `❓ UNKNOWN` | Purpose or exact rule not established |

Where a row describes *encoder* behaviour, that is the behaviour of the reference implementation.
The radio's own writes differ in several places (name padding in particular) — those differences are
called out inline. A rule that only the reference implementation exercises is **not** evidence of
what the radio requires.

**Implementation reference for this entire document**: `src/radios/dm32uv/structures.ts` in
[neonplug](https://neonplug.app) — the codecs live at the top of that file (`decodeBCDFrequency`,
`encodeBCDFrequency`, `decodeCTCSSDCS`, `encodeCTCSSDCS`) and the per-record string/integer
handling in the `parse*` / `encode*` functions further down. Codec unit tests:
`tests/unit/structures.test.ts`.

## The one rule that unifies every numeric codec

> **BCD digits are packed most-significant-first into a word, and the word is then stored
> little-endian.**

Both the 4-byte frequency codec and the 2-byte CTCSS/DCS codec obey it. If you implement that one
rule you can express both as `store_le(bcd_pack(value))`. Everything else in this document is
either a sentinel value, a bit-field selector layered on top of the high byte, or a string/integer
convention.

Plain (non-BCD) integers are a **separate** convention: they are ordinary little-endian binary,
never BCD. See [Integer Encoding and Endianness](#integer-encoding-and-endianness).

---

## BCD Frequency Encoding

### Format

Frequencies are stored as **BCD (Binary Coded Decimal)** in 4 bytes with **little-endian** byte order.

- Scale: **frequency in MHz × 100 000** → an 8-digit integer (resolution 10 Hz)
- Those 8 decimal digits are packed two-per-byte, most-significant digit first
- The resulting 32-bit BCD word is written **little-endian**
- Net display format: `XXX.XXXXX` MHz

### Algorithm

```
Frequency: 145.350 MHz
Step 1: Scale by 100000 and round:  14535000
Step 2: Group as BCD nibbles (MSD first): 14 53 50 00
Step 3: Store little-endian (reverse the 4 bytes): 00 50 53 14
```

**Encoding Steps:**
1. Convert frequency to 8-digit integer (drop decimal point)
   - Example: 145.350 MHz → 14535000
   - Use round-half-away-from-zero, **not** truncation — the reference implementation uses
     `Math.round(frequency * 100000)`. Truncating loses the last digit on values such as
     462.5625 MHz because of binary floating-point representation.
2. Convert to BCD (4 bytes, big-endian)
   - Each byte = two BCD digits (high nibble, low nibble)
   - Example: 14535000 → `14 53 50 00`
3. Reverse to little-endian
   - Example: `00 50 53 14`

**Decoding Steps:**
1. Reverse from little-endian to big-endian
2. Extract BCD digits (two per byte)
3. Insert decimal point at position 3
   - Example: `14535000` → `145.35000`

### Worked examples

| Frequency | Scaled (×100000) | BCD word (MSD first) | Stored bytes (little-endian) | Source |
|-----------|------------------|----------------------|------------------------------|--------|
| 145.35000 MHz | 14535000 | `14 53 50 00` | `00 50 53 14` | worked above |
| 146.52000 MHz | 14652000 | `14 65 20 00` | `00 20 65 14` | unit-tested |
| 440.00000 MHz | 44000000 | `44 00 00 00` | `00 00 00 44` | unit-tested |
| **432.02500 MHz** | 43202500 | `43 20 25 00` | **`00 25 20 43`** | **hardware** — OEM CPS read capture, VFO B RX at block `0x41` offset `0x0FDF` |
| 462.56250 MHz | 46256250 | `46 25 62 50` | `50 62 25 46` | round-trip tested |
| 87.50000 MHz | 8750000 | `08 75 00 00` | `00 00 75 08` | round-trip tested |
| 0.00000 MHz | 0 | `00 00 00 00` | `00 00 00 00` | unit-tested |

The 432.025 MHz row is the strongest evidence in this document: it is a real byte sequence a real
DM-32UV-family radio returned, and it independently confirms byte order, nibble packing, digit
count and scale all at once.

### Sentinels and edge cases

| Case | Bytes | Meaning | Confidence |
|------|-------|---------|------------|
| TX frequency all `0xFF` | `FF FF FF FF` | **"No TX"** — receive-only channel. The reference implementation surfaces this to the UI as the out-of-band magic value `1666.666` MHz | CONFIRMED |
| All zeros | `00 00 00 00` | 0.0 MHz — an empty/unprogrammed frequency, not a sentinel | CONFIRMED |
| Fewer than 4 bytes supplied | — | Decoder **must** reject. Reference implementation throws `BCD frequency must be 4 bytes` | CONFIRMED |

**When the "No TX" sentinel is written**: only when the RX frequency is in the
**87 MHz ≤ f < 136 MHz** receive-only band (aviation / broadcast FM) **and** the channel's
Forbid TX flag is set. Otherwise the real TX frequency is BCD-encoded even for a forbid-TX channel.
Round-tripping a `FF FF FF FF` TX field through a decoder that does not special-case it produces
garbage, because `1666.666` MHz does not fit in 8 BCD digits.

⚠️ DERIVED: `0xFF` nibbles are not valid BCD, so the sentinel is unambiguous, but the radio's
behaviour with *partially* invalid BCD (a nibble in `A`–`F`) is ❓ UNKNOWN. The reference
implementation does not validate nibbles; it will happily compute `high*10 + low` for nibble
values above 9 and return a nonsense frequency.

**No range check on encode.** Neither the reference implementation nor the code below rejects a
frequency above 999.99999 MHz. Such a value silently overflows the 8-digit field and decodes back
as a completely different frequency. Validate before encoding.

### C/C++ Implementation

This implementation matches the reference implementation byte-for-byte.

```c
#include <stdint.h>
#include <string.h>
#include <stdio.h>
#include <math.h>

/**
 * Encode frequency (MHz) as 4-byte BCD little-endian
 * 
 * @param freq_mhz Frequency in MHz (e.g., 145.350)
 * @param out Output buffer (must be 4 bytes)
 */
void encode_bcd_frequency(double freq_mhz, uint8_t *out) {
    // Convert to 8-digit integer (145.35000 MHz → 14535000)
    // NOTE: round, do NOT truncate. (uint32_t)(462.5625 * 100000.0) truncates to
    // 46256249 on typical IEEE-754 doubles and encodes the wrong frequency.
    uint32_t freq_int = (uint32_t)llround(freq_mhz * 100000.0);
    
    // Convert to BCD (big-endian)
    uint8_t bcd[4];
    for (int i = 3; i >= 0; i--) {
        uint8_t low = freq_int % 10;
        freq_int /= 10;
        uint8_t high = freq_int % 10;
        freq_int /= 10;
        bcd[i] = (high << 4) | low;
    }
    
    // Reverse to little-endian
    for (int i = 0; i < 4; i++) {
        out[i] = bcd[3 - i];
    }
}

/**
 * Decode 4-byte BCD little-endian to frequency (MHz)
 * 
 * @param data Input data (4 bytes)
 * @return Frequency in MHz
 */
double decode_bcd_frequency(const uint8_t *data) {
    // Reverse from little-endian
    uint8_t bcd[4];
    for (int i = 0; i < 4; i++) {
        bcd[i] = data[3 - i];
    }
    
    // Extract BCD digits
    uint32_t freq_int = 0;
    for (int i = 0; i < 4; i++) {
        uint8_t high = (bcd[i] >> 4) & 0x0F;
        uint8_t low = bcd[i] & 0x0F;
        freq_int = freq_int * 100 + high * 10 + low;
    }
    
    // Convert to MHz (14535000 → 145.35000)
    return freq_int / 100000.0;
}
```

### Go Implementation

```go
package dm32

import (
    "fmt"
    "math"
)

// EncodeBCDFrequency encodes frequency (MHz) as 4-byte BCD little-endian
func EncodeBCDFrequency(freqMHz float64) []byte {
    // Convert to 8-digit integer (145.35000 → 14535000)
    freqInt := uint32(math.Round(freqMHz * 100000))
    
    // Convert to BCD (big-endian)
    bcd := make([]byte, 4)
    for i := 3; i >= 0; i-- {
        low := freqInt % 10
        freqInt /= 10
        high := freqInt % 10
        freqInt /= 10
        bcd[i] = byte((high << 4) | low)
    }
    
    // Reverse to little-endian
    reversed := make([]byte, 4)
    for i := 0; i < 4; i++ {
        reversed[i] = bcd[3-i]
    }
    
    return reversed
}

// DecodeBCDFrequency decodes 4-byte BCD little-endian to frequency (MHz)
func DecodeBCDFrequency(data []byte) (float64, error) {
    if len(data) != 4 {
        return 0, fmt.Errorf("BCD frequency must be 4 bytes, got %d", len(data))
    }
    
    // Reverse from little-endian
    bcd := make([]byte, 4)
    for i := 0; i < 4; i++ {
        bcd[i] = data[3-i]
    }
    
    // Extract BCD digits
    var freqInt uint32
    for i := 0; i < 4; i++ {
        high := (bcd[i] >> 4) & 0x0F
        low := bcd[i] & 0x0F
        freqInt = freqInt*100 + uint32(high)*10 + uint32(low)
    }
    
    // Convert to MHz
    return float64(freqInt) / 100000.0, nil
}
```

## CTCSS/DCS Encoding

> **This section was substantially wrong in earlier revisions of this document.** The stated
> *examples* were right but the stated *formulas* and the reference code below them were not — the
> CTCSS nibbles were transposed and the high byte was written as a plain binary integer instead of
> BCD. See [Previously documented incorrectly](#previously-documented-incorrectly) for the exact
> diff. The rules below are now unit-tested over **all 104 standard DCS codes × both polarities**
> plus four CTCSS tones (67.0, 100.0, 127.3, 203.5 Hz — the only tones the test suite asserts
> individually; the radio's 40-entry CTCSS list is not exhaustively covered).

### Format

CTCSS tones and DCS codes share **one** 2-byte field. The field is a **16-bit BCD word stored
little-endian**, exactly like the frequency codec — with the top two bits of the high byte
repurposed as a type/polarity selector.

```
memory:   byte[0] = LOW byte      byte[1] = HIGH byte
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ little-endian: LSB first
```

| High byte | Type | |
|-----------|------|---|
| `0x00`–`0x7F` | **CTCSS** | BCD hundreds/tens digits |
| `0x80`–`0xBF` | **DCS, Normal** polarity | `0x80 \| hundreds_digit` |
| `0xC0`–`0xFF` | **DCS, Inverted** polarity | `0xC0 \| hundreds_digit` |

The discriminator works because the highest standard CTCSS tone is 250.3 Hz, whose high byte is
`0x25` — a BCD hundreds digit can never exceed 2, so bit 7 is always clear for CTCSS.

### CTCSS

`value = round(tone_Hz × 10)` → 4 BCD digits → little-endian.
Equivalently, per nibble:

| Byte | Nibble | Digit |
|------|--------|-------|
| `byte[1]` (high) | 7–4 | hundreds |
| `byte[1]` (high) | 3–0 | tens |
| `byte[0]` (low) | 7–4 | ones |
| `byte[0]` (low) | 3–0 | tenths |

`tone_Hz = (hundreds × 100 + tens × 10 + ones) + tenths / 10`

**Worked examples** (all unit-tested in both directions):

| Tone | ×10 | BCD word | Stored `[low, high]` |
|------|-----|----------|----------------------|
| 67.0 Hz | 670 | `0x0670` | `70 06` |
| 100.0 Hz | 1000 | `0x1000` | `00 10` |
| **127.3 Hz** | 1273 | `0x1273` | **`73 12`** |
| 203.5 Hz | 2035 | `0x2035` | `35 20` |
| 250.3 Hz | 2503 | `0x2503` | `03 25` ⚠️ DERIVED (arithmetic; highest standard tone, not individually asserted in tests) |

Walk 127.3 Hz by hand: hundreds `1`, tens `2`, ones `7`, tenths `3` → high byte `0x12`,
low byte `0x73` → stored `73 12`.

**Note on the tenths digit.** The reference implementation does not scale by 10 up front; it
computes the tenths digit as `Math.round((tone_Hz − floor(tone_Hz)) × 10)`. That is equivalent to
`round(tone_Hz × 10)` for every tone in the standard set — the fatal variant is the *truncating*
one, `int((tone_Hz − floor(tone_Hz)) × 10)`, which yields 2 for 127.3 Hz because the subtraction
produces 2.999…. Either rounded form is safe; the truncating form is not.

### DCS

DCS codes are conventionally written as three **octal** digits (`D023`, `D754`). They are stored as
three **BCD** digits — which for octal input is the same nibble packing, since every digit is 0–7.

| Byte | Nibble | Content |
|------|--------|---------|
| `byte[1]` (high) | 7–6 | Polarity selector: `10` = Normal (`0x80`), `11` = Inverted (`0xC0`) |
| `byte[1]` (high) | 3–0 | hundreds digit |
| `byte[0]` (low) | 7–4 | tens digit |
| `byte[0]` (low) | 3–0 | ones digit |

`code = (high & 0x0F) × 100 + ((low >> 4) & 0x0F) × 10 + (low & 0x0F)`

❓ UNKNOWN: high-byte bits 5–4 are never set by the reference implementation (its encoder ORs only
`0x80` / `0xC0` with the hundreds digit). There is **no observational evidence either way**: every
tone field in the hardware read capture is the "no tone" sentinel `FF FF`, so neither capture
contains a single DCS-coded channel. *Previously documented as "always 0 in the observed data" —
unsupported; there is no observed DCS data.*

**Worked examples** (all unit-tested; the full 104-code × 2-polarity matrix round-trips):

| Code | Digits | Polarity | High byte | Low byte | Stored `[low, high]` |
|------|--------|----------|-----------|----------|----------------------|
| D023N | 0,2,3 | Normal | `0x80 \| 0x0` = `0x80` | `(2<<4)\|3` = `0x23` | `23 80` |
| D023I | 0,2,3 | Inverted | `0xC0 \| 0x0` = `0xC0` | `0x23` | `23 C0` |
| D754N | 7,5,4 | Normal | `0x80 \| 0x7` = `0x87` | `(5<<4)\|4` = `0x54` | `54 87` |
| **D754I** | 7,5,4 | Inverted | `0xC0 \| 0x7` = `0xC7` | `0x54` | **`54 C7`** |

**Terminology note**: the OEM CPS labels inverted polarity `I`. The neonplug reference
implementation's data model calls the same thing `'P'`. They are the same bit; only the label
differs.

### None / Off — read and write are asymmetric

| Direction | Behaviour | Confidence |
|-----------|-----------|------------|
| Decode | `FF FF` → None | CONFIRMED |
| Decode | `00 00` → None (falls into the CTCSS branch, computes 0.0 Hz, which is treated as None) | CONFIRMED |
| Decode | fewer than 2 bytes available → None | CONFIRMED |
| **Encode** | writers in the wild emit `00 00`; the radio itself stores `FF FF` | CONFIRMED |

A decoder **must** accept both `FF FF` and `00 00` as "no tone". `FF FF` is what the radio itself
stores (the read capture has `FF FF FF FF` at `0x21`–`0x24` on every tone-less channel); `00 00`
also occurs in the wild. Both spellings are valid.

⚠️ DERIVED: the claim that the OEM CPS CSV export spells this state `00.0` has no source in the
reference implementation or in either capture; treat it as folklore until someone re-checks a CPS
export.

⚠️ DERIVED: because encode collapses None to `00 00`, a channel read from a radio that stored
`FF FF` will not byte-match after a write-back. The field's *meaning* round-trips; its *bytes* do
not.

### Decoding

- If both bytes are `0xFF`: None/Off
- If high byte >= `0x80`: decode as DCS (>= `0xC0` means inverted)
- Otherwise decode as CTCSS; if the resulting frequency is 0.0, report None/Off

### Previously documented incorrectly

Kept here so readers who cached an older revision can spot the difference.

| Item | Previously documented as | Correct | Effect of the old rule |
|------|--------------------------|---------|------------------------|
| CTCSS low byte | `low = (tenths << 4) \| (integer % 10)` | `low = (ones << 4) \| tenths` | **nibbles transposed** — 127.3 Hz encoded as `0x37`/`0x27` instead of `0x73` |
| CTCSS high byte | `high = integer / 10` (plain binary) | `high = (hundreds << 4) \| tens` (BCD) | 127.3 Hz encoded high byte `0x0C` instead of `0x12` |
| CTCSS decode | `hz = (high × 10 + (low & 0x0F)) + ((low >> 4) / 10)` | see nibble table above | `73 12` decoded as 183.7 Hz instead of 127.3 Hz |
| None/Off | "Both bytes = `0xFF`" only | `FF FF` **and** `00 00`; encoders emit `00 00` | `00 00` misread as a 0.0 Hz tone |
| DCS digits | described as hex parsing of the code string | BCD digit nibbles | *no functional difference* — every DCS digit is octal (0–7), so hex-parsing a DCS code string coincidentally produces the same nibbles. The old **DCS** code was therefore correct by accident; the description was not |

The DCS-by-accident case is exactly why this class of bug survives: decode and encode were mutually
consistent, so round-trip tests passed while the bytes on the radio were wrong. The current test
suite asserts **absolute byte values**, not just round-trips.

### Python Implementation

Verified against the reference implementation's unit-test vectors (four CTCSS tones and all 104
DCS codes in both polarities).

```python
def decode_ctcss_dcs(data: bytes) -> tuple:
    """Decode CTCSS/DCS from 2 bytes

    Returns: (ctcss_hz, dcs_code) where at most one is not None.
             (None, None) means "no tone".
    """
    if len(data) < 2:
        return (None, None)

    low_byte = data[0]
    high_byte = data[1]

    # Sentinel: unprogrammed field
    if low_byte == 0xFF and high_byte == 0xFF:
        return (None, None)

    # DCS (high byte >= 0x80)
    if high_byte >= 0x80:
        is_inverted = high_byte >= 0xC0
        hundreds = high_byte & 0x0F
        tens = (low_byte >> 4) & 0x0F
        ones = low_byte & 0x0F
        code = hundreds * 100 + tens * 10 + ones
        # Display form DXXXN / DXXXI
        dcs_code = f"D{code:03d}{'I' if is_inverted else 'N'}"
        return (None, dcs_code)

    # CTCSS (high byte < 0x80) — four BCD digits, MSD in the high byte
    hundreds = (high_byte >> 4) & 0x0F
    tens = high_byte & 0x0F
    ones = (low_byte >> 4) & 0x0F
    tenths = low_byte & 0x0F

    ctcss_hz = (hundreds * 100 + tens * 10 + ones) + (tenths / 10.0)

    # 0.0 Hz is the other "no tone" spelling (typically bytes 00 00)
    if ctcss_hz == 0.0:
        return (None, None)

    return (ctcss_hz, None)

def encode_ctcss(tone_hz: float) -> bytes:
    """Encode CTCSS tone as 2 bytes. 127.3 -> b'\\x73\\x12'"""
    # Scale to tenths and round; float subtraction alone loses the tenths digit
    # (e.g. 127.3 - 127 == 0.2999... -> int() would yield 2, not 3).
    scaled = int(round(tone_hz * 10))
    hundreds = (scaled // 1000) % 10
    tens = (scaled // 100) % 10
    ones = (scaled // 10) % 10
    tenths = scaled % 10

    low_byte = (ones << 4) | tenths
    high_byte = (hundreds << 4) | tens

    return bytes([low_byte, high_byte])

def encode_dcs(dcs_code: str) -> bytes:
    """Encode DCS code as 2 bytes (e.g., 'D023N' -> b'\\x23\\x80')"""
    is_inverted = dcs_code[4] == 'I'
    # DCS digits are BCD, not hex. For canonical octal DCS codes the two happen
    # to agree, but parse as decimal digits so an out-of-range code fails loudly.
    code = int(dcs_code[1:4], 10)

    hundreds = (code // 100) % 10
    tens = (code // 10) % 10
    ones = code % 10

    low_byte = (tens << 4) | ones
    high_byte = (0xC0 if is_inverted else 0x80) | hundreds

    return bytes([low_byte, high_byte])

def encode_none() -> bytes:
    """Encode "no tone". The reference implementation always writes 00 00,
    never FF FF, even though decoders must accept both."""
    return bytes([0x00, 0x00])
```

### C Implementation

```c
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <math.h>

/**
 * Encode CTCSS tone as 2 bytes
 * @param tone_hz CTCSS tone in Hz (e.g., 127.3)
 * @param out Output buffer (2 bytes)
 *
 * 127.3 Hz -> out = { 0x73, 0x12 }
 */
void encode_ctcss(double tone_hz, uint8_t *out) {
    // Scale to tenths and round. Never *truncate* the tenths digit out of a
    // subtraction: (int)((127.3 - 127) * 10) == 2 in IEEE-754, not 3.
    // (The reference implementation rounds that subtraction, which is equivalent.)
    int scaled   = (int)lround(tone_hz * 10.0);
    int hundreds = (scaled / 1000) % 10;
    int tens     = (scaled / 100)  % 10;
    int ones     = (scaled / 10)   % 10;
    int tenths   =  scaled         % 10;

    out[0] = (uint8_t)((ones << 4) | tenths);       /* low  byte */
    out[1] = (uint8_t)((hundreds << 4) | tens);     /* high byte */
}

/**
 * Encode DCS code as 2 bytes
 * @param dcs_code DCS code string (e.g., "D023N" or "D754I")
 * @param out Output buffer (2 bytes)
 *
 * "D023N" -> { 0x23, 0x80 }    "D754I" -> { 0x54, 0xC7 }
 */
void encode_dcs(const char *dcs_code, uint8_t *out) {
    // Parse "DXXXN" or "DXXXI"
    uint8_t is_inverted = (dcs_code[4] == 'I');

    // The three digits are BCD. Parsing them as base 10 and repacking as BCD is
    // equivalent to a base-16 parse for canonical (octal-digit) DCS codes, but
    // base 10 is the rule that is actually true.
    char code_str[4];
    code_str[0] = dcs_code[1];
    code_str[1] = dcs_code[2];
    code_str[2] = dcs_code[3];
    code_str[3] = '\0';

    int code     = (int)strtol(code_str, NULL, 10);
    int hundreds = (code / 100) % 10;
    int tens     = (code / 10)  % 10;
    int ones     =  code        % 10;

    out[0] = (uint8_t)((tens << 4) | ones);
    out[1] = (uint8_t)((is_inverted ? 0xC0 : 0x80) | hundreds);
}

/**
 * Encode "no tone". Always 00 00 - the reference implementation never emits FF FF.
 */
void encode_tone_none(uint8_t *out) {
    out[0] = 0x00;
    out[1] = 0x00;
}

/**
 * Decode CTCSS/DCS from 2 bytes
 * @param data Input data (2 bytes)
 * @param tone_hz Output CTCSS tone (set to 0 if DCS or none)
 * @param dcs_code Output DCS code buffer (>= 6 bytes, set to empty if CTCSS or none)
 * @return 0 for none, 1 for CTCSS, 2 for DCS
 */
int decode_ctcss_dcs(const uint8_t *data, double *tone_hz, char *dcs_code) {
    uint8_t low  = data[0];
    uint8_t high = data[1];

    *tone_hz = 0;
    dcs_code[0] = '\0';

    // Sentinel: unprogrammed field
    if (low == 0xFF && high == 0xFF) {
        return 0;
    }

    // DCS
    if (high >= 0x80) {
        uint8_t is_inverted = (high >= 0xC0);
        int hundreds = high & 0x0F;
        int tens     = (low >> 4) & 0x0F;
        int ones     = low & 0x0F;
        int code     = hundreds * 100 + tens * 10 + ones;

        sprintf(dcs_code, "D%03d%c", code, is_inverted ? 'I' : 'N');
        return 2;
    }

    // CTCSS - four BCD digits, most significant pair in the high byte
    int hundreds = (high >> 4) & 0x0F;
    int tens     =  high & 0x0F;
    int ones     = (low >> 4) & 0x0F;
    int tenths   =  low & 0x0F;

    double hz = (hundreds * 100 + tens * 10 + ones) + (tenths / 10.0);

    // 0.0 Hz is the other spelling of "no tone" (bytes 00 00)
    if (hz == 0.0) {
        return 0;
    }

    *tone_hz = hz;
    return 1;
}
```

---

## String and Name Encoding

### Character set

All text in the DM-32UV codeplug is **single-byte ASCII**, with **one exception**: the entry names
in block `0x03` (the unidentified "Call" list) are **UTF-16LE** — hardware-confirmed
(`43 00 61 00 6c 00 6c 00 20 00 31 00` = `"Call 1"`). See
[05-DATA-STRUCTURES.md § 0x03](05-DATA-STRUCTURES.md#0x03---call-list--layout-confirmed-purpose--unknown).

> Every *other* record type — channels, zones, scan lists, talk groups, contacts, messages,
> settings — uses plain byte-per-character ASCII. Name bytes observed in the captures
> (`"Channel 1"`, `"Zone 1"`, `"DEmer 1"`, `"Radio 1"`, `"How are you?"`) have no interleaved
> null bytes.

⚠️ **Encoder hazard — non-ASCII input overflows fixed fields.** Limiting a name by *character*
count while encoding it as UTF-8 produces more bytes than characters for non-ASCII input; in the
channel record the overflow runs past the 16-byte name field into the RX frequency BCD.
**Restrict names to ASCII `0x20`–`0x7E` and limit by byte count before encoding.**

### Decoding rule

For every string field:

1. Take the fixed-size field.
2. Find the first `0x00`. If present, the string is everything before it; if absent, the string is
   the whole field.
3. Strip any remaining `0x00` bytes, then trim leading/trailing whitespace.

Three record types deviate:

| Record | Deviation |
|--------|-----------|
| Talk Group (block `0x44`) name | stops at the first `0x00` **or** the first `0xFF` |
| DMR contact database callsign, city, province, country, remark | leading `0xFF` bytes are skipped before decoding; if no `0x00` is found, `0xFF` and `0x00` bytes are filtered out of the remainder |
| Quick message text (block `0x0A`) | terminated by **`0xFF`**, not `0x00`; embedded `0x00` padding is stripped |

If the decoded name is empty, the reference implementation substitutes a synthetic label
(`Channel N`, `Scan List N`, `RX Group N`, `DEmer N`, `ID <dmrid>`). That is a **UI convenience, not
stored data** — an empty name field on the radio stays empty. Zones are the exception: a zone whose
name decodes empty is **skipped entirely** rather than relabelled (and if the whole record is `0xFF`
or `0x00`, zone parsing stops there). *Previously documented as substituting `Zone N` — there is no
such fallback.*

### Field sizes and maximum lengths

The "max chars written", "terminator" and "pad byte" columns describe the **reference
implementation's encoder**, whose limit is usually one **less** than the field size because a
terminator is reserved. Getting this wrong is the most common source of names that look truncated
on the radio. Offsets and field sizes are structural; the encoder columns are not necessarily what
the radio itself writes — see the hardware notes under the table.

| Record | Field | Offset | Field size | Max chars written | Terminator | Pad byte | Confidence |
|--------|-------|--------|-----------|-------------------|------------|----------|------------|
| Channel (48 B) | name | `0x00` | 16 | **16** | `0x00` **only if** length < 16 | `0xFF` (radio writes `0x00` — see note) | CONFIRMED |
| Zone (145 B) | name | `0x00` | 11 | **10** | `0x00` always | `0xFF` | CONFIRMED (hardware: `"Zone 1" 00 FF FF FF FF`) |
| Scan list (57 B) | name | `0x00` | 11 | **10** (encoder) — the **radio writes 11 with no terminator** | `0x00` always (encoder only) | `0x00` | ⚠️ see note |
| DMR contact (92 B) | name | `0x00` | 16 | **15** | `0x00` always | `0xFF` | CONFIRMED |
| DMR contact | callsign | `0x14` | 8 | **7** | `0x00` always | `0xFF` | CONFIRMED |
| DMR contact | city | `0x1C` | 16 | **15** | `0x00` always | `0xFF` | CONFIRMED |
| DMR contact | province/state | `0x2C` | 16 | **15** | `0x00` always | `0xFF` | CONFIRMED |
| DMR contact | country | `0x3C` | 16 | **15** | `0x00` always | `0xFF` | CONFIRMED |
| DMR contact | remark | `0x4C` | 16 | **15** | `0x00` always | `0xFF` | CONFIRMED |
| Talk group (block `0x44`) | name | `+0x01` (entry 2+; entry 1 is at block offset `0x002` because it carries an extra header byte) | 16 | **16** | separate `0x00` byte at `+0x11`, *outside* the 16-byte field | `0x00` | CONFIRMED (hardware: `"Contacts 1"`@`0x002`, `"Contacts 2"`@`0x01A`, stride 24) |
| RX group (block `0x0F`, 109 B) | name | `0x00` (entries start at block offset `0x11`) | 11 | **10** | `0x00` always | `0x00` | CONFIRMED (hardware: `"RX Group 1"`@`0x011`) |
| DMR Radio ID (block `0x67`, 16 B) | name | `+0x03` | 12 | **11** | `0x00` always | `0x00` | CONFIRMED (hardware: `"Radio 1"`@`0x013`, entries from `0x010`, stride 16) |
| Quick message (block `0x0A`, 129 B) | text | `+0x01` | 128 | **127 effective** (see note) | `0xFF` | `0xFF` (radio writes `0x00`) | CONFIRMED |
| Radio settings (block `0x04`) | power-on display line 1 | `0x01` | 14 | **13** | `0x00` always | `0x00` | CONFIRMED (hardware: `"Welcome"`@`0x001`) |
| Radio settings | power-on display line 2 | `0x0F` | 14 | **13** | `0x00` always | `0x00` | CONFIRMED (hardware: `"DM-32UV"`@`0x00F`) |
| Radio settings | latitude | `0x306` | **9** (read path) / 14 (encoder — overruns) | **13** | `0x00` always | `0x00` | ⚠️ DERIVED — contradictory, see note |
| Radio settings | longitude | `0x310` | **9** (read path) / 14 (encoder — overruns) | **13** | `0x00` always | `0x00` | ⚠️ DERIVED — contradictory, see note |
| Encryption key (block `0x10`) | name | `+0x01` | 10 | **10** | none — zero-padded to 10 | `0x00` | ⚠️ DERIVED |
| Digital emergency (block `0x10`, 20 B) | name | `+0x00` | 10 | **10** | none — zero-padded to 10 | `0x00` | see note |
| Analog emergency (block `0x10`, 36 B) | name | `+0x00` | 17 | **16** | `0x00` always | `0x00` | ⚠️ DERIVED |

**Digital emergency note**: the 8 × 20-byte layout with the name at `+0x00` is **hardware
CONFIRMED** — the OEM CPS **read** capture page tagged `0x10` (at `0x00C000`) contains
`"DEmer 1"` … `"DEmer 8"` at a 20-byte stride, with the region after `0x0A0` (= 8 × 20) all zero:

```
0x000  44 45 6d 65 72 20 31 00 00 00 01 03 00 00 05 05   |DEmer 1.........|
0x010  00 00 01 00 44 45 6d 65 72 20 32 00 00 00 02 03   |....DEmer 2.....|
```

*Previously documented as coming from the **write** capture — incorrect: the write capture's `0x10`
page has only entry 1 populated and `0xFF` from `0x014` onward, so it does not establish the
stride.* Separately, the reference implementation's *parser* for this block is a stub that returns
an empty list; only the encoder is written. Layout confirmed, parser not implemented — both facts
are true.

**Scan-list name note**: hardware disagrees with the encoder. The read capture's scan-list block
(tag `0x11`) stores `53 63 61 6e 20 4c 69 73 74 20 31` = `"Scan List 1"` — **11 characters filling
the whole field with no null terminator**, immediately followed by the channel count at `+0x0B`.
The reference implementation writes at most 10 characters plus a `0x00`, and its *parser* requires a
`0x00` (a name with no terminator decodes as empty and is replaced by the synthetic label
`Scan List N`). Implementers should write ≤10 + terminator for CPS compatibility but **must** accept
a full 11-byte unterminated name on read.

**Quick-message note**: the encoder writes up to 128 text bytes, but when the text is exactly 128
bytes it overwrites byte 127 with the `0xFF` terminator — so at most **127** characters survive a
round-trip. The radio's own blocks pad with `0x00` after the text (read capture: status byte
`0x0C` = 12 = length of `"How are you?"`, then zeros), while the reference implementation pads with
`0xFF`; the parser accepts both.

**Latitude/longitude note**: the reference implementation's read path parses **9** bytes at `0x306`
and `0x310`, which is the only self-consistent reading — the latitude direction byte sits at `0x30F`
(= `0x306` + 9) and the longitude field starts at `0x310`. Its write path nevertheless emits **14**
bytes at each offset, which overruns `0x30F` and runs latitude into the longitude field. One of the
two is wrong and there is no capture evidence for either; treat the geometry as ⚠️ DERIVED and do
not copy the 14-byte write.

### The 16-character channel name has no terminator

**Channel-name padding differs between the radio and the reference implementation.** The radio pads
the 16-byte name field with `0x00`: the read capture's first channel record is
`43 68 61 6e 6e 65 6c 20 31 00 00 00 00 00 00 00` (`"Channel 1"` + seven `0x00`). The reference
implementation `0xFF`-fills the whole 48-byte record first and writes a single `0x00` terminator, so
its padding is `0xFF`. Both decode identically under the rule above; do not treat either pad byte as
a structural requirement.

The channel encoder writes the null terminator **only when the name is shorter than 16 bytes**. A
name that exactly fills the field is stored with no terminator, and decoders must fall back to
"consume the whole field". Any decoder that requires a terminator will read the RX frequency BCD
bytes as text.

### ASCII used as an enum

A few fields store an ASCII **letter** rather than a numeric code:

| Field | Offset (block `0x04`) | Values | Confidence |
|-------|----------------------|--------|------------|
| Latitude direction | `0x30F` | `0x4E` = `'N'`, `0x53` = `'S'` | ⚠️ DERIVED |
| Longitude direction | `0x319` | `0x45` = `'E'`, `0x57` = `'W'` | ⚠️ DERIVED |

---

## Integer Encoding and Endianness

Plain integers are **binary little-endian**, never BCD. Only frequencies and CTCSS/DCS use BCD.

```
uint16 LE:  value = b[0] | (b[1] << 8)
uint24 LE:  value = b[0] | (b[1] << 8) | (b[2] << 16)
uint32 LE:  value = b[0] | (b[1] << 8) | (b[2] << 16) | (b[3] << 24)
```

### Worked examples

| Value | Width | Bytes | Where | Confidence |
|-------|-------|-------|-------|------------|
| 25 | uint16 LE | `19 00` | channel count, block `0x12` offset `0x000` | **hardware** (read capture) |
| 128 | uint16 LE | `80 00` (then `ff ff` fill) | channel count, block `0x12` offset `0x000` | **hardware** (write capture) |
| 50 000 | 3-byte LE | `50 C3 00` | V-frame `0x10` reply payload — max contact count (the reference implementation reads this field as a uint32 LE) | **hardware** (both captures) |
| 3 023 401 | uint24 LE | `29 22 2E` | Talk Group contact number, block `0x44` entry `+0x12` | CONFIRMED |
| 1 | uint24 LE | `01 00 00` | Talk Group contact number | CONFIRMED |
| 0x0C8FFF | 3-byte LE address | `FF 8F 0C` | wire address field in a `0x52`/`0x57` frame | **hardware** |
| 0x1000 (4096) | uint16 LE | `00 10` | wire length field of every block write | **hardware** |

### Where each width is used

| Width | Field | Location |
|-------|-------|----------|
| uint8 | Zone channel count | zone record `+0x10` |
| uint8 | Zone count (**global** total, not per-block) | first zone block, byte `0x00` (hardware: `02`, and exactly two zones — `"Zone 1"`, `"Func Demo"` — are present) |
| uint8 | Scan list count | block `0x11`, byte `0x00` |
| uint8 | Scan list channel count | scan-list entry `+0x0B` |
| uint8 | Quick message count | block `0x0A`, byte `0x00` |
| uint8 | Quick message character count (status byte) | quick-message entry `+0x00` |
| uint8 | DMR Radio ID count | block `0x67`, byte `0x00`. ⚠️ Some in-tree constants call this a 4-byte DWORD; the working read **and** write paths both use 1 byte. Max is 250, so one byte suffices either way |
| uint8 | ❓ UNKNOWN count | block `0x0B`, byte `0x04`. One byte (the `0xFF` at `0x05` rules out a 16-bit field). The reference implementation writes the *Private Call* count here, but the factory image contradicts that: it reads `1` on a radio with 4 Private Call talk groups and 1 All Call — the byte matches the **All Call** count. Treat as unknown; see [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md#0x0b---quick-access-contact-list-talk-group-index-tables) |
| uint8 | Talk group counter | block `0x06`, offset `0x1FF`. ⚠️ DERIVED — the hardware read capture shows `7` here alongside exactly 7 analog/DTMF contact names at `0x200`, so this byte may be the **analog contact count**, not the talk-group count. Written as `count & 0xFF`, so any value ≥ 256 silently truncates |
| uint16 LE | Zone channel list entries (64 × uint16) | zone record `+0x11` |
| uint16 LE | Scan list channel list entries — **15 × uint16 at `+0x1A`**. The word at `+0x18`–`+0x19` is a separate ❓ UNKNOWN field, not the first channel slot: in the OEM write capture all nine scan lists store `00 00` there followed by exactly the channels their names imply. Preserve `+0x18` verbatim. Full analysis in [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md#scan-list-structure-57-bytes) | scan-list entry `+0x1A` |
| uint16 LE | Scan list priority / designated-TX channel numbers | scan-list entry `+0x0F`, `+0x11`, `+0x13` |
| uint16 LE | Total contact count, group call count | block `0x0B`, `0x00`–`0x01` and `0x02`–`0x03` |
| uint24 LE | DMR Radio ID | block `0x67` entry `+0x00` |
| uint24 LE | Talk group contact number | block `0x44` entry `+0x12` |
| uint24 LE | RX group members (32 × uint24). ⚠️ DERIVED — the reference implementation calls the array `talkGroupIndices` but comments each element "DMR ID (contactNumber)". Hardware shows `01 00 00 / 02 00 00 / 03 00 00 …`, and on this factory radio the talk groups have DMR IDs 1, 2, 3 …, so the capture cannot distinguish "DMR ID" from "1-based talk-group slot index" | block `0x0F` entry `+0x0B` |
| uint16 LE | Channel count | block `0x12`, `0x000`–`0x001` — **2 bytes**. The write capture settles it: `80 00 ff ff` = 128, matching the 128 channel records it then writes; a uint32 read would give 4,294,901,888. Bytes `0x002`–`0x00F` are fill (`0xFF` from the OEM CPS, `0x00` in the factory image). *Previously documented as uint32 — incorrect.* |
| uint32 LE | Active-RX-group bitmask | block `0x0F`, `0x00`–`0x03` |
| uint32 LE | DMR contact database count | contact region base `+0x00` |
| uint24 LE | Contact DMR ID | contact entry `+0x10`. ⚠️ The reference implementation reads 4 bytes here, but the only hardware entry is `01 00 00 f0` — ID 1 plus an unexplained `0xF0` at `+0x13`, which a uint32 reading turns into the out-of-range ID 4,026,531,841. Read 3 bytes; preserve `+0x13` |
| uint32 LE ×2 | V-frame memory-range replies (start, end) | V-frame payload |
| 3-byte LE | Wire address | `0x52` read and `0x57` write frame headers |
| uint16 LE | Wire length | `0x52` read and `0x57` write frame headers |

**No signed integers, no big-endian fields, and no BCD integers have been observed anywhere in the
codeplug.**

### Bitmask sense is inverted in one place

Block `0x0B`'s 16-byte slot-usage bitmask at `0x10`–`0x1F` uses **`0` = slot used, `1` = slot
free** — the opposite of the intuitive reading. The block is initialised to all-`0xFF` and bits are
*cleared* as slots are occupied. Hardware agrees: the read capture's `0x0B` block has total-contact
count `0A 00` (10) at `0x00` and bitmask bytes `00 FC FF FF …` at `0x10` — eight cleared bits in the
first byte plus two in the second, exactly the 10 occupied slots. The `0x0F` RX-group bitmask at
`0x00`–`0x03` uses the normal sense (`1` = active).

---

## Nibble and Bit Packing

Beyond the BCD codecs, several fields pack two values into one byte or split one value across two.
Channel and settings bit-field maps live in
[05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md); this section covers the packings that are *codec*
rules rather than record layout.

### 12-bit talk-group index split across 2 bytes (blocks `0x42` / `0x43`)

This is the only field in the codeplug that is neither byte-aligned nor little-endian in the usual
sense — the high nibble of byte 0 carries the **high** bits.

```
byte0:  bits 7-4 = index bits 11-8
        bits 3-1 = ❓ UNKNOWN — carry data on hardware (OEM CPS writes 111); preserve
        bit  0   = digital flag (1 = digital, 0 = analog/mixed)
byte1:  bits 7-0 = index bits 7-0
```

```
decode:  index = ((byte0 >> 4) & 0x0F) << 8 | byte1        # 0-4095, 0 = None
         digital = (byte0 & 0x01) != 0
encode:  byte0 = ((index >> 8) & 0x0F) << 4 | (digital ? 0x01 : 0x00)
         byte1 = index & 0xFF
```

**Worked example**: talk group index 300 (`0x12C`), digital →
`byte0 = (0x1 << 4) | 0x01 = 0x11`, `byte1 = 0x2C` → stored `11 2C`.
Decoding `11 2C`: `(0x11 >> 4) << 8 | 0x2C = 0x100 | 0x2C = 300`, digital because bit 0 is set.

### Two 4-bit values in one byte

| Byte | Low nibble (bits 3–0) | High nibble (bits 7–4) | Confidence |
|------|----------------------|------------------------|------------|
| Scan-list entry `+0x0E` | priority 1 type | priority 2 type | CONFIRMED |
| Channel `0x27` | signaling type | step frequency | CONFIRMED |
| Channel `0x29` | ❓ UNKNOWN — two 2-bit sub-fields | PTT ID type | CONFIRMED |

### Two 2-bit values in one byte

| Byte | Bits 1–0 | Bits 3–2 | Confidence |
|------|----------|----------|------------|
| Scan-list entry `+0x0C` | CTC scan mode | scan TX mode | CONFIRMED |

### Low-nibble-only fields

The six display colour settings in block `0x04` (`0x34`, `0x35`, `0x38`, `0x39`, `0x3A`, `0x3B`)
store the colour index in the **low nibble only**; the high nibble is ❓ UNKNOWN. It is masked off on
read (`raw & 0x0F`) and **preserved** on write (`(raw & 0xF0) | value`), so whatever it means
survives a round-trip — do not zero it.

---

## Stored vs Displayed Value Conventions

Several fields do not store the value a user sees. Getting the direction of the adjustment backwards
produces settings that are off by one step — the single most common class of CPS-compatibility bug.

### Numeric offsets

| Field | Location | Stored | Displayed | Decode | Encode | Confidence |
|-------|----------|--------|-----------|--------|--------|------------|
| Backlight brightness | settings `0x30` | 0–5 | 1–6 | `raw + 1` | `value - 1` | CONFIRMED |
| Auto backlight duration | settings `0x31` | 0–5 | 5–30 s (step 5) | `(raw + 1) × 5` | `floor(sec / 5) - 1` | CONFIRMED |
| Long press time | settings `0x93` | 0–4 | 1–5 | `raw + 1` | `value - 1` | CONFIRMED |
| Active wait time | settings `0x62` | raw byte; a source comment says `raw = combo_idx + 1` | ms | `raw` (no conversion) | `value & 0xFF` (no conversion) | ⚠️ DERIVED |
| Pre-carrier time | settings `0x64` | raw byte; a source comment says `raw = combo_idx` | ms | `raw` (no conversion) | `value & 0xFF` (no conversion) | ⚠️ DERIVED |
| Scan list — designated TX channel | scan-list entry `+0x11` | channel − 2 | channel | `raw + 2` | `max(0, ch - 2)` | ⚠️ DERIVED |
| Scan list — priority channel 2 | scan-list entry `+0x13` | channel − 2 | channel | `raw + 2` | `max(0, ch - 2)` | ⚠️ DERIVED |
| Scan list — priority channel 1 | scan-list entry `+0x0F` | channel, **no offset** | channel | `raw` | `ch` | ⚠️ DERIVED |

⚠️ The scan-list asymmetry is real and unexplained: priority channel **1** is stored directly while
priority channel **2** and the designated TX channel are stored with a **−2** bias. Both directions
of the reference implementation agree, so a codeplug written by it round-trips, but the −2 bias has
no hardware capture behind it and no explanation in the source. Treat it as DERIVED and verify
before relying on it.

**Fields with no adjustment** (raw byte *is* the value — listed because they sit next to fields
that do have one): GPS report interval (settings `0x42`, 5–255 s), call hold time (`0x61`, 0–61 s),
active retries (`0x63`, count 1–8), TX dwell time (`0x81`), auto keypad lock delay (`0x86`,
seconds), menu exit time (`0x36`, 1–30), UTC zone (`0x41`, 0–25).

### Index base conventions

Mixing 0-based and 1-based indices between record types is the second most common bug.

| Reference | Base | "None" value | Confidence |
|-----------|------|--------------|------------|
| Channel numbers in zone channel lists | **1-based**, stored as-is, no conversion | the count at `+0x10` governs; hardware has `0x0000` after the last entry, but see the note below | CONFIRMED (hardware) |
| Channel numbers in scan-list channel lists | **1-based**, stored as-is | `0x0000` (factory) or `0xFFFF` (OEM CPS) after the last entry | CONFIRMED (hardware; the write capture's lists resolve to their named channels) |
| TX contact → talk group (blocks `0x42`/`0x43`) | **1-based** physical slot in block `0x44` | `0` | CONFIRMED |
| DMR Radio ID index (channel byte `0x2B`) | **contested** — the reference implementation treats it as 0-based, but the OEM write capture (113 channels storing `0x01` against a one-entry Radio ID list) implies 1-based. Preserve the byte; see [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md) | `0xFF` | ⚠️ DERIVED |
| Scan list ID (channel `0x19` bits 5–2) | `0` = None, so effectively 1-based | `0` | CONFIRMED |
| RX group list ID (channel `0x1F` bits 5–0, digital) | `0` = None, so effectively 1-based | `0` | CONFIRMED |
| Quick message entry number | **1-based** in the offset formula; the reference implementation's UI model is 0-based | — | CONFIRMED |
| Encryption key ID (channel `0x2A`, digital) | 1–8 | `0` | ⚠️ DERIVED |

**Synthetic identifiers that are NOT stored on the radio**: `Zone.id` and `Contact.id` in the
reference implementation's data model are generated in software for UI bookkeeping and have no
on-radio representation. Do not look for them in the byte layout.

### Zone channel lists have no terminator inside the record

The zone record stores its channel count at `+0x10` and the radio uses that count to decide how many
`uint16` entries to read. **Hardware read capture** (zone block, tag `0x5C`): zone 1 has
`+0x10 = 0x10` (16) followed by channels 1–16 and then `00` bytes to the end of the record; zone 2
has `+0x10 = 0x09` followed by channels `0x11`–`0x19` and then `00` bytes. An entirely unused zone
slot is all-`0xFF`.

*Previously documented as "unused entry slots are padded with `0xFF`, not `0x0000` — hardware
observed" — that is **the reference implementation's** rule, and the radio's own records contradict
it: the radio pads inside a populated record with `0x00`.* The reference implementation deliberately
pads with `0xFF` and its source records a real regression ("writing `0x0000` caused the radio to
show null slots / lose channels"). Both statements are recorded because both are sourced; the
conflict is ⚠️ **unexplained**. The safe rule that satisfies both is: **set the count at `+0x10`
correctly and do not rely on any padding value** to delimit the list.

The reference implementation also writes a `0x0000` after the last zone *record* in the block, as a
record-list terminator. ⚠️ DERIVED — the hardware capture shows the slot after the last zone is
all-`0xFF`, with no `0x0000` terminator, so this appears to be an implementation convention rather
than a radio requirement.

---

## Fill-byte Conventions

Which byte means "empty" is **not** uniform across record types. Using the wrong one can make the
radio treat a populated record as blank.

**This table is what the reference implementation writes.** The radio's own fill differs in places
(it pads channel names and quick-message text with `0x00`, and leaves `0xFF` inside talk-group name
fields), so a mismatch here is not by itself a bug — but it does mean a re-write never byte-matches.

| Record / block | Fill byte (reference implementation) | Confidence |
|----------------|-----------|------------|
| Channel record (48 B) | `0xFF` | CONFIRMED (radio pads names with `0x00`) |
| Zone record (145 B) | `0xFF` | CONFIRMED (hardware agrees) |
| DMR contact entry (92 B) | `0xFF` | CONFIRMED |
| Quick message text | `0xFF` | CONFIRMED (radio pads with `0x00`) |
| Channel block, whole 4 KB on write | `0xFF` | CONFIRMED |
| Scan-list entry (57 B) | **`0x00`** | CONFIRMED |
| Talk group block `0x44`, whole 4 KB | **`0x00`** | CONFIRMED (hardware shows `0xFF` padding inside entry-2+ name fields) |
| RX group entry (109 B) | **`0x00`** | CONFIRMED |
| RX group *unused trailing* slots | `0xFF` | CONFIRMED |
| DMR Radio ID entry (16 B) and block `0x67` | **`0x00`** | CONFIRMED (hardware agrees) |

**Empty-record detection on read** correspondingly accepts either: a channel slot is empty when
every byte is `0xFF` *or* `0x00`; a contact entry is empty when its first name byte is `0xFF` *or*
`0x00`; a DMR Radio ID entry is empty when all 16 bytes are `0xFF` *or* all `0x00`. Implement the
permissive test in both directions.

⚠️ DERIVED (hardware, page level, distinct from the record level above): across the 200 pages of
the `0x001000`–`0x0C8FFF` region, the page-tag byte at `+0xFFF` takes the value `0xFF` on 114
contiguous high-region pages and `0x00` on 15 pages scattered among live ones. That pattern is
consistent with `0xFF` = erased/free page and `0x00` = allocated-then-invalidated page under a
wear-levelling copy-on-write scheme. Both are "not a live page", but they are probably **not** the
same state. Not yet confirmed by experiment.

---

## Round-trip Fidelity

A record decoded into a field model and re-encoded does **not** in general byte-match what the
radio originally stored: fill bytes differ by writer (`0x00` vs `0xFF`), "None" values have two
spellings (`FF FF` vs `00 00`), and every `❓ UNKNOWN` byte or bit reverts to a default unless it
is explicitly carried through. The captures prove many of those unknown positions carry real data
(see the per-record guidance in [05-DATA-STRUCTURES.md](05-DATA-STRUCTURES.md)).

**If byte-exact preservation matters, read-modify-write the raw block and patch only the fields
you intend to change, rather than decode-then-re-encode.**
