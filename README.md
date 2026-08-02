# Automotive Dual-CAN Gateway, Filter & Telemetry Board

[![KiCad Version](https://img.shields.io/badge/KiCad-v8.0%2B-blue.svg)](https://kicad.org)
[![Hardware License](https://img.shields.io/badge/License-CERN--OHL--P-green.svg)](https://cern-ohl.web.cern.ch/)
[![Architecture](https://img.shields.io/badge/Architecture-Dual--STM32%20%2B%20ESP32-orange.svg)]()
[![Platform](https://img.shields.io/badge/Platform-ADAM%20Hardware%20Engine-purple.svg)]()

An automotive-grade multi-purpose CAN module, Man-in-the-Middle (MitM) filter, and wireless telemetry gateway. Designed by **ADAM Hardware Engine**, this board bridges both **High-Speed CAN** and **Fault-Tolerant Low-Speed CAN** networks with dedicated real-time microcontrollers and an ESP32 Wi-Fi/Bluetooth gateway.

![Automotive Dual-CAN Gateway PCB Preview](preview.png)

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Operating Modes](#-operating-modes)
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Hardware & Functional Zones](#-hardware--functional-zones)
- [Pinout & Interface Definitions](#-pinout--interface-definitions)
- [Power Management & Auto Sleep/Wakeup](#-power-management--auto-sleepwakeup)
- [Bill of Materials (BOM) Summary](#-bill-of-materials-bom-summary)
- [PCB Layout & Physical Specifications](#-pcb-layout--physical-specifications)
- [Getting Started & Development](#-getting-started--development)
- [License & Acknowledgments](#-license--acknowledgments)

---

## 🛠 Overview

Modern vehicle architectures utilize multiple CAN physical layers and baud rates—typically **High-Speed CAN** (500 kbps / 1 Mbps for powertrain and chassis) and **Fault-Tolerant Low-Speed CAN** (up to 125 kbps for body electronics, doors, HVAC, and interior systems).

The **Automotive Dual-CAN Gateway & Filter** is a modular, high-reliability PCB platform designed for advanced automotive reverse engineering, message manipulation, custom ECU addition, and wireless telemetry. With two dedicated 32-bit ARM Cortex-M4 microcontrollers (STM32F446) and an ESP32 wireless co-processor, it provides strict real-time deterministic performance across all CAN channels alongside wireless connectivity.

---

## ⚙️ Operating Modes

The board is engineered for extreme flexibility across three primary operational deployment modes:

```
─────────────────────────────────────────────────────────────────────────────────────────────
 Mode A: Additional ECU (Listen-Only / Sniffer)
 [ Vehicle CAN Bus ] ──► ( LISTEN Channel ) ──► [ Dual-CAN Gateway ] (Passive / Wi-Fi Log)

 Mode B: Inline Man-in-the-Middle (MitM Filter)
 [ Vehicle CAN Bus ] ──► ( Input Channel ) ──► [ Real-Time Filter ] ──► ( Output Channel ) ──► [ Target ECU ]

 Mode C: Permanent In-Vehicle Installation
 [ BATT +12V / IGN ] ──► [ Auto Sleep / FT-CAN Wakeup ] ──► Ultra-Low Standby Power
─────────────────────────────────────────────────────────────────────────────────────────────
```

### 1. Additional ECU Mode (Listen / Sniffer / Telemetry)
- **Operation**: Connects to the target CAN bus using **only the LISTEN channels** (`CAN-HS-1` and/or `CAN-FT-1`).
- **Use Case**: Acts as a passive sniffer, logging tool, or custom secondary ECU (e.g. adding aftermarket sensors, ambient lighting control, or performance telemetry).
- **Advantage**: The transmit lines on the second channel remain unused or isolated, preventing any accidental bus disruption or error frame generation on the factory vehicle network.

### 2. Inline Man-in-the-Middle (MitM / Message Filter) Mode
- **Operation**: The board is wired **in series between the vehicle bus and an OEM ECU** (e.g., placing `CAN-HS-1` on the main bus side and `CAN-HS-2` on the target ECU side).
- **Use Case**: Real-time filtering, message blocking, payload modification, speed limit removal, ID translation, or diagnostic command injection.
- **Hardware Failsafe / Bypass**: Features physical jumper headers (`JP-MITM` for High-Speed, `JP-MITL` for Fault-Tolerant) allowing direct physical bypass lines to pass signals straight through during setup or failsafe recovery.

### 3. Permanent In-Vehicle Installation (Auto Sleep & Wakeup)
- **Operation**: Designed to be permanently wired into the vehicle's unswitched battery supply (`ALWAYS_ON` / `12V_IN`).
- **Use Case**: Invisible, permanent installation inside the vehicle dashboard, cluster, or door trim.
- **Power Management**: Utilizing the built-in low-power standby modes of the `TJA1055T` Fault-Tolerant transceivers and dedicated power circuitry, the module automatically powers down when vehicle CAN activity stops and wakes up instantaneously when network activity is detected on the FT-CAN bus.

---

## ✨ Key Features

- **Dual STM32F446 Real-Time Filtering Core**:
  - Two ARM Cortex-M4 microcontrollers running at up to 180 MHz (512 KB Flash, 128 KB SRAM each).
  - MCU 1 dedicated to Low-Speed Fault-Tolerant CAN processing.
  - MCU 2 dedicated to High-Speed CAN processing.
- **Multi-Physical Layer Transceiver Matrix**:
  - **High-Speed CAN (up to 5 Mbps, CAN FD Ready)**: 2× NXP `TJA1044GT-3` / `TJA1051T-3` transceivers.
  - **Fault-Tolerant Low-Speed CAN**: 2× NXP `TJA1055T` transceivers supporting single-wire fallback mode and bus wake-up detection.
- **Wireless Telemetry & Control Co-Processor**:
  - `ESP32-WROOM-32E` co-processor with Wi-Fi (802.11 b/g/n) and Bluetooth / BLE.
  - Concurrent dual high-speed UART links connecting ESP32 to MCU 1 and MCU 2 independently.
  - Hosts real-time web dashboards, WebSocket streams, or CAN-over-IP telemetry bridges.
- **Automotive Power Supply & Protection**:
  - Wide input voltage range from 12V automotive battery and ignition lines (`12V_IN`, `ALWAYS_ON`, `AUTO_POWER`).
  - Active reverse-polarity protection using a P-Channel MOSFET (`AO3401A`).
  - Overvoltage and surge protection via Transient Voltage Suppressor (`SMCJ24CA` TVS + `SS34` Schottky).
  - High-efficiency primary step-down buck converter (`LM2596S-5` @ 5V, 3A capacity).
  - Secondary low-noise LDO regulator (`AMS1117-3.3` @ 3.3V, 1A capacity).
- **Diagnostic & Bypass Jumpers**:
  - Hardware MitM bypass headers (`JP-MITM`, `JP-MITL`).
  - Standardized SWD debug headers for both STM32 microcontrollers.
  - Serial programming and boot selection headers for the ESP32 module.

---

## 📐 System Architecture

```
                                  AUTOMOTIVE POWER (12V BATT / IGN)
                                                  │
                            ┌─────────────────────┴─────────────────────┐
                            ▼                                           ▼
                 [ AO3401A Reverse MOSFET ]               [ TVS Surge Clamping ]
                            │                                           │
                            └─────────────────────┬─────────────────────┘
                                                  │
                                                  ▼
                                       [ LM2596S-5 Buck Regulator ] ──► +5V System Rail
                                                  │
                                                  ▼
                                      [ AMS1117-3.3 LDO Regulator ] ──► +3.3V Logic Rail
                                                  │
               ┌──────────────────────────────────┼──────────────────────────────────┐
               │                                  │                                  │
               ▼                                  ▼                                  ▼
    ┌──────────────────────┐           ┌──────────────────────┐           ┌──────────────────────┐
    │  STM32F446 #1 (MCU1) │           │  STM32F446 #2 (MCU2) │           │  ESP32-WROOM-32E     │
    │ Low-Speed FT-CAN MCU │           │   High-Speed CAN MCU │           │ Wireless Gateway     │
    └──────────┬───────────┘           └──────────┬───────────┘           └──────────┬───────────┘
               │ (Rx/Tx)                          │ (Rx/Tx)                          │
         ┌─────┴─────┐                      ┌─────┴─────┐              ┌─────────────┴───────────┐
         ▼           ▼                      ▼           ▼              ▼                         ▼
     [TJA1055T]  [TJA1055T]            [TJA1044]   [TJA1044]     [ UART Link 1 ]          [ UART Link 2 ]
     (FT-CAN 1)  (FT-CAN 2)            (HS-CAN 1)  (HS-CAN 2)    (ESP32 ◄► MCU1)          (ESP32 ◄► MCU2)
         │           │                      │           │              │                         │
         ▼           ▼                      ▼           ▼              ▼                         ▼
    ┌──────────────────────┐           ┌──────────────────────┐   ┌────────────────────────────────────┐
    │ Low-Speed FT-CAN Bus │           │ High-Speed CAN Bus   │   │ Wi-Fi / BLE Wireless Bridge        │
    │ (Body / Interior)    │           │ (Powertrain/Chassis) │   │ (Web UI / Telemetry / ADAM Engine) │
    └──────────────────────┘           └──────────────────────┘   └────────────────────────────────────┘
```

---

## 🔍 Hardware & Functional Zones

### Zone 1: Automotive Power Regulation & Protection
- **Input Channels**: `12V_IN`, `ALWAYS_ON`, `AUTO_POWER`, `GND`.
- **Protection**:
  - `Q1 (AO3401A)` P-channel MOSFET provides low voltage drop reverse-polarity protection.
  - `D1 (SMCJ24CA)` TVS diode absorbs load dumps and inductive voltage spikes common on automotive 12V rails.
  - `D2 (SS34)` Schottky barrier diode prevents reverse current flow.
- **Power Rails**:
  - `U8 (LM2596S-5)` high-efficiency 5V 3A buck converter steps down raw automotive battery voltage to 5V.
  - `U9 (AMS1117-3.3)` linear regulator provides noise-free 3.3V power to the STM32 MCUs, ESP32 co-processor, and transceivers.

### Zone 2: MCU #1 – Fault-Tolerant Low-Speed CAN Core
- **Microcontroller**: `U1` (`STM32F446RETx` LQFP-64).
- **Transceivers**: `U6` & `U7` (`NXP TJA1055T` SOIC-14).
- **Function**: Manages low-speed interior/body CAN communication (door controls, instrument clusters, HVAC, body control modules).
- **Pin Map**:
  - CAN1 RX: `PA11`, TX: `PA12`
  - CAN2 RX: `PB8`, TX: `PB9`
  - ESP32 Inter-MCU Bridge: `MCU1_UART_TX` / `MCU1_UART_RX`

### Zone 3: MCU #2 – High-Speed CAN Core
- **Microcontroller**: `U2` (`STM32F446RETx` LQFP-64).
- **Transceivers**: `U4` & `U5` (`NXP TJA1044GT-3` / `TJA1051T-3` SOIC-8).
- **Function**: Handles latency-critical high-bandwidth powertrain, chassis, and OBD-II diagnostic CAN networks.
- **Pin Map**:
  - CAN1 RX: `PA11`, TX: `PA12`
  - CAN2 RX: `PB8`, TX: `PB9`
  - ESP32 Inter-MCU Bridge: `MCU2_UART_TX` / `MCU2_UART_RX`

### Zone 4: ESP32 Wireless Telemetry Co-Processor
- **Module**: `U3` (`ESP32-WROOM-32E`).
- **Function**: Serves a configuration web app, WebSocket live data feed, or CAN-over-IP telemetry channel over Wi-Fi and Bluetooth. Operates dual independent UART channels to MCU 1 and MCU 2.

---

## 🔌 Pinout & Interface Definitions

### 1. Power Connector Interface
| Terminal | Signal Name | Description |
| :--- | :--- | :--- |
| **`12V1`** | `+12V_IN` | Primary +12V Automotive Power Input |
| **`ALWAYS_ON1`** | `+12V_BATT` | Constant Unswitched Battery Supply (for Sleep/Wakeup) |
| **`AUTO_POWER1`** | `+12V_IGN` | Switched Ignition Voltage Supply |
| **`GND1..3`** | `GND` | Common Chassis / System Ground |

### 2. High-Speed CAN Connectors & MitM Jumpers
| Terminal / Header | Signal Name | Description |
| :--- | :--- | :--- |
| **`CAN-HS-H1` / `CAN-HS-L1`** | HS-CAN Channel 1 | Listen / Bus Input Channel |
| **`CAN-HS-H2` / `CAN-HS-L2`** | HS-CAN Channel 2 | Filtered Output / Target ECU Channel |
| **`JP-MITM-1` / `JP-MITM-2`** | Bypass Jumpers | Hardware Pass-Through Bypass for High-Speed CAN |

### 3. Fault-Tolerant Low-Speed CAN Connectors & MitM Jumpers
| Terminal / Header | Signal Name | Description |
| :--- | :--- | :--- |
| **`CAN-FT-H1` / `CAN-FT-L1`** | FT-CAN Channel 1 | Listen / Bus Input Channel |
| **`CAN-FT-H2` / `CAN-FT-L2`** | FT-CAN Channel 2 | Filtered Output / Target ECU Channel |
| **`JP-MITL-1` / `JP-MITL-2`** | Bypass Jumpers | Hardware Pass-Through Bypass for Fault-Tolerant CAN |

### 4. Programming & Debug Headers
- **STM32 MCU #1 (Low-Speed)**: `STM_1_SWCLK1`, `STM_1_SWID1`, `STM_1_NRST1`, `GND`
- **STM32 MCU #2 (High-Speed)**: `STM_2_SWCLK1`, `STM_2_SWID1`, `STM_2_NRST1`, `GND`
- **ESP32 Co-Processor**: `ESP_TXD0`, `ESP_RXD0`, `ESP_EN1`, `ESP_ID0` (GPIO0 Boot Mode)

---

## ⚡ Power Management & Auto Sleep/Wakeup

The module is specifically optimized for permanent installation inside modern vehicles without running down the battery:

1. **Auto Sleep Trigger**: When the vehicle is parked and CAN bus activity ceases, the `TJA1055T` Fault-Tolerant CAN transceivers automatically transition into low-power standby mode.
2. **Low-Power State**: Microcontroller power consumption is minimized using low-power sleep modes, disabling non-essential peripherals while maintaining context in RAM.
3. **Instantaneous Wakeup**: Upon detecting a dominant state transition or bus activity on the FT-CAN bus, the transceiver signals an interrupt that wakes up the system, restoring full operational mode within milliseconds.

---

## 📦 Bill of Materials (BOM) Summary

| Ref Des | Component / Value | Package / Footprint | Qty | Description |
| :--- | :--- | :--- | :---: | :--- |
| **U1, U2** | `STM32F446RETx` | LQFP-64 (10×10mm) | 2 | 32-bit ARM Cortex-M4 MCU (180 MHz, 512KB Flash) |
| **U3** | `ESP32-WROOM-32E` | Module (18×25.5mm) | 1 | Dual-Core Wi-Fi & BLE Co-Processor |
| **U4, U5** | `TJA1044GT-3` | SOIC-8 | 2 | High-Speed CAN Transceiver (CAN FD ready) |
| **U6, U7** | `TJA1055T` | SOIC-14 | 2 | Fault-Tolerant Low-Speed CAN Transceiver |
| **U8** | `LM2596S-5` | TO-263-5 | 1 | 5V 3A Switching Buck Converter |
| **U9** | `AMS1117-3.3` | SOT-223 | 1 | 3.3V 1A Linear LDO Regulator |
| **Q1** | `AO3401A` | SOT-23 | 1 | P-Channel MOSFET (Reverse Polarity Protection) |
| **Q2** | `2N7002` | SOT-23 | 1 | N-Channel Signal MOSFET |
| **D1** | `SMCJ24CA` | SMA / D_SMA | 1 | Transient Voltage Suppression (TVS) Diode |
| **D2** | `SS34` | D_SMA | 1 | 3A 40V Schottky Barrier Rectifier |
| **D3, D4** | `BAT54C / BAT54A` | SOT-23 | 2 | Dual Schottky Barrier Diodes |
| **L1** | `33-68 uH` | Radial D8.7mm | 1 | Power Inductor for Buck Converter |
| **FB1, FB2**| `Ferrite Bead` | SMD 0805 | 2 | Power Rail EMI Filtering Beads |
| **R5, R6** | `120 Ω` | SMD 0805 | 2 | 120 Ω High-Speed CAN Bus Termination Resistors |
| **R1 - R4** | `560 Ω` | SMD 0805 | 4 | Fault-Tolerant Low-Speed CAN Bias/Termination Resistors |
| **Passives**| `Capacitors / Resistors` | SMD 0805 | Various | Decoupling, Pull-Ups/Downs & Status LEDs |

---

## 📏 PCB Layout & Physical Specifications

- **Board Dimensions**: **165.00 mm × 105.00 mm**
- **Layer Count**: **2 Layers** (`F.Cu` Top Layer, `B.Cu` Bottom Layer)
- **Ground Planes**: Continuous ground plane fill on bottom layer for signal integrity and EMC compliance.
- **Assembly Friendly**: Standard 0805 SMD passive footprints designed for easy hand-soldering and prototype assembly.
- **Total Footprint Count**: 90 placed PCB footprints.

---

## 🚀 Getting Started & Development

### 1. Opening the KiCad Project
1. Install **KiCad 8.0** or newer.
2. Clone the repository:
   ```bash
   git clone https://github.com/dexus1337/dual-can-gateway.git
   ```
3. Open `dual-can-gateway.kicad_pro` in KiCad to view or edit the schematic and PCB layout.

### 2. Firmware Development & Flashing
- **STM32 Microcontrollers (MCU 1 & MCU 2)**:
  - Supports STM32CubeIDE, PlatformIO, or Keil.
  - Flash using an ST-LINK V2/V3 connected to `SWCLK`, `SWDIO`, `NRST`, and `GND` headers.
- **ESP32 Co-Processor**:
  - Built using ESP-IDF or Arduino-ESP32.
  - Flash using a standard 3.3V USB-to-UART adapter connected to `ESP_TXD0`, `ESP_RXD0`, `ESP_EN1`, and `ESP_ID0`.

---

## 📄 License & Acknowledgments

- **Owner**: ADAM Hardware Engine
- **License**: CERN Open Hardware Licence Version 2 - Permissive ([CERN-OHL-P](https://cern-ohl.web.cern.ch/))
- **Design / Author**: dexus1337 / ADAM Hardware Engineering Team
