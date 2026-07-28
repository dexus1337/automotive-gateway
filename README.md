# Automotive Dual-CAN Filter & Wi-Fi Gateway

[![KiCad Version](https://img.shields.io/badge/KiCad-v8.0%2B-blue.svg)](https://kicad.org)
[![Hardware License](https://img.shields.io/badge/License-CERN--OHL--P-green.svg)](https://cern-ohl.web.cern.ch/)
[![Architecture](https://img.shields.io/badge/Architecture-Dual--STM32%20%2B%20ESP32-orange.svg)]()
[![Platform](https://img.shields.io/badge/Platform-ADAM%20Hardware%20Engine-purple.svg)]()

An automotive-grade dual-channel CAN bus filter, Man-in-the-Middle (MitM) payload analyzer, and wireless telemetry gateway. Designed by **ADAM Hardware Engine**, this board bridges both High-Speed CAN and Fault-Tolerant Low-Speed CAN networks with dedicated real-time microcontrollers and an ESP32 Wi-Fi/Bluetooth gateway.

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Hardware & Functional Zones](#-hardware--functional-zones)
- [Bill of Materials (BOM) Summary](#-bill-of-materials-bom-summary)
- [PCB Layout & Specifications](#-pcb-layout--specifications)
- [Pinout & Interface Definitions](#-pinout--interface-definitions)
- [Repository Structure](#-repository-structure)
- [Getting Started & Development](#-getting-started--development)
- [License & Acknowledgments](#-license--acknowledgments)

---

## 🛠 Overview

Modern automotive networks rely heavily on Controller Area Network (CAN) buses operating at different speeds and physical layers (High-Speed CAN for powertrain/chassis vs. Fault-Tolerant Low-Speed CAN for body control). 

The **Automotive Dual-CAN Filter & Wi-Fi Gateway** provides a dual-controller, isolated Man-in-the-Middle (MitM) topology:
1. **Low-Speed Fault-Tolerant CAN Filtering**: Dedicated STM32 MCU interacting with twin TJA1055 transceivers.
2. **High-Speed CAN Filtering**: Dedicated STM32 MCU interacting with twin TJA1044/TJA1051 transceivers.
3. **Wireless Telemetry Bridge**: ESP32 module routing inter-MCU UART streams to host systems or the ADAM Engine via Wi-Fi (802.11 b/g/n) or BLE.
4. **Automotive Power Conditioning**: Integrated reverse polarity protection, TVS surge clamping, and dual buck/LDO voltage regulation.

---

## ✨ Key Features

- **Dual STM32F446 Real-Time Filtering Core**:
  - Two high-performance ARM Cortex-M4 microcontrollers operating at up to 180 MHz.
  - Dedicated hardware CAN controllers running low-latency filtering, message injection, and payload manipulation routines.
- **Multi-Physical Layer CAN Support**:
  - **High-Speed CAN (up to 5 Mbps)**: 2× NXP `TJA1044GT-3` / `TJA1051T-3` transceivers.
  - **Fault-Tolerant Low-Speed CAN**: 2× NXP `TJA1055T` fault-tolerant transceivers.
- **Wireless Telemetry & Control Bridge**:
  - `ESP32-WROOM-32E` wireless co-processor for Wi-Fi and Bluetooth connectivity.
  - Dual independent high-speed UART links connecting ESP32 to MCU 1 and MCU 2.
- **Robust Automotive Power Supply**:
  - Operating Input Range: 12V Nominal Automotive Battery / Ignition rails (`12V_IN`, `ALWAYS_ON`, `AUTO_POWER`).
  - Active Reverse Polarity Protection via P-Channel MOSFET (`AO3401A`).
  - Overvoltage & Transient Voltage Suppression (TVS `SMCJ24CA` / Zener + Schottky `SS34`).
  - High-Efficiency Primary Step-Down Buck Regulator (`LM2596S-5` @ 5V, 3A capability).
  - Clean Secondary LDO Regulator (`AMS1117-3.3` @ 3.3V) for microcontrollers and transceivers.
- **Hardware MitM & Diagnostic Headers**:
  - Individual hardware bypass/test headers (`JP-MITM`, `JP-MITL`).
  - SWD programming headers for both STM32 MCUs.
  - Serial debug and flash mode access headers for ESP32.

---

## 📐 System Architecture

```
                                  AUTOMOTIVE POWER (12V)
                                             │
                       ┌─────────────────────┴─────────────────────┐
                       ▼                                           ▼
            [ AO3401A Reverse MOSFET ]               [ TVS Surge Protection ]
                       │                                           │
                       └─────────────────────┬─────────────────────┘
                                             │
                                             ▼
                                  [ LM2596S-5 Buck Regulator ] ──► 5V Power Rail
                                             │
                                             ▼
                                 [ AMS1117-3.3 LDO Regulator ] ──► 3.3V Power Rail
                                             │
           ┌─────────────────────────────────┼─────────────────────────────────┐
           │                                 │                                 │
           ▼                                 ▼                                 ▼
┌──────────────────────┐          ┌──────────────────────┐          ┌──────────────────────┐
│  STM32F446 #1 (MCU1) │          │  STM32F446 #2 (MCU2) │          │  ESP32-WROOM-32E     │
│ Low-Speed FT-CAN MCU │          │   High-Speed CAN MCU │          │ Wireless Telemetry   │
└──────────┬───────────┘          └──────────┬───────────┘          └──────────┬───────────┘
           │ (Rx/Tx)                         │ (Rx/Tx)                         │
     ┌─────┴─────┐                     ┌─────┴─────┐              ┌────────────┴───────────┐
     ▼           ▼                     ▼           ▼              ▼                        ▼
 [TJA1055T]  [TJA1055T]           [TJA1044]   [TJA1044]      [ UART Link 1 ]         [ UART Link 2 ]
 (FT-CAN 1)  (FT-CAN 2)           (HS-CAN 1)  (HS-CAN 2)     (ESP32 ◄► MCU1)         (ESP32 ◄► MCU2)
     │           │                     │           │              │                        │
     ▼           ▼                     ▼           ▼              ▼                        ▼
┌──────────────────────┐          ┌──────────────────────┐  ┌────────────────────────────────────┐
│ Low-Speed CAN Bus    │          │ High-Speed CAN Bus   │  │ Wi-Fi / BLE Wireless Gateway       │
│ (Fault-Tolerant)     │          │ (Powertrain/Chassis) │  │ (Host / ADAM Hardware Engine)      │
└──────────────────────┘          └──────────────────────┘  └────────────────────────────────────┘
```

---

## 🔍 Hardware & Functional Zones

### Zone 1: Power Management & Protection
- **Inputs**: `12V_IN`, `ALWAYS_ON`, `AUTO_POWER`, `GND`.
- **Protection Circuitry**:
  - `Q1 (AO3401A)` P-channel MOSFET ensures low voltage drop reverse-polarity immunity.
  - `D1 (SMCJ24CA / SMA Zener)` clamps inductive spikes and load dumps common in automotive electrical systems.
  - `D2 (SS34)` Schottky diode prevents back-feeding.
- **Regulators**:
  - `U8 (LM2596S-5)` switching step-down converter steps down +12V battery power to a regulated 5V system rail with low thermal dissipation.
  - `U9 (AMS1117-3.3)` converts 5V to 3.3V power for internal digital ICs.

### Zone 2: MCU #1 – Low-Speed Fault-Tolerant CAN Filter
- **Microcontroller**: `U1` (`STM32F446RETx` in LQFP-64 package).
- **Transceivers**: `U6` & `U7` (`NXP TJA1055T` Fault-Tolerant CAN Transceivers, SOIC-14).
- **Function**: Intercepts and filters messages on low-speed body/interior networks (e.g., HVAC, door modules, instrument cluster).
- **Pin Mapping**:
  - CAN1 RX: `PA11`, TX: `PA12`
  - CAN2 RX: `PB8`, TX: `PB9`
  - UART Bridge: Dedicated UART connection to ESP32 (`MCU1_UART_TX` / `MCU1_UART_RX`).

### Zone 3: MCU #2 – High-Speed CAN Filter
- **Microcontroller**: `U2` (`STM32F446RETx` in LQFP-64 package).
- **Transceivers**: `U4` & `U5` (`NXP TJA1044GT-3` / `TJA1051T-3` High-Speed CAN Transceivers, SOIC-8).
- **Function**: Handles latency-critical, high-bandwidth CAN channels (e.g., powertrain, vehicle dynamics, OBD-II diagnostic port).
- **Pin Mapping**:
  - CAN1 RX: `PA11`, TX: `PA12`
  - CAN2 RX: `PB8`, TX: `PB9`
  - UART Bridge: Dedicated UART connection to ESP32 (`MCU2_UART_TX` / `MCU2_UART_RX`).

### Zone 4: ESP32 Wireless Gateway
- **Module**: `U3` (`ESP32-WROOM-32E`).
- **Function**: Hosts a Web UI, WebSocket streaming, or CAN-over-IP bridge for the ADAM Engine. Communicates concurrently with MCU 1 and MCU 2 over dual serial channels.

---

## 📦 Bill of Materials (BOM) Summary

| Ref Des | Component / Value | Footprint / Package | Quantity | Description |
| :--- | :--- | :--- | :---: | :--- |
| **U1, U2** | `STM32F446RETx` | LQFP-64 (10×10mm) | 2 | 32-bit ARM Cortex-M4 MCU (180 MHz, 512KB Flash) |
| **U3** | `ESP32-WROOM-32E` | Module (18×25.5mm) | 1 | Wi-Fi & BLE Dual-Core Wireless Transceiver |
| **U4, U5** | `TJA1044GT-3` | SOIC-8 | 2 | High-Speed CAN Transceiver (CAN FD capable) |
| **U6, U7** | `TJA1055T` | SOIC-14 | 2 | Fault-Tolerant Low-Speed CAN Transceiver |
| **U8** | `LM2596S-5` | TO-263-5 | 1 | 5V 3A Step-Down Switching Regulator |
| **U9** | `AMS1117-3.3` | SOT-223 | 1 | 3.3V 1A Linear LDO Voltage Regulator |
| **Q1** | `AO3401A` | SOT-23 | 1 | P-Channel MOSFET (Reverse Polarity Protection) |
| **Q2** | `2N7002` | SOT-23 | 1 | N-Channel MOSFET |
| **D1** | `D_Zener / SMCJ24CA` | SMA / D_SMA | 1 | TVS Overvoltage Transient Protection Diode |
| **D2** | `SS34` | D_SMA | 1 | 3A 40V Schottky Barrier Rectifier |
| **D3** | `BAT54C` | SOT-23 | 1 | Dual Anode Schottky Barrier Diode |
| **D4** | `BAT54A` | SOT-23 | 1 | Dual Cathode Schottky Barrier Diode |
| **L1** | `33-68 uH` | Radial D8.7mm | 1 | Power Inductor for Buck Regulator |
| **FB1, FB2**| `Ferrite Bead` | SMD 0805 | 2 | Power Supply Noise Filtering Bead |
| **R5, R6** | `120 Ω` | SMD 0805 | 2 | 120 Ω High-Speed CAN Termination Resistors |
| **R1 - R4** | `560 Ω` | SMD 0805 | 4 | Fault-Tolerant CAN Termination / Bias Resistors |
| **Passives**| `R / C` | SMD 0805 | Various | Filtering Capacitors, Pull-ups, & Status Indicators |

---

## 📏 PCB Layout & Specifications

- **Dimensions**: **165.00 mm × 105.00 mm**
- **Layer Count**: **2 Layers** (`F.Cu` Top Layer, `B.Cu` Bottom Layer)
- **Copper Planes**: Continuous Ground Plane fill on Bottom layer for signal integrity and EMI control.
- **Component Packaging**: 0805 SMD passive footprints throughout for hand-soldering and prototype assembly accessibility.
- **Total PCB Footprints**: 90 placed components.

---

## 🔌 Pinout & Interface Definitions

### Power Connectors
- **`12V1`**: +12V Automotive Power Input
- **`ALWAYS_ON1`**: Constant Battery Supply (+12V BATT)
- **`AUTO_POWER1`**: Switched Ignition Power (+12V IGN)
- **`GND1`, `GND2`, `GND3`**: System Ground Return

### CAN Bus Connections
- **High-Speed CAN Header**:
  - `CAN-HS-H1` / `CAN-HS-L1`: Listen Channel (Channel 1 Input)
  - `CAN-HS-H2` / `CAN-HS-L2`: MitM / Filtered Channel (Channel 2 Output)
- **Fault-Tolerant Low-Speed CAN Header**:
  - `CAN-FT-H1` / `CAN-FT-L1`: Listen Channel (Channel 1 Input)
  - `CAN-FT-H2` / `CAN-FT-L2`: MitM / Filtered Channel (Channel 2 Output)
- **Hardware MitM Jumpers**:
  - `JP-MITM-1` / `JP-MITM-2`: High-Speed CAN MitM Hardware Bypass Jumpers
  - `JP-MITL-1` / `JP-MITL-2`: Fault-Tolerant CAN MitM Hardware Bypass Jumpers

### Programming & Debugging Test Points
- **STM32 MCU #1 (Low-Speed)**:
  - `STM_1_SWCLK1`: Serial Wire Clock
  - `STM_1_SWID1`: Serial Wire Data I/O
  - `STM_1_NRST1`: MCU Reset Line
- **STM32 MCU #2 (High-Speed)**:
  - `STM_2_SWCLK1`: Serial Wire Clock
  - `STM_2_SWID1`: Serial Wire Data I/O
  - `STM_2_NRST1`: MCU Reset Line
- **ESP32 Wireless Module**:
  - `ESP_TXD0` / `ESP_RXD0`: Serial Console & Programming UART
  - `ESP_EN1`: Chip Enable / Reset Pin
  - `ESP_ID0`: Boot Mode Selector (GPIO0)

---

## 📂 Repository Structure

```
dual-can-gateway/
├── dual-can-gateway.kicad_sch   # KiCad Schematic file (Schematic Version 8/10)
├── dual-can-gateway.kicad_pcb   # KiCad Board Layout file
├── dual-can-gateway.kicad_pro   # KiCad Project settings file
├── dual-can-gateway.kicad_prl   # KiCad Local Workspace settings
├── generate_schematic.py        # Schematic generation helper script
└── README.md                    # Project documentation (this file)
```

---

## 🚀 Getting Started & Development

### 1. Opening in KiCad
1. Install **KiCad 8.0** or newer.
2. Clone this repository:
   ```bash
   git clone https://github.com/dexus1337/dual-can-gateway.git
   ```
3. Double-click `dual-can-gateway.kicad_pro` to open the full project in KiCad.

### 2. Firmware Development Setup
- **STM32 Microcontrollers (MCU 1 & MCU 2)**:
  - Developed with STM32CubeIDE / STM32CubeHAL or PlatformIO.
  - Flash using ST-LINK V2 / V3 connected to the SWD test points (`SWCLK`, `SWDIO`, `NRST`, `GND`).
- **ESP32 Module**:
  - Developed using ESP-IDF or Arduino-ESP32.
  - Flash using a USB-to-UART adapter (3.3V logic) connected to `ESP_TXD0`, `ESP_RXD0`, `ESP_EN1`, and `ESP_ID0`.

---

## 📄 License & Acknowledgments

- **Company / Project Owner**: ADAM Hardware Engine
- **License**: CERN Open Hardware Licence Version 2 - Permissive ([CERN-OHL-P](https://cern-ohl.web.cern.ch/))
- **Author / Designer**: dexus1337 / ADAM Hardware Engineering Team

---
*Generated automatically by scanning `dual-can-gateway.kicad_sch`, `dual-can-gateway.kicad_pcb`, and project configuration files.*
