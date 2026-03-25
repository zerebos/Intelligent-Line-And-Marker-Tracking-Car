# Intelligent Line and Marker Tracking Car

An autonomous robotic car that follows a black line on a track and uses a Pixy camera to detect colored markers at intersections to determine navigation direction. Built on the NXP FRDM-K64F development board and written in C using the Kinetis SDK.

## Overview

The car uses a 128-pixel line scan camera (LSC) to detect and track a black line. At intersections, a Pixy color blob detection camera reads pairs of colored markers to determine which direction to turn. Steering is controlled by a servo motor and a PID controller, while two DC motors drive the wheels independently via an H-bridge circuit.

## Hardware

| Component | Description |
|---|---|
| **MCU** | NXP MK64F12 (FRDM-K64F, 120 MHz ARM Cortex-M4) |
| **Line Scan Camera** | 128-pixel 1D image sensor (ADC channel, FTM clock) |
| **Pixy Camera** | Color blob detection camera, I²C address `0x54` |
| **LCD Display** | I²C LCD with color backlight and button panel, address `0x20` |
| **Steering Servo** | PWM servo, ±90° range |
| **DC Motors** | Two independently driven wheels via H-bridge |
| **Status LEDs** | Port B22 (red), B21 (green), E26 (blue) |
| **Direction Blinkers** | Port B18 (left), B19 (right) |

### Schematics

Circuit diagrams are located in [`Documentation/Schematics/`](Documentation/Schematics/):

- `TopLevel.png` — Full system overview
- `H-bridge.PNG` — Motor driver circuit
- `Line Camera.png` — Line scan camera wiring
- `Servo.png` — Servo motor wiring
- `Stearing Complete.png` — Complete steering circuit
- `LCD_Schematic.png` — LCD display wiring
- `Voltage Regulator.PNG` — Power supply circuit

## Operating Modes

The car operates as a state machine with four modes, selectable via the LCD button panel:

| Mode | Description |
|---|---|
| **STANDBY** | Car is stopped; waiting for mode selection via LCD buttons |
| **ACCURACY** | Follows the line at reduced speed (50%) for 2 laps |
| **SPEED** | Follows the line at full speed (100%) for 2 laps |
| **DISCOVERY** | Scans left and right to locate the line, then transitions to ACCURACY mode |

The active mode is indicated by the on-board status LEDs. The LCD's left/right buttons cycle through modes and the select button confirms.

## Navigation Logic

### Line Following
The line scan camera captures a 128-pixel frame at a 15 ms interval. Each pixel is thresholded (8-bit ADC, ~0.325 V threshold) to produce a binary black/white array. The firmware finds the center of the black line and feeds it to a PID steering controller, which drives the servo to keep the line centered (setpoint = pixel 64).

When the PID output saturates (i.e., the servo is at its limit), differential motor speed control kicks in via two additional PID controllers — one per wheel — to assist with tight turns.

If the line is lost entirely, the car performs a hard turn in the last known overflow direction until the line is re-acquired.

### Intersection Handling
An intersection is detected when the measured line width spans ≥ 125 pixels. The car pauses, reads the current turning direction (supplied by the Pixy camera), executes the appropriate turn, and then resumes line following.

### Start/Stop Line Detection
A wide transverse line at the start/finish of the track is recognized by the line scan logic. The lap counter increments on each crossing; when the goal lap count is reached the car stops and returns to STANDBY.

### Pixy Marker Detection
Colored post markers on either side of the track are read every 20 ms via I²C. Two signatures are used:

| Marker configuration | Direction |
|---|---|
| Signature A left of Signature B | Turn RIGHT |
| Signature B left of Signature A | Turn LEFT |
| Two Signature A posts side by side | Continue FORWARD |
| Two Signature B posts side by side | Turn BACKWARD (U-turn) |

## Software Architecture

### Source Files

| File | Description |
|---|---|
| `main.c` | Main loop, mode dispatcher, LCD button timer (PIT2 @ 50 ms) |
| `Common.c / Common.h` | Hardware abstraction: FTM, PIT, ADC, GPIO |
| `DataTypes.h` | Shared type definitions and structs |
| `DrivingControl.c / .h` | Motor control, line following, intersection handling, blinkers |
| `LSC.c / .h` | Line scan camera driver (FTM3 clock, PIT0 SI trigger, ADC1) |
| `PID.c / .h` | Generic PID controller (steering and per-wheel motor PID) |
| `PIXY.c / .h` | Pixy camera I²C driver; marker analysis and direction decoding |
| `LCD.c / .h` | I²C LCD driver; color backlight, character output, button reading |
| `ModeControl.c / .h` | Mode state machine, LED indicators |
| `Servo.c / .h` | Servo PWM driver (FTM0 CH2, ±90°) |

### Key PID Parameters

| Controller | Kp | Ki | Kd | Sample Time |
|---|---|---|---|---|
| Steering | 45 | 10 | 0 | 15 ms |
| Motor (left/right) | 15 | 15 | 60 | 15 ms |

PID output for steering is mapped to servo angle (−90° to +90°). Motor PID outputs are mapped to PWM duty cycles (10%–100%).

### Interrupt Priority Structure

| Interrupt | Source | Priority |
|---|---|---|
| PIT0 | Line camera SI trigger (15 ms) | Default |
| FTM3 | Line camera clock (pixel clock) | Default |
| ADC1 | Line camera pixel sample | Default |
| PIT1 | Pixy read + blinker update (20 ms) | 1 |
| PIT2 | LCD button poll (50 ms) | 5 |
| PORTC | Physical mode-change button | Default |

## Repository Structure

```
├── main.c
├── Common.c
├── DataTypes.h
├── DrivingControl.c
├── LSC.c
├── ModeControl.c
├── PID.c
├── PIXY.c
├── Servo.c
├── LCD.c
├── Header Files/
│   ├── Common.h
│   ├── DataTypes.h (see root)
│   ├── DrivingControl.h
│   ├── LCD.h
│   ├── LSC.h
│   ├── ModeControl.h
│   ├── PID.h
│   ├── PIXY.h
│   └── Servo.h
├── Documentation/
│   ├── Schematics/
│   ├── SOC_Final_Report.docx
│   ├── Detailed Design I.docx
│   ├── System Design.docx
│   ├── Final Test Plan.docx
│   ├── Design Changes & Tuning.docx
│   ├── Intelligent Car Presentation.pptx
│   └── SOC_Wall_Poster.pptx
└── Demo & Proof/
    ├── SOC Final Video.mp4
    └── IMG_244x.JPG (build and run photos)
```

## Documentation

Full design documents, test plans, and a final report are available in the [`Documentation/`](Documentation/) folder. Build photos and a demonstration video are in [`Demo & Proof/`](Demo%20%26%20Proof/).

## Author

**Zack Rauen** — developed November–December 2015 as part of a Systems-on-Chip (SOC) course project.
