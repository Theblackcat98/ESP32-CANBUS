
Engineering Analysis and Development Framework for Legacy Automotive Interfacing: The 2000 Mazda Miata OBD-II Project
1. Introduction and Project Scope
The modernization of legacy automotive systems through embedded electronics represents a significant intersection of computer science, electrical engineering, and automotive diagnostics. The request to develop a Minimum Viable Product (MVP) interface for a 2000 Mazda Miata (NB generation) presents a distinct set of challenges that differ fundamentally from contemporary automotive hacking. While modern vehicles (post-2008) rely on the standardized Controller Area Network (CAN) for both diagnostics and control, vehicles from the transitional era of the late 1990s and early 2000s utilize a fragmented landscape of protocols.
This report provides an exhaustive technical analysis for the design, fabrication, and programming of an On-Board Diagnostics (OBD-II) interface device specifically for the 2000 Mazda Miata. The project's core objectives—reading telemetry and executing state changes (active testing)—require a rigorous understanding of the ISO 9141-2 protocol, microcontroller architecture, and discrete analog circuitry. Contrary to the user's initial hypothesis regarding "CANbus messages," the 2000 Miata does not utilize a user-accessible CAN bus for diagnostics. Instead, it relies on the K-Line asynchronous serial interface, necessitating a complete shift in the proposed hardware and software stack.
The following analysis dictates that the ESP32 microcontroller is the optimal hardware platform due to its real-time I/O capabilities, which are essential for the timing-sensitive initialization sequences of legacy protocols. Furthermore, the report establishes that "changing settings" on this vehicle architecture is best achieved not through software commands to the Engine Control Unit (ECU), which are often read-only or undocumented, but through a hybrid approach utilizing the vehicle's specific diagnostic hardware pins (TFA) for actuator control.
2. Automotive Network Architecture: The 2000 Mazda Miata Context
To successfully interface with the 2000 Mazda Miata, one must first deconstruct the vehicle's digital nervous system. The term "OBD-II" refers to a standard for the physical connector (SAE J1962) and a set of emissions-related data parameters, but it does not mandate a single language. Between 1996 and 2008, manufacturers selected from five distinct signaling protocols, creating a complex environment for aftermarket development.
2.1 The Myth of the CAN Bus in Legacy Diagnostics
A primary misconception in early automotive hacking is the assumption that the OBD-II port always carries CAN bus traffic. The user's query specifically requests information on "canbus messages." However, authoritative research confirms that the 2000 Mazda Miata does not utilize the ISO 15765-4 (CAN) standard for its diagnostic interface.1 While later Mazda models (specifically the NC generation starting in 2006 and fully mandated in 2008) adopted CAN, the NB generation (1999–2005) relies on the ISO 9141-2 protocol.3
This distinction is not merely semantic; it is a fundamental engineering constraint. A CAN transceiver, such as the MCP2515 often used with Arduino or Raspberry Pi projects, operates on a differential signal pair (CAN High and CAN Low). If a developer were to connect such a device to pins 6 and 14 of the Miata's OBD port, they would encounter high-impedance open circuits or silence.5 The 2000 Miata communicates via a single-ended, high-voltage line known as the K-Line, located at Pin 7 of the SAE J1962 connector.1
2.2 Detailed Analysis of ISO 9141-2
The ISO 9141-2 protocol is a legacy asynchronous serial communication standard that shares more lineage with RS-232 serial ports than with modern packet-switched networks. Understanding its physical and data link layers is critical for writing the interface code.
2.2.1 Physical Layer (Layer 1)
The K-Line operates on battery voltage logic. When the bus is idle, it is pulled up to the vehicle's battery voltage ($V_{batt}$), which typically ranges from 12.6V (engine off) to 14.4V (alternator running).1 A logical "0" (dominant bit) is represented by pulling the line to ground (0V), while a logical "1" (recessive bit) is represented by allowing the line to float back to $V_{batt}$ via pull-up resistors within the ECU.7
This high-voltage environment ($0V - 14.4V$) is incompatible with the 3.3V logic levels of modern microcontrollers like the ESP32 or Raspberry Pi. Direct connection would result in immediate catastrophic failure of the microcontroller's GPIO pins. Therefore, a robust transceiver circuit is required to shift these voltage levels bi-directionally.7
2.2.2 Data Link Layer (Layer 2) and Initialization
Unlike CAN, where a device can simply attach to the bus and start listening to broadcast frames, ISO 9141-2 requires a deliberate "handshake" or initialization sequence to wake the ECU and establish communication parameters. This is known as the 5-Baud Initialization.10
The sequence is timing-critical and proceeds as follows:
Idle: The bus must be idle (High) for a specified period (typically 300ms to 3s).
Address Byte: The diagnostic tool must manually drive the K-Line Low and High to transmit the address byte 0x33 at an extremely slow rate of 5 bits per second (200ms per bit).
Synchronization: If the ECU detects this sequence, it wakes up and responds with a synchronization byte (0x55) at the standard communication rate of 10,400 baud.12
Key Bytes: The ECU sends two "Key Bytes" (typically 0x08, 0x08 for Mazda) to indicate protocol specifics.
Inversion: The tool must invert the second key byte and send it back to the ECU as an acknowledgment.
Active: Only after this sequence does the ECU enter a state where it accepts PID requests.10
This initialization requirement is the primary reason why standard serial libraries often fail with ISO 9141. The hardware must be capable of switching between "bit-banging" (manual GPIO toggling) for the 5-baud sequence and hardware-driven UART communication for the 10.4 kbps data stream.11
2.3 The "Settings" and Bi-Directional Control Landscape
The user desires to "change a setting" on the car, citing "auto-start-stop" as an example. It is crucial to manage expectations regarding the capabilities of the 2000 Miata's ECU. Vehicles of this era typically do not support user-configurable "settings" via the OBD port in the way modern cars do (e.g., coding features via BimmerCode or FORScan). The ISO 9141 implementation on the NB Miata is primarily a read-only interface designed for emissions compliance (Mode 01) and clearing fault codes (Mode 04).13
However, "Active Tests" or bi-directional controls (Mode 08) do exist for diagnostic purposes, such as commanding the fuel pump, toggling solenoids, or controlling the radiator fan.13 Unfortunately, documentation for these specific commands on the Mazda ISO 9141 implementation is sparse and often proprietary. Consequently, reliance solely on software commands to "change a setting" introduces high development risk and complexity.
To satisfy the MVP requirement of "sending a signal" to change a physical state, this report advocates for a hybrid approach: utilizing the TFA (Test Fan) pin located in the vehicle's under-hood diagnostic box.15 By electronically grounding this pin, the device can force the radiator fan to activate, providing a tangible, verifiable "write" operation that bypasses the limitations of the software protocol.
3. Hardware Architecture and Component Selection
The selection of the computing core is the pivotal decision in the hardware design process. The user inquired about the suitability of the Raspberry Pi Zero 2W versus the ESP32. Based on the specific requirements of ISO 9141 initialization and power management, the ESP32 is the unequivocally superior choice.
3.1 Microcontroller Analysis: ESP32 vs. Raspberry Pi Zero 2W
3.1.1 The Determinism of Real-Time Operations
The 5-baud initialization sequence requires the transmission of bits with a duration of exactly 200 milliseconds. While this seems slow, the transition accuracy must be maintained. The Raspberry Pi Zero 2W runs a full Linux operating system. Linux is a general-purpose OS, not a Real-Time Operating System (RTOS). Background processes, kernel housekeeping, and thread scheduling can introduce unpredictable latency (jitter) into GPIO operations.17 If the Linux kernel preempts the GPIO toggling process during the initialization sequence, the timing will drift, the ECU will reject the handshake, and the connection will fail.
In contrast, the ESP32 is a microcontroller that runs code on bare metal or atop FreeRTOS. This architecture allows for deterministic control of GPIO pins with microsecond precision.18 The ESP32 can disable interrupts or use hardware timers to ensure the 5-baud sequence is generated with perfect fidelity.
3.1.2 Power Consumption and Boot Time
An automotive device is a parasitic load on the vehicle's battery. The NB Miata is equipped with a relatively small lead-acid battery. A Raspberry Pi Zero 2W, even at idle, consumes significantly more current (100mA+) than an ESP32, which can be put into deep sleep modes consuming mere microamps.18 Furthermore, the boot time of a Raspberry Pi (30–60 seconds) is undesirable for a device that should be active the moment the ignition key is turned. The ESP32 boots and begins code execution in milliseconds.
3.1.3 Integrated Peripherals
The ESP32 features three hardware UARTs (Universal Asynchronous Receiver/Transmitter).20 This is critical for the ISO 9141 protocol. After the bit-banged initialization, the communication switches to 10,400 baud. The ESP32's hardware serial ports can be configured to non-standard baud rates (like 10,400) with high stability, offloading the CPU from managing bit timing.
Conclusion: The ESP32 (specifically the ESP32-WROOM-32 or ESP32-DevKitC) is the required device for this project. It offers the necessary real-time control, low power consumption, and hardware UART support that the Raspberry Pi lacks for this specific application.
3.2 The Physical Interface: Transceiver Selection
Since the ESP32 cannot tolerate the 12V signals of the K-Line, a transceiver is required.
3.2.1 The Dedicated IC Solution (Recommended)
For an MVP, reliability is paramount. Dedicated K-Line transceiver ICs are designed to handle the noisy, spike-prone electrical environment of a vehicle. The STMicroelectronics L9637D is the industry standard for this application.9
Function: The L9637D converts the bidirectional, single-wire 12V K-Line signal into separate TX and RX lines at logic levels compatible with the microcontroller (typically 5V or 3.3V).
Protection: It includes built-in protection against short circuits to the battery, reverse battery polarity, and thermal overload.9
Implementation: Since the L9637D is a surface-mount component (SO-8 package), it is advisable to purchase a breakout board. Modules labeled "K-Line Transceiver" or "ISO 9141 Interface" often contain this chip or the equivalent NXP MC33660.22
3.2.2 The Discrete Component Solution (Alternative)
If a dedicated IC cannot be sourced, a transceiver can be constructed using discrete components, specifically the 2N3904 NPN transistor.7
Transmission (TX): The ESP32's TX pin drives the base of the NPN transistor. The collector is connected to the K-Line, and the emitter to ground. When the ESP32 outputs a logic High, the transistor saturates, pulling the K-Line to ground (Logic 0). When the ESP32 outputs Low, the transistor turns off, and a 510$\Omega$ pull-up resistor pulls the K-Line to 12V (Logic 1). Note: This inverts the logic, which must be handled in software.
Reception (RX): A voltage divider (e.g., 22k$\Omega$ and 10k$\Omega$) is used to step down the 12V K-Line voltage to approximately 3.3V for the ESP32's RX pin. Alternatively, an optocoupler can be used for isolation.
3.3 Active Control Hardware: The Relay Module
To fulfill the requirement of changing a setting (turning the fan on), the device requires an actuation circuit. The 2000 Miata's diagnostic box contains a pin labeled TFA (Test Fan). Grounding this pin completes the circuit for the cooling fan relay, forcing the fan to run.15
The ESP32 cannot sink the current required to drive the vehicle's relay coil directly, nor can it handle the voltage. Therefore, a 5V Relay Module or a MOSFET module is required.24
Relay Module: A standard 1-channel 5V relay module compatible with Arduino/ESP32 is ideal. The ESP32 GPIO drives the module's input (often via an optocoupler on the module), and the relay's "Normally Open" (NO) and "Common" (COM) contacts are used to switch the TFA pin to chassis ground.
4. Software Engineering and Implementation
The software stack must handle the precise timing of the initialization, the parsing of the serial data, and the logic for the active test. The recommended language is C++, utilizing the Arduino framework for the ESP32.
4.1 Development Environment
The Arduino IDE or PlatformIO (running within VS Code) are the standard environments for ESP32 development. They provide access to extensive libraries that abstract much of the low-level hardware complexity.
4.2 The OBD2_KLine_Library
Writing a raw ISO 9141 stack from scratch is error-prone due to the strict inter-byte timing requirements (P3 and P4 timings) mandated by the standard. Fortunately, the open-source community has produced robust libraries. The OBD2_KLine_Library by user muki01 is explicitly designed for ESP32 and ISO 9141/KWP2000 protocols.7
Key Features of the Library:
Initialization Handling: It automates the 5-baud "bit-banging" sequence on the TX pin before seamlessly handing control over to the hardware UART.
Timing Compliance: It manages the inter-byte delays required by the ECU, preventing communication errors where the ECU "times out" waiting for a request.
Protocol Abstraction: It provides high-level functions like obd.readPID(), allowing the developer to request data (e.g., RPM) without manually constructing the hex packet and calculating the checksum.
4.3 Data Acquisition Strategy (Reading Data)
To read data, the software must send a Mode 01 request.
Request Structure: [Header][Mode][Checksum]
Header: 0x68 0x6A 0xF1 (Priority, Target: ECU, Source: Scan Tool).
Mode: 0x01 (Show Current Data).
PID: Parameter ID.
Key PIDs for Miata:
0x0C: Engine RPM.
0x0D: Vehicle Speed.
0x05: Engine Coolant Temperature.
0x10: MAF Air Flow Rate.
Example Logic:
The code will initialize the connection. Once successful, the loop() function will periodically call obd.readPID(PID_RPM) and obd.readPID(PID_COOLANT_TEMP). The library handles the raw serial transmission (at 10,400 baud) and returns the integer value.
4.4 Active Control Strategy (The "Write" Operation)
The "Write" operation involves controlling the radiator fan via the relay connected to the TFA pin.
Software Logic:
Trigger Condition: The code monitors a variable (e.g., fanState) or listens for a user command via Serial/Bluetooth.
Actuation:
Turn Fan ON: Set the GPIO pin connected to the Relay Module to HIGH. The relay closes, connecting the TFA pin to Ground. The car's fan activates.
Turn Fan OFF: Set the GPIO pin to LOW. The relay opens, disconnecting TFA. The car's fan returns to ECU control.
4.5 Information Sources for Code (CANbus Messages & PIDs)
The user asked: "Where can I find the information to put in my code that tells me exactly what settings to change on a specific car? (canbus messages)"
Since the car is not CAN-based, "CANbus messages" (DBC files, Arbitration IDs) are irrelevant. The developer needs OBD-II PID Lists for ISO 9141.
Primary Sources:
SAE J1979 Standard: Defines the generic PIDs (Mode 01) supported by all compliant vehicles (RPM, Speed, Temp).26 Wikipedia's "OBD-II PIDs" page is a sufficient reference for these standard commands.
RomRaider & OpenECU Forums: These communities specialize in reverse-engineering Subaru and Mazda ECUs. They contain definition files (XML) that list specific memory addresses and extended PIDs for logging.27
Mazda Miata Enthusiast Forums (Miata.net): The "ECU & Tuning" sections often contain threads where users have sniffed the K-Line traffic to identify proprietary commands.29
GitHub Repositories: Searching for "Mazda OBD PIDs" or "Miata ISO 9141" on GitHub reveals repositories like python-OBD or torque-pid-plugins which often contain CSV files listing known PIDs for specific vehicles.31
Addressing the "Settings" Misconception:
It is critical to note that extensive lists of "settings to change" (like changing door lock behavior or turn signal timing) likely do not exist for the 2000 Miata's ECU. These features were hard-coded or controlled by separate, non-networked modules (like the LC-144 timer module) in this era. The "settings" available to change are limited to active actuator tests (Fan, Fuel Pump, Solenoids) typically accessible only via proprietary dealer tools or the hardware workarounds (TFA pin) proposed here.
5. Detailed Implementation Plan
5.1 Bill of Materials (BOM)
The following components are required to build the device:
Component
Specification
Purpose
Microcontroller
ESP32 DevKit V1 (ESP-WROOM-32)
Core processing, logic control, WiFi/BLE.
Transceiver
L9637D Breakout Board (or MC33660)
Converts 12V K-Line signals to 3.3V logic.
Actuator
5V Relay Module (1-Channel)
Switches the high-current/12V TFA pin signal.
Power Supply
LM2596 Buck Converter Module
Steps down 12V car battery to 5V for ESP32.
Connector
OBD-II Male Plug (J1962)
Physical connection to the car's cabin port.
Wiring
22 AWG Wire
Connections between modules.
Passive
1kΩ Resistors (Optional)
For transistor base protection (if using discrete).

5.2 Circuit Diagram Description
Power:
OBD Pin 16 (Battery +): Connect to LM2596 Input +.
OBD Pin 4/5 (Ground): Connect to LM2596 Input -.
LM2596 Output + (5V): Connect to ESP32 VIN and Relay VCC.
LM2596 Output - (GND): Connect to ESP32 GND and Relay GND.
Diagnostic Communication:
L9637D VCC: Connect to 5V (or 3.3V if supported).
L9637D GND: Connect to ESP32 GND.
L9637D RX: Connect to ESP32 GPIO 16 (RX2).
L9637D TX: Connect to ESP32 GPIO 17 (TX2).
L9637D K-Line: Connect to OBD Pin 7.
Active Control:
ESP32 GPIO 26: Connect to Relay Module IN pin.
Relay COM: Connect to Chassis Ground.
Relay NO: Connect via long wire to the TFA Pin in the diagnostic box under the hood.
5.3 Software Logic Walkthrough
The software will be written in C++. The flow is as follows:
Setup Phase:
Initialize Serial Debugging (USB to Computer) at 115200 baud.
Configure GPIO 26 (Relay) as OUTPUT and set to LOW (Fan Off).
Call obd.begin() from the library to start the K-Line interface.
Execute the 5-baud init sequence (handled by library). Wait for connection confirmation.
Loop Phase:
Reading: Every 100ms, send a Mode 01 PID 0x0C request. Wait for response. Parse the two data bytes ($A$ and $B$). Calculate $RPM = ((A \times 256) + B) / 4$. Print to Debug Serial.
Writing: Check for a trigger (e.g., if RPM > 3000, or if a character is received on Serial).
If Trigger = TRUE: Set GPIO 26 HIGH. (Relay clicks, Fan turns ON).
If Trigger = FALSE: Set GPIO 26 LOW. (Relay releases, Fan turns OFF).
6. Testing, Validation, and Troubleshooting
6.1 Validation of the Physical Layer
Before connecting the ESP32, use a multimeter to verify the power supply.
Check voltage at OBD Pin 16: Should be ~12V.
Check output of LM2596: Must be adjusted to 5.0V before connecting to ESP32.
Check continuity between the TFA wire and the relay Common pin.
6.2 Troubleshooting Communication
If the device fails to connect ("Connection Failed" in logs):
Ignition State: ISO 9141 requires the ECU to be powered. The key must be in the ON position.
Baud Rate: Ensure the library is set to 10,400 baud, not 9600.
5-Baud Timing: If using a custom bit-banging routine (not the library), verify the 200ms delay is accurate using an oscilloscope. Linux-based devices (Pi) often fail here due to jitter.
TX/RX Swap: It is a common error to swap TX and RX lines. Try reversing GPIO 16 and 17 definitions.
6.3 Safety Considerations
TFA Pin: Grounding the TFA pin bypasses the ECU's thermal control logic. While safe for testing, leaving the fan forced ON indefinitely could mask thermostat issues or drain power. The code should include a fail-safe to turn the relay off after a set duration.
Voltage Isolation: The ESP32 is not 5V tolerant on its data pins. If powering the L9637D with 5V, ensure the RX line (going into the ESP32) passes through a voltage divider or logic level shifter to bring the signal down to 3.3V.
7. Conclusion
The development of an OBD-II interface for the 2000 Mazda Miata is a sophisticated exercise in legacy system integration. The project requires discarding the modern "CAN bus" paradigm in favor of the ISO 9141-2 K-Line protocol. By utilizing an ESP32 microcontroller for its deterministic timing and low power profile, paired with an L9637D transceiver, the MVP can successfully read standard telemetry like RPM and Coolant Temperature.
Crucially, the requirement to "change a setting" is satisfied not by risky and undocumented ECU memory writes, but by a reliable hardware-bridged approach utilizing the TFA diagnostic pin. This architecture provides a robust, safe, and verifiable method for bi-directional control, fulfilling the user's MVP goals while navigating the constraints of 25-year-old automotive technology. This project serves as a foundational platform, expandable to wireless logging, remote start (via other ignition harness taps), or digital dashboard applications.
7.1 Summary of Recommendations
Device: ESP32 (Do not use Raspberry Pi Zero 2W).
Protocol: ISO 9141-2 (Do not use CAN/ISO 15765).
Language: C++ (Arduino Framework).
Information Source: SAE J1979 (Standard PIDs), Miata.net (TFA Pin).
MVP "Write" Action: Control Radiator Fan via Relay to TFA Pin.
Works cited
OBD II Protocols Explained, accessed December 15, 2025, https://www.obdexperts.com/obd-ii-protocols-explained/
OBD2 protocols - OBDTester, accessed December 15, 2025, https://www.obdtester.com/obd2_protocols
How to Identify OBD-II in Vehicles | PDF - Scribd, accessed December 15, 2025, https://www.scribd.com/document/339011826/does-my-car-have-obd-pdf
5 Common OBD2 Protocols - Sinocastel, accessed December 15, 2025, https://www.sinocastel.com/5-common-obd2-protocols/
OBD2 pinout explained. Major car brands pinouts - FlexiHub, accessed December 15, 2025, https://www.flexihub.com/oobd2-pinout/
Diagnostic port 2000 NB : r/Miata - Reddit, accessed December 15, 2025, https://www.reddit.com/r/Miata/comments/1fac4ze/diagnostic_port_2000_nb/
muki01/OBD2 K-Line - PlatformIO Registry, accessed December 15, 2025, https://registry.platformio.org/libraries/muki01/OBD2%20K-Line
ISO 9141 K-Line Communication - ATI Accurate Technologies, accessed December 15, 2025, https://www.accuratetechnologies.com/blog/post/iso-9141-k-line
L9637 - Monolithic bus driver with ISO 9141 interface - STMicroelectronics, accessed December 15, 2025, https://www.st.com/resource/en/datasheet/l9637.pdf
Cheap OBD2 Communications on K-line (ISO 9141-2 and ISO 14230-4) - Instructables, accessed December 15, 2025, https://www.instructables.com/Low-Cost-OBD2-Communications-on-K-line-ISO-9141-2-/
Producing K-line initialization sequence - General Discussion - Macchina, accessed December 15, 2025, https://forum.macchina.cc/t/producing-k-line-initialization-sequence/764
K-line Communication Description - National OBD Clearinghouse, accessed December 15, 2025, https://www.obdclearinghouse.com/Files/viewFile?fileID=1380
Mode 8 - The bidirectional controls in OBD2 - OBD Auto Doctor, accessed December 15, 2025, https://www.obdautodoctor.com/blog/mode-8-bidirectional-controls-in-obd2/
What is an Active Test/Bidirectional Control Scanner? — OBDPRICE-AU, accessed December 15, 2025, https://au.obdprice.com/blogs/news/what-is-an-active-test-bidirectional-control-scanner
Testing the radiator fan - Engine, Transmission, Exhaust etc - MX-5 Owners Club Forum, accessed December 15, 2025, https://forum.mx5oc.co.uk/t/testing-the-radiator-fan/75846
Need help with fan/overheating issue : r/Miata - Reddit, accessed December 15, 2025, https://www.reddit.com/r/Miata/comments/oubj84/need_help_with_fanoverheating_issue/
esp32 vs pi zero w. serious question, what would you use one for vs. the other? - Reddit, accessed December 15, 2025, https://www.reddit.com/r/arduino/comments/65dp98/esp32_vs_pi_zero_w_serious_question_what_would/
Help me choose between esp32 and raspberry in terms of efficiency - Reddit, accessed December 15, 2025, https://www.reddit.com/r/esp32/comments/1faq4tr/help_me_choose_between_esp32_and_raspberry_in/
Raspberry Pi Pico vs ESP32 - Which one is faster? - YouTube, accessed December 15, 2025, https://www.youtube.com/watch?v=hNEXW9hmo00
ESP32 UART - Serial Communication, Send and Receive Data (Arduino IDE), accessed December 15, 2025, https://randomnerdtutorials.com/esp32-uart-communication-serial-arduino/
E-L9637D - eStore - STMicroelectronics, accessed December 15, 2025, https://estore.st.com/en/e-l9637d-cpn.html
MC33660, ISO K line serial link interface - Data Sheet - NXP Semiconductors, accessed December 15, 2025, https://www.nxp.com/docs/en/data-sheet/MC33660.pdf
MX 5 mk 2.5 2002 cooling fan sensor,where is it?, accessed December 15, 2025, https://forum.mx5oc.co.uk/t/mx-5-mk-2-5-2002-cooling-fan-sensor-where-is-it/85139
ESP32 Relay Module - Control AC Appliances (Web Server) - Random Nerd Tutorials, accessed December 15, 2025, https://randomnerdtutorials.com/esp32-relay-module-ac-web-server/
muki01/OBD2_KLine_Library: OBD2 K-Line library (ISO9141, ISO14230) for Arduino, ESP32, and other compatible boards. - GitHub, accessed December 15, 2025, https://github.com/muki01/OBD2_KLine_Library
OBD-II PIDs - Wikipedia, accessed December 15, 2025, https://en.wikipedia.org/wiki/OBD-II_PIDs
handmade0octopus/ds2: DS2 K-line library for Arduino and ESP32 - GitHub, accessed December 15, 2025, https://github.com/handmade0octopus/ds2
VersaTuner, accessed December 15, 2025, https://www.versatuner.com/
Mazda OBD2 Codes | PDF - Slideshare, accessed December 15, 2025, https://www.slideshare.net/slideshow/obd2-codes-of-mazda/236664117
Mazda OBD-II Trouble Codes | PDF | Throttle | Mechanical Engineering - Scribd, accessed December 15, 2025, https://www.scribd.com/document/751155087/Mazda-OBD-II-Trouble-Codes
Mazda 3 PIDs (Rough List) - GitHub Gist, accessed December 15, 2025, https://gist.github.com/agronick/4d01cfe7f94dd41eeeb46f5e5dd204b8
obd-ii-pids · GitHub Topics, accessed December 15, 2025, https://github.com/topics/obd-ii-pids

