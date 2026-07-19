# Parkinson's Movement Classification System

Real-time classification of Parkinson's-related movement disorders on an **Adafruit Circuit Playground Classic** using FFT-based frequency analysis. The device distinguishes between resting tremor, dyskinesia, and normal movement, and communicates the result through color-coded NeoPixel LEDs scaled to movement intensity.

---

## Overview

Parkinson's disease produces two distinct involuntary movement patterns that appear at different frequency bands:

| Movement Type | Frequency Range | LED Color |
|---|---|---|
| Normal | — | Green |
| Tremor | 3–5 Hz | Yellow |
| Dyskinesia | 5–7 Hz | Red |
| Both | 3–7 Hz | Purple |

The device continuously samples the onboard 3-axis accelerometer, computes the vector magnitude, runs an FFT, and classifies the dominant frequency band — all within a 3-second window.

---

## Hardware

- **Adafruit Circuit Playground Classic** (ATmega32u4, onboard accelerometer + 10 NeoPixels)

No additional components required. The board's built-in accelerometer and LED ring are the only hardware needed.

---

## Signal Processing Pipeline

```
Accelerometer (X, Y, Z)
        |
  Vector Magnitude  ──  removes orientation dependence
        |
  64-sample buffer  ──  sampled at ~21.33 Hz over 3 seconds
        |
    DC Removal       ──  strips the gravity component
        |
  Hann Windowing     ──  reduces spectral leakage
        |
      FFT            ──  64-point real FFT (~0.333 Hz/bin)
        |
  Moving Average     ──  smooths the magnitude spectrum (window = 2)
        |
  Energy Ratio       ──  tremorEnergy / totalEnergy, dyskinesiaEnergy / totalEnergy
        |
  Classification     ──  label + intensity → LED color + brightness
```

### Classification Logic

- **Minimum energy gate:** total spectral energy must exceed 50 to register any classification; below this the signal is treated as rest/normal.
- **Tremor:** energy ratio in the 3–5 Hz band ≥ 0.33
- **Dyskinesia:** energy ratio in the 5–7 Hz band ≥ 0.33
- **Both:** both bands carry ≥ 0.16 of total energy simultaneously
- **Intensity** maps directly to NeoPixel brightness, giving a continuous severity indication within each classification.

---

## Project Structure

```
Embedded Project/
├── src/
│   ├── main.cpp                    # Arduino setup/loop
│   ├── tremor_detection/
│   │   ├── tremor.cpp              # FFT pipeline and classifier
│   │   └── tremor.h
│   └── led_logic/
│       ├── led.cpp                 # NeoPixel color/brightness handlers
│       └── led.h
└── platformio.ini                  # Board and library config
```

---

## Dependencies

Managed automatically by PlatformIO:

| Library | Version |
|---|---|
| Adafruit Circuit Playground | ^1.12.0 |
| ArduinoFFT | ^2.0.4 |

---

## Build & Flash

Requires [PlatformIO](https://platformio.org/) (VS Code extension or CLI).

```bash
# Build
pio run

# Upload to connected Circuit Playground
pio run --target upload
```

---

## Design Decisions

- **Vector magnitude over single axis** — makes the classifier orientation-agnostic so the device works regardless of how it is worn or positioned.
- **Hann window** — chosen over rectangular and Hamming windows during testing; it yielded fewer spectral leakage artifacts on short, non-stationary motion bursts.
- **64-sample FFT at ~21.33 Hz** — balances frequency resolution (~0.333 Hz/bin, enough to separate the 3–5 Hz and 5–7 Hz bands) against the 3-second sampling window needed for real-time feedback.
- **Energy ratio thresholds** — normalizing by total energy makes the classifier amplitude-invariant; it responds to *what kind* of movement, not just *how much*.
