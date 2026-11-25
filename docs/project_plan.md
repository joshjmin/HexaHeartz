# Real-Time Song Analyzer — Project Plan & Block Diagram

## ECE243 Final Project — DE1-SoC (Nios V)

---

## 1. Project Overview

This project performs real-time audio analysis on music fed through the DE1-SoC's audio
line-in. The system computes a **fixed-point FFT (Fast Fourier Transform)** on incoming audio
to produce a full frequency spectrum, from which it extracts **energy levels** and
**bass/mid/treble band power**. It estimates **BPM (beats per minute)** using an
**autocorrelation algorithm** on the energy signal. Live statistics are displayed on the board's
**7-segment displays**, while synchronized, mood-based visuals — including a real-time
**spectrum visualizer** — are rendered on the **VGA display**. The user can control visual modes
and settings via a **PS/2 keyboard**. Audio input is handled via **interrupts** for responsive,
real-time processing.

---

## 2. Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          NIOS V PROCESSOR                              │
│                                                                        │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────────────┐  │
│  │              │    │                  │    │                       │  │
│  │  AUDIO INPUT │    │  SIGNAL ANALYSIS │    │    DISPLAY OUTPUT     │  │
│  │  (Block A)   │───>│  ENGINE          │───>│    CONTROLLER         │  │
│  │              │    │  (Block B)       │    │    (Block D)          │  │
│  └──────┬───────┘    │                  │    │                       │  │
│         │  ^         └──────┬───────────┘    │  ┌─────────┐         │  │
│         │  │                │                │  │ 7-Seg   │         │  │
│  ┌──────┴──┴─────┐   ┌──────┴───────────┐    │  │ Driver  │         │  │
│  │  Audio CODEC  │   │                  │    │  └────┬────┘         │  │
│  │  WM8731       │   │  BEAT / BPM      │    │       │              │  │
│  │  (Interrupt)  │   │  DETECTOR        │    │  ┌────┴────┐         │  │
│  └──────┬────────┘   │  (Block C)       │    │  │ VGA     │         │  │
│         │            │                  │    │  │ Renderer│         │  │
│         │            └──────────────────┘    │  └────┬────┘         │  │
│         │                                    └───────┼──────────────┘  │
│         │  ┌──────────────────────────────┐          │                 │
│         └─>│  AUDIO OUTPUT / PLAYBACK     │          │                 │
│            │  (Block F)                   │          │                 │
│            │  Passthrough + line-out write│          │                 │
│            └──────────────┬───────────────┘          │                 │
│                           │                          │                 │
│  ┌──────────────┐         │         ┌────────────────┘                 │
│  │  PS/2        │    ┌────┴─────────┴──┐                               │
│  │  KEYBOARD    │───>│  USER INPUT     │                               │
│  │  INPUT       │    │  HANDLER        │                               │
│  │  (Interrupt) │    │  (Block E)      │                               │
│  └──────────────┘    └─────────────────┘                              │
│                                                                        │
└──────────────┬────────────────┬──────────────┬──────────────────────────┘
               │                │              │
         ┌─────┴─────┐    ┌─────┴─────┐  ┌─────┴─────┐
         │  Audio    │    │  HEX 7-   │  │   VGA     │
         │  Line-Out │    │  Segment  │  │  Display  │
         │ (WM8731)  │    │  Displays │  │ 320x240   │
         └───────────┘    └───────────┘  └───────────┘
```

---

## 3. Block Descriptions

### Block A — Audio Input (Interrupt-Driven)

| Attribute | Detail |
|---|---|
| **Hardware** | WM8731 Audio CODEC via Audio Port (base address `0xFF203040`) |
| **Method** | Interrupt-driven: ISR fires when audio FIFO has samples ready |
| **Output** | Raw 16-bit signed PCM samples written into a circular sample buffer |
| **Buffer Size** | 1024 samples (~23 ms at 44.1 kHz) — double-buffered so analysis runs on one buffer while the other fills |
| **Test** | Read samples and echo them to audio-out to verify correct capture |

### Block B — Signal Analysis Engine (FFT-Based)

| Attribute | Detail |
|---|---|
| **Input** | Completed sample buffer (1024 samples) from Block A |
| **Processing** | 1. **Energy (RMS):** Sum of squared samples, then integer square root approximation. Produces a 0–100 energy value. |
|  | 2. **Fixed-Point Radix-2 FFT:** In-place 1024-point Cooley-Tukey FFT using Q15 fixed-point arithmetic. Produces 512 frequency magnitude bins (0 Hz to Nyquist). All twiddle factors are pre-computed as Q15 constants in a lookup table. Butterfly operations use 32-bit intermediate products to prevent overflow. |
|  | 3. **Frequency Band Extraction:** Sum the magnitude of FFT bins within each range — **Bass** (bins 0–37, ~0–300 Hz), **Mid** (bins 38–256, ~300–2000 Hz), **Treble** (bins 257–512, ~2000–4000 Hz). Normalize each to 0–100. |
|  | 4. **Spectrum Data:** The raw 512 magnitude bins are also passed to the VGA renderer for the spectrum visualizer mode. |
| **Output** | `energy_level` (0–100), `bass_energy`, `mid_energy`, `treble_energy` (each 0–100), `fft_magnitudes[512]` |
| **Test** | Feed known sine waves (e.g., 440 Hz) and verify the FFT peak appears at the correct bin. Feed silence and verify all bins near zero. |

**FFT Algorithm Detail:**

The Cooley-Tukey radix-2 FFT breaks the 1024-point DFT into progressively smaller DFTs:
1. **Bit-reversal permutation** of input samples (pre-computed index table)
2. **10 stages** of butterfly operations (log2(1024) = 10)
3. Each butterfly: `X[k] = E[k] + W * O[k]`, where W = twiddle factor
4. Twiddle factors `W_N^k = cos(2*pi*k/N) - j*sin(2*pi*k/N)` stored as Q15
5. Final magnitude: `|X[k]| = isqrt(real[k]^2 + imag[k]^2)` (approximation acceptable)

### Block C — Beat / BPM Detector (Autocorrelation-Based)

| Attribute | Detail |
|---|---|
| **Input** | `bass_energy` values over time from Block B (bass band carries the beat) |
| **Algorithm** | **Two-stage approach: onset detection + autocorrelation for tempo.** |
|  | **Stage 1 — Onset Detection:** |
|  | 1. Maintain a running average of bass energy over ~0.5 seconds. |
|  | 2. A **beat onset** is detected when bass energy exceeds the running average by a threshold (e.g., 1.5x). |
|  | 3. This produces the `beat_flag` for visual sync. |
|  | **Stage 2 — Autocorrelation BPM:** |
|  | 1. Maintain a circular buffer of the last ~4 seconds of bass energy values (~32 values at ~8 analysis frames/sec). |
|  | 2. Compute the **autocorrelation** R(lag) = sum of `energy[i] * energy[i+lag]` for each lag value. |
|  | 3. Search lags corresponding to 60–200 BPM for the **peak** in R(lag). The lag with the highest autocorrelation gives the dominant period. |
|  | 4. Convert: `BPM = 60 / (peak_lag * frame_period)`. |
|  | 5. Apply exponential smoothing to the BPM estimate to avoid jitter. |
| **Output** | `bpm` (integer, 40–220 range), `beat_flag` (1 on the frame a beat is detected, 0 otherwise) |
| **Timer** | Uses the Interval Timer at `0xFF202000` for precise timestamps |
| **Test** | Play a metronome track at known BPM (e.g., 120 BPM), verify autocorrelation peak aligns and displayed BPM converges within ±5 BPM |

**Why Autocorrelation over simple peak-counting:**
Autocorrelation is robust to missed beats, syncopation, and noisy energy signals. It finds the
dominant *periodicity* rather than relying on every individual beat being detected. This is the
same core approach used in professional BPM detection software.

### Block D — Display Output Controller

#### D1: 7-Segment Driver

| Attribute | Detail |
|---|---|
| **Hardware** | HEX5–HEX0 displays (base addresses `0xFF200020`, `0xFF200030`) |
| **Display Layout** | HEX5–HEX4: BPM value (e.g., `120`). HEX3: current mode indicator. HEX2–HEX0: energy level (e.g., `075`). |
| **Input** | `bpm`, `energy_level`, `current_mode` from other blocks |
| **Test** | Write known values and visually verify correct segments |

#### D2: VGA Renderer

| Attribute | Detail |
|---|---|
| **Hardware** | VGA pixel buffer (320x240, base `0x08000000`) + character buffer (base `0x09000000`) |
| **Rendering** | Double-buffered (front/back buffer swap via pixel buffer control register). Draw to back buffer, then swap. |
| **Visual Modes** | See Section 4 below |
| **Input** | `energy_level`, `bass/mid/treble_energy`, `beat_flag`, `bpm`, `current_mode` |
| **Frame Rate** | Target ~30 FPS (one analysis + render per ~33 ms) |
| **Test** | Render each mode with static test values; verify no tearing with buffer swap |

### Block E — User Input Handler (PS/2 Keyboard, Interrupt-Driven)

| Attribute | Detail |
|---|---|
| **Hardware** | PS/2 Keyboard port (base address `0xFF200100`) |
| **Method** | Interrupt-driven: ISR reads scan code on key press |
| **Controls** | `1`–`4`: Switch visual mode. `↑`/`↓`: Adjust sensitivity threshold for beat detection. `Space`: Pause/resume analysis. `R`: Reset BPM calculation. |
| **Output** | Updates `current_mode`, `sensitivity`, `paused` state variables |
| **Test** | Press keys, verify state variables update correctly; print to character buffer for debug |

### Block F — Audio Output / Playback

| Attribute | Detail |
|---|---|
| **Hardware** | WM8731 Audio CODEC line-out (same base address `0xFF203040`, write-side registers `wsrc`/`wslc`) |
| **Method** | Polled: before storing each incoming sample, check write FIFO space (`wsrc > 0 && wslc > 0`) and write the sample to both `ldata` and `rdata` registers |
| **Input** | Raw PCM samples from Block A (captured from line-in) |
| **Output** | Audio passthrough to headphone/line-out jack — the input signal is heard in real-time on the output |
| **Latency** | One FIFO depth (~8 samples at 8 kHz ≈ 1 ms) — effectively zero-latency passthrough |
| **Test** | Connect a music source to line-in and headphones to line-out; confirm audio is audible and unclipped |

---

## 4. VGA Visual Modes

| Mode | Name | Description |
|---|---|---|
| 1 | **Spectrum Visualizer** | Classic FFT spectrum display — 32 vertical bars representing frequency bins from low to high. Bar height = FFT magnitude. Color gradient from red (bass) through green (mid) to blue (treble). Bars smoothly decay with a "peak hold" dot that falls slowly. Flashes on beat detection. This is the signature visual that directly showcases the FFT algorithm. |
| 2 | **Bar Graph** | Three large vertical bars for bass, mid, treble energy (derived from FFT band sums). Bars grow/shrink in real-time. Color shifts with energy (green → yellow → red). Pulses/flashes white on beat detection. |
| 3 | **Waveform** | Scrolling waveform of raw audio samples across the screen (oscilloscope style). Color tinted by mood (warm colors for high energy, cool for low). |
| 4 | **Mood Screen** | Full-screen color wash that smoothly transitions based on energy and frequency balance. Text overlay shows BPM and energy stats via character buffer. Background "particles" drift with velocity proportional to energy. |

---

## 5. Mood / Color Mapping Logic

The "mood" is derived from the frequency band balance and overall energy:

```
if bass_dominant AND high_energy   → RED/ORANGE tones   ("intense/hype")
if mid_dominant  AND medium_energy → GREEN/YELLOW tones  ("balanced/groovy")
if treble_dominant AND low_energy  → BLUE/PURPLE tones   ("chill/ambient")
```

This mapping is used across all visual modes to tint colors and select palettes.

---

## 6. Data Flow Summary

```
Audio Line-In
     │
     ▼
[Audio Capture] ──passthrough──> [Audio Output / Playback] ──> Audio Line-Out
     │                            (Block F: write sample          (WM8731 DAC,
     │                             to CODEC ldata/rdata)          headphones)
     ▼
[Circular Sample Buffer (double-buffered)]
                    │
                    ▼
        [Signal Analysis Engine]
         ├── 1024-pt Fixed-Point FFT
         │    └── fft_magnitudes[512]
         ├── Band Extraction from FFT bins
         │    ├── bass_energy  (bins 0–37)
         │    ├── mid_energy   (bins 38–256)
         │    └── treble_energy (bins 257–512)
         └── RMS energy_level
                    │
                    ▼
          [BPM Detector]
           ├── Onset detection (beat_flag)
           └── Autocorrelation tempo (bpm)
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
[7-Seg Driver]  [VGA Renderer]  [PS/2 Handler]
        │          │    │           │
        ▼          │    ▼           │
   HEX Displays   │  VGA Display   │
                   │    ▲           │
        fft_magnitudes   └───────────┘
        (spectrum viz)  (mode/settings)
```

---

## 7. Fixed-Point Arithmetic Strategy

The Nios V has **no hardware floating-point unit**, so all signal processing uses **Q15 fixed-point**
(16-bit signed, 15 fractional bits):

- Multiply two Q15 values: `result = (int32_t)a * b >> 15`
- FFT twiddle factors (sin/cos) pre-computed as Q15 constants in a 512-entry lookup table
- FFT butterfly operations use 32-bit intermediate products to prevent overflow
- Energy accumulation uses 32-bit integers to avoid overflow
- Integer square root via Newton's method (5–6 iterations sufficient)
- Autocorrelation accumulates into 32-bit sums (energy values are small enough)

**FFT Memory Budget:**
- Input buffer: 1024 × 4 bytes (real) + 1024 × 4 bytes (imaginary) = **8 KB**
- Twiddle table: 512 × 2 × 2 bytes (sin + cos, Q15) = **2 KB**
- Bit-reversal table: 1024 × 2 bytes = **2 KB**
- Output magnitudes: 512 × 4 bytes = **2 KB**
- **Total: ~14 KB** — fits comfortably in SDRAM

---

## 8. Interrupt Architecture

| Source | Priority | Purpose |
|---|---|---|
| Audio CODEC | High | Fill sample buffer — must not miss samples or audio glitches occur |
| PS/2 Keyboard | Medium | Read key scan codes — latency-tolerant |
| Interval Timer | Low | Periodic tick for BPM timestamp tracking and frame rate pacing |

All interrupts configured through the Nios V interrupt controller. Main loop polls a
`buffer_ready` flag set by the audio ISR, then runs analysis and rendering.

---

## 9. Memory Map

| Resource | Base Address | Usage |
|---|---|---|
| Audio Port | `0xFF203040` | Read audio samples, write audio config |
| Interval Timer | `0xFF202000` | BPM timing, frame pacing |
| PS/2 Keyboard | `0xFF200100` | Key input |
| HEX3–HEX0 | `0xFF200020` | Energy display |
| HEX5–HEX4 | `0xFF200030` | BPM display |
| Switches | `0xFF200040` | (Optional) additional controls |
| LEDs | `0xFF200000` | (Optional) beat indicator LEDs |
| VGA Pixel Buffer | `0x08000000` | Back pixel buffer |
| VGA Char Buffer | `0x09000000` | Text overlay |
| Pixel Buffer Ctrl | `0xFF203020` | Buffer swap control |

---

## 10. Weekly Plan & Milestones

### Week 1 (March 16) — Foundation

**Goal:** Audio capture working, basic energy calculation, 7-seg output.

| Task | Owner | Details |
|---|---|---|
| Audio ISR + circular buffer | Partner A | Set up interrupt-driven audio input; echo audio to output to verify |
| Energy (RMS) calculation | Partner A | Compute energy on each filled buffer; output to 7-seg HEX2–0 |
| VGA initialization + clear screen | Partner B | Set up double-buffered VGA; draw a test pattern |
| PS/2 keyboard ISR | Partner B | Read scan codes; print to character buffer for debug |
| 7-seg digit rendering utility | Partner B | Function to display a 3-digit decimal number on HEX displays |

**Deliverable for TA:** Block diagram (this document), audio input working, energy value on 7-seg.

### Week 2 (March 23) — Core Algorithms

**Goal:** FFT working, BPM via autocorrelation, spectrum visualizer on VGA.

| Task | Owner | Details |
|---|---|---|
| Fixed-point radix-2 FFT | Partner A | Implement 1024-pt Cooley-Tukey FFT in Q15. Pre-compute twiddle table and bit-reversal indices. Test with known sine waves — verify peak at correct bin. |
| FFT band extraction | Partner A | Sum FFT magnitude bins for bass/mid/treble. Replace energy-only display on HEX with band values. |
| Autocorrelation BPM | Partner A | Circular buffer of bass energy history. Compute autocorrelation, find peak lag, convert to BPM. Display on HEX5–4. |
| Spectrum Visualizer mode | Partner B | 32 bars from FFT magnitudes on VGA. Color gradient bass→treble. Peak-hold dots. |
| Bar Graph visual mode | Partner B | 3 large bars for bass/mid/treble from FFT band sums. Beat flash. |
| PS/2 mode switching | Partner B | Keys 1–4 switch visual mode; up/down adjust sensitivity |

**Deliverable for TA:** FFT spectrum visible on VGA, BPM on 7-seg, keyboard switching modes.

### Week 3 (March 30) — Polish & Demo

**Goal:** All 4 visual modes, mood mapping, robustness, demo.

| Task | Owner | Details |
|---|---|---|
| Waveform visual mode | Partner B | Scrolling oscilloscope display |
| Mood Screen visual mode | Partner B | Color wash + particles |
| Mood color mapping | Partner A | Implement mood logic from FFT band balance; integrate across all modes |
| FFT optimization | Partner A | Optimize butterfly inner loop in assembly if needed for frame rate |
| Edge case handling | Both | Handle silence, very loud input, no-beat music gracefully |
| Performance tuning | Both | Ensure 30 FPS; profile FFT vs. rendering time budget |
| Demo preparation | Both | Pick good demo songs (beat-heavy for BPM, varied for spectrum); rehearse |

**Stretch Goals (if time permits):**
- LED bar: light up LEDs 0–9 proportional to energy level (easy, ~30 min)
- Audio-out: output the filtered bass/mid/treble bands to headphones (moderate)
- Song history: track average BPM and energy across last 30 seconds on char buffer

---

## 11. Key Technical Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| FFT too slow for real-time | Dropped frames, laggy visuals | Profile on hardware; if needed, reduce to 512-point FFT or optimize butterfly loop in assembly. 1024-pt FFT is ~5000 multiplies — feasible at 50 MHz clock. |
| Fixed-point overflow in FFT | Corrupted spectrum | Use 32-bit intermediates for butterfly multiply-accumulate; scale down input samples before FFT |
| Autocorrelation gives wrong BPM | Unreliable BPM display | Constrain search to 60–200 BPM range; use exponential smoothing; demo with beat-heavy songs |
| VGA rendering too slow | Low frame rate, tearing | Keep draw routines simple; use `memset` for clears; double-buffer swap |
| Audio ISR misses samples | Glitchy audio, wrong analysis | Keep ISR minimal (just copy to buffer + set flag); do all FFT/analysis in main loop |
| Memory overflow | Crashes | Budget: 14 KB for FFT + 4 KB audio buffers + 150 KB VGA = ~170 KB total, fits in 64 MB SDRAM |

---

## 12. Hardware I/O Devices Used

1. **Audio CODEC** (WM8731) — audio line-in input (interrupt-driven) **and line-out playback** (polled write)
2. **VGA Display** — pixel buffer for visuals + character buffer for text (double-buffered)
3. **PS/2 Keyboard** — user control input (interrupt-driven)
4. **7-Segment Displays** (HEX0–HEX5) — numeric readout of BPM and energy
5. **Interval Timer** — timing for BPM calculation and frame pacing
6. *(Optional)* **LEDs** — beat indicator
7. *(Optional)* **Switches** — additional controls

This gives **5+ hardware devices** and **2 interrupt sources** (audio + PS/2), which aligns with
the rubric's "High" system complexity rating.

---

## 13. File Structure (Planned)

```
project/
├── main.c              # Main loop: analysis pipeline, rendering dispatch
├── audio.c / audio.h   # Audio ISR, circular buffer, codec init
├── fft.c / fft.h       # Fixed-point radix-2 FFT, twiddle tables, bit-reversal
├── analysis.c / .h     # RMS energy, FFT band extraction, spectrum output
├── bpm.c / bpm.h       # Beat detection, BPM estimation
├── vga.c / vga.h       # VGA drawing primitives, double-buffer swap
├── visuals.c / .h      # Visual mode renderers (bar, rings, wave, mood)
├── keyboard.c / .h     # PS/2 ISR, key mapping, state management
├── hex.c / hex.h       # 7-segment display driver
├── fixed_point.h       # Q15 macros and utility functions
└── isr.c               # Top-level interrupt handler, IRQ dispatch
```
