# Ultrasonic Sonar Radar

Real-time ultrasonic proximity mapping system built on the Intel DE1-SoC FPGA. An HC-SR04 sensor mounted on a servo sweeps 0–180°, and a Nios V soft processor running embedded C computes obstacle positions and renders a live polar sonar display on VGA.

[▶ Demo (coming soon)]()

---

## Overview

This project implements a scanning sonar radar — similar in concept to marine or air-traffic radar — using low-cost ultrasonic hardware and an FPGA-based soft processor. The servo sweeps the sensor through a semicircle; at each angle the HC-SR04 fires an ultrasonic pulse and measures the echo return time. Distance readings are streamed over UART to the DE1-SoC, where the Nios V processor clusters them into discrete objects, converts polar coordinates to Cartesian via precomputed lookup tables, and drives a real-time polar plot on a 320×240 VGA display.

---

## System Architecture

```
┌──────────────┐    UART (9600 baud)    ┌──────────────────────────┐
│  Arduino Uno │ ──────────────────────► │  DE1-SoC (Cyclone V)     │
│              │                         │                          │
│  HC-SR04     │                         │  Nios V Soft Processor   │
│  + Servo     │                         │  ├─ UART RX (parsed)     │
│              │                         │  ├─ Object clustering     │
│  Sensor      │                         │  ├─ Polar → Cartesian    │
│  timing &    │                         │  ├─ Trig lookup tables   │
│  servo PWM   │                         │  └─ VGA pixel buffer     │
└──────────────┘                         │                          │
       5V logic                          │  VGA Controller ──► Monitor
                                         └──────────────────────────┘
                    5V ↔ 3.3V level shifting
                    (voltage divider on Echo/RX lines)
```

**Arduino front-end** — handles the timing-critical HC-SR04 trigger/echo measurement (10 μs trigger pulse, echo pulse width → distance) and servo PWM. Sends `angle,distance\n` pairs over UART at 9600 baud.

**DE1-SoC back-end** — Nios V reads UART data via interrupt-driven I/O, parses ASCII distance values, runs nearest-neighbor clustering to group readings into objects, and renders the polar map on VGA.

---

## Key Technical Details

### Sensor Interfacing & Level Shifting
The HC-SR04 operates at 5V; the Cyclone V GPIO is 3.3V. A resistive voltage divider (1 kΩ / 2 kΩ) on the Echo → FPGA path drops the signal to safe levels. The 3.3V trigger output from the FPGA is sufficient to register as logic HIGH on the 5V sensor.

### Object Clustering Algorithm
Raw (angle, distance) readings are grouped into discrete objects using a nearest-neighbor approach with configurable thresholds:
- **Angular tolerance:** ±5° — readings at similar angles are candidates for the same object
- **Distance tolerance:** ±1 cm — readings at similar distances confirm a match
- **Border filtering:** readings beyond a configurable maximum range are discarded as noise

Objects are stored in a fixed-size struct array with fields for distance, angle, ID, and active status. A periodic cleanup pass merges duplicates and expires stale objects that haven't been updated in recent sweeps.

### Coordinate Conversion
Polar-to-Cartesian conversion uses precomputed integer sin/cos lookup tables (0–180°, scaled ×1000) to avoid expensive floating-point trig calls on the Nios V. Each conversion reduces to a table lookup and an integer multiply.

### VGA Rendering
A real-time polar sonar display rendered to the 320×240 VGA pixel buffer. The sweep line rotates to show the current sensor heading; detected objects are plotted as points with distance-proportional positioning from the origin.

---

## Hardware

| Component | Role |
|---|---|
| DE1-SoC (Cyclone V FPGA) | Nios V processor, VGA controller, UART peripheral |
| HC-SR04 Ultrasonic Sensor | Time-of-flight distance measurement (2 cm – 400 cm) |
| SG90 Micro Servo | 0–180° angular sweep for the sensor |
| Arduino Uno R3 | Sensor timing, servo PWM, UART bridge to FPGA |
| Voltage Divider (1 kΩ + 2 kΩ) | 5V → 3.3V level shifting on UART/Echo lines |

---

## Software Structure

```
sonar-radar/
├── arduino/
│   └── sonar_sweep.ino        # HC-SR04 timing + servo + UART TX
├── niosv/
│   ├── main.c                 # Main loop: UART RX → clustering → VGA
│   ├── uart.c / uart.h        # Interrupt-driven UART read/write
│   ├── objects.c / objects.h   # Object array, clustering, cleanup
│   ├── trig_table.c / .h      # Precomputed sin/cos lookup (×1000)
│   └── vga.c / vga.h          # Pixel buffer writes, polar plot rendering
├── quartus/
│   ├── sonar_radar.qpf        # Quartus project
│   └── platform_designer/     # Nios V system with UART peripheral
└── README.md
```

---

## Building & Running

**Arduino:**
```bash
# Open arduino/sonar_sweep.ino in Arduino IDE
# Select Board: Arduino Uno, Port: COMx
# Upload
```

**DE1-SoC:**
```bash
# 1. Open quartus/sonar_radar.qpf in Quartus Prime
# 2. Compile: Processing → Start Compilation
# 3. Program FPGA: Tools → Programmer → Start
# 4. Build Nios V software in Eclipse / Ashling IDE
# 5. Run on hardware — VGA monitor shows the polar display
```

**Testing in CPUlator (no hardware needed):**
```
# Load the Nios V binary in CPUlator (DE1-SoC configuration)
# Type simulated readings into the JTAG UART terminal:
#   45,24.5
#   90,18.3
#   135,30.1
# The algorithm processes them as if they came from the sensor
```

---

## Skills Demonstrated

`Embedded C` `Nios V (RISC-V)` `UART Serial Communication` `Interrupt-Driven I/O` `Fixed-Point Arithmetic` `VGA Display` `Sensor Interfacing` `Level Shifting` `FPGA (Cyclone V)` `Quartus Prime` `Platform Designer` `Arduino`

---

## About

Personal project built during 2nd year ECE at the University of Toronto (Winter 2026). Sensor front-end and VGA rendering developed collaboratively; object clustering algorithm and UART integration implemented independently.

**Author:** Daniel (Lung Shing) Lee · [linkedin.com/in/lee-lung](https://linkedin.com/in/lee-lung) · [daniellee.lovable.app](https://daniellee.lovable.app)
