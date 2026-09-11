Code 3 MX7000 Lightbar Controller

Arduino GIGA R1 WiFi + GIGA Display Shield project for controlling a Code 3 MX7000 lightbar, displaying engine gauges, and providing a built-in GIGA↔Nano serial monitor.

The GIGA acts as an additional control source and display only. Physical switches remain independent through the Nano, and the Nano remains the central command/arbitration controller for the Code 3 MX7000 lightbar system.

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
* Provides touchscreen controls for the Code 3 MX7000
* Receives physical/GIGA state information from the Nano
* Sends commands to the Nano
* Monitors GIGA↔Nano serial traffic internally
* Does not directly control the Mega or lightbar outputs