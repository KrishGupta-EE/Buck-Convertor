# DC-DC Buck Converter — 12V → 5V, 1A

**Portfolio Project | University of Manitoba — Electrical Engineering Co-op**  
**Domain: Power Electronics · PCB Design · Analog Control**

---

## Overview

A fully custom DC-DC buck converter designed from scratch using the LM2596-ADJ switching regulator IC. The circuit steps 12V input down to a regulated 5V output at up to 1A. There is no microcontroller or firmware — all control is implemented in analog hardware via the LM2596's internal error amplifier, oscillator, and gate driver.

This project is explicitly a power electronics and PCB design showcase, distinct from embedded or firmware work.

| Specification | Target | Simulated |
|---|---|---|
| Input Voltage | 12V DC | 12V DC |
| Output Voltage | 5V DC | 4.8V DC* |
| Output Current (max) | 1A continuous | 1A (5Ω load) |
| Switching Frequency | 150 kHz | **149.6 kHz** ✅ |
| Duty Cycle | 41.7% | **42.6%** ✅ |
| Startup Overshoot | < 10% | **22% (6.1V peak)** — see note |
| Startup Settling Time | < 1 ms | **~0.5 ms** ✅ |
| Inductor Current (steady state) | ~1A avg | **~1A avg** ✅ |
| PCB | 2-layer, 80×60 mm, JLCPCB | — |
| Total BOM Cost | ~$35 CAD | — |

> *The 4.8V steady-state output (vs. 5.0V target) is expected in this open-loop behavioral simulation. The fixed duty cycle D=0.417 does not compensate for inductor DCR and switch losses. The real LM2596 closed-loop circuit regulates to exactly 5.0V regardless of losses.
>
> *Startup overshoot of 22% is a known artifact of the underdamped LC resonance during the first few switching cycles with no load pre-charge. The LM2596's internal soft-start circuitry limits this in the real hardware.

---

## How It Works

The LM2596 drives an internal MOSFET at 150 kHz. During the ON phase, current ramps up through the inductor, storing energy in its magnetic field while supplying the load. During the OFF phase, the inductor reverses polarity and forward-biases the flyback Schottky diode, releasing stored energy into the output — keeping current flowing continuously to the load.

The output voltage is set by a resistor divider that scales V_out down to the LM2596's internal 1.23V reference. The IC's built-in error amplifier continuously adjusts PWM duty cycle in real time:

```
V_out = 1.23 × (1 + R1/R2)
D = t_on / T = V_out / V_in  (ideal)
```

For this design: R1 = 24.3 kΩ, R2 = 7.87 kΩ → V_out = 5.00V, D ≈ 42%

---

## Simulation Results

All simulations run in **LTSpice XVII** using a behavioral switch model (ideal SW + PWM source) with verified component values. Screenshots in `/simulation/`.

### V_out Transient — Startup and Steady State

![Vout Transient](simulation/vout_transient.png)

- Starts at 0V, rises and overshoots to **~6.1V** during the first 0.2ms (LC resonance on startup)
- Settles to steady-state **~4.8V** within **~0.5ms**
- Flat DC line after settling — no visible ripple at this timescale (ripple is in the mV range, resolved at higher zoom)

### Switch Node Waveform

![Switch Node](simulation/switch_node.png)

- Clean square wave switching between **0V and 12V**
- Switching frequency measured: **149.6 kHz** (T = 6.684 µs) — target was 150 kHz ✅
- Sharp edges with no sustained ringing — indicates clean behavioral model

### Duty Cycle Measurement

![On Time](simulation/on_time.png) ![Total Cycle](simulation/total_cycle_time.png)

Measured directly from the switch node waveform using LTSpice cursors:

| Parameter | Measured | Target |
|---|---|---|
| Total period T | 6.684 µs | 6.667 µs (150 kHz) |
| ON time t_on | 2.849 µs | 2.778 µs |
| Duty cycle D = t_on/T | **42.6%** | **41.7%** |
| Frequency | **149.6 kHz** | **150 kHz** |

Duty cycle within 1% of theoretical — confirms the behavioral model is running at the correct operating point.

### Inductor Current

![Inductor Current](simulation/inductor_current.png)

- Startup inrush peaks at **~4.5A** during LC resonance (first 0.5ms)
- Settles to steady-state average of **~1.0A** — matches expected output current ✅
- Continuous conduction mode (CCM) confirmed — current never reaches 0A at steady state
- Ripple visible at the 1A average line in steady state (~0.3A pk-pk as designed)

### Simulation Summary

| Check | Result | Pass |
|---|---|---|
| Switching frequency | 149.6 kHz (target: 150 kHz) | ✅ |
| Duty cycle | 42.6% (target: 41.7%) | ✅ |
| Steady-state output voltage | 4.8V (closed-loop target: 5.0V) | ✅* |
| Startup settling time | ~0.5 ms (target: < 1 ms) | ✅ |
| Steady-state inductor current | ~1.0A avg (target: 1A) | ✅ |
| Continuous conduction mode | Confirmed | ✅ |

*Open-loop behavioral model; closed-loop LM2596 hardware will regulate to 5.0V.

---

## Design Calculations

### Duty Cycle
```
D = V_out / V_in = 5 / 12 = 0.417  (41.7%)
Simulated: D = t_on / T = 2.849µs / 6.684µs = 42.6%  ✅
```

### Inductor Sizing
Target 30% current ripple relative to 1A output:

```
ΔiL = 0.30 × 1A = 0.3A
L = (Vin - Vout) × D / (ΔiL × f_sw)
L = (12 - 5) × 0.417 / (0.3 × 150,000) = 65 µH  →  use 68 µH
```

Inductor rated > 1.5A saturation, DCR < 200 mΩ, shielded SMD.

### Output Capacitor Sizing
Target ripple < 50 mV:

```
C_out = ΔiL / (8 × f_sw × ΔVout)
C_out = 0.3 / (8 × 150,000 × 0.05) = 5 µF  →  use 47 µF low-ESR
```

### Feedback Divider
LM2596-ADJ V_ref = 1.23V:

```
V_out = 1.23 × (1 + R1/R2)
R1/R2 = (5/1.23) − 1 = 3.07
R1 = 24.3 kΩ (1%), R2 = 7.87 kΩ (1%)  →  V_out = 5.027V (0.5% error)
```

1% tolerance resistors required — 5% tolerance produces unacceptable output voltage error.

---

## Bill of Materials

| # | Component | Value / Part | Qty | Est. CAD | Source |
|---|---|---|---|---|---|
| 1 | Switching IC | LM2596-ADJ | 1 | $2.00 | Digikey |
| 2 | Inductor | 68 µH, 2A shielded SMD | 1 | $3.50 | Digikey |
| 3 | Output capacitor | 47 µF 50V low-ESR | 2 | $1.50 | LCSC |
| 4 | Input capacitor | 100 µF 50V | 1 | $1.00 | LCSC |
| 5 | Ceramic decoupling | 100 nF X7R | 10 | $1.00 | LCSC |
| 6 | Flyback diode | SS34 Schottky 40V 3A | 2 | $0.80 | LCSC |
| 7 | Feedback R1 | 24.3 kΩ 1% 0805 | 2 | $0.50 | LCSC |
| 8 | Feedback R2 | 7.87 kΩ 1% 0805 | 2 | $0.50 | LCSC |
| 9 | Test load resistor | 5 Ω 5W wirewound | 1 | $1.50 | Digikey |
| 10 | Screw terminals | 2-pin & 3-pin 5mm pitch | 4 | $2.00 | LCSC |
| 11 | Pin headers | 2.54 mm male/female | 1 pk | $1.50 | LCSC |
| 12 | PCB (5 pcs) | 2-layer 80×60 mm, JLCPCB | 5 | $8.00 | JLCPCB |
| 13 | Shipping (JLCPCB) | Standard to Canada | — | $3.00 | JLCPCB |
| 14 | Shipping (Digikey) | Standard | — | $8.00 | Digikey |
| | **Total** | | | **~$35 CAD** | |

---

## PCB Layout Strategy

The switching current loop (Vin → MOSFET → Inductor → Output Cap → GND) changes at 150 kHz every cycle. A large loop area radiates EMI proportional to loop area × di/dt. The entire layout strategy is built around minimizing this loop.

**Key decisions:**
- Input capacitor, MOSFET, and flyback diode placed within 5 mm of each other — minimum switching loop area
- High-current paths (Vin, switch node, Vout) routed as copper pours, not traces
- Switch node kept as a small copper island — not a large pour — to reduce antenna area
- Feedback resistors placed far from the inductor to avoid magnetically induced noise on the high-impedance sense line
- Solid GND pour on bottom layer across the entire board — low-impedance return path for every component
- 100 nF ceramic decoupling cap directly at each IC power pin, trace < 2 mm

---

## Bench Verification (Pending PCB)

To be completed once PCB arrives from JLCPCB. Verification will use the oscilloscope in the IEEE student lab at the University of Manitoba.

**Planned probe setup:**
- CH1: V_out at output capacitor positive terminal — 50 mV/div to resolve ripple
- CH2: Switch node (MOSFET source / diode cathode / inductor input) — 5V/div
- Trigger: CH2 rising edge

**Target measurements:**
- V_out: 5.0V ± 5% (closed-loop LM2596 will regulate regardless of losses)
- Switch node duty cycle: ~42% at ~150 kHz
- Output ripple: < 100 mV pk-pk
- Efficiency η = (V_out × I_out) / (V_in × I_in) > 85%

---

## Repository Structure

```
buck-converter/
├── README.md
├── simulation/
│   ├── buck.asc                  ← LTSpice XVII schematic (open directly)
│   ├── vout_transient.png        ← V_out startup and steady-state
│   ├── inductor_current.png      ← I(L1) — startup inrush and steady state
│   ├── switch_node.png           ← V(n002) — PWM square wave at switch node
│   ├── on_time.png               ← cursor measurement: t_on = 2.849 µs
│   └── total_cycle_time.png      ← cursor measurement: T = 6.684 µs (149.6 kHz)
├── hardware/
│   ├── buck.kicad_sch            ← KiCad schematic
│   ├── buck.kicad_pcb            ← KiCad PCB layout
│   └── gerbers/                  ← fabrication files (JLCPCB)
└── docs/
    └── design_calculations.md    ← full worked equations
```

---

## Project Context

This is the second project in a deliberate analog/power portfolio build:

| Project | Skills | Status |
|---|---|---|
| ESP32 Auger Controller | Embedded firmware, I2C, motor control, GitHub | ✅ Complete |
| **DC-DC Buck Converter** | **Power electronics, PCB layout, LTSpice, analog control** | **← Current** |
| Synchronous Buck (planned) | 2nd MOSFET, gate drive timing, efficiency optimization | Planned |
| 3-Phase BLDC Inverter (planned) | Buck + gate driver PCB + firmware integration | Planned |

**Why this is relevant to power electronics roles:**
- **Manitoba Hydro:** Grid-tied inverters, HVDC converters, and substation power supplies all use switching regulator topologies. Duty cycle control, inductor sizing, and feedback compensation demonstrated here transfer directly.
- **Motor Coach Industries:** EV auxiliary power systems (12V bus from HV traction battery) use high-power buck converters. This project is a scaled-down version of that exact circuit topology.

---

## Tools Used

| Tool | Purpose |
|---|---|
| LTSpice XVII | Transient simulation, waveform verification, cursor measurements |
| KiCad 8 | Schematic capture and PCB layout |
| JLCPCB | PCB fabrication |
| Bench oscilloscope (IEEE lab, U of M) | Hardware verification (pending) |

---

*University of Manitoba — Electrical Engineering, Co-op Program*  
*Power Electronics Portfolio Project*
