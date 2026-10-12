# Multi-Domain 4-Layer PCB (Teensy 4.0 | CAN Bus | Analog Filtering | Hall-Effect Switching)

A 4-layer mixed-signal PCB designed end to end in **Altium Designer**, from schematic capture through layout, routing, and design rule checks. Built as part of my onboarding with the **University of Alberta Autonomous Robotic Vehicle Project (ARVP)**, where the same circuit blocks are used on the team's underwater robot.

> **Fill-in guide:** anything marked `✏️ FILL IN` needs your own words or numbers. Delete the prompt text once you've written your response. Delete this note before publishing.

---

## Table of Contents

- [Overview](#overview)
- [Board at a Glance](#board-at-a-glance)
- [System Architecture](#system-architecture)
- [Subsystems](#subsystems)
- [PCB Design Details](#pcb-design-details)
- [Verification](#verification)
- [Challenges and How I Approached Them](#challenges-and-how-i-approached-them)
- [Key Takeaways](#key-takeaways)
- [Future Improvements](#future-improvements)
- [Repository Structure](#repository-structure)
- [Tools and Resources](#tools-and-resources)
- [Acknowledgments](#acknowledgments)
- [Contact](#contact)

---

## Overview

This board integrates four circuit domains around a Teensy 4.0 microcontroller: digital communication (CAN bus), I2C sensing, analog signal conditioning, and magnetically triggered high-side load switching. Each block mirrors a subsystem used in ARVP's robot (control board, sonar board, and the magnetic kill switch).

**Why this project:**
I took on ARVP’s electrical onboarding project because I wanted to learn how a real PCB gets designed, not just how circuits work on paper. Going from schematic to a manufacturable 4-layer board in Altium, the same workflow ARVP uses on its robots, was the best way to get there. It was also the requirement for joining the electrical team, which I wanted anyway.

**My role:**
I did the full design myself, start to finish. That covers the schematic (Teensy 4.0, CAN bus transceiver, I2C temperature sensor, hall-effect-triggered high-side load switch, active low-pass filter, power input, and payload connector), the 4-layer PCB layout and routing, and the design rule and electrical checks. I also simulated the filter in LTspice, did the silkscreen, and kept design documentation along the way. The team gave me a high-level block diagram, a component library, and tutorials to follow, so the system requirements and part selection came from them. The pin mapping, filter values, placement, routing, and fixing every DRC/ERC error were on me.

---

## Board at a Glance

| Item | Detail |
|---|---|
| EDA tool | Altium Designer |
| Layer count | 4 (Signal / Ground / Power / Signal) |
| Microcontroller | Teensy 4.0 |
| Supply input | +5 V via 2-pin Molex connector |
| Logic level | 3.3 V (Teensy pins are not 5 V tolerant) |
| Communication | CAN bus, I2C, plus UART/SPI/I2C payload header |
| Analog | Active low-pass filter into Teensy ADC |
| Board dimensions | 2.34 in x 1.94 in |
| Component count | 24 |

---

## System Architecture

```mermaid
flowchart LR
    PWR[+5V Power Input] --> REG[3.3V Rail]
    PWR --> CAN
    REG --> MCU[Teensy 4.0]
    MCU <-->|I2C| TEMP[Temperature Sensor]
    MCU <-->|TX/RX| CAN[CAN Transceiver]
    CAN <--> BUS[External CAN Bus]
    SIG[Noisy Analog Input] --> LPF[Active Low-Pass Filter]
    LPF --> MCU
    HALL[Hall-Effect Sensor] --> PFET[P-Channel MOSFET High-Side Switch]
    PFET --> LOAD[LED Load]
    MCU --- PAY[Payload Header: UART / SPI / I2C]
```

> ✏️ FILL IN: Replace or supplement the diagram above with your own high-level diagram if you made one, and confirm how the 3.3 V rail is generated on your board (Teensy onboard regulator or another source) so the diagram is accurate.

---

## Subsystems

### 1. Microcontroller: Teensy 4.0
Plug-in system-on-module that handles control, communication, and ADC sampling. Pin assignments were chosen from the Teensy pinout to match each peripheral's required function (I2C, CAN TX/RX, ADC, SPI, UART).

✏️ FILL IN: Which pins you assigned to what, and why.

### 2. I2C Temperature Sensor: TCN75A
Digital temperature sensor on the I2C bus.

✏️ FILL IN: I2C address, pull-up resistor values, and how you chose them.

### 3. CAN Bus: TCAN1042HV transceiver
Gives the Teensy CAN communication over a differential CAN-H / CAN-L pair on a 2-pin header. The transceiver's **VIO pin** is used to level-shift the RXD/TXD lines so the Teensy only sees 3.3 V logic while the transceiver runs from 5 V.

✏️ FILL IN: Termination approach, how you read the datasheet to set up VIO, anything you verified.

### 4. Active Low-Pass Filter: LMV321 op-amp
Non-inverting, unity-gain op-amp filter that passes a **1 kHz** sine wave and rejects **15 kHz** noise before the signal reaches a Teensy analog input. Input swings between 1 V and 3 V, so no negative supply is needed and the output stays inside the Teensy's 0 to 3.3 V range.

| Parameter | Value |
|---|---|
| Topology | ✏️ FILL IN (first-order / second-order) |
| Cutoff frequency | ✏️ FILL IN Hz |
| Resistor / capacitor values | ✏️ FILL IN |
| Simulated attenuation at 15 kHz | ✏️ FILL IN dB (LTSpice) |

✏️ FILL IN: How you picked the cutoff and component values, and add your LTSpice plot to `docs/images/`.

### 5. Hall-Effect Load Switch: DRV5033 + BSS84 P-MOSFET
A hall-effect sensor with an open-drain output controls a p-channel MOSFET used as a **high-side switch**. When a magnet is near the sensor, its output pulls low, the MOSFET turns on, and the LED lights. When the magnet is removed, the output goes high-impedance, a pull-up resistor turns the MOSFET off, and the LED goes dark. This is the same principle as ARVP's magnetic kill switch.

✏️ FILL IN: Your resistor choices (pull-up, LED current limiting) and the reasoning behind them.

### 6. Power Input and Payload Connector
- **Power:** 2-pin Molex connector supplying +5 V.
- **Payload header:** 8-pin header breaking out a UART TX/RX pair, an SPI bus (MISO, MOSI, SCLK) with a chip-select GPIO, and the shared I2C bus, so functionality can be added later without respinning the board.

---

## PCB Design Details

### Layer Stackup

| Layer | Name | Purpose |
|---|---|---|
| 1 | Top Signal | Components, signal routing, wide power traces to vias |
| 2 | Ground | Solid ground plane (reference for top-layer signals) |
| 3 | Power | +5 V and +3.3 V pours |
| 4 | Bottom Signal | Signal routing, referenced to the power plane |

### Design Rules
- Minimum clearance: 0.2 mm
- Power traces on the top layer: at least 0.5 mm (20 mil) before dropping to the power plane
- Signal traces: around 0.25 mm (10 mil)
- Bottom-layer signals routed over a +3.3 V pour so the reference plane matches the signal's voltage
- Ground connections via down to the layer 2 plane

✏️ FILL IN: Any rules you added or deviated from, and why.

### Layout Approach
✏️ FILL IN: How you placed components (Teensy first, then grouping by voltage and signal flow), where you put the analog filter relative to noisy digital sections, and where decoupling capacitors sit.

### Images

> ✏️ FILL IN: Export these from Altium and drop them in `docs/images/`, then uncomment the lines below.

<!--
![Schematic](docs/images/schematic.png)
![PCB Layout - Top](docs/images/pcb_top.png)
![PCB Layout - Bottom](docs/images/pcb_bottom.png)
![3D Render](docs/images/3d_render.png)
![Filter Simulation](docs/images/ltspice_filter.png)
-->

---

## Verification

| Check | Result |
|---|---|
| Schematic ERC | ✏️ FILL IN (errors / warnings) |
| PCB DRC | ✏️ FILL IN (violations; note any ignored items such as solder mask slivers and why) |
| LTSpice filter simulation | ✏️ FILL IN |
| Design review | ✏️ FILL IN (who reviewed, what feedback you got, what you changed) |
| Physical testing | ✏️ FILL IN, or write "Not fabricated" if that's the case. Don't claim results you don't have. |

---

## Challenges and How I Approached Them

> ✏️ FILL IN: This is the section employers read most closely. Pick 2-4 real problems. For each one, use the structure below. Specific beats impressive.

### Challenge 1: ✏️ FILL IN (short title)
- **Problem:** What went wrong or what was hard?
- **Why it mattered:** What would have happened if you got it wrong?
- **What I tried:** Including things that didn't work.
- **Solution:** What you ended up doing and why it worked.
- **What I'd do differently:**

### Challenge 2: ✏️ FILL IN
- **Problem:**
- **Why it mattered:**
- **What I tried:**
- **Solution:**
- **What I'd do differently:**

### Challenge 3: ✏️ FILL IN
- **Problem:**
- **Why it mattered:**
- **What I tried:**
- **Solution:**
- **What I'd do differently:**

<details>
<summary>Prompts if you're stuck</summary>

- Level shifting between 5 V and 3.3 V logic
- Choosing filter component values from the ARVP library
- Keeping signal return paths clean on a 4-layer stack
- Routing around the Teensy footprint
- Fixing DRC violations without breaking clearance rules
- Reading datasheets for pin functions and application circuits
- Learning Altium with no prior experience

</details>

---

## Key Takeaways

✏️ FILL IN: 3-5 bullets on what you learned technically and professionally. Example directions: reading datasheets, why stackup and reference planes matter, documenting as you work, getting feedback from design reviews.

---

## Future Improvements

✏️ FILL IN: What you'd change on a second revision. Ideas to react to, keep only what's true for you: add test points, add reverse-polarity or overcurrent protection, add ESD protection on external connectors, improve filter order, add status LEDs, tighten board outline, fabricate and bring up the board.

---

## Repository Structure

> ✏️ FILL IN: Adjust to match what you actually upload.

```
.
├── README.md
├── hardware/
│   ├── schematic/        # Altium schematic files / PDF export
│   ├── pcb/              # Altium PCB files
│   ├── gerbers/          # Manufacturing outputs (if generated)
│   └── bom/              # Bill of materials
├── simulation/
│   └── lowpass_filter.asc   # LTSpice files
└── docs/
    ├── images/
    └── design_notes.md
```

---

## Tools and Resources

- **Altium Designer**: schematic capture, PCB layout, ERC/DRC
- **LTSpice**: low-pass filter simulation
- **Datasheets**: Teensy 4.0, TCN75A, TCAN1042HV, LMV321, BSS84AK, DRV5033
- **Phil's Lab** Altium / STM32 PCB tutorial series for workflow reference

---

## Acknowledgments

- **University of Alberta ARVP** electrical team for the onboarding program and design reviews
- ✏️ FILL IN: Name anyone who reviewed your work, if they're fine with being named

*This is a personal portfolio project built during ARVP onboarding. It is not an official ARVP release, and ARVP names and logos belong to the team.*

---

## Contact

**Nathan Lin**
Electrical Engineering (Co-op), University of Alberta
[LinkedIn](https://linkedin.com/in/nathanlin27) | [GitHub](https://github.com/njlin664) | nathanlin727@gmail.com
