# Code 3 MX7000 Controller

Custom Arduino-based controller and touchscreen interface for a **Code 3 MX7000** installed on a **1996 Ford F-350 7.3L Power Stroke**.

The system uses three Arduino boards:

- **Arduino Nano ESP32** — Central controller / physical switch inputs
- **Arduino Mega 2560** — Lightbar output controller
- **Arduino GIGA R1 WiFi + GIGA Display Shield** — Touchscreen, gauges, diagnostics, and lightbar controls

---

# System Layout

```text
                    PHYSICAL SWITCHES
                           │
                           ▼
                  ┌─────────────────┐
                  │   NANO ESP32    │
                  │ Central Control │
                  └───────┬─────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
           UART                    UART
         38,400                  38,400
              │                       │
              ▼                       ▼
     ┌─────────────────┐     ┌─────────────────┐
     │    MEGA 2560    │     │    GIGA R1      │
     │ Output Control  │     │ Touchscreen UI  │
     └────────┬────────┘     └─────────────────┘
              │
              ▼
        RELAY / DRIVER
              │
              ▼
        CODE 3 MX7000
```

The **Nano is the central controller**.

The GIGA does not directly control lightbar outputs. It sends requests to the Nano, and the Nano determines what the Mega should do.

---

# Arduino Nano ESP32

## Physical Switch Inputs

All physical inputs are:

- Active LOW
- Ground triggered
- `INPUT_PULLUP`
- 50 ms debounce

| Nano Pin | Function |
|---|---|
| D2 | OUTERS |
| D3 | INTERMEDIATES |
| D4 | INNERS |
| D5 | TRAFFICBREAKERS |
| D6 | LEDS |
| D7 | TLEFT |
| D8 | TRIGHT |
| D9 | TOUT |
| A0 | LEFTSIGNAL |
| A1 | RIGHTSIGNAL |

Do **NOT** apply 12 V directly to Nano GPIO pins.

---

## Nano ↔ Mega UART

**38,400 baud**

| Nano ESP32 | Mega 2560 |
|---|---|
| D10 TX | D19 RX1 |
| D11 RX | D18 TX1 |
| GND | GND |

### Important

The Mega uses **5 V logic** while the Nano ESP32 uses **3.3 V logic**.

Therefore:

```text
Nano D10 TX (3.3V)
        │
        └──────────────> Mega D19 RX1

Mega D18 TX (5V)
        │
        ▼
 Level Shifter /
 Resistor Divider
        │
        ▼
Nano D11 RX (3.3V)
```

Do **not** connect Mega D18 directly to Nano D11 without reducing the voltage.

### Installed Wire Colors

| Connection | Wire |
|---|---|
| Mega D19 RX1 | Brown |
| Mega D18 TX1 | Yellow |
| Ground | Green |

---

## Nano ↔ GIGA UART

**38,400 baud**

| GIGA | Nano ESP32 |
|---|---|
| D1 TX | D12 RX |
| D0 RX | D13 TX |
| GND | GND |

Both boards use 3.3 V logic, so no level shifter is required.

---

# Arduino Mega 2560

The Mega controls the actual lightbar output drivers.

Outputs are **active LOW**:

```text
LOW  = Output ON
HIGH = Output OFF
```

The Arduino pins should operate suitable relays/MOSFET drivers.

They should **NOT directly power the lightbar loads**.

---

## Traffic Advisor Outputs

| Mega Pin | Function |
|---|---|
| D23 | TA1 — Far Left |
| D25 | TA2 |
| D27 | TA3 |
| D29 | TA4 |
| D31 | TA5 |
| D33 | TA6 |
| D35 | TA7 |
| D37 | TA8 — Far Right |

---

## Main Lightbar Outputs

| Mega Pin | Function |
|---|---|
| D39 | TRAFFICBREAKERS |
| D41 | OUTERS |
| D43 | INTERMEDIATES |
| D45 | INNERS |
| D47 | LEDS |
| D49 | Unassigned |
| D51 | Unused |
| D53 | Unused |

---

## Physical Relay Wiring Reference

These colors are only installation/wiring references.

They have **no meaning in the software**.

### Set 1

| Mega Pin | Wire |
|---|---|
| D23 | Green |
| D25 | Red |
| D27 | White |
| D29 | Black |

### Set 2

| Mega Pin | Wire |
|---|---|
| D31 | Green |
| D33 | Red |
| D35 | White |
| D37 | Black |

### Set 3

| Mega Pin | Wire |
|---|---|
| D39 | Green |
| D41 | Red |
| D43 | White |
| D45 | Black |

### Set 4

| Connection | Wire |
|---|---|
| D47 | Green |
| +5 V | Red |
| D49 | White |
| 5 V Ground | Black |

---

# Arduino GIGA R1 WiFi

The GIGA uses the **GIGA Display Shield** as the user interface.

It provides:

- 7.3 Power Stroke gauges
- Lightbar touchscreen controls
- Physical-switch status
- Communication status
- Serial monitor
- Startup logo
- Physical switch override controls

The GIGA does **not directly drive the MX7000**.

Commands go:

```text
GIGA
  │
  ▼
NANO
  │
  ▼
MEGA
  │
  ▼
LIGHTBAR
```

---

# GIGA Screens

## Startup

At startup, the GIGA displays the custom DBB logo for approximately **5 seconds**.

Communication with the Nano can initialize while the splash screen is displayed.

After the splash screen:

```text
STARTUP LOGO
      │
      ▼
GAUGE SCREEN
```

---

## Gauge Screen

Displays:

- IPR — 0–100%
- ICP — 0–4000 PSI

The current code can simulate engine data for UI testing.

```cpp
bool simulateEngineData = true;
```

When real truck data is implemented, this should be changed to:

```cpp
bool simulateEngineData = false;
```

A flashing red gauge border indicates either:

- Engine-data simulation is enabled
- Real truck data has timed out

Loss of Nano/Mega communication does **not** trigger the gauge-screen alarm.

---

## Lightbar Screen

Layout:

```text
┌────────────┬────────────┬────────────┬────────────┬────────────┐
│  TRAFFIC   │   INNERS   │ INTERMED.  │   OUTERS   │    LEDS    │
│  BREAKER   │            │            │            │            │
├────────────┼────────────┼────────────┼────────────┼────────────┤
│   CLOSE    │  TA LEFT   │   TA OUT   │  TA RIGHT  │  MONITOR   │
└────────────┴────────────┴────────────┴────────────┴────────────┘
```

A secret two-finger downward swipe from the gauge screen opens the lightbar controller.

`CLOSE` returns to the gauge screen.

`MONITOR` opens the serial monitor.

The Serial Monitor `X` returns to the Lightbar screen.

---

# Touchscreen Button Colors

| Color | Meaning |
|---|---|
| Dark | Inactive |
| Flashing Blue | Physical switch ON during startup delay |
| Blue | Physical switch active |
| Flashing Purple | Waiting for command confirmation |
| Solid Purple | Command failed/timed out |
| Green | GIGA command confirmed by Mega |
| Red | Physical switch is being suppressed by GIGA override |

The green used by the lightbar controls matches the green used by the OBS-style gauges.

---

# Touchscreen Command Behavior

A normal touchscreen command is **not sent when the finger first touches the screen**.

The GIGA waits until the finger is released before executing the command.

```text
Finger Down
     │
     ▼
Button Identified
     │
     │  No command yet
     ▼
Finger Released
     │
     ▼
Release Confirmed
     │
     ▼
Send Command ONCE
```

This helps prevent one touchscreen tap from being interpreted as multiple presses due to the behavior of the GIGA touchscreen.

---

# Physical Switch Priority / Override

Physical switches and GIGA requests are tracked separately.

Normally:

```text
Physical Request
       OR
GIGA Request
       │
       ▼
Effective Output
```

If a physical switch is ON, its GIGA button appears **blue**.

Holding that touchscreen button for approximately **2 seconds** activates the physical-switch override.

The button becomes **red**.

```text
Physical Switch ON
       │
       ▼
     BLUE
       │
   Hold ~2 sec
       │
       ▼
      RED
       │
       ▼
Physical contribution suppressed
```

Long-holding the red button again removes the override.

Turning the physical switch OFF automatically clears its override.

The override suppresses only the **physical source**. A GIGA request may still keep the same output active.

---

# Startup-Held Physical Switches

If a physical switch is already ON when the Nano boots, the Nano recognizes it but does not immediately activate that output.

The startup delay is:

```text
120 seconds
```

During this period, the corresponding GIGA button flashes blue.

If the switch remains ON when the delay expires, it becomes a normal active physical request.

If the physical switch is turned OFF during the delay, the startup hold is canceled.

Turning it back ON is then treated as a new intentional activation.

---

# Traffic Advisor

The Traffic Advisor uses eight outputs:

```text
TA1 TA2 TA3 TA4 TA5 TA6 TA7 TA8
LEFT                     RIGHT
```

Sequence timing:

```text
500 ms per step
```

## Left

```text
1+2
2+3
3+4
4+5
5+6
6+7
7+8
REPEAT
```

## Right

```text
8+7
7+6
6+5
5+4
4+3
3+2
2+1
REPEAT
```

## Out

```text
4+5
3+6
2+7
1+8
REPEAT
```

Only one Traffic Advisor pattern is active at a time.

The most recently activated eligible TA request wins.

---

# Turn Signals

Physical turn-signal inputs:

| Nano Pin | Function |
|---|---|
| A0 | LEFTSIGNAL |
| A1 | RIGHTSIGNAL |

When no Traffic Advisor pattern is active:

```text
LEFTSIGNAL  -> TA1
RIGHTSIGNAL -> TA8
```

Traffic Advisor patterns have priority over the turn-signal output.

When the TA pattern stops, an active turn signal can resume.

---

# Communication

## GIGA → Nano

Examples:

```text
GIGA_PING

START OUTERS
STOP OUTERS

SYNC_REQUEST

OVERRIDE OUTERS ON
OVERRIDE OUTERS OFF
```

## Nano → GIGA

Examples:

```text
GIGA_PONG

ACK START OUTERS
ACK STOP OUTERS

CONFIRMED START OUTERS
CONFIRMED STOP OUTERS

ACK OVERRIDE OUTERS ON
CONFIRMED OVERRIDE OUTERS ON

STATE OUTERS P=1 G=0 E=1 M=1 H=0 O=0

NANO_BOOT

MEGA_LINK_LOST
MEGA_LINK_RESTORED
```

---

# STATE Message

Example:

```text
STATE OUTERS P=1 G=0 E=1 M=1 H=0 O=0
```

Meaning:

| Field | Meaning |
|---|---|
| P | Physical request |
| G | GIGA request |
| E | Effective desired state |
| M | Mega-confirmed state |
| H | Startup hold |
| O | Physical override |

The Nano's `STATE` message is the authoritative state reported to the GIGA.

---

# Command Confirmation

A GIGA request travels through the complete system.

Example:

```text
GIGA
 │
 │ START OUTERS
 ▼
NANO
 │
 │ ACK START OUTERS
 ▼
GIGA

NANO
 │
 │ START OUTERS
 ▼
MEGA
 │
 │ ACK START OUTERS
 ▼
NANO
 │
 │ CONFIRMED START OUTERS
 ▼
GIGA
```

The GIGA flashes the button purple while waiting for confirmation.

A confirmed GIGA command becomes green.

The GIGA retries an unconfirmed START/STOP command every:

```text
5 seconds
```

It gives up after:

```text
30 seconds
```

A timed-out command becomes solid purple.

---

# Communication Watchdogs

The system uses heartbeats so communication failures do not leave the controller blindly assuming another board is working.

The Nano periodically communicates with the Mega.

The GIGA periodically communicates with the Nano.

The Mega has its own failsafe behavior if communication with the Nano disappears.

On the GIGA:

- Gauge alarm is reserved for truck/engine-data status.
- Lightbar and Serial Monitor alarms indicate Nano/Mega communication problems.

---

# Serial Monitor

The GIGA contains a built-in serial diagnostics screen.

It shows traffic between the GIGA and Nano and diagnostics forwarded from the Mega.

Direction colors:

| Color | Traffic |
|---|---|
| Green | GIGA → Nano |
| Blue | Nano → GIGA |
| Red | Error |
| Gray | Timestamp |

It also displays:

```text
NANO: OK / LOST
MEGA: OK / LOST
```

---

# Important Electrical Notes

### Nano ESP32

The Nano ESP32 is a **3.3 V device**.

Do not connect:

- 12 V vehicle signals
- 5 V Mega TX

directly to its GPIO pins.

### Mega Outputs

Mega GPIO pins should only control appropriate driver circuitry.

Do not attempt to power MX7000 lamps or other high-current loads directly from Arduino pins.

### Grounds

The communicating boards require a common signal ground:

```text
Nano GND
   │
   ├──── Mega GND
   │
   └──── GIGA GND
```

Use appropriate automotive electrical protection when interfacing the Arduino system with the truck's 12 V electrical system.

---

# Project Summary

```text
PHYSICAL SWITCHES
       │
       ▼
   NANO ESP32  ◄────────►  GIGA DISPLAY
       │                   Touchscreen
       │                   Gauges
       │                   Diagnostics
       │
       ▼
   MEGA 2560
       │
       ▼
 RELAY / DRIVER
       │
       ▼
 CODE 3 MX7000
```

The main design principle is:

**The Nano decides, the Mega drives, and the GIGA displays and requests.**

This keeps the physical lightbar controls functional independently of the touchscreen while still allowing the GIGA to provide touchscreen control, status monitoring, diagnostics, and vehicle gauges.
