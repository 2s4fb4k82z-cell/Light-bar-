# Lightbar Controller

A modular, Arduino-based vehicle lightbar control system designed around separate **input**, **output**, and future **touchscreen/vehicle-data** controllers.

The current system uses:

* **Arduino Nano ESP32** — physical input controller
* **Arduino Mega 2560** — lightbar output controller

Future development will add:

* **Arduino GIGA R1 WiFi**
* **Touchscreen interface**
* **Touchscreen lightbar controls**
* **OBD-II / CAN vehicle gauges and diagnostics**

The design intentionally separates physical controls from the future touchscreen system so that physical lightbar controls can continue operating if the touchscreen or GIGA becomes unavailable.

---

# System Overview

## Arduino Nano ESP32 — Input Controller

The Nano ESP32 monitors 10 ground-triggered physical control inputs.

It is responsible for:

* Monitoring physical vehicle/control inputs
* Debouncing input signals
* Detecting ON/OFF state changes
* Sending START/STOP commands
* Sending a heartbeat (`PING`) every 5 seconds
* Communicating with the Mega over hardware UART
* Monitoring messages returned by the Mega

---

## Arduino Mega 2560 — Output Controller

The Mega is responsible for controlling the actual lightbar functions.

It:

* Receives commands from the Nano
* Controls 8 individual Traffic Advisor outputs
* Controls 5 direct lightbar outputs
* Generates Traffic Advisor patterns
* Handles Traffic Advisor priority
* Integrates left/right turn signals with the TA outputs
* Monitors the Nano heartbeat
* Provides communication-loss warning
* Shuts down all lightbar outputs after a communication timeout

---

## Arduino GIGA R1 WiFi — Planned Touchscreen / Vehicle Controller

A future Arduino GIGA R1 WiFi with touchscreen will provide an in-cab graphical interface.

The GIGA will have two primary functions:

### Lightbar Control

The touchscreen will provide an additional method of controlling the lightbar system.

The existing physical controls connected to the Nano will remain available independently.

### OBD-II / CAN Gauge Display

The GIGA will also communicate with the truck's vehicle network and display available OBD-II/CAN information.

Planned functionality includes:

* Digital gauges
* Vehicle information
* Warning indicators
* Diagnostic trouble codes
* Vehicle-specific CAN information where available

---

# Current System Architecture

```text
             PHYSICAL VEHICLE CONTROLS
                       │
                       ▼
             ┌───────────────────┐
             │  INPUT INTERFACE  │
             │    12V → 3.3V     │
             └─────────┬─────────┘
                       │
                       ▼
             ┌───────────────────┐
             │   NANO ESP32      │
             │                   │
             │ Physical Inputs   │
             │ Debouncing        │
             │ Command Generator │
             │ Heartbeat         │
             └─────────┬─────────┘
                       │
                  UART 38400
                       │
                       ▼
             ┌───────────────────┐
             │    MEGA 2560      │
             │                   │
             │ Command Processor │
             │ TA Controller     │
             │ Output Controller │
             │ Watchdog          │
             └─────────┬─────────┘
                       │
                       ▼
             ┌───────────────────┐
             │  OUTPUT DRIVERS   │
             └─────────┬─────────┘
                       │
                       ▼
                    LIGHTBAR
```

---

# Planned System Architecture

The completed system is expected to use three primary controllers:

```text
                   PHYSICAL CONTROLS
                          │
                          ▼
                 ┌─────────────────┐
                 │   NANO ESP32    │
                 │                 │
                 │ Physical Input  │
                 │ Controller      │
                 └────────┬────────┘
                          │
                          │ Commands
                          ▼
                 ┌─────────────────┐
                 │    MEGA 2560    │
                 │                 │
                 │ Lightbar Output │
                 │ Controller      │
                 └────────┬────────┘
                          │
                          ▼
                   OUTPUT DRIVERS
                          │
                          ▼
                       LIGHTBAR


                 ┌─────────────────┐
                 │  ARDUINO GIGA   │
                 │    R1 WiFi      │
                 │                 │
                 │   Touchscreen   │
                 │   Controller    │
                 └────────┬────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
                ▼                   ▼
         LIGHTBAR CONTROL       OBD-II / CAN
                │                   │
                ▼                   ▼
        NANO / CONTROL         TRUCK ECU /
            SYSTEM             VEHICLE CAN
```

The GIGA is intended to act as an **additional command source**, rather than replacing the physical Nano controls.

This prevents the touchscreen from becoming a single point of failure.

---

# Hardware Communication

## Nano ESP32 ↔ Mega 2560

The Nano and Mega communicate using hardware UART.

### UART Settings

```text
Baud Rate: 38400
Data Bits: 8
Parity:    None
Stop Bits: 1
```

### Wiring

| Nano ESP32 | Mega 2560 |
| ---------- | --------- |
| D10 TX     | D19 RX1   |
| D11 RX     | D18 TX1   |
| GND        | GND       |

```text
NANO ESP32                    MEGA 2560

D10 TX  ------------------->  D19 RX1

D11 RX  <-------------------  D18 TX1

GND     --------------------  GND
```

---

# Nano ESP32 Inputs

The Nano monitors 10 physical inputs.

All inputs use:

```cpp
INPUT_PULLUP
```

Therefore:

```text
HIGH = OFF
LOW  = ACTIVE / GROUNDED
```

## Input Pinout

| Input | Function        | Nano Pin |
| ----: | --------------- | -------- |
|     1 | OUTERS          | D2       |
|     2 | INTERMEDIATES   | D3       |
|     3 | INNERS          | D4       |
|     4 | TRAFFICBREAKERS | D5       |
|     5 | LEDS            | D6       |
|     6 | TLEFT           | D7       |
|     7 | TRIGHT          | D8       |
|     8 | TOUT            | D9       |
|     9 | LEFTSIGNAL      | A0       |
|    10 | RIGHTSIGNAL     | A1       |

---

# Input Debouncing

All 10 Nano inputs use software debouncing.

Current debounce time:

```text
50 ms
```

When an input remains changed for at least 50 ms, the Nano accepts the new state and sends the appropriate command to the Mega.

---

# Mega Output Configuration

All Mega lightbar outputs are **active LOW**.

```text
LOW  = OUTPUT ON
HIGH = OUTPUT OFF
```

The Mega GPIO pins are intended to control appropriate external driver circuitry.

They should not directly drive high-current lightbar loads.

---

# Traffic Advisor Outputs

The Traffic Advisor consists of eight individually controlled outputs.

| Mega Pin | Output | Physical Position |
| -------- | ------ | ----------------- |
| D23      | TA1    | Far Left          |
| D25      | TA2    |                   |
| D27      | TA3    |                   |
| D29      | TA4    |                   |
| D31      | TA5    |                   |
| D33      | TA6    |                   |
| D35      | TA7    |                   |
| D37      | TA8    | Far Right         |

Physical arrangement:

```text
LEFT                                      RIGHT

TA1   TA2   TA3   TA4   TA5   TA6   TA7   TA8
 │     │     │     │     │     │     │     │
D23   D25   D27   D29   D31   D33   D35   D37
```

---

# Direct Lightbar Outputs

The remaining lightbar functions use direct outputs from the Mega.

| Mega Pin | Function        |
| -------- | --------------- |
| D39      | TRAFFICBREAKERS |
| D41      | OUTERS          |
| D43      | INTERMEDIATES   |
| D45      | INNERS          |
| D47      | LEDS            |

Currently unused:

```text
D49
D51
D53
```

These pins are available for future expansion.

---

# Traffic Advisor Operation

The Traffic Advisor currently supports three modes:

* `TLEFT`
* `TRIGHT`
* `TOUT`

Only **one Traffic Advisor mode can be active at a time**.

If another TA command is activated while a pattern is running, the newest TA command takes priority.

Each Traffic Advisor step lasts:

```text
500 ms
```

Exactly **two TA outputs are ON at any given time** while a pattern is active.

---

## TLEFT

```text
TA1 + TA2
TA2 + TA3
TA3 + TA4
TA4 + TA5
TA5 + TA6
TA6 + TA7
TA7 + TA8
REPEAT
```

---

## TRIGHT

```text
TA8 + TA7
TA7 + TA6
TA6 + TA5
TA5 + TA4
TA4 + TA3
TA3 + TA2
TA2 + TA1
REPEAT
```

---

## TOUT

```text
TA4 + TA5
TA3 + TA6
TA2 + TA7
TA1 + TA8
REPEAT
```

---

# Turn Signal Integration

The two outside Traffic Advisor outputs are also used for the turn signals.

```text
TA1 = LEFTSIGNAL
TA8 = RIGHTSIGNAL
```

When no Traffic Advisor mode is active:

```text
LEFTSIGNAL  → TA1
RIGHTSIGNAL → TA8
```

## Priority

Traffic Advisor operation has priority over the turn signals.

```text
TRAFFIC ADVISOR ACTIVE
          │
          ▼
TA CONTROLS TA1–TA8
          │
          ▼
TURN SIGNALS CANNOT OVERRIDE TA
```

When the Traffic Advisor stops, any turn signal that is still active is automatically restored.

---

# Command Protocol

Communication between controllers uses simple text-based commands.

## START

Format:

```text
START <COMMAND>
```

Examples:

```text
START OUTERS
START INTERMEDIATES
START INNERS
START TRAFFICBREAKERS
START LEDS

START TLEFT
START TRIGHT
START TOUT

START LEFTSIGNAL
START RIGHTSIGNAL
```

---

## STOP

Format:

```text
STOP <COMMAND>
```

Examples:

```text
STOP OUTERS
STOP INTERMEDIATES
STOP INNERS
STOP TRAFFICBREAKERS
STOP LEDS

STOP TLEFT
STOP TRIGHT
STOP TOUT

STOP LEFTSIGNAL
STOP RIGHTSIGNAL
```

---

## Heartbeat

The Nano sends:

```text
PING
```

every:

```text
5 seconds
```

The Mega uses this heartbeat to verify that the Nano is still operating and communication remains available.

---

# Communication Watchdog

The Mega contains a communication watchdog.

## Normal Operation

After the first valid `PING`:

```text
Watchdog LED = SOLID ON
Outputs       = ENABLED
```

## Warning

If the Mega has not received a heartbeat for more than:

```text
6 seconds
```

the watchdog LED begins flashing.

Outputs remain operational during the warning period.

## Communication Failure

If no heartbeat has been received for:

```text
60 seconds
```

the Mega:

1. Turns all lightbar outputs OFF
2. Turns the watchdog LED OFF
3. Marks communication as lost
4. Prevents normal lightbar operation until communication returns

## Recovery

When communication returns and a new `PING` is received:

1. Communication is restored
2. The watchdog LED turns solid ON
3. Normal command processing resumes

---

# Watchdog Summary

| Condition              | Watchdog LED | Lightbar        |
| ---------------------- | ------------ | --------------- |
| Startup / no PING      | OFF          | Disabled        |
| Normal communication   | Solid ON     | Enabled         |
| 6–60 sec without PING  | Flashing     | Enabled         |
| 60+ sec without PING   | OFF          | All outputs OFF |
| Communication restored | Solid ON     | Enabled         |

---

# Serial Debugging

## Nano USB Serial

```text
115200 baud
```

The Nano reports:

* Startup information
* Pin configuration
* Initial input states
* Input activation/deactivation
* Commands sent to Mega
* Heartbeats
* Messages received from Mega

Example:

```text
INPUT 6 - TLEFT ACTIVE
NANO -> MEGA: START TLEFT
```

---

## Mega USB Serial

```text
115200 baud
```

The Mega provides debugging information for:

* Startup
* UART communication
* Received commands
* Traffic Advisor operation
* Watchdog state
* Communication failure
* Communication recovery

---

# Complete Nano ESP32 Pin Reference

| Pin | Function              |
| --- | --------------------- |
| D2  | OUTERS input          |
| D3  | INTERMEDIATES input   |
| D4  | INNERS input          |
| D5  | TRAFFICBREAKERS input |
| D6  | LEDS input            |
| D7  | TLEFT input           |
| D8  | TRIGHT input          |
| D9  | TOUT input            |
| D10 | UART TX → Mega D19    |
| D11 | UART RX ← Mega D18    |
| A0  | LEFTSIGNAL input      |
| A1  | RIGHTSIGNAL input     |
| GND | Common ground         |

---

# Complete Mega 2560 Pin Reference

| Pin | Function        |
| --- | --------------- |
| D18 | TX1 → Nano D11  |
| D19 | RX1 ← Nano D10  |
| D23 | TA1             |
| D25 | TA2             |
| D27 | TA3             |
| D29 | TA4             |
| D31 | TA5             |
| D33 | TA6             |
| D35 | TA7             |
| D37 | TA8             |
| D39 | TRAFFICBREAKERS |
| D41 | OUTERS          |
| D43 | INTERMEDIATES   |
| D45 | INNERS          |
| D47 | LEDS            |
| D49 | Unused          |
| D51 | Unused          |
| D53 | Unused          |
| GND | Common ground   |

---

# Electrical / Safety Notes

## Nano ESP32 Inputs

The Nano ESP32 uses **3.3 V GPIO logic**.

**Do not connect a 12 V vehicle signal directly to a Nano GPIO pin.**

Vehicle signals should pass through an appropriate interface, such as:

* Optocoupler
* Transistor interface
* Appropriate level-shifting circuit
* Other automotive-rated input-conditioning circuit

---

## Mega Outputs

The Mega GPIO pins are logic-level outputs.

They should not directly power lightbar loads or switch high-current circuits.

Use appropriately rated external output drivers, such as:

* MOSFETs
* Transistors
* Driver ICs
* Solid-state switching devices
* Relays where appropriate

Driver circuitry should be selected according to the actual electrical characteristics of the lightbar/control inputs.

---

## Ground

The Nano and Mega require a common ground for UART communication.

```text
Nano GND ───────── Mega GND
```

Proper automotive power regulation and protection should be used when powering the controllers from the truck's electrical system.

---

# Project Status

**Current Status: Functional Prototype**

## Current Lightbar Controller

* [x] Arduino Nano ESP32 input controller
* [x] Arduino Mega 2560 output controller
* [x] 10 Nano physical inputs
* [x] Ground-triggered input operation
* [x] 50 ms input debouncing
* [x] Hardware UART communication
* [x] 38400 baud Nano ↔ Mega communication
* [x] Text-based START/STOP command protocol
* [x] 5-second heartbeat
* [x] Mega communication watchdog
* [x] 6-second watchdog warning
* [x] 60-second communication shutdown
* [x] Automatic watchdog recovery
* [x] 8-output Traffic Advisor
* [x] TLEFT pattern
* [x] TRIGHT pattern
* [x] TOUT pattern
* [x] Two-output TA sequencing
* [x] 500 ms TA timing
* [x] TA command priority
* [x] Left turn signal integration
* [x] Right turn signal integration
* [x] Turn signal restoration after TA operation
* [x] Five direct lightbar outputs
* [x] Active-low Mega output control
* [x] Nano serial debugging
* [x] Mega serial debugging

---

# Planned Arduino GIGA / Touchscreen Controller

The project will eventually incorporate an **Arduino GIGA R1 WiFi with touchscreen** as an in-cab control and information system.

## GIGA Hardware / Core System

* [ ] Add Arduino GIGA R1 WiFi
* [ ] Add touchscreen display
* [ ] Design GIGA power supply and automotive protection
* [ ] Establish communication between GIGA and lightbar control system
* [ ] Define communication protocol for GIGA commands
* [ ] Add GIGA communication heartbeat/watchdog
* [ ] Add communication-loss handling
* [ ] Ensure failure of GIGA does not disable Nano physical controls

## Touchscreen Lightbar Controller

* [ ] Develop main touchscreen user interface
* [ ] Create dedicated lightbar control page
* [ ] Add OUTERS touchscreen control
* [ ] Add INTERMEDIATES touchscreen control
* [ ] Add INNERS touchscreen control
* [ ] Add TRAFFICBREAKERS touchscreen control
* [ ] Add LEDS touchscreen control
* [ ] Add TLEFT touchscreen control
* [ ] Add TRIGHT touchscreen control
* [ ] Add TOUT touchscreen control
* [ ] Display current lightbar command states
* [ ] Display active Traffic Advisor mode
* [ ] Display communication status
* [ ] Define priority between touchscreen and physical inputs
* [ ] Allow touchscreen commands to coexist safely with physical controls
* [ ] Add touchscreen startup/self-test page

---

# Planned OBD-II / CAN Gauge System

The GIGA touchscreen will also function as an OBD-II/CAN gauge and vehicle-information display for the truck.

## OBD-II / CAN Hardware

* [ ] Add suitable OBD-II/CAN interface
* [ ] Establish communication with truck ECU
* [ ] Identify supported standard OBD-II PIDs
* [ ] Investigate truck-specific CAN messages
* [ ] Isolate vehicle-data functionality from critical lightbar control

## Gauge Interface

* [ ] Create touchscreen gauge page
* [ ] Display engine RPM
* [ ] Display vehicle speed
* [ ] Display engine coolant temperature
* [ ] Display engine load
* [ ] Display throttle position
* [ ] Display intake air temperature
* [ ] Display battery/control-module voltage where available
* [ ] Add additional supported vehicle parameters
* [ ] Create configurable gauge layouts
* [ ] Add day/night gauge layouts
* [ ] Add user-selectable gauges

## Vehicle Warnings / Diagnostics

* [ ] Add configurable warning thresholds
* [ ] Add visual warning indicators
* [ ] Add diagnostic trouble code display
* [ ] Display active DTCs
* [ ] Display pending DTCs
* [ ] Add DTC descriptions
* [ ] Add vehicle/CAN communication status display

---

# Future Features

Potential future expansion includes:

* [ ] Automatic touchscreen brightness
* [ ] Day/night display mode
* [ ] User-configurable lightbar screen layout
* [ ] User-configurable gauge layout
* [ ] System diagnostics page
* [ ] Nano communication status
* [ ] Mega communication status
* [ ] GIGA communication status
* [ ] Output feedback/verification
* [ ] Output-driver fault detection
* [ ] ACK responses between controllers
* [ ] Startup/self-test sequence
* [ ] Store configuration in non-volatile memory
* [ ] Vehicle data logging
* [ ] Lightbar command/event logging
* [ ] Diagnostic event logging
* [ ] Additional Traffic Advisor patterns
* [ ] Additional Mega outputs using D49/D51/D53
* [ ] USB configuration
* [ ] Wi-Fi configuration
* [ ] Network-based software/configuration updates
* [ ] Additional truck-specific CAN monitoring

---

# Design Philosophy

The project is being designed around **modular controllers with defined responsibilities**.

### Nano ESP32

```text
PHYSICAL INPUTS
      ↓
INPUT PROCESSING
      ↓
COMMAND GENERATION
```

### Mega 2560

```text
COMMANDS
    ↓
OUTPUT LOGIC
    ↓
LIGHTBAR
```

### Arduino GIGA

```text
          TOUCHSCREEN
              │
       ┌──────┴──────┐
       ▼             ▼
LIGHTBAR UI       GAUGE UI
       │             │
       ▼             ▼
CONTROL SYSTEM    OBD-II/CAN
```

A major design goal is to avoid making the touchscreen a single point of failure.

The Nano's physical controls should remain capable of controlling the lightbar independently of the GIGA touchscreen.

---

# Recommended Repository Structure

```text
Lightbar-Controller/
│
├── README.md
│
├── Nano/
│   └── Lightbar_Nano.ino
│
├── Mega/
│   └── Lightbar_Mega.ino
│
├── GIGA/
│   ├── README.md
│   └── GIGA_Controller.ino
│
├── Documentation/
│   ├── Wiring.md
│   ├── Pinout.md
│   ├── Protocol.md
│   ├── TrafficAdvisor.md
│   ├── Watchdog.md
│   └── OBD2_CAN.md
│
└── Hardware/
    ├── Input_Interface/
    ├── Output_Drivers/
    ├── GIGA/
    └── Schematics/
```

---

# Development Status

This project is under active development.

The **Nano ESP32 and Mega 2560 lightbar control system is currently functional**, while the **Arduino GIGA touchscreen and OBD-II/CAN system are planned future additions**.

Pin assignments, communication protocols, hardware interfaces, and planned features may change as development continues.
