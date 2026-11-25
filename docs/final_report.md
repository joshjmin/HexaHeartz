# ECE243 Final Project Report
## Real-Time Song Analyzer

**Group Members:** Aidan Tran · Joshua Min  
**Submission Date:** April 7, 2026

---

## 1. Project Description

This project is a real-time audio analyzer that runs on the DE1-SoC board, executing entirely on the Nios V processor. Music is fed through the board's audio line-in jack, and the system continuously captures 1024-sample frames at 8 kHz. Each frame is processed through a custom-written, in-place 1024-point radix-2 Cooley-Tukey FFT using Q15 fixed-point arithmetic (no floating-point hardware is available on the Nios V). The FFT output is used to compute the overall RMS energy level and to split the spectrum into three frequency bands — bass (0–290 Hz), mid (295 Hz – 2 kHz), and treble (2–4 kHz). A beat/BPM detector processes the audio in 64-sample hops: it computes the average absolute sample value per hop, tracks a slow EMA baseline, and fires a beat when the hop energy exceeds the baseline by a sensitivity-controlled threshold (with a minimum refractory gap between beats). Beat-to-beat intervals are averaged with an EMA and converted directly to a BPM value. All analysis results are displayed live: energy and BPM are shown as 3-digit values on the six 7-segment displays (HEX0–HEX5), energy level is shown as a bar graph on the ten LEDs, and a 320×240 double-buffered VGA display renders one of four selectable visual modes. A PS/2 keyboard controls mode switching, beat sensitivity, pause/resume, and state reset. A built-in test signal generator (controlled by slide switches) can inject four types of synthetic signals — a single 440 Hz tone, a dual-tone (220 Hz + 3520 Hz), a 120 BPM kick pulse pattern, and a sweeping sine — for verifying individual subsystems without external audio hardware.

---

## 2. Block Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              NIOS V PROCESSOR                              │
│                                                                            │
│  ┌─────────────────┐      ┌──────────────────────────────────────────────┐ │
│  │  AUDIO INPUT    │      │            SIGNAL ANALYSIS ENGINE            │ │
│  │  (Block A)      │─────>│                                              │ │
│  │                 │      │  ┌──────────────┐   ┌────────────────────┐  │ │
│  │  WM8731 CODEC   │      │  │ RMS Energy   │   │  1024-pt FFT       │  │ │
│  │  Line-In, 8kHz  │      │  │ (log scale)  │   │  Q15 fixed-point   │  │ │
│  │  Polled FIFO    │      │  │  0–100       │   │  Hann windowed     │  │ │
│  └─────────────────┘      │  └──────┬───────┘   └────────┬───────────┘  │ │
│                           │         │                    │               │ │
│  ┌─────────────────┐      │         │           ┌────────┴──────────┐    │ │
│  │  TEST SIGNAL    │      │         │           │  Band Extraction  │    │ │
│  │  GENERATOR      │─────>│         │           │  Bass / Mid /     │    │ │
│  │  (SW0–SW2)      │      │         │           │  Treble (0–100)   │    │ │
│  └─────────────────┘      │         │           └────────┬──────────┘    │ │
│                           └─────────┼─────────────────── ┼──────────────┘ │
│                                     │                    │                │
│                           ┌─────────┴────────────────────┴──────────────┐ │
│                           │              BPM DETECTOR  (Block C)        │ │
│                           │  Cascaded LP envelope · onset peak-pick     │ │
│                           │  Interval → candidate → median/MAD BPM      │ │
│                           └──────────────────────┬───────────────────────┘ │
│                                                  │                        │
│  ┌─────────────────┐      ┌─────────────────┐    │   ┌─────────────────┐  │
│  │  PS/2 KEYBOARD  │─────>│  USER INPUT     │    │   │  DISPLAY        │  │
│  │  (Block E)      │      │  HANDLER        │    ├──>│  CONTROLLER     │  │
│  │  1–4/↑↓/Spc/R   │      │  mode·sens·     │    │   │  (Block D)      │  │
│  └─────────────────┘      │  pause·reset    │    │   │                 │  │
│                           └────────┬────────┘    │   │  7-Seg HEX0–5   │  │
│                                    │             │   │  LED Bar (×10)  │  │
│                                    │             │   │  VGA 320×240    │  │
│                                    └─────────────┴──>│  4 Visual Modes │  │
│                                                      └─────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
         │                              │                   │
    ┌────┴────┐                  ┌──────┴──────┐     ┌──────┴──────┐
    │  Audio  │                  │  HEX0–HEX5  │     │  VGA 320×240│
    │  Line-In│                  │  LEDs ×10   │     │  + Char Buf │
    └─────────┘                  └─────────────┘     └─────────────┘
```

**Block Descriptions**

| Block | Function |
|-------|----------|
| **A – Audio Input** | Polls the WM8731 FIFO; averages left/right channels to mono; accumulates into a 1024-sample circular buffer. WM8731 configured over I2C to 0 dB line-in gain with microphone path powered down. |
| **B – Signal Analysis** | On each full buffer: (1) DC-blocked RMS energy on a log₂ scale → 0–100; (2) Hann-windowed, auto-scaled Q15 FFT produces 512 magnitude bins; (3) band-summed bass/mid/treble → 0–100 via √-compression. |
| **C – BPM Detector** | Processes 64-sample hops; computes average absolute energy per hop; fires a beat when hop energy exceeds a slow EMA baseline scaled by the sensitivity threshold, with a minimum refractory gap; averages beat-to-beat intervals via EMA and folds the result into the 60–200 BPM range. |
| **D – Display Controller** | HEX5-4-3 = BPM (hundreds/tens/ones); HEX2-1-0 = energy. LEDs = energy bar. VGA double-buffered (back buffer in SDRAM); renders Spectrum / Bar Graph / Waveform / Mood Screen based on current mode. |
| **E – User Input** | PS/2 state machine handles break/extended prefixes. Keys 1–4 switch visual mode, ↑/↓ adjust beat sensitivity (110–300), Space toggles pause, R resets BPM and all smoothing state. SW0 enables test signal mode; SW1/SW2 select signal type. |

---

## 3. How to Operate

### Hardware Setup
1. Connect the DE1-SoC to a VGA monitor and a PS/2 keyboard.
2. Plug an audio source (e.g., phone or laptop) into the **line-in** jack (blue) on the board.
3. Power on and load the program.

### Normal Operation
- Set **SW0 to the down (off) position** to use real audio input.
- Play music on the connected audio source. The program will immediately begin analyzing.
- **HEX5–HEX3** display the current BPM estimate (e.g., `120` = 120 BPM). BPM shows `000` until enough beats have been detected to form a consensus estimate (typically 3–5 seconds of steady rhythmic music).
- **HEX2–HEX0** display the current energy level (0–100), updated every analysis frame.
- **The 10 red LEDs** show energy as a proportional bar (1 LED per 10 energy units).
- **The VGA display** shows the current visual mode (default: Spectrum Visualizer). A white flash border appears on each detected beat.

### Keyboard Controls

| Key | Action |
|-----|--------|
| `1` | Switch to **Spectrum Visualizer** — 32 FFT frequency bars (red=bass, green=mid, blue=treble) with peak-hold dots |
| `2` | Switch to **Band Bar Graph** — three large bars (bass/mid/treble); color shifts green→yellow→red with energy; flashes white on beats |
| `3` | Switch to **Waveform (Oscilloscope)** — raw audio samples drawn as a connected trace; color tinted by mood |
| `4` | Switch to **Mood Screen** — full-screen gradient + 20 drifting particles; text overlay shows BPM, energy, band levels, and current mood (INTENSE / GROOVY / CHILL) |
| `↑` (Up arrow) | Increase beat detection sensitivity threshold (makes detector less sensitive; range 110–300) |
| `↓` (Down arrow) | Decrease beat detection sensitivity threshold (makes detector more sensitive) |
| `Space` | Pause / resume analysis and visualization |
| `R` | Reset BPM detector and all smoothing state (useful when switching songs) |

### Test Signal Mode (Slide Switches)
Set **SW0 up** to enable the built-in test signal generator. SW1 and SW2 select the signal:

| SW2 | SW1 | Signal |
|-----|-----|--------|
| 0 | 0 | **Single tone** — 440 Hz sine wave. One bar lights up in the bass region of the spectrum. |
| 0 | 1 | **Dual tone** — 220 Hz + 3520 Hz. Two separate bars light up (bass + treble) in the spectrum and bar-graph modes. |
| 1 | 0 | **Beat pulse** — 120 BPM kick bursts. The BPM display should converge to approximately 120 after a few seconds. LEDs flash on each beat. |
| 1 | 1 | **Frequency sweep** — sine wave sweeping from bass to treble. All 32 spectrum bars animate sequentially from left to right. |

Set **SW0 back down** to return to live audio input.

### Tips for Best Results
- Use music with a strong, consistent beat (e.g., electronic or pop) for the most reliable BPM detection.
- Press `R` when switching songs to clear stale BPM candidates.
- The Mood Screen classification stabilizes after approximately 1.5 seconds of sustained audio. INTENSE = bass-dominant + high energy; GROOVY = mid-dominant; CHILL = treble or low energy.

---

## 4. Attribution Table

| Task | Aidan Tran | Joshua Min | Notes |
|------|:-----------:|:-----------------:|-------|
| WM8731 codec configuration (I2C registers, line-in setup) | ✓ | | |
| Audio FIFO polling and mono mix | ✓ | | |
| RMS energy computation and log-scale mapping | ✓ | | Initial dB threshold parameters tuned collaboratively |
| Q15 fixed-point FFT (twiddle tables, bit-reversal, butterfly loop) | ✓ | | |
| Hann window, DC blocking, auto-scaling pre-conditioning | ✓ | | |
| FFT magnitude computation and band extraction (bass/mid/treble) | ✓ | | |
| BPM detection (hop energy threshold, EMA interval averaging) | ✓ | | |
| Mood classification system (slow EMA + hysteresis) | ✓ | | |
| Test signal generator (4 modes via slide switches) | ✓ | | |
| VGA initialization and double-buffering (SDRAM back buffer) | ✓ | | |
| VGA drawing primitives (plot_pixel, draw_rect_fill, draw_hline) | ✓ | | |
| Mode 1: Spectrum Visualizer (bars, peak-hold, beat flash) | ✓ | | |
| Mode 2: Band Bar Graph (3-bar display, energy color coding) | ✓ | | |
| Mode 3: Waveform oscilloscope | ✓ | | |
| Mode 4: Mood Screen (gradient background, particle system, char overlay) | ✓ | | |
| 7-segment display driver (BPM + energy formatting) | | ✓ | |
| LED energy bar graph | | ✓ | |
| PS/2 keyboard scan code state machine and key mapping | | ✓ | |
| Smoothing / display decay functions (smooth_display, decay_display) | | ✓ | |
| Project plan, block diagram documentation | | ✓ | |
| Code comments and documentation throughout main.c | ✓ | ✓ | Shared |

### Contribution Summary

| Partner | Fraction | Primary Responsibilities |
|---------|----------|--------------------------|
| **Aidan Tran** | ~70% | Entire signal processing pipeline: audio codec setup, Q15 FFT from scratch, energy computation, band extraction, BPM detection (hop energy threshold + EMA intervals), mood classification, all four VGA visual mode renderers, VGA double-buffering infrastructure, test signal generator |
| **Joshua Min** | ~30% | 7-segment display driver, LED bar output, PS/2 keyboard handler, display smoothing functions, project documentation and block diagram, code commenting and parameter review during Week 1 |

---

*Report submitted to Quercus by April 7, 2026 at 6:00 PM.*
