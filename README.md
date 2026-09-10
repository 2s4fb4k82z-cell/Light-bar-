Arduino Lightbar Controller

A modular Arduino-based vehicle lightbar control system using an Arduino Nano ESP32, Arduino Mega 2560, and planned Arduino GIGA R1 WiFi with GIGA Display Shield.

The system separates physical control inputs, touchscreen control, output control, Traffic Advisor sequencing, communication monitoring, and future vehicle diagnostic functions across multiple controllers.

⸻

System Architecture

Current / planned architecture:

PHYSICAL SWITCHES
       │
       ▼
┌─────────────────────┐
│  Arduino Nano ESP32 │◄──────── UART ──────── Arduino GIGA
│                     │                         + Display Shield
│  Input Controller   │                         Touch Controller
│  Command Arbiter    │                         Diagnostic Gauges
│  Watchdog Manager   │
└──────────┬──────────┘
           │
           │ UART
           ▼
┌─────────────────────┐
│ Arduino Mega 2560   │
│                     │
│ Output Controller   │
│ Traffic Advisor     │
│ Mega Watchdog       │
└──────────┬──────────┘
           │
           ▼
       LIGHTBAR

The Nano ESP32 is the central command controller.

It:

* Reads the physical switches.
* Receives commands from the GIGA.
* Keeps physical and GIGA requests separate.
* Determines the effective requested state.
* Sends final commands to the Mega.
* Sends a heartbeat to the Mega.
* Monitors the GIGA heartbeat.
* Protects physical controls from GIGA communication failures.

The Mega 2560 controls the actual lightbar outputs and Traffic Advisor sequences.

⸻

Arduino Nano ESP32

Physical Inputs

The Nano has 10 ground-triggered physical inputs.

HIGH = Input OFF
LOW  = Input ON / Grounded

All physical inputs use:

INPUT_PULLUP

Nano Input Pinout

Input	Function	Nano Pin
1	OUTERS	D2
2	INTERMEDIATES	D3
3	INNERS	D4
4	TRAFFICBREAKERS	D5
5	LEDS	D6
6	TLEFT	D7
7	TRIGHT	D8
8	TOUT	D9
9	LEFTSIGNAL	A0
10	RIGHTSIGNAL	A1

Input debounce time:

50 ms

⸻

Nano UART Connections

The Nano communicates with both the Mega and GIGA using separate hardware UART connections.

Nano ↔ Mega UART

Nano D10 TX ─────────► Mega D19 RX1
Nano D11 RX ◄───────── Mega D18 TX1
Nano GND ───────────── Mega GND

UART configuration:

Baud:      38,400
Data:      8 bits
Parity:    None
Stop bits: 1

Nano code:

HardwareSerial MegaSerial(1);
const int MEGA_RX_PIN = D11;
const int MEGA_TX_PIN = D10;

⸻

Nano ↔ GIGA UART

The GIGA communicates with the Nano through a second hardware UART.

GIGA TX ─────────────► Nano D12 RX
GIGA RX ◄───────────── Nano D13 TX
GIGA GND ───────────── Nano GND

UART configuration:

Baud:      38,400
Data:      8 bits
Parity:    None
Stop bits: 1

Nano code:

HardwareSerial GigaSerial(2);
const int GIGA_RX_PIN = D12;
const int GIGA_TX_PIN = D13;

D12 / D13 Note

D12 and D13 are reserved for GIGA communication in this project.

On the Nano ESP32:

D12 = GPIO47
D13 = GPIO48

These pins also have SPI functionality.

D13/GPIO48 is also associated with the Nano’s built-in LED.

Because this project is using D12 and D13 for the GIGA UART, SPI and the built-in LED should not be assigned to these pins while this UART configuration is in use.

⸻

Nano USB Serial

The Nano USB serial monitor operates at:

115,200 baud

It is used for debugging communication, physical inputs, watchdog activity, and commands sent between controllers.

Example output:

INPUT 1 - OUTERS ACTIVE
NANO -> MEGA: START OUTERS
GIGA -> NANO: GIGA_PING
GIGA communication established.
GIGA -> NANO: START LEDS
GIGA REQUEST ON: LEDS
NANO -> MEGA: START LEDS

⸻

Command Protocol

Commands between controllers use simple text messages terminated with a newline.

START Command

START <FUNCTION>

Example:

START OUTERS

STOP Command

STOP <FUNCTION>

Example:

STOP OUTERS

Supported functions:

OUTERS
INTERMEDIATES
INNERS
TRAFFICBREAKERS
LEDS
TLEFT
TRIGHT
TOUT
LEFTSIGNAL
RIGHTSIGNAL

The GIGA can request all 10 functions.

⸻

Source-Aware Command Arbitration

The Nano keeps physical switch requests and GIGA requests separate.

For example:

OUTERS
Physical Request = OFF
GIGA Request     = ON
Effective State = ON

Likewise:

OUTERS
Physical Request = ON
GIGA Request     = OFF
Effective State = ON

The effective state for normal functions is:

Physical Request OR GIGA Request

Therefore:

Physical	GIGA	Effective Output
OFF	OFF	OFF
ON	OFF	ON
OFF	ON	ON
ON	ON	ON

This prevents one control source from accidentally disabling another.

⸻

Nano → Mega Heartbeat

The Nano sends:

PING

to the Mega every:

5 seconds

The Mega uses this heartbeat to determine whether communication with the Nano is still operating correctly.

⸻

Mega Watchdog

The Mega monitors the Nano heartbeat.

Startup

Watchdog LED = OFF

After the first valid Nano PING:

Watchdog LED = ON

Normal Communication

Less than 6 seconds since the last heartbeat:

Watchdog LED = Solid ON
Outputs = Enabled

Communication Warning

6–60 seconds since the last heartbeat:

Watchdog LED = Flashing
Outputs = Still Enabled

Communication Failure

60 seconds or more since the last heartbeat:

Watchdog LED = OFF
All lightbar outputs = OFF

When communication returns, a new valid PING reactivates the watchdog system.

⸻

GIGA → Nano Heartbeat

The GIGA sends:

GIGA_PING

to the Nano.

The Nano records the most recent valid GIGA heartbeat.

The Nano will not accept GIGA control commands until at least one valid:

GIGA_PING

has been received.

⸻

GIGA Watchdog

The Nano has a separate watchdog specifically for GIGA communication.

Timeout:

60 seconds

If the Nano does not receive another:

GIGA_PING

within 60 seconds, the GIGA is considered disconnected.

The Nano then clears all requests originating from the GIGA.

This includes:

OUTERS
INTERMEDIATES
INNERS
TRAFFICBREAKERS
LEDS
TLEFT
TRIGHT
TOUT
LEFTSIGNAL
RIGHTSIGNAL

The watchdog does not clear physical Nano switch requests.

⸻

GIGA Watchdog Example

Suppose the GIGA requests OUTERS:

Physical OUTERS = OFF
GIGA OUTERS     = ON
Effective OUTERS = ON

If GIGA communication fails:

60 seconds without GIGA_PING
             │
             ▼
GIGA OUTERS = OFF
             │
             ▼
Physical OUTERS = OFF
             │
             ▼
Nano sends:
STOP OUTERS

However, if the physical switch is also active:

Physical OUTERS = ON
GIGA OUTERS     = ON

After a GIGA timeout:

Physical OUTERS = ON
GIGA OUTERS     = OFF
Effective OUTERS = ON

The Nano does not send STOP OUTERS.

The physical switch retains control.

⸻

Turn Signal Watchdog Behavior

Turn signals follow the same source-aware watchdog rules.

The GIGA can send:

START LEFTSIGNAL
STOP LEFTSIGNAL
START RIGHTSIGNAL
STOP RIGHTSIGNAL

If a turn signal was requested only by the GIGA and communication fails:

GIGA LEFTSIGNAL = ON
Physical LEFTSIGNAL = OFF
        ↓
60-second GIGA timeout
        ↓
GIGA LEFTSIGNAL = OFF
        ↓
Nano sends:
STOP LEFTSIGNAL

If the physical turn signal is also active:

GIGA LEFTSIGNAL = ON
Physical LEFTSIGNAL = ON
        ↓
60-second GIGA timeout
        ↓
GIGA LEFTSIGNAL = OFF
Physical LEFTSIGNAL = ON
        ↓
LEFTSIGNAL remains active

Therefore a GIGA failure cannot disable an active physical turn signal.

⸻

Traffic Advisor

The Traffic Advisor consists of eight outputs.

Mega Traffic Advisor Pinout

Position	Function	Mega Pin
1	TA1 — Far Left	D23
2	TA2	D25
3	TA3	D27
4	TA4	D29
5	TA5	D31
6	TA6	D33
7	TA7	D35
8	TA8 — Far Right	D37

Traffic Advisor commands:

TLEFT
TRIGHT
TOUT

Only one Traffic Advisor pattern can run at a time.

The most recently activated Traffic Advisor request receives priority.

⸻

TLEFT Pattern

Exactly two adjacent Traffic Advisor lights are active at a time.

TA1 + TA2
TA2 + TA3
TA3 + TA4
TA4 + TA5
TA5 + TA6
TA6 + TA7
TA7 + TA8
Repeat

Each step lasts:

500 ms

⸻

TRIGHT Pattern

TA8 + TA7
TA7 + TA6
TA6 + TA5
TA5 + TA4
TA4 + TA3
TA3 + TA2
TA2 + TA1
Repeat

Each step lasts:

500 ms

⸻

TOUT Pattern

TA4 + TA5
TA3 + TA6
TA2 + TA7
TA1 + TA8
Repeat

Each step lasts:

500 ms

⸻

Traffic Advisor Source Arbitration

The Nano tracks Traffic Advisor requests from both sources:

Physical switches
GIGA touchscreen

The most recently activated Traffic Advisor request wins.

Example:

Physical TLEFT = ON
        ↓
TLEFT running
        ↓
GIGA sends START TRIGHT
        ↓
TRIGHT becomes active

If the GIGA then loses communication and its TRIGHT request is removed:

GIGA watchdog timeout
        ↓
GIGA TRIGHT request removed
        ↓
Physical TLEFT still active
        ↓
Nano sends:
STOP TRIGHT
START TLEFT

The physical Traffic Advisor request automatically resumes.

⸻

Turn Signal / Traffic Advisor Priority

Physical turn signal inputs:

A0 = LEFTSIGNAL
A1 = RIGHTSIGNAL

When no Traffic Advisor pattern is active:

LEFTSIGNAL  → TA1 ON
RIGHTSIGNAL → TA8 ON

Traffic Advisor patterns have priority over the turn-signal display.

If a Traffic Advisor pattern starts while a turn signal is active, the Traffic Advisor controls the TA outputs.

When the Traffic Advisor stops, an active turn signal is restored.

⸻

Mega Direct Outputs

Function	Mega Pin
TRAFFICBREAKERS	D39
OUTERS	D41
INTERMEDIATES	D43
INNERS	D45
LEDS	D47

Currently unused:

D49
D51
D53

⸻

Mega Output Logic

The Mega lightbar outputs are:

ACTIVE LOW

Meaning:

OUTPUT_ON  = LOW;
OUTPUT_OFF = HIGH;

The Arduino GPIO pins should not directly carry lightbar power or high-current ground loads.

Appropriate output drivers should be used.

Example:

Mega GPIO
    │
    ▼
MOSFET / Transistor / Driver
    │
    ▼
Lightbar Control Input

⸻

Electrical Safety

Nano ESP32 Inputs

The Nano ESP32 uses 3.3 V GPIO logic.

Never apply vehicle 12 V directly to a Nano GPIO pin.

For vehicle-derived 12 V signals, use an appropriate interface such as:

Optocoupler
Transistor interface
Protected voltage-level interface
Automotive-rated input circuit

The exact interface should provide the Nano with a safe 3.3 V-compatible signal.

⸻

Common Ground

For the UART connections:

Nano GND
Mega GND
GIGA GND

must share a common electrical reference unless an isolated communication interface is used.

⸻

Planned Arduino GIGA Controller

Future hardware:

Arduino GIGA R1 WiFi
Arduino GIGA Display Shield

The GIGA will provide:

* Touchscreen lightbar controls
* Vehicle diagnostic gauges
* Communication with the Nano
* GIGA heartbeat
* Future user-interface features

The GIGA will communicate with the Nano, not directly with the Mega.

GIGA
  │
  │ UART
  ▼
NANO
  │
  │ UART
  ▼
MEGA

⸻

Planned GIGA Display

The normal/default screen will be the diagnostic gauge screen.

Two gauges are currently planned:

IPR Duty Cycle
0–100 %
ICP Pressure
0–4000 PSI

Both gauges will have:

* Analog-style needle
* Scale markings
* Digital value readout

⸻

Planned Screen Navigation

There will be no permanent navigation button between the gauge and lightbar screens.

Screen switching will use a two-finger gesture:

Finger 1 starts on left side
Finger 2 starts on right side
Both swipe downward together

Screen flow:

GAUGE SCREEN
     │
     │ Two-finger downward swipe
     ▼
LIGHTBAR SCREEN
     │
     │ Two-finger downward swipe
     ▼
GAUGE SCREEN

⸻

Planned GIGA Lightbar Controls

The touchscreen lightbar screen is planned to control:

OUTERS
INTERMEDIATES
INNERS
TRAFFICBREAKERS
LEDS
TLEFT
TRIGHT
TOUT

The communication protocol also supports:

LEFTSIGNAL
RIGHTSIGNAL

if turn-signal touchscreen controls are added.

⸻

Planned Vehicle Diagnostics

Target vehicle:

1996 Ford F-350
7.3L Power Stroke Diesel

The two planned diagnostic values are:

IPR Duty Cycle
ICP Pressure

The truck uses Ford-era SAE J1850 PWM/SCP diagnostic communication rather than modern CAN-based OBD communication.

The final diagnostic interface will require an appropriate J1850 PWM-compatible vehicle interface/transceiver.

⸻

Repository Structure

Recommended repository structure:

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
│   └── VehicleDiagnostics.md
│
└── Hardware/
    ├── Input_Interface/
    ├── Output_Drivers/
    ├── GIGA/
    └── Schematics/

⸻

Project Checklist

Nano ESP32

* [x]	Configure 10 ground-triggered physical inputs
* [x]	Add 50 ms input debounce
* [x]	Configure Nano → Mega UART
* [x]	Send START / STOP commands to Mega
* [x]	Send 5-second heartbeat to Mega
* [x]	Add separate GIGA UART
* [x]	Reserve D12/D13 for GIGA communication
* [x]	Add GIGA_PING heartbeat monitoring
* [x]	Add 60-second GIGA watchdog
* [x]	Separate physical and GIGA command requests
* [x]	Prevent GIGA timeout from disabling physical requests
* [x]	Include GIGA-controlled turn signals in watchdog
* [x]	Add Traffic Advisor source arbitration
* [ ]	Hardware-test Nano ↔ GIGA UART
* [ ]	Hardware-test GIGA watchdog
* [ ]	Hardware-test simultaneous physical/GIGA requests
* [ ]	Hardware-test Traffic Advisor source restoration

Mega 2560

* [x]	Configure lightbar outputs
* [x]	Configure Traffic Advisor outputs
* [x]	Implement TLEFT
* [x]	Implement TRIGHT
* [x]	Implement TOUT
* [x]	Add turn-signal priority handling
* [x]	Add Nano heartbeat watchdog
* [x]	Add communication-loss shutdown
* [ ]	Final vehicle hardware testing

Arduino GIGA

* [ ]	Obtain Arduino GIGA R1 WiFi
* [ ]	Obtain GIGA Display Shield
* [ ]	Configure landscape display
* [ ]	Configure LVGL
* [ ]	Configure GIGA → Nano UART
* [ ]	Implement GIGA_PING
* [ ]	Send GIGA heartbeat periodically
* [ ]	Implement START / STOP command protocol
* [ ]	Build lightbar touchscreen
* [ ]	Build IPR gauge
* [ ]	Build ICP gauge
* [ ]	Implement two-finger screen-change gesture
* [ ]	Add communication status indication
* [ ]	Test GIGA disconnect/reconnect behavior

Vehicle Diagnostics

* [ ]	Verify diagnostic connector pin population
* [ ]	Verify J1850 PWM communication
* [ ]	Verify IPR live-data request
* [ ]	Verify ICP live-data request
* [ ]	Select J1850 PWM interface/transceiver
* [ ]	Implement diagnostic communication on GIGA
* [ ]	Validate IPR percentage
* [ ]	Validate ICP PSI
* [ ]	Add stale-data detection
* [ ]	Add diagnostic communication failure indication

Hardware

* [ ]	Design protected 12 V → 3.3 V Nano input interface
* [ ]	Design Mega output driver circuits
* [ ]	Add appropriate fusing
* [ ]	Add reverse-polarity protection
* [ ]	Add automotive transient protection
* [ ]	Build final wiring harness
* [ ]	Label Nano/Mega/GIGA UART wiring
* [ ]	Create final wiring schematic
* [ ]	Bench-test complete system
* [ ]	Install in vehicle

⸻

Current Project Status

The Nano/Mega lightbar controller is currently a functional prototype.

The latest Nano architecture adds:

Physical control inputs
        +
GIGA touchscreen requests
        ↓
Source-aware arbitration
        ↓
Final command to Mega

Two independent communication watchdog layers are planned/currently implemented:

GIGA
 │
 │ GIGA_PING
 ▼
NANO
 │
 │ PING
 ▼
MEGA

This provides protection against either the GIGA or Nano disappearing while preserving physical control whenever possible.

The Arduino GIGA touchscreen and vehicle diagnostic system are the next major development stages.