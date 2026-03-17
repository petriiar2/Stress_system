# Stress_system
Description of a device for measuring patient's stress and pain at a dentist's appointment

# BM-Dentist — Physiological Monitoring System
### Hardware Design & Sensor Integration

> **PCC1 Project | KTH Royal Institute of Technology | MSc Medical Engineering | Stockholm, 2026**  
> Team: Naya Nasr, Iaroslav Petrishchev, Yanzi Yu, Lalith Rahul  
> *This repository covers the hardware design, sensor selection, wiring, and embedded firmware — the physical core of the system.*

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Internal Station — Patient Side](#3-internal-station--patient-side)
4. [External Station — Dentist Side](#4-external-station--dentist-side)
5. [Sensor Reference](#5-sensor-reference)
   - [Pressure Sensor](#51-pressure-sensor--abpdrrv060mgaa5)
   - [GSR Sensor](#52-gsr-sensor--galvanic-skin-response)
   - [Moisture Sensor](#53-moisture-sensor--adafruit-stemma)
   - [PPG Sensor](#54-ppg-sensor--max30102)
6. [Microcontrollers & Communication](#6-microcontrollers--communication)
7. [Power Supply](#7-power-supply)
8. [PCB & Enclosure](#8-pcb--enclosure)
9. [Current Limitations & Future Work](#9-current-limitations--future-work)

---

## 1. Project Overview

Dental patients cannot speak during procedures. The traditional workaround — raising a hand to signal pain — forces dentists to split their attention between precise work under magnifying loupes and watching for a hand signal outside their field of view. It is also binary: the signal says *stop*, but conveys nothing about intensity or rising discomfort.

**BM-Dentist** is a Bluetooth-enabled physiological monitoring system that gives patients a richer, low-effort communication channel while providing dentists with objective, real-time data about patient comfort:

- A **squeeze ball** lets the patient signal pain by squeezing harder or with squeeze patterns (1 squeeze = mild, 2 = moderate, 3+ = severe).
- **Involuntary physiological sensors** (GSR + skin moisture) detect stress independently of the patient's conscious action — catching anxiety before it reaches the threshold of deliberate signalling.
- **LED ring indicators** on the dentist's side are positioned within their natural visual field and communicate pain and stress levels through colour and intensity without requiring the dentist to look away.
- A **web dashboard** logs all signals in real time and generates post-procedure statistics for quality improvement.

The system is designed as a prototype towards clinical use — not yet a certified medical device.

---

## 2. System Architecture

The system consists of three interconnected subsystems:

```
┌──────────────────────────────┐        BLE 5.0         ┌──────────────────────────────────────┐
│     INTERNAL STATION         │ ────────────────────►  │         EXTERNAL STATION             │
│        (Patient Side)        │                        │          (Dentist Side)              │
│                              │                        │                                      │
│  Pressure Sensor             │                        │  nRF52840 #1  ──UART──  nRF52840 #2  │
│  GSR Sensor                  │  XIAO nRF52840         │  LED Pain Ring          OLED Display │
│  Moisture Sensor             │  (BLE Peripheral)      │  LED Stress Ring        Buttons      │
│  MAX30102 (PPG)              │                        │  (BLE Central)          (BLE→USB)    │
│                              │                        │                                      │
└──────────────────────────────┘                        └──────────────┬───────────────────────┘
                                                                       │ USB Serial
                                                              ┌────────▼────────┐
                                                              │   WEB APP        │
                                                              │  (HTML/JS)       │
                                                              │  Real-time plot  │
                                                              │  Post-proc stats │
                                                              └─────────────────┘
```

**Data flow summary:**

1. Sensors → Internal XIAO nRF52840 via analog ADC / I²C
2. Internal nRF → External nRF #1 via BLE UART (20 Hz)
3. External nRF #1 → LED rings (pain & stress visualisation) + External nRF #2 via UART
4. External nRF #2 → Computer via USB Serial → Web app

The web/software layer (dashboard, statistical analysis, CSV export) was developed by other team members and is referenced here for context only.

---

## 3. Internal Station — Patient Side

The internal station is the patient-held device. It houses all sensors, the microcontroller, and the BLE radio inside a custom 3D-printed enclosure designed for comfortable grip. Sensors that must contact the patient's skin (electrodes, moisture pad) exit the enclosure via JST connectors and are attached externally.

### What it does

- Reads all four sensors at **20 Hz** via a continuous polling loop
- Packages readings into a CSV string: `pressure,gsr,moisture\n`
- Transmits over **BLE UART** to the external station
- Performs **min-max normalisation** against individually calibrated baseline values
- Drives a **status LED** (pin D10) to confirm active transmission

### Calibration

Before a procedure, a 5-second calibration sequence is triggered from the dentist side:

| Phase | Patient action | What is recorded |
|-------|---------------|-----------------|
| Resting | Hold balloon without squeezing | `P_min` — baseline pressure |
| Max squeeze | Squeeze as hard as possible | `P_max` — individual squeeze ceiling |
| Resting (continuous) | Stay relaxed | `GSR_baseline` — calm-state conductance average |

Subsequent readings are normalised as:

```
pressure_norm = (P_current - P_min) / (P_max - P_min)
stress_norm   = |GSR_current - GSR_baseline| / GSR_range
```

Patient gender and age category are also collected at calibration time (they affect pain thresholds), though in the current prototype the demographic adjustment coefficients are set to neutral (A=1.0, B=0.0) pending clinical data.

---

## 4. External Station — Dentist Side

The dentist-side station receives data wirelessly and translates it into immediate visual feedback — no screen-watching required.

### Dual-microcontroller architecture

Two XIAO nRF52840 boards are connected by a UART link (TX/RX crossed: D6↔D7):

**nRF52840 #1 — Signal processing & LED control**

- Acts as **BLE Central**: scans for and connects to the internal station (`"SensorNode"`)
- Parses the incoming CSV string and normalises sensor values
- Maps normalised pain to the number of illuminated LEDs on the **green pain ring**
- Maps normalised stress to the number of illuminated LEDs on the **blue stress ring**
- Flips stress ring colour to **red** when stress normalised value > 75% (urgent alert)
- Supports two pain-signalling modes (selectable at runtime):
  - **Direct mode**: squeeze force → proportional LED count
  - **Pattern mode**: counts squeeze edges in a sliding time window → 1 squeeze = few LEDs, 2 = moderate, 3+ = many
- Forwards raw data to nRF #2 via UART during working mode

**nRF52840 #2 — Interface & data relay**

- Drives the **OLED menu display** and three navigation buttons (Up / Down / Enter)
- Sends configuration commands to nRF #1:
  - `CALIB,<gender>,<age>` — start calibration
  - `RESET_CALIB` — wipe calibration
  - `WORKCFG` — start working mode with selected options
  - `WORK_END` — stop working mode, return to sleep
- Relays raw sensor data to the computer via BLE → USB Serial

### Operating modes

| Mode | Description |
|------|-------------|
| **Sleep** | BLE connection active, command exchange only — no data streaming or LED updates |
| **Working** | Full data streaming at 20 Hz, LED feedback active, data sent to computer |

---

## 5. Sensor Reference

### 5.1 Pressure Sensor — ABPDRRV060MGAA5

| Property | Value |
|----------|-------|
| Type | Analog MEMS pressure sensor |
| Interface | Analog voltage → ADC pin D2 |
| Supply | 3.3 V |
| ADC resolution | 12-bit (0–4095) |
| Physical connection | Flexible tubing → squeeze balloon (water bottle) |
| Sampling rate | 20 Hz |

**Purpose:** The primary intentional pain-signalling channel. The patient squeezes a balloon; the pressure sensor sits inside the enclosure connected by tubing, so there is no direct electronic contact with the patient's hand. This improves hygiene and allows the balloon to be replaced/sterilised between patients.

**What we get:** An analogue voltage proportional to balloon pressure. After calibration, this maps linearly to a 0–100% pain scale. In pattern mode, rising edges are counted over a sliding window to detect discrete squeeze events.

**Why this sensor:** The specific sensor was supplied to the team and is well-matched to the use case — it captures a deliberate, high signal-to-noise signal that requires no inference to interpret.

---

### 5.2 GSR Sensor — Galvanic Skin Response

| Property | Value |
|----------|-------|
| Type | Skin conductance, analog |
| Interface | Analog voltage → ADC pin D0 |
| Supply | 3.3 V |
| Electrode placement | Index finger + middle finger |
| Sampling rate | 20 Hz |

**Purpose:** Continuous, involuntary stress monitoring. Galvanic skin response measures changes in sweat gland activity driven by the sympathetic nervous system. It captures anticipatory anxiety — the patient may start showing stress before consciously deciding to squeeze.

**What we get:** A slow-varying analogue voltage that rises as skin conductance increases (higher stress → more perspiration → lower resistance → higher conductance). The deviation from the calibrated calm baseline is used as the stress metric.

**Why this sensor:** GSR is a well-established proxy for sympathetic arousal. Practical alternatives (e.g., dedicated EDA modules) had longer delivery times; the chosen module was available immediately and is straightforward to integrate with a single ADC pin.

**Wiring:** Two finger-clip electrodes connect to the sensor module via wires that exit the enclosure through a JST socket. The module output goes to pin D0.

---

### 5.3 Moisture Sensor — Adafruit STEMMA

| Property | Value |
|----------|-------|
| Type | Capacitive soil moisture sensor (repurposed) |
| Interface | Analog voltage → ADC pin D1 |
| Supply | 3.3 V |
| Placement | Palm contact (external, JST connector) |
| Sampling rate | 20 Hz |

**Purpose:** Secondary stress indicator via perspiration. Stress-induced sweating changes the dielectric properties of the skin surface, which the capacitive sensor detects as a change in output voltage.

**What we get:** An analogue voltage that varies with surface moisture. In our pilot validation, moisture features showed near-complete binary separation between no-stress and stress conditions (AUC ≈ 0.02–0.035, Cliff's δ ≈ −0.93 to −0.96 — moisture *decreases* in our sensor's output with higher sweat, reflecting sensor polarity).

**Important caveat:** This sensor is designed for soil, not human skin. Its readings are highly sensitive to contact area and the rigid structure makes consistent placement difficult. It is treated as an **exploratory channel** in the current prototype. A future iteration should replace it with a skin-validated conductance or hydration sensor.

**Wiring:** The sensor pad exits the enclosure via JST connector and is held against the patient's palm.

---

### 5.4 PPG Sensor — MAX30102

| Property | Value |
|----------|-------|
| Type | Photoplethysmographic (optical heart rate + SpO₂) |
| Interface | I²C → pins D4 (SDA) / D5 (SCL) |
| Supply | 3.3 V |
| Interrupt pin | D3 (INT, optional) |

**Purpose:** Heart rate and pulse-related metrics are strong indicators of physiological stress. HRV (heart rate variability) in particular is sensitive to autonomic nervous system state.

**Current status:** ⚠️ **Not used in the current validated prototype.** One of the two purchased modules was defective (internal short). The second produced unstable readings. The JST connector and I²C lines remain present on the PCB for future integration. Results in this report do not rely on PPG data.

**Why this sensor:** Selected primarily on cost, feature set, and community support under a limited budget. A future iteration should source a verified module or use an alternative such as the MAX86150.

---

## 6. Microcontrollers & Communication

### Seeed Studio XIAO nRF52840

Used for all three microcontroller roles (internal station + both external boards).

| Spec | Value |
|------|-------|
| CPU | ARM Cortex-M4 @ 64 MHz |
| BLE | 5.0 integrated |
| ADC | 12-bit, 6 analog pins |
| Size | 20 × 17.5 mm |
| Supply | 3.3 V logic; powered via VBUS (5 V) |
| I²C | Hardware I²C on D4/D5 |
| UART | Hardware Serial1 on D6/D7 |

### BLE Configuration

- Internal board advertises as peripheral under name `"SensorNode"`
- External board #1 acts as central, scans and connects to `"SensorNode"`
- BLE UART service emulates serial over Bluetooth
- Data transmitted at 20 Hz; format: `"pressure,gsr,moisture\n"`
- ADC resolution set explicitly: `analogReadResolution(12)`

### UART Between External Boards

- Pins: `D6(#1) → D7(#2)`, `D7(#1) → D6(#2)` (TX/RX crossed)
- Baud rate: **115200**
- Commands flow #2 → #1; raw data flows #1 → #2
- In sleep mode: command exchange only. In working mode: full 20 Hz data stream.

---

## 7. Power Supply

### MB102 Breadboard Power Module

Used on both internal and external stations.

| Output | Used for |
|--------|----------|
| 5 V | XIAO VBUS pin, LED rings (5 V NeoPixel-type) |
| 3.3 V | All sensors (pressure, GSR, moisture, PPG) |

- Each XIAO board provides its own 3.3 V rail from its onboard regulator — two separate 3.3 V rails on the external station to distribute load.
- All boards and sensors share a **common ground**.
- MB102 includes overcharge/overdischarge protection.

---

## 8. PCB & Enclosure

### PCB Design

Schematics and PCB layouts were designed in **KiCad**. Key design decisions:

- **JST connectors** used extensively throughout — sensors on the internal board and interface elements on the external board are all physically external to the PCB, so connectors allow clean cable management and easy swapping.
- PCB fabrication within university facilities was not feasible in the project timeline, so components were soldered onto **prototyping (perfboard) boards** following the KiCad layout.

KiCad project files are included in `/hardware/kicad/`.

### Enclosure

Designed in **Solid Edge CAD**. Simple box form with openings for:
- Power cable
- Sensor cable bundle (JST exit points)
- Interface elements (buttons, display) on external box surface

Due to 3D printer availability constraints during the holiday period, only the **internal station box** was printed for the submission deadline. The external station box remains as a CAD design.

- PCBs are fixed inside using **gomme fix** (repositionable adhesive) — easy to remove for debugging.
- CAD files are in `/hardware/enclosure/`.

---

## 9. Current Limitations & Future Work

| Area | Current state | Planned improvement |
|------|--------------|---------------------|
| PPG (heart rate) | Unstable — excluded from prototype | Source verified MAX30102 or MAX86150; repeat validation |
| Moisture sensor | Exploratory — not skin-specific | Replace with validated skin conductance/hydration sensor |
| Demographic calibration | Coefficients set to neutral (A=1, B=0) | Add clinically derived thresholds per gender/age group |
| Clinical validation | Simulated stress (video induction), 1 subject | Ethically approved trials with real dental patients |
| Enclosure | Internal box only printed | Print external box; ergonomic redesign for sterile grip |
| Accessibility | Hand squeeze only | Add foot-operated or capacitive-touch alternative inputs |
| Sterilisation | Not yet validated | Material selection and sterilisation protocol testing |

---

*For questions about the hardware design specifically, feel free to open an issue or reach out directly.*
