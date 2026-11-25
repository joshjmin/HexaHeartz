# Synthetic Test Signal Reference

Controlled by slide switches (SW0–SW2) in CPUlator's Switches widget.
**SW0 must be ON** to enable test mode. SW1/SW2 select the signal.

## Switch Settings

| SW2 | SW1 | SW0 | Signal |
|-----|-----|-----|--------|
| 0 | 0 | 0 | Normal mode (real audio) |
| 0 | 0 | 1 | Single Tone — 440 Hz |
| 0 | 1 | 1 | Dual Tone — 220 Hz + 3520 Hz |
| 1 | 0 | 1 | Beat Pulse — ~120 BPM |
| 1 | 1 | 1 | Sweep — bass to treble |

---

## Expected Outputs

### VGA Display

Bars use **attack/decay smoothing** — instant rise on peaks, 75% decay per frame. This prevents flickering on beat pulse and sweep signals.

| Signal | Spectrum Mode | Bar Graph Mode |
|--------|--------------|----------------|
| **Single Tone** (440 Hz) | One sharp spike near bar 3–4 | Bass bar lit (~60–80%), mid/treble near 0 |
| **Dual Tone** (220 Hz + 3520 Hz) | Two spikes: bar ~2 (220 Hz) and bar ~28 (3520 Hz) | Bass bar + treble bar lit, mid near 0 |
| **Beat Pulse** (~120 BPM) | Bar 1 pulses up on beat, decays smoothly between beats. White border flashes. | Bass bar pulses with smooth decay, flashes white on each beat |
| **Sweep** (bass → treble) | Smooth left-to-right sweep with visible falloff tails | All 3 bars animate smoothly as frequency sweeps through each band |

### BPM (HEX5–HEX3)

| Signal | Expected BPM | Notes |
|--------|-------------|-------|
| **Single Tone** | 0 | No periodic beat, BPM stays at 0 |
| **Dual Tone** | 0 | No beat rhythm detected |
| **Beat Pulse** | ~117–120 | Converges over ~5–10 seconds. Uses `SAMPLE_RATE`-derived constant (469 at 8 kHz) |
| **Sweep** | Unstable | Not meaningful — no consistent beat |

### Energy (HEX2–HEX0)

| Signal | Expected Energy | Notes |
|--------|----------------|-------|
| **Single Tone** | ~30–50 | Low amplitude (~6% full-scale), tests lower energy range |
| **Dual Tone** | ~30–50 | Two tones at low amplitude, tests band separation at low energy |
| **Beat Pulse** | ~70–100 on beat, ~0–10 between beats | High amplitude burst, tests upper energy range and beat flash trigger |
| **Sweep** | ~70–100 (slowly varying) | High amplitude, tests full spectrum display at high energy |

---

## Band Boundaries (from `band_extract`)

At 8 kHz (CPUlator), bin resolution = 7.8 Hz/bin, Nyquist = 4 kHz.

| Band | Bins | Freq Range (8 kHz) | Freq Range (44.1 kHz) | Notes |
|------|------|-------------------|----------------------|-------|
| Bass | 0–37 | 0–289 Hz | 0–1590 Hz | 220 Hz and 440 Hz fall here at 8 kHz |
| Mid | 38–256 | 297–2000 Hz | 1633–11025 Hz | 3520 Hz falls here at 44.1 kHz; above Nyquist at 8 kHz |
| Treble | 257–511 | 2008–3992 Hz | 11068–22007 Hz | At 8 kHz this covers the upper half of audible range |

---

## Notes

- **Beat flash** (white VGA border) fires when `bass_energy > bass_avg * 1.5` and `bass_energy > 10`.
- **BPM smoothing**: uses exponential average `bpm = (3 * old + new) / 4` — needs several beats to stabilize.
- **3520 Hz** from the dual-tone signal lands in the **mid** band, not treble, despite the name. The bar graph mid bar will light up, not treble.
- Press **R** on the PS/2 keyboard to reset BPM history when switching between test signals.
