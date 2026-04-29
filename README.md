# Precision Weighing Subsystem — Ratiometric Load Cell Interface

A high-resolution weighing system designed for automated food dispensing applications, built around the ADS1220 24-bit Σ-Δ ADC and STM32G431 microcontroller. Achieves **0.005 g/step resolution** through ratiometric measurement and precision analog front-end design.

---

## System Overview

```
                        ┌──────────────────────────────────────────────┐
                        │              STM32G431 MCU                   │
                        │                                              │
  ┌─────────────┐       │   ┌──────────┐    ┌──────────┐    ┌──────┐  │
  │  Tedea-1042  │──M12──│──►│ TVS+RC   │───►│ ADS1220  │───►│ SPI  │  │
  │  Load Cell   │       │   │ Filter   │    │ 24-bit   │    │ +DMA │  │
  │  (5 kg, IP67)│       │   └──────────┘    │ PGA=128× │    └──────┘  │
  └─────────────┘       │                    └────┬─────┘              │
                        │                         │                    │
                        │   ┌──────────┐    ┌─────┴─────┐             │
                        │   │TPS7A3301 │    │  REF3033   │             │
                        │   │  LDO     │    │  Voltage   │─── Excites  │
                        │   │ 4.4µVrms │    │  Reference │    Load Cell│
                        │   └──────────┘    │  ±0.05%   │             │
                        │                    └───────────┘             │
                        └──────────────────────────────────────────────┘
```

## Key Specifications

| Parameter | Value |
|---|---|
| **Resolution** | 0.005 g/step (20.4 noise-free bits) |
| **Capacity** | 5 kg (Tedea-Huntleigh 1042) |
| **ADC** | ADS1220 — 24-bit Σ-Δ, SPI, PGA = 128× |
| **Data Rate** | 20 SPS (configurable up to 2000 SPS) |
| **Excitation** | REF3033 precision reference (±0.05%, 10 ppm/°C) |
| **MCU** | STM32G431 (bare-metal, SPI + DMA) |
| **Environment** | IP67 load cell, industrial-grade components (-40 to +125°C) |
| **Accuracy** | ±1 g (system level, after calibration) |

---

## Signal Chain

```
Food on pan
    │
    ▼
Load Cell (Wheatstone Bridge)
    │  Output: 0 – 6.6 mV differential
    │  (2 mV/V × 3.3V excitation)
    ▼
Shielded Cable (4-wire, PTFE jacket)
    │
    ▼
M12 Connector (IP67, gold contacts)
    │
    ▼
TVS Diodes (PESD5V0S1BSF) — ESD protection
    │
    ▼
RC Filter (100Ω + 10nF, fc ≈ 159 kHz) — EMI rejection
    │
    ▼
ADS1220 PGA (128×) — 6.6 mV → 845 mV
    │
    ▼
ADS1220 Σ-Δ ADC — 24-bit digitization
    │  50/60 Hz rejection enabled
    ▼
SPI + DMA → STM32G431
    │
    ▼
Firmware: Tare → Calibration → Moving Average (N=10)
    │
    ▼
Weight output (grams)
```

---

## Hardware Design

### Component Selection

| Component | Part Number | Why Selected |
|---|---|---|
| **Load Cell** | Tedea-Huntleigh 1042 | IP67 stainless steel, OIML C3 certified, ±0.016% creep, 300% ultimate overload |
| **ADC** | TI ADS1220 | 24-bit Σ-Δ, external reference input (ratiometric), PGA 1×–128×, SPI interface |
| **Voltage Reference** | TI REF3033 | ±0.05% accuracy, 10 ppm/°C drift, 15 µVpp noise — excites load cell AND serves as ADC reference |
| **LDO** | TI TPS7A3301 | 4.4 µVrms output noise, 72 dB PSRR — clean digital power for MCU and ADC |
| **ESD Protection** | Nexperia PESD5V0S1BSF | TVS diodes on analog inputs — clamp transients before ADC |
| **Connector** | M12 (IP67) | Screw-lock, gold contacts — reliable in humid kitchen environment |

### Ratiometric Measurement

The REF3033 simultaneously:
1. **Excites** the load cell bridge (E+ / E-)
2. **Serves as** the ADC's voltage reference (REFP0 / REFN0)

If the reference drifts by +0.1%:
- Load cell output increases by 0.1%
- ADC reference increases by 0.1% → reads 0.1% lower
- **Errors cancel → zero drift in weight reading**

This is the single most important design decision — it eliminates reference voltage drift as an error source.

### Power Architecture

```
5V Input (Robot PSU)
    │
    ├──► TPS7A3301 LDO ──► 3V3 (Digital)
    │         │                  └── STM32G431, ADS1220 DVDD, LEDs
    │         │
    └──► REF3033 ──► VREF_3V3 (Precision Analog)
                         ├── Load Cell Excitation (E+)
                         └── ADC Reference (REFP0)
```

> **VREF_3V3 and 3V3 are separate nets.** Digital 3V3 has ~4.4 µVrms noise (acceptable for logic). VREF_3V3 has 15 µVpp noise (0.1–10 Hz) — required for exciting a µV-level sensor.

---

## Firmware

### Architecture

```
main.c
├── ads1220_driver.c/h    — SPI register config, DMA read, DRDY interrupt
├── weight.c/h            — Tare, calibration, filtering, gram conversion
├── uart_cli.c/h          — Serial command interface ($TARE, $CAL, $READ)
└── system_init.c/h       — Clock, GPIO, SPI, DMA peripheral setup
```

### ADS1220 Configuration

```c
// CONFIG0: AIN0/AIN1 differential, PGA=128×, PGA enabled
ads1220_write_reg(CONFIG0, 0x0E);

// CONFIG1: 20 SPS, normal mode, continuous conversion
ads1220_write_reg(CONFIG1, 0x04);

// CONFIG2: External reference (REFP0/REFN0), 50/60 Hz rejection
ads1220_write_reg(CONFIG2, 0x40);

// CONFIG3: DRDY on dedicated pin
ads1220_write_reg(CONFIG3, 0x00);
```

### Weight Calculation

```c
// Read 24-bit signed ADC value via SPI+DMA
int32_t raw = ads1220_read();

// Apply tare offset (captured at startup with empty pan)
int32_t tared = raw - tare_offset;

// Convert to grams using calibration factor
// cal_factor = (known_weight_raw - tare_raw) / known_weight_grams
float weight_g = (float)tared / cal_factor;

// Moving average filter (N=10) for noise smoothing
float filtered = moving_avg_update(&filter, weight_g);
```

### Serial Command Interface

| Command | Function |
|---|---|
| `$TARE` | Zero the scale with current load |
| `$CAL,<grams>` | Two-point calibration with known weight |
| `$READ` | Single filtered weight reading |
| `$RATE,<sps>` | Change ADC data rate (20/45/90/175/330/600) |
| `$RAW` | Read raw ADC counts (debugging) |

---

## PCB Design

### Layout Guidelines Followed

- **Separated analog and digital ground domains** — joined at a single point under the ADC
- **REF3033 and ADS1220 decoupling caps** placed within 3 mm of power pins (100nF C0G + 10µF X7R)
- **No signal traces routed under ADS1220** on bottom layer — unbroken ground plane for clean return paths
- **Analog input traces (AIN0/AIN1)** routed away from SPI clock — minimum 2 mm clearance
- **Guard ring** around analog inputs to prevent leakage current
- **M12 connector shield** tied to chassis ground through 0Ω resistor (configurable)

### Design Rules

| Parameter | Value |
|---|---|
| Layers | 4-layer (SIG-GND-PWR-SIG) |
| Min trace width | 6 mil (signal), 15 mil (power) |
| Min clearance | 6 mil |
| Via size | 0.3 mm drill / 0.6 mm pad |
| Impedance control | Not required (low-speed SPI ≤ 2 MHz) |

---

## Load Cell vs ADC Trade-off Analysis

### Load Cell Comparison

| Model | IP Rating | Accuracy | Creep (30 min) | Overload | Score |
|---|---|---|---|---|---|
| **Tedea 1042** ✅ | **IP67** | **±0.02% (OIML C3)** | **±0.016%** | **300%** | **4.55** |
| HBM SP4MC3 | IP67 | ±0.017% | ±0.014% | 300% | 4.20 |
| TAL220B | None | ±0.1% | ±0.05% | 200% | 2.05 |

### ADC Comparison

| ADC | Resolution | Interface | Ext. Ref | PGA | Score |
|---|---|---|---|---|---|
| **ADS1220** ✅ | **24-bit (20.4 NFB)** | **SPI** | **Yes** | **1–128×** | **4.65** |
| AD7190 | 24-bit (21.7 NFB) | SPI | Yes | 1–128× | 4.55 |
| HX711 | 24-bit (18 NFB) | Proprietary | No | 64/128× | 1.65 |

---

## Resolution Budget

```
Load Cell:     5000 g full scale
Sensitivity:   2 mV/V × 3.3V = 6.6 mV full-scale output
PGA:           128× → 845 mV at ADC input
ADC:           20.4 noise-free bits → 1,384,000 effective levels

Resolution = 5000 g ÷ 1,384,000 = 0.0036 g/step ≈ 0.005 g (conservative)

At 1g load:    Output = 1.32 µV | ADC noise floor = 0.09 µV | SNR = 14.7:1 ✓
```

---

## Environment & Reliability

| Challenge | Mitigation |
|---|---|
| Kitchen humidity & grease | IP67 load cell + M12 sealed connector |
| Heat from cooking (200°C pan) | PTFE thermal insulator between pan and load cell |
| Motor vibration | Rubber isolators + digital moving-average filter |
| Mains EMI (50/60 Hz) | ADS1220 built-in 50/60 Hz rejection + shielded cable |
| ESD from user contact | TVS diodes (PESD5V0S1BSF) on all analog inputs |
| 15 kg accidental overload | 300% ultimate overload rating + mechanical hard stops |

---

## Tools Used

- **MCU IDE:** STM32CubeIDE (bare-metal, no HAL for SPI — register-level driver)
- **PCB Design:** Altium Designer
- **Simulation:** LTspice (analog front-end noise analysis)
- **Component Sourcing:** Mouser/Digikey (genuine TI, Vishay, Nexperia parts)
- **Version Control:** Git

---

## References

- [ADS1220 Datasheet](https://www.ti.com/lit/ds/symlink/ads1220.pdf) — TI, 24-bit Σ-Δ ADC
- [REF3033 Datasheet](https://www.ti.com/lit/ds/symlink/ref3033.pdf) — TI, 3.3V Precision Reference
- [TPS7A3301 Datasheet](https://www.ti.com/lit/ds/symlink/tps7a33.pdf) — TI, Low-Noise LDO
- [Tedea-Huntleigh 1042 Datasheet](https://www.vishay.com/docs/12082/1042.pdf) — Vishay, Single-Point Load Cell
- [OIML R60 Standard](https://www.oiml.org/en/files/pdf_r/r060-e00.pdf) — Metrological Regulation for Load Cells
