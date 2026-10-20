# ESP32-CANBUS

Research notes and experiments on interfacing with car data buses — primarily the ESP32's CAN bus peripheral, with Raspberry Pi and pre-CAN approaches covered where they fit the car better.

**Status: research phase — notes only, no code yet.**

## Contents

### [research/crosstrek.md](research/crosstrek.md) — 2024 Subaru Crosstrek

Interfacing with the Crosstrek over OBD-II / CAN:

- Hardware options: Raspberry Pi Zero 2 W with a PiCAN2 CAN hat vs. an ESP32 paired with a CAN transceiver (or a ready-made adapter like WiCAN)
- Languages: Python on the Pi; Arduino C++ (or MicroPython) on the ESP32
- Reading standard OBD-II data (RPM, speed, etc.) is the straightforward MVP; toggling auto start-stop is the stretch goal
- The Crosstrek's specific CAN messages aren't publicly documented — expect to reverse-engineer them by logging bus traffic; starting points include comma.ai's OpenDBC database, Subaru CAN ID repositories, and car-hacking forums

### [research/miata.md](research/miata.md) — 2000 Mazda Miata (NB)

Engineering analysis and development framework for an OBD-II interface for a pre-CAN car:

- The NB Miata predates standardized CAN diagnostics — it uses the fragmented transitional-era protocols (ISO 9141-2), which changes the hardware design fundamentally
- Covers telemetry reading and active testing (state changes), microcontroller architecture, and the discrete analog circuitry an interface needs

### [research/tools.md](research/tools.md) — Tools & hardware links

- [SavvyCAN](https://www.savvycan.com/) — CAN bus capture and analysis
- [PiCAN2](https://copperhilltech.com/pican-2-can-bus-interface-for-raspberry-pi/) — CAN bus interface board for Raspberry Pi
- [Adafruit product 5708](https://www.adafruit.com/product/5708) — CAN hardware
