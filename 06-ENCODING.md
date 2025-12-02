# DM-32UV Encoding and Decoding

This document provides complete implementations for all encoding/decoding algorithms used in the DM-32UV protocol.

## BCD Frequency Encoding

### Format

Frequencies are stored as **BCD (Binary Coded Decimal)** in 4 bytes with **little-endian** byte order.

### Algorithm

```
Frequency: 145.350 MHz
Step 1: Split into digits: 1 4 5 . 3 5 0 0 0
Step 2: Group as BCD nibbles: 14 53 50 00
Step 3: Store little-endian: 00 50 53 14
```

### Algorithm

**Encoding Steps:**
1. Convert frequency to 8-digit integer (drop decimal point)
   - Example: 145.350 MHz → 14535000
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

### C/C++ Implementation

```c
#include <stdint.h>
#include <string.h>
#include <stdio.h>

/**
 * Encode frequency (MHz) as 4-byte BCD little-endian
 * 
 * @param freq_mhz Frequency in MHz (e.g., 145.350)
 * @param out Output buffer (must be 4 bytes)
 */
void encode_bcd_frequency(double freq_mhz, uint8_t *out) {
    // Convert to 8-digit integer (145.35000 MHz → 14535000)
    uint32_t freq_int = (uint32_t)(freq_mhz * 100000.0);
    
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

### Format

CTCSS tones and DCS codes are stored in 2 bytes:
- **CTCSS**: High byte < 0x80, format: `<tone_low> <tone_high>`
- **DCS Normal**: High byte >= 0x80 and < 0xC0
- **DCS Inverted**: High byte >= 0xC0
- **None/Off**: Both bytes = 0xFF

### Algorithm

**CTCSS Encoding:**
- High byte < 0x80
- Format: `XXX.Y Hz` → `low_byte = (Y << 4) | (X % 10)`, `high_byte = XX`
- Example: 127.3 Hz → `0x73 0x12`

**DCS Encoding:**
- High byte >= 0x80
- Format: `DXXXN` or `DXXXI` → `low_byte = XX`, `high_byte = 0x80/0xC0 | X`
- Normal: High byte = 0x80-0xBF
- Inverted: High byte = 0xC0-0xFF
- Example: D023N → `0x23 0x80`

**Decoding:**
- If high byte < 0x80: Decode as CTCSS
- If high byte >= 0x80: Decode as DCS (check if >= 0xC0 for inverted)
- If both bytes = 0xFF: None/Off

### Python Implementation

```python
def decode_ctcss_dcs(data: bytes) -> tuple:
    """Decode CTCSS/DCS from 2 bytes
    
    Returns: (ctcss_hz, dcs_code) where one is None
    """
    low_byte = data[0]
    high_byte = data[1]
    
    # Check for None/Off
    if low_byte == 0xFF and high_byte == 0xFF:
        return (None, None)
    
    # Check for DCS (high byte >= 0x80)
    if high_byte >= 0x80:
        is_inverted = high_byte >= 0xC0
        high_nibble = high_byte & 0x0F
        
        # Build DCS code: DXXXN/I
        dcs_code = f"D{high_nibble:01X}{low_byte:02X}{'I' if is_inverted else 'N'}"
        return (None, dcs_code)
    
    # CTCSS (high byte < 0x80)
    decimal_part = (low_byte >> 4) & 0x0F
    low_digit = low_byte & 0x0F
    high_digit = high_byte
    
    ctcss_hz = float(high_digit * 10 + low_digit) + (decimal_part / 10.0)
    return (ctcss_hz, None)

def encode_ctcss(tone_hz: float) -> bytes:
    """Encode CTCSS tone as 2 bytes"""
    integer_part = int(tone_hz)
    decimal_part = int((tone_hz - integer_part) * 10)
    
    low_byte = (decimal_part << 4) | (integer_part % 10)
    high_byte = integer_part // 10
    
    return bytes([low_byte, high_byte])

def encode_dcs(dcs_code: str) -> bytes:
    """Encode DCS code as 2 bytes (e.g., 'D023N')"""
    is_inverted = dcs_code[4] == 'I'
    code_hex = int(dcs_code[1:4], 16)
    
    low_byte = code_hex & 0xFF
    high_byte = (0xC0 if is_inverted else 0x80) | ((code_hex >> 8) & 0x0F)
    
    return bytes([low_byte, high_byte])
```

### C Implementation

```c
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

/**
 * Encode CTCSS tone as 2 bytes
 * @param tone_hz CTCSS tone in Hz (e.g., 127.3)
 * @param out Output buffer (2 bytes)
 */
void encode_ctcss(double tone_hz, uint8_t *out) {
    int integer_part = (int)tone_hz;
    int decimal_part = (int)((tone_hz - integer_part) * 10);
    
    out[0] = (decimal_part << 4) | (integer_part % 10);
    out[1] = integer_part / 10;
}

/**
 * Encode DCS code as 2 bytes
 * @param dcs_code DCS code string (e.g., "D023N")
 * @param out Output buffer (2 bytes)
 */
void encode_dcs(const char *dcs_code, uint8_t *out) {
    // Parse "DXXXN" or "DXXXI"
    uint8_t is_inverted = (dcs_code[4] == 'I');
    
    // Extract digits (023 → 0x23)
    char code_str[4];
    code_str[0] = dcs_code[1];
    code_str[1] = dcs_code[2];
    code_str[2] = dcs_code[3];
    code_str[3] = '\0';
    
    uint16_t code = (uint16_t)strtol(code_str, NULL, 16);
    
    out[0] = (uint8_t)(code & 0xFF);
    out[1] = (is_inverted ? 0xC0 : 0x80) | ((code >> 8) & 0x0F);
}

/**
 * Decode CTCSS/DCS from 2 bytes
 * @param data Input data (2 bytes)
 * @param tone_hz Output CTCSS tone (set to 0 if DCS)
 * @param dcs_code Output DCS code buffer (6 bytes, set to empty if CTCSS)
 * @return 0 for none, 1 for CTCSS, 2 for DCS
 */
int decode_ctcss_dcs(const uint8_t *data, double *tone_hz, char *dcs_code) {
    uint8_t low = data[0];
    uint8_t high = data[1];
    
    // Check for none/off
    if (low == 0xFF && high == 0xFF) {
        *tone_hz = 0;
        dcs_code[0] = '\0';
        return 0;
    }
    
    // Check for DCS
    if (high >= 0x80) {
        uint8_t is_inverted = (high >= 0xC0);
        uint8_t high_nibble = high & 0x0F;
        
        sprintf(dcs_code, "D%01X%02X%c", high_nibble, low, is_inverted ? 'I' : 'N');
        *tone_hz = 0;
        return 2;
    }
    
    // CTCSS
    uint8_t decimal = (low >> 4) & 0x0F;
    uint8_t low_digit = low & 0x0F;
    
    *tone_hz = (high * 10 + low_digit) + (decimal / 10.0);
    dcs_code[0] = '\0';
    return 1;
}
```
