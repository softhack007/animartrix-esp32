# ANIMartRIX.h Array Bounds Violation Analysis

## Executive Summary

After a thorough review of ANIMartRIX.h for potential array and vector bounds violations, **NO CRITICAL BUGS** were found. However, there are some observations and recommendations for improved safety and clarity.

## Areas Analyzed

### 1. Polar Lookup Tables (polar_theta and distance)
**Status: SAFE** ✅

- **Declaration:** Lines 105-106
  ```cpp
  std::vector<std::vector<float>> polar_theta;
  std::vector<std::vector<float>> distance;
  ```

- **Initialization:** Lines 342-357 in `render_polar_lookup_table()`
  ```cpp
  polar_theta.resize(num_x, std::vector<float>(num_y, 0.0f));
  distance.resize(num_x, std::vector<float>(num_y, 0.0f));
  
  for (int xx = 0; xx < num_x; xx++) {
    for (int yy = 0; yy < num_y; yy++) {
      distance[xx][yy]    = hypotf(dx, dy);
      polar_theta[xx][yy] = atan2f(dy, dx);
    }
  }
  ```

- **Access Pattern:** All animation functions use loops bounded by `num_x` and `num_y`:
  ```cpp
  for (int x = 0; x < num_x; x++) {
    for (int y = 0; y < num_y; y++) {
      // Access polar_theta[x][y] and distance[x][y]
    }
  }
  ```

**Analysis:** The vectors are properly sized and all accesses are within bounds. The loop boundaries match the vector dimensions.

### 2. Oscillator Arrays (timings and move structs)
**Status: SAFE** ✅

- **Definition:** Line 30
  ```cpp
  #define num_oscillators 10
  ```

- **Arrays:**
  ```cpp
  struct oscillators {
    float offset[num_oscillators];  // indices 0-9
    float ratio[num_oscillators];   // indices 0-9
  };
  
  struct modulators {
    float linear[num_oscillators];      // indices 0-9
    float radial[num_oscillators];      // indices 0-9
    float directional[num_oscillators]; // indices 0-9
    float noise_angle[num_oscillators]; // indices 0-9
  };
  ```

- **Access Pattern in calculate_oscillators():** Lines 267-277
  ```cpp
  for (int i = 0; i < num_oscillators; i++) {
    move.linear[i]      = (runtime + timings.offset[i]) * timings.ratio[i];
    move.radial[i]      = fmodf(move.linear[i], 2 * PI);
    move.directional[i] = sinf(move.radial[i]);
    move.noise_angle[i] = PI * (1 + pnoise(move.linear[i], 0, 0));
  }
  ```

- **Manual Access Analysis:** Comprehensive grep revealed that all manual accesses use indices 0-9 only. Examples:
  - `move.radial[0]` through `move.radial[9]`
  - `move.linear[0]` through `move.linear[5]`
  - `move.directional[0]` through `move.directional[6]`
  - `move.noise_angle[0]` through `move.noise_angle[8]`

**Analysis:** All oscillator array accesses are within valid bounds [0-9].

### 3. Perlin Noise Array (pNoise)
**Status: SAFE** ✅

- **Declaration:** Lines 73-88
  ```cpp
  static const byte pNoise[] = { 151,160,137,91,... }; // 256 elements
  ```

- **Size:** 256 elements (indices 0-255)

- **Access Pattern:** Lines 232-260
  ```cpp
  #define P(x) pNoise[(x) & 255]
  ```

**Analysis:** The P() macro uses bitwise AND with 255 (`& 255`) which ensures the index is always in the range [0-255], preventing any out-of-bounds access regardless of input value. This is a safe implementation of the Perlin noise permutation table.

**Note:** While intermediate calculations in pnoise() (like `A = P(X)+Y`) can produce values > 255, these values are always passed through the P() macro which applies the & 255 mask before array access.

### 4. Special Cases

#### a. Half-resolution loop (line 1923)
```cpp
for (int x = 0; x < num_x/2; x++) {
  for (int y = 0; y < num_y/2; y++) {
    // Accesses distance[x][y] and polar_theta[x][y]
  }
}
```
**Status: SAFE** ✅ - Accesses only half the indices, which are clearly within bounds.

#### b. XY indexing function (lines 417-422)
```cpp
uint16_t xy(uint8_t x, uint8_t y) {
  if (serpentine &&  y & 1)
    return (y + 1) * num_x - 1 - x;
  else
    return y * num_x + x;
}
```
**Potential Issue:** This function doesn't validate that x < num_x and y < num_y. If called with out-of-bounds values, it will return an invalid LED index.

**Recommendation:** Add bounds checking or document that callers must ensure valid inputs:
```cpp
uint16_t xy(uint8_t x, uint8_t y) {
  // Caller must ensure: x < num_x && y < num_y
  if (serpentine &&  y & 1)
    return (y + 1) * num_x - 1 - x;
  else
    return y * num_x + x;
}
```

## Potential Issues Identified

### 1. No bounds checking in xy() function (LOW PRIORITY)
- **Location:** Lines 417-422
- **Risk:** Low - All observed calls use validated loop indices
- **Impact:** Could cause LED buffer overflow if called with invalid coordinates
- **Recommendation:** Add documentation or assertions

### 2. Implicit trust in num_x and num_y (LOW PRIORITY)
- **Observation:** The code assumes polar_theta and distance vectors are properly initialized before use
- **Risk:** Low - init() is called in constructors
- **Recommendation:** Add null checks or assertions in animation functions if paranoid validation is needed

## Recommendations

### High Priority: None
No critical issues found.

### Low Priority:
1. **Add bounds checking or documentation to xy() function**
   - Document that callers must validate inputs
   - Or add runtime assertions in debug builds

2. **Consider adding defensive checks**
   ```cpp
   void Rotating_Blob() {
     // Add at start of animation functions in debug builds
     assert(polar_theta.size() == num_x);
     assert(distance.size() == num_x);
     // ...
   }
   ```

3. **Consider const correctness**
   - The pNoise array is correctly marked as `static const`
   - Consider marking polar_theta and distance as const after initialization

## Conclusion

**The ANIMartRIX.h code is memory-safe with respect to array and vector bounds access.** All loops use correct boundaries, the Perlin noise implementation has proper masking, and oscillator arrays are accessed within valid indices. The observed "random behaviour" mentioned in the issue is unlikely to be caused by array bounds violations in the core library code.

Possible alternative causes for random behavior:
1. Uninitialized memory in user code (CRGB leds array)
2. Insufficient power supply causing LED glitches
3. Timing issues with millis() overflow
4. Memory constraints on embedded platform
5. Stack overflow from large automatic arrays
6. Hardware issues (wiring, power, signal integrity)

## Files Reviewed
- `/home/runner/work/animartrix-esp32/animartrix-esp32/ANIMartRIX.h` (complete review, 3980 lines)

## Analysis Date
2026-01-19
