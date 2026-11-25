# main.c Code Breakdown

This document walks through every section of `src/main.c` — the sole source file for the Real-Time Song Analyzer running on the Nios V processor (DE1-SoC board).

---

## Table of Contents

1. [Hardware Base Addresses](#1-hardware-base-addresses)
2. [System Constants](#2-system-constants)
3. [Hardware Register Structs](#3-hardware-register-structs)
4. [Lookup Tables](#4-lookup-tables)
5. [Global State Variables](#5-global-state-variables)
6. [Utility Functions](#6-utility-functions)
7. [Display Functions](#7-display-functions)
8. [FFT Functions](#8-fft-functions)
9. [BPM Detection Functions](#9-bpm-detection-functions)
10. [VGA Functions](#10-vga-functions)
11. [Visual Mode Renderers](#11-visual-mode-renderers)
12. [PS/2 Keyboard Input](#12-ps2-keyboard-input)
13. [Synthetic Test Signal Generator](#13-synthetic-test-signal-generator)
14. [Main Loop](#14-main-loop)

---

## 1. Hardware Base Addresses

```c
#define AUDIO_BASE          0xFF203040
#define HEX3_HEX0_BASE     0xFF200020
#define HEX5_HEX4_BASE     0xFF200030
#define LED_BASE            0xFF200000
#define SW_BASE             0xFF200040
#define PS2_BASE            0xFF200100
#define PIXEL_BUF_CTRL_BASE 0xFF203020
#define TIMER_BASE          0xFF202000
```

These are **memory-mapped I/O addresses** for the DE1-SoC peripherals. On this board, peripherals don't use special I/O instructions — instead, each hardware device is assigned a fixed address in the processor's memory space. Reading from or writing to these addresses communicates directly with the hardware.

| Address | Peripheral | Purpose |
|---|---|---|
| `0xFF203040` | WM8731 Audio CODEC | Read/write audio samples to/from line-in and line-out |
| `0xFF200020` | HEX3-HEX0 | Lower four 7-segment displays |
| `0xFF200030` | HEX5-HEX4 | Upper two 7-segment displays |
| `0xFF200000` | LEDs | 10 red LEDs |
| `0xFF200040` | Slide Switches | 10 slide switches (read-only) |
| `0xFF200100` | PS/2 | Keyboard controller |
| `0xFF203020` | VGA Pixel Buffer Controller | Controls double-buffered VGA output |
| `0xFF202000` | Timer | Hardware interval timer (defined but unused) |

---

## 2. System Constants

### Buffer and FFT Sizing

```c
#define BUFFER_SIZE     1024
#define FFT_SIZE        1024
#define FFT_HALF        512
#define FFT_LOG2        10
```

- The audio buffer holds **1024 samples**. This is also the FFT window size.
- `FFT_LOG2 = 10` because 2^10 = 1024. The FFT has exactly 10 butterfly stages.
- `FFT_HALF = 512`: The FFT of a real signal is symmetric, so only the first 512 bins contain unique frequency information (the Nyquist theorem).

### Visualization Constants

```c
#define NUM_BARS        32
#define BINS_PER_BAR    16
```

The 512 FFT bins are grouped into **32 display bars**, each covering 16 bins (32 x 16 = 512). This maps the full spectrum to a VGA bar chart.

### BPM Detection Constants

```c
#define BPM_HISTORY_LEN 32
#define BPM_MIN         60
#define BPM_MAX         200
#define SAMPLE_RATE     8000
#define BPM_CONST       ((60 * SAMPLE_RATE + BUFFER_SIZE / 2) / BUFFER_SIZE)
#define BPM_LAG_MIN     (BPM_CONST / BPM_MAX)
#define BPM_LAG_MAX_RAW (BPM_CONST / BPM_MIN)
#define BPM_LAG_MAX     (BPM_LAG_MAX_RAW < BPM_HISTORY_LEN / 2 ? BPM_LAG_MAX_RAW : BPM_HISTORY_LEN / 2)
```

**How BPM detection maps to lag values:**

Each time the 1024-sample buffer fills, it represents one "frame" of audio. The frame rate is `SAMPLE_RATE / BUFFER_SIZE`. At 8 kHz, that's about 7.8 frames per second.

`BPM_CONST` converts between BPM and lag (in frames): `BPM = BPM_CONST / lag`. At 8 kHz, `BPM_CONST = 469`.

So a song at 120 BPM would have a beat every `469 / 120 ≈ 4` frames. The autocorrelation searches lag values corresponding to 60-200 BPM. `BPM_LAG_MAX` is clamped to half the history length to ensure enough overlap for meaningful correlation.

### Q15 Fixed-Point Arithmetic

```c
#define Q15_MUL(a, b) ((int)(((long long)(a) * (long long)(b)) >> 15))
```

**This is critical for the FFT.** The Nios V processor has no floating-point unit, so all math uses **Q15 fixed-point format**:

- A Q15 number uses 16 bits: 1 sign bit + 15 fractional bits.
- The value `1.0` is represented as `32767` (0x7FFF), and `-1.0` as `-32768`.
- Multiplying two Q15 numbers produces a Q30 result in a 32-bit (or 64-bit) word. The `>> 15` shift converts it back to Q15.
- The cast to `long long` (64-bit) prevents overflow: two 16-bit Q15 values multiplied can produce up to a 31-bit result, but intermediate FFT values can grow, so 64-bit intermediate is essential.

**Example:** `Q15_MUL(16384, 16384)` = `(16384 * 16384) >> 15` = `268435456 >> 15` = `8192`. In decimal: `0.5 * 0.5 = 0.25`, and `8192/32768 = 0.25`. Correct.

### VGA Color Format

```c
#define RGB565(r, g, b) ((((r) & 0x1F) << 11) | (((g) & 0x3F) << 5) | ((b) & 0x1F))
```

The VGA buffer uses **RGB565** format — each pixel is a 16-bit value:
- Bits 15-11: Red (5 bits, 0-31)
- Bits 10-5: Green (6 bits, 0-63 — human eyes are more sensitive to green)
- Bits 4-0: Blue (5 bits, 0-31)

---

## 3. Hardware Register Structs

### Audio CODEC Struct

```c
struct audio_t {
    volatile unsigned int control;    // +0
    volatile unsigned char rarc;      // +4: read available (right channel)
    volatile unsigned char ralc;      // +5: read available (left channel)
    volatile unsigned char wsrc;      // +6: write space (right channel)
    volatile unsigned char wslc;      // +7: write space (left channel)
    volatile unsigned int ldata;      // +8: left channel data
    volatile unsigned int rdata;      // +12: right channel data
};
```

This struct overlays the audio CODEC registers in memory. The `volatile` keyword is essential — it tells the compiler that these values can change at any time (because the hardware updates them), so the compiler must never cache or optimize away reads/writes.

The **FIFOSPACE** register at offset +4 is split into four `unsigned char` fields. Each reports how many samples are available to read from (or can be written to) the audio FIFOs. The code checks all four are nonzero before reading/writing a sample.

### VGA Pixel Buffer Controller Struct

```c
struct pixel_buf_ctrl_t {
    volatile unsigned int buffer;       // +0: front buffer address; write 1 to swap
    volatile unsigned int back_buffer;  // +4: back buffer address
    volatile unsigned int resolution;   // +8
    volatile unsigned int status;       // +C: bit 0 = swap pending
};
```

This controls double-buffered VGA output. Writing `1` to the `buffer` register triggers a swap at the next VSync. The `status` bit 0 stays `1` until the swap actually occurs.

---

## 4. Lookup Tables

### 7-Segment Display Encoding

```c
static const unsigned char seg7[10] = {
    0x3F, 0x06, 0x5B, 0x4F, 0x66, 0x6D, 0x7D, 0x07, 0x7F, 0x6F
};
```

Each byte encodes which of the 7 segments (a-g) to light up for digits 0-9. For example, `0x3F = 0b0111111` lights segments a-f (displaying "0"), while `0x06 = 0b0000110` lights segments b and c (displaying "1").

### FFT Twiddle Factor Tables

```c
static const short twiddle_cos[512];  // cos(2*pi*k/1024) in Q15
static const short twiddle_sin[512];  // -sin(2*pi*k/1024) in Q15
```

These are **precomputed trigonometric values** used by the FFT butterfly operations. Computing `cos()` and `sin()` at runtime would be far too slow on the Nios V, so all 512 values are stored in a lookup table.

The twiddle factor is `W_N^k = e^{-j*2*pi*k/N} = cos(2*pi*k/N) - j*sin(2*pi*k/N)`. The sin table stores `-sin` directly so the butterfly code can use addition instead of subtraction.

Values are in Q15 format: `cos(0) = 32767` (≈1.0), `cos(pi/2) = 0`, `cos(pi) = -32767` (≈-1.0).

### Bit-Reversal Permutation Table

```c
static const short bit_rev[1024];
```

The Cooley-Tukey FFT requires the input to be in **bit-reversed order**. For a 1024-point FFT (10-bit indices), each index has its binary representation reversed:

| Index | Binary | Reversed Binary | Bit-Reversed Index |
|---|---|---|---|
| 0 | `0000000000` | `0000000000` | 0 |
| 1 | `0000000001` | `1000000000` | 512 |
| 2 | `0000000010` | `0100000000` | 256 |
| 3 | `0000000011` | `1100000000` | 768 |

This reordering allows the FFT to be computed **in-place** (without extra memory) using the iterative Cooley-Tukey algorithm. The table is precomputed to avoid reversing bits at runtime.

---

## 5. Global State Variables

### Audio Buffers

```c
int sample_buf[BUFFER_SIZE];  // 1024 raw audio samples
int buf_index = 0;            // current write position
int new_data_ready = 0;       // flag: buffer is full, ready for analysis
```

`sample_buf` is a linear buffer (not circular) that fills up with 1024 samples. Once full, `new_data_ready` is set and analysis runs. Then `buf_index` resets to 0 and filling starts over.

### FFT Working Arrays

```c
int fft_real[FFT_SIZE];   // real part of FFT (1024 elements)
int fft_imag[FFT_SIZE];   // imaginary part of FFT (1024 elements)
int fft_mag[FFT_HALF];    // magnitude of first 512 bins
```

These hold the FFT computation results. `fft_real` and `fft_imag` are modified in-place during the butterfly stages. `fft_mag` stores `sqrt(real^2 + imag^2)` for display.

### VGA Double Buffer

```c
short pixel_buf_back[512 * 240];  // 245,760 bytes in SDRAM
```

This is the **second pixel buffer** for double-buffered VGA. The default hardware only has one buffer at `0xC8000000`, so drawing and displaying happen on the same buffer, causing visible flicker. This array provides a separate off-screen buffer. The stride is 512 pixels per row (not 320) because the VGA controller uses a power-of-2 stride for efficient address calculation.

**Memory calculation:** 512 pixels/row x 240 rows x 2 bytes/pixel = **245,760 bytes** of SDRAM.

---

## 6. Utility Functions

### Integer Square Root (`isqrt`)

```c
int isqrt(int n) {
    if (n <= 0) return 0;
    if (n == 1) return 1;
    int x = n;
    int y = (x + 1) / 2;
    while (y < x) {
        x = y;
        y = (x + n / x) / 2;
    }
    return x;
}
```

Computes the integer square root using **Newton's method** (also called the Babylonian method). Since there's no `sqrt()` or floating-point on the Nios V, this is an iterative integer-only approach.

**Algorithm:**
1. Start with an initial guess `y = (n + 1) / 2`.
2. Repeatedly refine: `y_new = (y + n/y) / 2`.
3. Each iteration halves the error. Stop when the estimate stops decreasing.

This converges very quickly — typically 5-10 iterations even for large values. It's used by both the RMS energy calculation and the FFT magnitude computation.

### RMS to dB Scale (`rms_to_db_scale`)

```c
int rms_to_db_scale(int rms) {
    // ...
    int log2_val = 0;
    int temp = rms;
    while (temp > 1) { temp >>= 1; log2_val++; }

    int lower = 1 << log2_val;
    int frac_100 = ((rms - lower) * 100) / lower;
    int db_x100 = 6 * (log2_val * 100 + frac_100);
    int energy = db_x100 / 66;
    // ...
}
```

Converts an RMS amplitude value to a **0-100 logarithmic (dB) scale**. This is important because human hearing perception is logarithmic — doubling the amplitude sounds like a fixed increase in volume, not a doubling.

**How it works without floating-point:**

1. **Integer log2:** Count how many times you can right-shift before reaching 1. For `rms = 100`: `100 → 50 → 25 → 12 → 6 → 3 → 1`, so `log2_val = 6`.

2. **Fractional interpolation:** The RMS value falls between two powers of 2. `lower = 2^6 = 64`. The fractional part is `(100 - 64) * 100 / 64 = 56` (i.e., 56% of the way from 64 to 128).

3. **dB approximation:** `dB ≈ 6 * log2(rms)` (since 20*log10(2) ≈ 6 dB). With the fractional part: `db_x100 = 6 * (600 + 56) = 3936`, meaning ~39.36 dB.

4. **Scale to 0-100:** Divide by 66 (since max RMS of 128 produces ~42 dB, and `4200/66 ≈ 64`... tuned empirically so full-scale maps to ~100).

### Energy Computation (`compute_energy`)

```c
int compute_energy(int *samples, int count) {
    unsigned int sum_sq = 0;
    for (int i = 0; i < count; i++) {
        int s = samples[i] >> 16;
        sum_sq += (unsigned int)(s * s);
    }
    int rms = isqrt((int)(sum_sq / (unsigned int)count));
    return rms_to_db_scale(rms);
}
```

Computes the **RMS (Root Mean Square)** energy of the audio buffer. RMS is the standard measure of signal power.

**Step-by-step:**
1. **Shift samples** from 24-bit (in 32-bit word) down to ~8-bit range (`>> 16`). This prevents overflow when squaring — `255^2 * 1024 = 66,585,600` which fits in a 32-bit unsigned int.
2. **Sum of squares:** Accumulate `s^2` for all 1024 samples.
3. **Mean:** Divide by sample count.
4. **Root:** Take integer square root.
5. **Convert to dB scale** for perceptually meaningful display.

---

## 7. Display Functions

### LED Bar (`led_bar`)

```c
void led_bar(int energy) {
    volatile int *led_ptr = (int *)LED_BASE;
    int num_leds = energy / 10;
    int pattern = 0;
    for (int i = 0; i < num_leds && i < 10; i++)
        pattern |= (1 << i);
    *led_ptr = pattern;
}
```

Maps energy (0-100) to 0-10 LEDs. Each LED represents 10 units of energy. The bit pattern is built by OR-ing in one bit per LED (bit 0 = LED0, bit 1 = LED1, etc.), then written to the LED memory-mapped register in a single write.

### 7-Segment Display (`hex_show_full`)

Splits BPM and energy into individual digits, looks up each digit's 7-segment encoding from `seg7[]`, then packs them into the correct bit positions within the two HEX display registers.

**Register layout:**
- `HEX3_HEX0_BASE` (one 32-bit register): HEX3 at bits 30-24, HEX2 at bits 22-16, HEX1 at bits 14-8, HEX0 at bits 6-0.
- `HEX5_HEX4_BASE` (one 32-bit register): HEX5 at bits 14-8, HEX4 at bits 6-0.

BPM goes on HEX5-3 (hundreds, tens, ones), energy goes on HEX2-0.

---

## 8. FFT Functions

### The FFT Algorithm (`fft_compute`) — In Depth

This is a **1024-point radix-2 decimation-in-time (DIT) Cooley-Tukey FFT** implemented with Q15 fixed-point arithmetic.

#### What the FFT does

The FFT transforms 1024 time-domain audio samples into 1024 frequency-domain bins. Each bin tells you how much energy exists at a specific frequency. Bin `k` corresponds to frequency `k * SAMPLE_RATE / FFT_SIZE`. At 8 kHz sample rate: bin 0 = 0 Hz (DC), bin 1 = 7.8 Hz, bin 56 = 437.5 Hz (approximately A4 = 440 Hz), bin 512 = 4000 Hz (Nyquist).

#### Step 1: Bit-Reversal Permutation

```c
for (i = 0; i < FFT_SIZE; i++) {
    fft_real[bit_rev[i]] = samples[i] >> 16;
    fft_imag[bit_rev[i]] = 0;
}
```

The input samples are loaded into the FFT arrays in **bit-reversed order**. Sample `i` goes to position `bit_rev[i]`. The imaginary parts start at 0 (real input signal). The `>> 16` reduces 24-bit samples to ~8-bit range to prevent overflow in subsequent multiplications.

**Why bit-reversal?** The iterative Cooley-Tukey FFT processes butterflies in a specific order that requires this permutation. It's equivalent to recursively splitting even/odd indices — the bit-reversal achieves this non-recursively.

#### Step 2: 10 Stages of Butterfly Operations

```c
half_size = 1;
for (stage = 0; stage < FFT_LOG2; stage++) {
    size = half_size << 1;
    tw_step = FFT_HALF / half_size;

    for (group = 0; group < FFT_SIZE; group += size) {
        for (k = 0; k < half_size; k++) {
            tw_idx = k * tw_step;
            // ... butterfly computation ...
        }
    }
    half_size = size;
}
```

The FFT proceeds in **10 stages** (log2(1024) = 10). Each stage doubles the DFT size being computed:

| Stage | half_size | size | # Groups | Butterflies/Group | Twiddle Step |
|---|---|---|---|---|---|
| 0 | 1 | 2 | 512 | 1 | 512 |
| 1 | 2 | 4 | 256 | 2 | 256 |
| 2 | 4 | 8 | 128 | 4 | 128 |
| ... | ... | ... | ... | ... | ... |
| 9 | 512 | 1024 | 1 | 512 | 1 |

Each stage always performs exactly 512 butterfly operations total, but organized differently.

#### The Butterfly Operation

```c
// Twiddle multiply: W * odd element
tr = Q15_MUL(twiddle_cos[tw_idx], odd_r) -
     Q15_MUL(twiddle_sin[tw_idx], odd_i);
ti = Q15_MUL(twiddle_cos[tw_idx], odd_i) +
     Q15_MUL(twiddle_sin[tw_idx], odd_r);

// Butterfly: combine even and twiddle*odd
fft_real[group + k]             = ur + tr;
fft_imag[group + k]             = ui + ti;
fft_real[group + k + half_size] = ur - tr;
fft_imag[group + k + half_size] = ui - ti;
```

This is the core computation. Each butterfly:

1. Takes two complex numbers: the "even" element `(ur, ui)` and the "odd" element `(odd_r, odd_i)`.
2. Multiplies the odd element by a **twiddle factor** `W = cos(θ) - j*sin(θ)`:
   - `tr = cos*odd_r - (-sin)*odd_i = cos*odd_r + sin*odd_i`
   - Actually, since `twiddle_sin` stores `-sin`: `tr = cos*odd_r - (-sin)*odd_i`
3. Produces two outputs:
   - Top: `even + W*odd`
   - Bottom: `even - W*odd`

This is an **in-place** operation — the two inputs are overwritten with the two outputs.

### FFT Magnitude (`fft_magnitudes`)

```c
void fft_magnitudes(void) {
    for (int k = 0; k < FFT_HALF; k++) {
        int r = fft_real[k] >> 4;
        int im = fft_imag[k] >> 4;
        fft_mag[k] = isqrt(r * r + im * im);
    }
}
```

Computes `|X[k]| = sqrt(re^2 + im^2)` for each frequency bin. The `>> 4` shift before squaring prevents 32-bit overflow (without it, values up to ~8000 squared would exceed 2^31).

### Band Extraction (`band_extract`)

```c
// Bass:   bins 0-37   (~0-1590 Hz)
// Mid:    bins 38-256  (~1633-11025 Hz)
// Treble: bins 257-511 (~11068-22007 Hz)
```

Sums FFT magnitudes within three frequency ranges and normalizes each to 0-100. The divisors (20, 80, 40) are tuned empirically to account for the different number of bins per band and the typical spectral shape of music.

---

## 9. BPM Detection Functions

### Beat Detection (`beat_detect`)

```c
void beat_detect(void) {
    bass_avg = (bass_avg * 7 + bass_energy) >> 3;  // EMA
    int threshold = bass_avg * sensitivity / 100;
    if (bass_energy > threshold && bass_energy > 10)
        beat_flag = 1;
}
```

Uses an **exponential moving average (EMA)** to track the running average of bass energy. A beat is detected when the current bass energy exceeds the average by a factor of `sensitivity/100` (default 1.5x).

**The EMA `(avg * 7 + new) >> 3`** is equivalent to `avg = 0.875 * avg + 0.125 * new`. This is a common technique for implementing a low-pass filter with integer-only arithmetic. The `>> 3` (divide by 8) replaces floating-point division. The 7/8 weighting means the average adapts slowly, so brief peaks (beats) stand out clearly.

The absolute minimum threshold of `10` prevents false triggers during silence.

### BPM Estimation (`bpm_update`)

```c
void bpm_update(void) {
    bass_history[bass_hist_idx] = bass_energy;
    bass_hist_idx = (bass_hist_idx + 1) % BPM_HISTORY_LEN;

    for (lag = min_lag; lag <= max_lag; lag++) {
        corr = 0;
        for (i = 0; i < BPM_HISTORY_LEN - lag; i++) {
            int idx1 = (bass_hist_idx + i) % BPM_HISTORY_LEN;
            int idx2 = (bass_hist_idx + i + lag) % BPM_HISTORY_LEN;
            corr += (bass_history[idx1] * bass_history[idx2]) >> 8;
        }
        if (corr > best_corr) { best_corr = corr; best_lag = lag; }
    }

    new_bpm = BPM_CONST / best_lag;
    bpm = (bpm * 3 + new_bpm) >> 2;  // smooth
}
```

This uses **autocorrelation** to find the dominant periodicity in bass energy history.

**How autocorrelation works:**

Autocorrelation measures how similar a signal is to a delayed copy of itself. For each possible delay (lag), it multiplies the signal by the shifted version and sums the products. If there's a repeating pattern (like a beat), the autocorrelation will peak at lags that match the beat period.

**Step-by-step:**
1. **Push** current bass energy into a circular buffer of 32 frames.
2. **For each lag** in the valid BPM range: compute the sum of element-wise products between the original and shifted signal. The `>> 8` prevents overflow.
3. **Find the lag** with the highest correlation — this is the beat period in frames.
4. **Convert to BPM:** `BPM = BPM_CONST / lag`. At 8 kHz with 1024-sample buffers, `BPM_CONST = 469`, so `lag = 4` gives `469/4 = 117 BPM`.
5. **Smooth** with a 3/4 old + 1/4 new exponential filter to reduce jitter.

### Display Smoothing (`smooth_display` and `decay_display`)

These implement an **asymmetric envelope follower**:

- **Attack (rising):** Instant — if the new value is higher, use it immediately. This keeps the display responsive to beats and transients.
- **Decay (falling):** Gradual — multiply by 3/4 (75%) each analysis frame. This creates a smooth fade-out that prevents flickering.

`decay_display` runs on VSync frames where no new FFT data is available, using a gentler 7/8 (87.5%) decay tuned for the faster 60 fps display rate.

---

## 10. VGA Functions

### Pixel Plotting (`plot_pixel`)

```c
void plot_pixel(int x, int y, short color) {
    if (x < 0 || x >= SCREEN_W || y < 0 || y >= SCREEN_H) return;
    back_buf_ptr[y * 512 + x] = color;
}
```

Writes a single RGB565 pixel to the back buffer. The **stride is 512** (not 320) because the VGA controller expects power-of-2 row widths for efficient hardware address computation. This means each row has 192 unused pixels (wasted memory, but required by the hardware).

### Screen Clear (`clear_screen`)

```c
void clear_screen(void) {
    volatile int *p = (volatile int *)back_buf_ptr;
    int total = 512 * 240 / 2;
    for (int i = 0; i < total; i++) p[i] = 0;
}
```

Clears the back buffer by writing zeros. Uses **32-bit writes** to clear 2 pixels per write (each pixel is 16 bits), giving ~2x throughput compared to writing pixels individually. This writes `512 * 240 / 2 = 61,440` 32-bit words.

### VSync and Buffer Swap (`wait_for_vsync`)

```c
void wait_for_vsync(void) {
    ctrl->buffer = 1;                   // trigger swap
    while (ctrl->status & 0x01) ;       // wait for VSync
    back_buf_ptr = (volatile short *)(ctrl->back_buffer);  // update pointer
}
```

This implements **double-buffered rendering**:
1. Writing `1` to the buffer register requests a swap.
2. The hardware waits until the next **vertical blanking interval** (VSync) — the brief period between finishing one screen refresh and starting the next.
3. The swap happens atomically during VSync, so the viewer never sees a half-drawn frame.
4. After the swap, the roles reverse: what was the back buffer is now displayed, and the old front buffer becomes the new back buffer for drawing.

### VGA Initialization (`vga_init`)

Sets up proper double-buffering by pointing the back buffer register to `pixel_buf_back[]` (in SDRAM), separate from the default `0xC8000000`. Clears both buffers and performs an initial swap so the system starts in a clean state.

---

## 11. Visual Mode Renderers

### Mode 1: Spectrum Visualizer (`draw_spectrum`)

Draws 32 vertical bars representing the frequency spectrum from low to high:

- **Layout:** 32 bars x 10px each (8px bar + 2px gap) = 320px = full screen width.
- **Color gradient:** Bass bars (0-7) are red, mid bars (8-19) are green, treble bars (20-31) are blue, with intensity gradients within each region.
- **Peak-hold dots:** White horizontal lines that track the maximum height of each bar, hold for 15 frames, then fall at 2 pixels/frame. This mimics the classic peak meters seen on audio equipment.
- **Beat flash:** When `beat_flag` is set, a white border is drawn around the entire display.

### Mode 2: Bar Graph (`draw_bar_graph`)

Draws 3 large bars (80px wide) for bass, mid, and treble:

- Each bar height is `energy * 2` pixels (0-200 range).
- Color changes with energy level: green (low, <33) -> yellow (medium, 33-65) -> red (high, 66+).
- All three bars flash white on beat detection.
- Colored label markers at the bottom identify each bar (red=bass, green=mid, blue=treble).

---

## 12. PS/2 Keyboard Input

### Scan Code State Machine

PS/2 keyboards send **scan codes** with prefixes:
- `0xF0` = break prefix (the next byte is a key *release*, not a press)
- `0xE0` = extended prefix (for keys like arrows that have 2-byte codes)

The state machine tracks these prefixes with `ps2_break_flag` and `ps2_extended_flag`.

### Reading the PS/2 Port (`read_ps2_key`)

```c
int read_ps2_key(void) {
    volatile int *ps2_ptr = (int *)PS2_BASE;
    int data = *ps2_ptr;
    if (data & 0x8000)         // bit 15 = RVALID (data available)
        return data & 0xFF;    // bits 7-0 = scan code
    return -1;
}
```

Reads the PS/2 data register. Bit 15 (`RVALID`) indicates whether valid data is available. If set, the lower 8 bits contain the scan code. This is a **polled** (not interrupt-driven) approach.

### Key Bindings

| Key | Scan Code | Action |
|---|---|---|
| 1 | `0x16` | Spectrum mode |
| 2 | `0x1E` | Bar graph mode |
| 3 | `0x26` | Waveform mode |
| 4 | `0x25` | Mood mode |
| Space | `0x29` | Toggle pause |
| R | `0x2D` | Reset all state |
| Up Arrow | `0xE0, 0x75` | Increase beat sensitivity (+10) |
| Down Arrow | `0xE0, 0x72` | Decrease beat sensitivity (-10) |

Sensitivity ranges from 110 to 300 (representing 1.1x to 3.0x the average).

---

## 13. Synthetic Test Signal Generator

When `SW0` is flipped on, the system bypasses the audio CODEC and fills the sample buffer with a mathematically generated signal. This is invaluable for testing without a real audio source.

### Sine Wave Generation

All test signals use the `twiddle_cos[]` table as a sine/cosine lookup:

```c
test_phase_a = (test_phase_a + step) & 1023;
int idx = test_phase_a < 512 ? test_phase_a : 1023 - test_phase_a;
int s = twiddle_cos[idx];
```

**How this works:**
- `test_phase_a` is a **phase accumulator** that wraps at 1024 (one full cycle of the lookup table).
- The `step` value determines the frequency: `freq = step * SAMPLE_RATE / 1024`. For 440 Hz at 8 kHz: `step = 440 * 1024 / 8000 ≈ 56`.
- The `twiddle_cos` table only has 512 entries (half cycle). The `idx < 512 ? idx : 1023 - idx` mirrors the second half, exploiting the symmetry `cos(π + x) = cos(π - x)` to cover the full cycle.

### Four Test Modes (SW2:SW1)

| SW2 | SW1 | Signal | Purpose |
|---|---|---|---|
| 0 | 0 | 440 Hz sine (low amplitude) | Verify FFT shows single peak at bin 56, bass energy rises |
| 0 | 1 | 220 Hz + 3520 Hz dual tone (low amplitude) | Verify band separation — bass and treble both active |
| 1 | 0 | 120 BPM kick pattern (high amplitude) | Verify beat detection fires and BPM converges to ~120 |
| 1 | 1 | Frequency sweep (high amplitude) | Verify all spectrum bars animate sequentially |

The **beat pulse** signal is the most complex: it generates a short (50-sample) exponentially decaying sine burst at 100 Hz every 4000 samples (= 120 BPM at 8 kHz). The envelope linearly decays from full to zero over the burst duration, simulating a kick drum.

---

## 14. Main Loop

The main loop runs three phases every iteration:

### Phase 1: Audio Collection

```c
if (test_mode) {
    fill_test_signal(sw1, sw2);
    new_data_ready = 1;
} else {
    while (FIFO has data) {
        read sample → echo to output → store in buffer
        if (buffer full) { new_data_ready = 1; break; }
    }
}
```

In normal mode, the loop **drains all available samples** from the audio FIFO each iteration (rather than reading one at a time) to keep up with the sample rate. Once 1024 samples are collected, analysis triggers.

In test mode, the entire buffer is filled synthetically in one shot.

### Phase 2: Analysis (only when new data & not paused)

```c
energy_level = compute_energy(sample_buf, BUFFER_SIZE);
fft_compute(sample_buf);
fft_magnitudes();
band_extract();
beat_detect();
bpm_update();
smooth_display();
hex_show_full(energy_level, current_mode, bpm);
led_bar(energy_level);
```

The full DSP pipeline runs:
1. Compute overall energy (RMS → dB)
2. Run 1024-point FFT
3. Compute magnitudes of frequency bins
4. Extract bass/mid/treble band energies
5. Detect beats
6. Update BPM estimate
7. Smooth display values for flicker-free rendering
8. Update HEX displays and LEDs

### Phase 3: Render (every VSync frame)

```c
if (!paused) {
    if (!new_data_ready) decay_display();  // gentle fade when no new data
    clear_screen();
    switch (current_mode) { /* draw selected mode */ }
    wait_for_vsync();
    beat_flag = 0;
}
```

Rendering runs independently of analysis — it draws every VSync frame (~60 fps) even if no new audio data arrived. When there's no new data, `decay_display()` gently fades the bars to keep the animation smooth. After drawing, it waits for VSync (synchronizing to the display refresh) and clears the beat flash flag.

This separation of analysis rate (~7.8 fps at 8 kHz) from display rate (~60 fps) is what makes the visualization appear smooth despite the relatively slow audio frame rate.
