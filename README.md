# Automotive Dual-CAN Gateway, Filter & Telemetry Board

[![KiCad Version](https://img.shields.io/badge/KiCad-v8.0%2B-blue.svg)](https://kicad.org)
[![Architecture](https://img.shields.io/badge/Architecture-Dual--STM32%20%2B%20ESP32-orange.svg)]()

An automotive-grade multi-purpose CAN module, Man-in-the-Middle (MitM) filter, and wireless telemetry gateway. This board bridges both **High-Speed CAN** and **Fault-Tolerant Low-Speed CAN** networks with dedicated real-time microcontrollers and an ESP32 Wi-Fi/Bluetooth co-processor.

![Automotive Dual-CAN Gateway PCB Preview](preview.png)

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Operating Modes](#-operating-modes)
- [Deployment & Vehicle Integration](#-deployment--vehicle-integration)
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Hardware & Functional Zones](#-hardware--functional-zones)
- [Pinout & Interface Definitions](#-pinout--interface-definitions)
- [Bill of Materials (BOM) Summary](#-bill-of-materials-bom-summary)
- [PCB Layout & Physical Specifications](#-pcb-layout--physical-specifications)
- [Getting Started & Development](#-getting-started--development)
- [License & Acknowledgments](#-license--acknowledgments)

---

## 🛠 Overview

Modern vehicle architectures utilize multiple CAN physical layers and baud rates—typically **High-Speed CAN** (up to 1 Mbps for powertrain and chassis) and **Fault-Tolerant Low-Speed CAN** (up to 125 kbps for body control, doors, HVAC, and interior systems).

The **Automotive Dual-CAN Gateway & Filter** is a versatile PCB platform designed for automotive reverse engineering, real-time message manipulation, secondary ECU addition, and wireless telemetry. Equipped with two 32-bit ARM Cortex-M4 microcontrollers (STM32F446) and an ESP32 wireless co-processor, it provides real-time deterministic performance across both CAN physical layers.

---

## ⚙️ Operating Modes

The board operates in two primary functional modes:

```
─────────────────────────────────────────────────────────────────────────────────────────────
 Mode 1: Additional ECU (Listen-Only)
 [ Vehicle CAN Bus ] ──► ( LISTEN Channel Active ) ──► [ Dual-CAN Gateway ] (Telemetry / Passive Log)

 Mode 2: Inline Man-in-the-Middle (MitM Filter)
 [ Vehicle CAN Bus ] ──► ( Input Channel ) ──► [ Real-Time Filter ] ──► ( Output Channel ) ──► [ Target ECU ]
─────────────────────────────────────────────────────────────────────────────────────────────
```

### 1. Additional ECU Mode (Listen-Only)
- **Operation**: Only the **LISTEN channels** (`CAN-HS-1` and/or `CAN-FT-1`) are active on the vehicle bus.
- **Function**: Acts as a secondary ECU, passive data logger, or wireless bridge.
- **Safety**: Transmit channels remain inactive or isolated, guaranteeing zero disruption or error frame generation on factory vehicle networks.

### 2. Inline Man-in-the-Middle (MitM Filter) Mode
- **Operation**: Installed in series between the vehicle bus and a specific target ECU (e.g. `CAN-HS-1` on the bus side, `CAN-HS-2` on the target ECU side).
- **Function**: Intercepts frames in real time—enabling packet filtering, message modification, frame dropping, ID translation, or custom message injection before passing data to the destination ECU.
- **Hardware Failsafe / Bypass**: Features physical bypass jumpers (`JP-MITM` for High-Speed CAN, `JP-MITL` for Fault-Tolerant CAN) for direct pass-through testing or hardware failsafe bypass.

---

## 🚗 Deployment & Vehicle Integration

Regardless of whether running in **Additional ECU** or **Inline MitM Filter** mode, the board supports two deployment options:

### External / Bench / Temporary Diagnostic Use
- Connect on-demand for temporary bus logging, ECU bench testing, diagnostic manipulation, or temporary wireless debugging.

### Permanent In-Vehicle Residence (Auto Sleep & Wakeup)
- Can be permanently wired into the vehicle's unswitched battery supply (`ALWAYS_ON` / `12V_IN`).
- **Low Power & Auto Sleep/Wake**: Uses the native low-power standby features of the `TJA1055T` Fault-Tolerant Low-Speed CAN transceivers. When vehicle CAN bus activity stops, the module automatically enters an ultra-low quiescent current standby state. When network activity is detected on the FT-CAN bus, it wakes up instantaneously to resume active processing.

---

## ✨ Key Features

- **Dual STM32F446 Microcontroller Core**:
  - Two ARM Cortex-M4 microcontrollers @ 180 MHz (512 KB Flash, 128 KB SRAM each).
  - MCU 1: Low-Speed Fault-Tolerant CAN controller.
  - MCU 2: High-Speed CAN controller.
- **Multi-Physical Layer CAN Support**:
  - **High-Speed CAN (up to 5 Mbps, CAN FD Ready)**: 2× NXP `TJA1044GT-3` / `TJA1051T-3` transceivers.
  - **Fault-Tolerant Low-Speed CAN**: 2× NXP `TJA1055T` transceivers supporting single-wire fallback and automatic wake-up detection.
- **Wireless Telemetry Co-Processor**:
  - `ESP32-WROOM-32E` co-processor with Wi-Fi (802.11 b/g/n) and Bluetooth / BLE.
  - Dual independent high-speed UART links to MCU 1 and MCU 2.
- **Automotive Power & Protection**:
  - Broad input voltage handling (+12V BATT / IGN).
  - Active reverse-polarity protection via P-Channel MOSFET (`AO3401A`).
  - Overvoltage & surge protection via TVS (`SMCJ24CA`) and Schottky rectifier (`SS34`).
  - High-efficiency primary 5V step-down buck regulator (`LM2596S-5`, 3A capacity).
  - Secondary 3.3V LDO regulator (`AMS1117-3.3`, 1A capacity).
- **Hardware Diagnostic & Bypass Jumpers**:
  - MitM bypass headers (`JP-MITM`, `JP-MITL`).
  - Standardized SWD debug headers for MCU 1 & MCU 2.
  - Serial programming & boot pins for ESP32.

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
    │ (Body / Interior)    │           │ (Powertrain/Chassis) │   │ (Web UI / Live Telemetry Stream)   │
    └──────────────────────┘           └──────────────────────┘   └────────────────────────────────────┘
```

---

## 🔍 Hardware & Functional Zones

### Zone 1: Power Regulation & Protection
- **Input Rails**: `12V_IN`, `ALWAYS_ON`, `AUTO_POWER`, `GND`.
- **Protection**:
  - `Q1 (AO3401A)` P-channel MOSFET provides reverse polarity protection.
  - `D1 (SMCJ24CA)` TVS diode clamps load dump spikes.
  - `D2 (SS34)` Schottky diode prevents back-feeding.
- **Regulators**:
  - `U8 (LM2596S-5)` 5V 3A buck converter.
  - `U9 (AMS1117-3.3)` 3.3V 1A LDO regulator.

### Zone 2: MCU #1 – Low-Speed Fault-Tolerant CAN Core
- **MCU**: `U1` (`STM32F446RETx` LQFP-64).
- **Transceivers**: `U6` & `U7` (`NXP TJA1055T` SOIC-14).
- **Function**: Handles low-speed body/interior CAN networks.
- **Pin Mapping**:
  - CAN1 RX: `PA11`, TX: `PA12`
  - CAN2 RX: `PB8`, TX: `PB9`
  - ESP32 Serial Link: `MCU1_UART_TX` / `MCU1_UART_RX`

### Zone 3: MCU #2 – High-Speed CAN Core
- **MCU**: `U2` (`STM32F446RETx` LQFP-64).
- **Transceivers**: `U4` & `U5` (`NXP TJA1044GT-3` / `TJA1051T-3` SOIC-8).
- **Function**: Handles latency-critical High-Speed CAN networks.
- **Pin Mapping**:
  - CAN1 RX: `PA11`, TX: `PA12`
  - CAN2 RX: `PB8`, TX: `PB9`
  - ESP32 Serial Link: `MCU2_UART_TX` / `MCU2_UART_RX`

### Zone 4: ESP32 Wireless Telemetry Co-Processor
- **Module**: `U3` (`ESP32-WROOM-32E`).
- **Function**: Serves configuration pages, live CAN data over WebSockets/Wi-Fi/Bluetooth, and communicates with MCU 1 and MCU 2 over independent hardware UART links.

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
- **ESP32 Co-Processor**: `ESP_TXD0`, `ESP_RXD0`, `ESP_EN1`, `ESP_ID0` (GPIO0 Boot Pin)

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
  - Developed with STM32CubeIDE, PlatformIO, or Keil.
  - Flash using an ST-LINK V2/V3 connected to `SWCLK`, `SWDIO`, `NRST`, and `GND` headers.
- **ESP32 Co-Processor**:
  - Developed using ESP-IDF or Arduino-ESP32.
  - Flash using a standard 3.3V USB-to-UART adapter connected to `ESP_TXD0`, `ESP_RXD0`, `ESP_EN1`, and `ESP_ID0`.

---
