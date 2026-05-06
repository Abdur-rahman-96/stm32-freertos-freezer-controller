# STM32 FreeRTOS Freezer Controller

An STM32 ARM Cortex-M embedded controller for a solar-powered PCM freezer, built with Embedded C and FreeRTOS. The system manages temperature regulation, battery-aware compressor scheduling, I2C EEPROM data logging, and scheduled daily data rollover — all running on a custom 6-layer PCB designed in Altium Designer.

---

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│                   STM32 ARM Cortex-M                    │
│                                                         │
│  ┌─────────────────┐    ┌──────────────────────────┐   │
│  │ Temperature Task │    │  Battery Monitor Task    │   │
│  │  (HIGH PRIORITY) │    │  (MEDIUM PRIORITY)       │   │
│  │                 │    │                          │   │
│  │ Reads NTC via   │    │ Reads SOC via ADC        │   │
│  │ ADC, wakes from │    │ Blocks compressor if     │   │
│  │ sleep on exceed │    │ battery below 20%        │   │
│  └────────┬────────┘    └──────────┬───────────────┘   │
│           │                        │                    │
│           └────────────┬───────────┘                    │
│                        ▼                                │
│              ┌──────────────────┐                       │
│              │ Compressor Control│                       │
│              │ AD5231 via SPI   │                       │
│              └──────────────────┘                       │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │            Data Logger Task (LOW PRIORITY)        │   │
│  │                                                   │   │
│  │  Logs temperature, time, date, SOC to EEPROM     │   │
│  │  via I2C every cycle                              │   │
│  │                                                   │   │
│  │  Before midnight: backs up all EEPROM data       │   │
│  │  to main server via RS-485                        │   │
│  │  After midnight: rolls over to new daily record  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
         │           │           │           │
       NTC         EEPROM     RS-485      AD5231
    Thermistor    (I2C)      (UART)    Potentiometer
```

---

## Features

- **FreeRTOS task scheduling** with priority-based execution across three concurrent tasks
- **Low-power sleep/wake management** so the STM32 enters sleep mode during idle periods and wakes on temperature threshold or timer trigger
- **Battery-aware compressor control** so the compressor activates only when battery state-of-charge is above 20%, protecting the off-grid system from deep discharge
- **Temperature-based closed-loop regulation** using NTC thermistor ADC readings and AD5231 digital potentiometer to adjust compressor setpoints dynamically
- **I2C EEPROM data logging** with timestamped temperature, battery SOC, time, and date records persisted across power cycles
- **Daily data rollover** so all operational data is backed up to the main server via RS-485 before midnight each day, after which a new date-stamped logging cycle begins
- **Custom RS-485 firmware protocol** for reliable inter-board communication with packet framing and checksum validation
- **HM-10 Bluetooth** for wireless remote monitoring and setpoint adjustment

---

## FreeRTOS Task Architecture

| Task | Priority | Trigger | Responsibility |
|---|---|---|---|
| `TempControlTask` | HIGH | Periodic timer or threshold interrupt | Reads NTC, wakes STM32, controls compressor via AD5231 |
| `BatteryMonitorTask` | MEDIUM | Periodic ADC poll | Reads battery SOC, sets compressor enable/disable flag |
| `DataLoggerTask` | LOW | Periodic timer, midnight scheduler | Logs to EEPROM, triggers daily rollover and RS-485 backup |

---

## State Machine — Compressor Control Logic

```
         ┌─────────────────────────┐
         │       SLEEP MODE        │
         │  STM32 in low-power     │
         └──────────┬──────────────┘
                    │ Timer or temp threshold wakeup
                    ▼
         ┌─────────────────────────┐
         │    READ SENSORS         │
         │  NTC temperature + SOC  │
         └──────────┬──────────────┘
                    │
         ┌──────────▼──────────────┐
         │   Temperature > Setpoint│
         │   AND Battery SOC > 20% │
         └──────────┬──────────────┘
              YES   │        NO
        ┌───────────┘           └──────────────┐
        ▼                                      ▼
┌───────────────┐                   ┌──────────────────┐
│ ACTIVATE      │                   │ INHIBIT          │
│ COMPRESSOR    │                   │ COMPRESSOR       │
│ Adjust AD5231 │                   │ Log reason       │
│ setpoint      │                   │ Return to sleep  │
└───────┬───────┘                   └──────────────────┘
        │
        ▼
┌───────────────┐
│ LOG TO EEPROM │
│ Return to     │
│ SLEEP MODE    │
└───────────────┘
```

---

## Daily Data Rollover Schedule

```
11:50 PM  →  DataLoggerTask triggers backup routine
11:55 PM  →  All EEPROM records transmitted to server via RS-485
12:00 AM  →  EEPROM old records archived and reset
12:01 AM  →  New date-stamped logging cycle begins
```

---

## Hardware Platform

| Component | Part | Interface |
|---|---|---|
| Microcontroller | STM32 ARM Cortex-M | SWD/JTAG |
| Temperature Sensor | NTC Thermistor | ADC |
| Compressor Control | AD5231 Digital Potentiometer | SPI |
| Data Storage | I2C EEPROM | I2C |
| Communication | Custom RS-485 | UART |
| Wireless | HM-10 Bluetooth Module | UART |
| Power — 5V Rail | TPS5430DDAR Buck Converter | PCB |
| Power — 3.3V Rail | AMS1117 LDO | PCB |
| Timing | Crystal Resonator | Internal |
| PCB | Custom 6-Layer Board | Altium Designer |

---

## Repository Structure

```
stm32-freertos-freezer-controller/
├── README.md
├── docs/
│   ├── system_architecture.png
│   ├── task_diagram.png
│   ├── state_machine.png
│   └── pcb_overview.png
├── firmware/
│   ├── main.c
│   ├── app_tasks.c          # FreeRTOS task definitions
│   ├── temp_control.c       # NTC reading and compressor logic
│   ├── battery_monitor.c    # ADC SOC reading and threshold logic
│   ├── eeprom_logger.c      # I2C EEPROM read/write and rollover
│   ├── rs485_comm.c         # Custom RS-485 protocol driver
│   └── scheduler.c          # Midnight rollover and backup scheduler
├── include/
│   ├── app_tasks.h
│   ├── temp_control.h
│   ├── battery_monitor.h
│   ├── eeprom_logger.h
│   └── rs485_comm.h
└── examples/
    └── pseudocode_overview.md
```

---

## Tools Used

- **STM32CubeIDE** — firmware development and peripheral configuration
- **Embedded C** — all firmware written in C
- **FreeRTOS** — task scheduling, priorities, and sleep/wake management
- **SWD/JTAG** — firmware flashing and hardware debugging
- **GDB** — step-through firmware validation and fault tracing
- **Altium Designer** — 6-layer PCB layout and schematic capture
- **Oscilloscope and Logic Analyzer** — signal validation during bring-up

---

## Skills Demonstrated

- FreeRTOS task design with priority scheduling
- Low-power embedded design with sleep/wake management
- Battery-aware firmware decision logic
- I2C peripheral driver development
- Custom RS-485 protocol implementation in firmware
- EEPROM data persistence and rollover logic
- STM32 ADC, SPI, UART, and I2C peripheral integration
- Hardware bring-up and firmware debugging using SWD/JTAG and GDB

---

## Important Note on Code

This repository contains a clean-room demonstration of the architecture, logic, and firmware patterns used in the original project. No proprietary or employer-owned source code has been uploaded. All code in this repository was written fresh for demonstration purposes, using the same engineering concepts and design decisions from the original system.

---

## Author

**Abdur Rahman** — Embedded Systems Engineer
📧 arahman14@lamar.edu | 🔗 [LinkedIn](https://www.linkedin.com/in/abdur-rahman353/)
