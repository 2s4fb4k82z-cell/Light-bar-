GIGA Code 3 MX7000 Lightbar / Gauge Controller

Arduino GIGA R1 WiFi + GIGA Display Shield project for a vehicle-mounted Code 3 MX7000 lightbar control interface, engine gauge display, and built-in GIGA↔Nano serial monitor.

The GIGA acts as an additional control source and display only. Physical switches remain independent through the Nano, and the Nano remains the central command/arbitration controller for the lightbar system.

System Architecture

PHYSICAL SWITCHES
      │
      ▼
Arduino Nano ESP32  <──── UART ────> Arduino GIGA R1 WiFi + Display
      │
      │ UART
      ▼
Arduino Mega 2560
      │
      ▼
CODE 3 MX7000 LIGHTBAR

The Nano ESP32 remains the primary controller.

The GIGA:

* Displays engine gauges
* Provides touchscreen lightbar controls
* Receives physical/GIGA state information from the Nano
* Sends commands to the Nano
* Monitors GIGA↔Nano serial traffic internally
* Does not directly control the Mega or lightbar outputs

Controller Pinouts

Arduino GIGA R1 WiFi

The GIGA currently communicates only with the Nano ESP32 for lightbar control.

GIGA Pin	Function	Connects To
D0 / RX	Nano UART receive	Nano D13 / TX
D1 / TX	Nano UART transmit	Nano D12 / RX
GND	Common ground	Nano GND

Communication settings:

GIGA Serial1
Baud: 38400

Wiring:

GIGA D1 TX  ─────> Nano D12 RX
GIGA D0 RX  <───── Nano D13 TX
GIGA GND    ────── Nano GND

The GIGA and Nano ESP32 both use 3.3 V logic.

The GIGA Display Shield provides the touchscreen and 800×480 display used for the gauge, lightbar, and serial-monitor interfaces.

⸻

Arduino Nano ESP32

The Nano ESP32 is the central command controller.

It:

* Reads the physical switches
* Receives touchscreen commands from the GIGA
* Keeps physical and GIGA requests separate
* Performs source arbitration
* Sends effective lightbar commands to the Mega
* Monitors GIGA communication
* Sends state information back to the GIGA

Physical Switch Inputs

All physical switch inputs are active LOW.

HIGH = OFF
LOW  = ON / grounded

The inputs use:

INPUT_PULLUP

with approximately:

50 ms debounce

Nano Pin	Function
D2	OUTERS
D3	INTERMEDIATES
D4	INNERS
D5	TRAFFICBREAKERS
D6	LEDS
D7	TLEFT
D8	TRIGHT
D9	TOUT
A0	LEFTSIGNAL
A1	RIGHTSIGNAL

Nano ↔ Mega UART

Nano Pin	Function	Connects To
D10	TX to Mega	Mega D19 / RX1
D11	RX from Mega	Mega D18 / TX1
GND	Common ground	Mega GND

Nano firmware uses:

HardwareSerial MegaSerial(1);

Communication speed:

38400 baud

Wiring:

Nano D10 TX ─────> Mega D19 RX1
Nano D11 RX <───── Mega D18 TX1
                  │
                  └── LEVEL SHIFT / VOLTAGE DIVIDER REQUIRED
Nano GND    ────── Mega GND

Important: The Mega 2560 TX output is 5 V logic while the Nano ESP32 uses 3.3 V GPIO.

A suitable level shifter or voltage divider must therefore be installed between:

Mega D18 TX1
     ↓
Nano D11 RX

The Nano’s 3.3 V TX signal is generally sufficient for the Mega’s RX input.

Nano ↔ GIGA UART

Nano Pin	Function	Connects To
D12	RX from GIGA	GIGA D1 / TX
D13	TX to GIGA	GIGA D0 / RX
GND	Common ground	GIGA GND

Nano firmware uses:

HardwareSerial GigaSerial(2);

Communication speed:

38400 baud

Relevant Nano ESP32 GPIO Mapping

Nano Pin	ESP32 GPIO
D10	GPIO21
D11	GPIO38
D12	GPIO47
D13	GPIO48
A0	GPIO1
A1	GPIO2

D12 and D13 are reserved in this project for GIGA UART communication.

D13 / GPIO48 is also associated with LED_BUILTIN / SPI SCK, and D12 is associated with SPI CIPO. Those default functions should not be used simultaneously with the GIGA UART configuration.

⸻

Arduino Mega 2560

The Mega controls the physical Code 3 MX7000 lightbar outputs.

The Mega receives commands from the Nano and handles:

* Direct lightbar outputs
* Traffic Advisor sequencing
* Turn signal outputs
* Traffic Advisor priority over turn signals
* Nano heartbeat monitoring
* Output shutdown if Nano communication is lost

Nano Communication

Mega Pin	Function	Connects To
D18 / TX1	TX to Nano	Nano D11 / RX through level shifting
D19 / RX1	RX from Nano	Nano D10 / TX
GND	Common ground	Nano GND

Communication:

Mega Serial1
38400 baud

Traffic Advisor Outputs

Mega Pin	Output
D23	TA1 — Far Left
D25	TA2
D27	TA3
D29	TA4
D31	TA5
D33	TA6
D35	TA7
D37	TA8 — Far Right

Traffic Advisor layout:

LEFT                                         RIGHT
TA1   TA2   TA3   TA4   TA5   TA6   TA7   TA8
 │     │     │     │     │     │     │     │
D23   D25   D27   D29   D31   D33   D35   D37

Direct Lightbar Outputs

Mega Pin	Code 3 MX7000 Function
D39	TRAFFICBREAKERS
D41	OUTERS
D43	INTERMEDIATES
D45	INNERS
D47	LEDS
D49	Unused
D51	Unused
D53	Unused

The outputs are currently configured as active LOW:

OUTPUT_ON  = LOW;
OUTPUT_OFF = HIGH;

The Mega GPIO pins must not directly power the lightbar loads.

The Mega outputs should control suitable:

MOSFETs
Transistors
Relay/driver circuits
or other appropriate automotive output drivers

depending on the final lightbar interface hardware.

Complete Communication Overview

                 3.3 V UART
       ┌───────────────────────────┐
       │                           │
       │                           ▼
┌──────────────┐             ┌──────────────┐
│              │             │              │
│  Nano ESP32  │             │  GIGA R1     │
│              │             │  + Display   │
└──────┬───────┘             └──────────────┘
       │
       │ UART
       │
       │ Nano D10 TX ───────> Mega D19 RX1
       │ Nano D11 RX <─────── Mega D18 TX1
       │                       through level shifting
       ▼
┌──────────────┐
│              │
│ Mega 2560    │
│              │
└──────┬───────┘
       │
       │ Output drivers
       ▼
┌─────────────────────────┐
│                         │
│ CODE 3 MX7000 LIGHTBAR  │
│                         │
└─────────────────────────┘

Display Screens

The GIGA currently has three screens.

1. Engine Gauges

The default screen displays two gauges:

* IPR Duty Cycle
    * Range: 0–100%
* ICP Pressure
    * Range: 0–4000 PSI

The analog needles use smoothing:

displayedIPR +=
  (actualIPR - displayedIPR) * 0.15f;
displayedICP +=
  (actualICP - displayedICP) * 0.15f;

The digital values remain unsmoothed.

The current engine values are simulated for development.

Future vehicle diagnostic code will replace the engine-data source without requiring the rest of the display code to be redesigned.

Engine Vehicle Target

Current target vehicle:

1996 Ford F-350
7.3L Power Stroke

Planned diagnostic values:

IPR Duty Cycle
ICP Pressure

The 1996 7.3 Power Stroke uses Ford’s SAE J1850 PWM/SCP-era diagnostic system rather than modern CAN.

Actual J1850 diagnostic support has not yet been implemented.

2. Lightbar Control Screen

The touchscreen supports the following commands:

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

The GIGA does not directly activate lightbar outputs.

Instead:

GIGA
  │
  │ START / STOP command
  ▼
Nano ESP32
  │
  │ arbitration
  ▼
Mega 2560
  │
  ▼
Code 3 MX7000 Lightbar

Lightbar Button Colors

The lightbar screen uses color-only status indications.

Dark

Inactive

Blue

Physical Nano switch is active

Green

GIGA request is active and confirmed by the Nano

Flashing Purple

GIGA command did not receive an acknowledgement from the Nano

If both a physical switch and a GIGA request are active at the same time, blue takes priority so the physical source remains obvious.

Communication Loss Warning

If the GIGA loses communication with the Nano, a flashing red border appears around the entire display.

The warning is shown on:

* Gauge screen
* Lightbar screen
* Serial monitor screen

The GIGA considers the Nano offline after:

15 seconds without GIGA_PONG

The GIGA continues transmitting heartbeat requests so communication can recover automatically.

Screen Gestures

There are no permanent navigation buttons on the display.

Gauges ↔ Lightbar

Place two fingers on opposite horizontal sides of the display:

One finger on left side
One finger on right side

Then swipe both fingers downward.

Requirements:

Left finger begins in outer 30%
Right finger begins in outer 30%
Both move at least 100 pixels downward

This toggles:

GAUGES
   ↕
LIGHTBAR

Serial Monitor Gesture

To open the built-in serial monitor:

One finger near TOP of screen
One finger near BOTTOM of screen

Swipe both fingers left.

Requirements:

Top finger begins in top 30%
Bottom finger begins in bottom 30%
Both move at least 100 pixels left

Using the same gesture while the serial monitor is open returns to the previous normal screen.

Example:

GAUGES
  │
  └── top/bottom swipe left
             ↓
      SERIAL MONITOR
             │
             └── same gesture
                      ↓
                   GAUGES

If the monitor was opened from the lightbar screen, it returns to the lightbar screen.

Built-In Serial Monitor

The GIGA internally records everything it sends to and receives from the Nano.

No additional UART hardware or passive monitoring wires are required.

Direction labels:

G>N = GIGA to Nano
N>G = Nano to GIGA

Example:

0000012500  G>N  GIGA_PING
0000012510  N>G  GIGA_PONG
0000016200  G>N  START OUTERS
0000016210  N>G  ACK START OUTERS
0000016220  N>G  STATE OUTERS P=0 G=1 E=1

The monitor stores approximately the most recent:

48 messages

using a circular buffer.

The newest messages appear at the bottom of the screen.

Timestamps are milliseconds since GIGA startup.

GIGA ↔ Nano Protocol

GIGA → Nano

GIGA_PING
START <COMMAND>
STOP <COMMAND>
SYNC_REQUEST

Examples:

START OUTERS
STOP TLEFT
SYNC_REQUEST

Nano → GIGA

GIGA_PONG
ACK START <COMMAND>
ACK STOP <COMMAND>
SYNC_BEGIN
STATE <COMMAND> P=<0|1> G=<0|1> E=<0|1>
SYNC_END

Example:

STATE OUTERS P=1 G=0 E=1

State meanings:

P = physical switch request on Nano
G = GIGA request currently stored by Nano
E = effective state Nano is commanding toward the Mega

Command Acknowledgement

When the GIGA sends a command, it waits for an acknowledgement from the Nano.

Example:

GIGA:
START OUTERS

Expected Nano response:

ACK START OUTERS

ACK timeout:

1.5 seconds

Maximum retries:

2 retries

This means a command can be transmitted up to three times:

Original transmission
Retry 1
Retry 2

If no acknowledgement is received after all attempts, the affected button flashes purple.

Duplicate commands are safe because the Nano acknowledges duplicate START/STOP requests.

STATE Messages

A valid STATE message is treated as authoritative.

For example:

STATE OUTERS P=0 G=1 E=1

This tells the GIGA that:

Physical switch = OFF
GIGA request = ON
Effective output = ON

A valid STATE message also clears an ACK-failure warning because it proves the Nano received and processed the command state even if the ACK itself was lost.

Synchronization

After first connecting or recovering communication, the GIGA requests a complete state synchronization.

Sequence:

GIGA_PING
   ↓
GIGA_PONG
   ↓
SYNC_REQUEST
   ↓
SYNC_BEGIN
STATE OUTERS ...
STATE INTERMEDIATES ...
STATE INNERS ...
STATE TRAFFICBREAKERS ...
STATE LEDS ...
STATE TLEFT ...
STATE TRIGHT ...
STATE TOUT ...
STATE LEFTSIGNAL ...
STATE RIGHTSIGNAL ...
SYNC_END

This allows the GIGA display to rebuild its state from the Nano instead of assuming previous conditions.

Physical Switch Independence

Physical controls do not depend on the GIGA.

For normal commands:

Effective request =
Physical request OR GIGA request

Example:

Physical ON
GIGA OFF
=
ON
Physical OFF
GIGA ON
=
ON
Physical ON
GIGA ON
=
ON
Physical OFF
GIGA OFF
=
OFF

If GIGA communication fails, physical switch operation remains available.

GIGA Failure Behavior

The GIGA does not force outputs off merely because communication is lost.

The Nano independently manages GIGA-originated requests.

The Nano’s GIGA watchdog is responsible for clearing GIGA-generated requests after its timeout period.

Physical requests are never cleared by loss of GIGA communication.

Traffic Advisor Behavior

The GIGA supports:

TLEFT
TRIGHT
TOUT

Only one GIGA Traffic Advisor request is kept active at a time.

When a new GIGA TA mode is selected, the GIGA sends STOP commands for its previous TA request before sending the new START request.

Final arbitration still occurs on the Nano.

This is important because a physical Traffic Advisor request may also be active.

The Nano tracks request order and decides which active TA source has priority.

Touchscreen Coordinate Mapping

The GIGA Display is used in landscape orientation:

display.setRotation(1);

Display resolution:

800 × 480

Current touch transform:

x = rawY;
y = 479 - rawX;

This mapping is provisional until tested on the physical Display Shield.

If touch is rotated or mirrored incorrectly, the preferred solution is to modify only the touch-coordinate transform rather than changing all UI button and gesture coordinates.

Current Engine Data

The engine gauge values are currently generated by a simulator.

Example:

float ipr =
  35.0f +
  20.0f * sinf(seconds * 0.70f);
float icp =
  1200.0f +
  900.0f * sinf(seconds * 0.45f);

This exists only so the gauge display can be developed and tested before the J1850 interface is complete.

Arduino Libraries

The project currently uses:

#include <Arduino_GigaDisplay_GFX.h>
#include <Arduino_GigaDisplayTouch.h>

Install the corresponding Arduino GIGA Display graphics and touch libraries before compiling.

Current Single-File Firmware

The current development version is intentionally contained in one Arduino .ino file.

This makes it easier to:

* Copy into Arduino IDE
* Compile early versions
* Troubleshoot hardware
* Share complete firmware
* Make changes before final project structure is established

The code may later be separated into modules again once the hardware and communication system are fully tested.

Current Development Status

Implemented:

* GIGA Display support
* Landscape UI
* IPR gauge
* ICP gauge
* Simulated engine data
* Lightbar touchscreen controls
* Physical/GIGA source-state display
* Nano heartbeat
* Nano communication-loss detection
* Flashing red communication-loss border
* START/STOP commands
* Nano ACK processing
* Automatic command retries
* Flashing purple failed-command indication
* Full state synchronization
* Traffic Advisor request handling
* Built-in GIGA↔Nano serial monitor
* Circular serial log
* Multi-touch screen navigation

Still to be tested on physical hardware:

* GIGA Serial1 pin behavior with D0/D1
* Display rendering
* Touch coordinate orientation
* Multi-touch gesture reliability
* Nano↔GIGA UART operation
* Full synchronization behavior
* Lightbar button response
* ACK retry behavior

Still planned:

* SAE J1850 PWM interface
* Real 7.3 Power Stroke IPR data
* Real 7.3 Power Stroke ICP data
* Vehicle testing
* UI refinement after physical display testing

Important Electrical Notes

The GIGA and Nano ESP32 use 3.3 V logic.

Do not connect vehicle 12 V directly to any GIGA or Nano GPIO.

The Mega 2560 uses 5 V logic.

Any Mega TX signal entering a 3.3 V device should use appropriate level shifting or voltage reduction.

Project Philosophy

The system is designed so that the touchscreen is an additional control source rather than a single point of failure.

Core design goals:

Physical switches continue working without GIGA
GIGA can control the same functions independently
Nano performs source arbitration
Mega controls the actual outputs
Communication failures fail safely
UI clearly identifies command source
Troubleshooting information is available directly on the display

Disclaimer

This project is under active development and has not yet been fully verified on the final vehicle hardware.

Bench-test all outputs, voltage levels, communication links, watchdog behavior, and touchscreen controls before connecting the system to the Code 3 MX7000 lightbar.