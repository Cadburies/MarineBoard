## Technical Specifications for Marine Generic Board

Based on the provided schematic screenshots, PCB layout rules, and component specifications in the instruction set, the design is modularized into logical blocks for easier schematic organization, PCB zoning, testing, and potential reuse. Each block represents a functional subunit with defined inputs/outputs, power requirements, and interfaces. Blocks are separated to minimize noise coupling (e.g., high-current from low-signal), facilitate EMI mitigation, and allow for independent verification. For example:

- **High-current blocks** (e.g., PWM drivers) use thicker traces and separate zones.
- **Analog-sensitive blocks** (e.g., INA226 monitors) are isolated with ground splits and bypass caps.
- **Digital blocks** (e.g., MCU) are centralized with I2C/one-wire buses shared where possible.
- **Power block** handles conditioning and distribution, with ferrite beads for EMI.

Below is a list of separable logical blocks, updated to match the exact components, labels, and configurations from the provided schematic screenshots (PWM Drivers, Power and Filtering, Levels Monitor, Battery Charger, CAN Interface, 12V Battery Monitor). Descriptions incorporate key details like part numbers (e.g., U6 for TPS5430DDA), values (e.g., capacitors like C3=4.7uF), and connections (e.g., I2C to SDA_GPIO/SCL_GPIO). All names are synchronized for consistency (e.g., Ux for ICs, consistent addrs like 0x40, GPIO refs matched to instruction set). Input/output silk names are shortened to ≤4 letters where possible (e.g., BPOS for BATT_POS, BNEG for BATT_NEG, SH+ unchanged as already short, PWM1 for PWM1_OUT, ENBL unchanged, TMP1 unchanged, etc.) for easy reading while retaining sense. Unused blocks from prior versions (e.g., detailed EMI/Protection as a separate block) are consolidated into relevant sections where they appear in schematics (e.g., ferrites in Power, ESD in Levels).

### 1. Power and Filtering Block

#### Description

This block handles the conversion and filtering of the variable marine DC input (9-32V, typically from a battery bank in 12V or 24V systems) to stable regulated voltages required by the system. It generates a 3.3V rail for the ESP32 MCU, logic circuits, and low-power components, while providing fused +12V outputs for sensor power (SPWR, suitable for 4-20mA pressure sensors requiring 12-30V) and general use. The design emphasizes robustness in marine environments, including protection against voltage transients (e.g., load dumps from alternators), reverse polarity, overcurrent, and EMI/noise from engine rooms or nearby high-current circuits. EMI filtering is critical to prevent noise propagation to sensitive analog/digital blocks, using ferrite beads (e.g., BLM21PG331SN1D), bypass capacitors, and efficient switching regulation with low EMI. Additional conditioning includes enhanced input/output filtering for stability, ripple reduction, and transient suppression to ensure reliable operation under variable voltages and harsh conditions (vibration, humidity, salt exposure). The use of a buck converter improves efficiency (up to 95%) and reduces heat dissipation compared to linear regulators, making it suitable for enclosed marine setups with limited cooling.

#### Key Components

- **U6: TPS5430DDA** (Buck converter, adjustable output set to 3.3V): Wide input (5.5-36V), high-efficiency (up to 95%), 3A capable switching regulator for handling ESP32 peaks (e.g., Wi-Fi TX up to 355mA) in noisy environments; low EMI with internal compensation.
- **F1: 5A Fuse** (2C17700142 3): Overcurrent protection on input.
- **D5: 1N5819HW-7-F** (Schottky diode): Reverse polarity protection on input (minimal voltage drop ~0.3V).
- **L1, L4, L7: BLM21PG331SN1D** (Ferrite beads, 330Ω at 100MHz): EMI filters on input power lines and outputs to suppress high-frequency noise.
- **C3: 4.7uF** (C23733), **C4: 0.1uF** (C1525): Input capacitors for decoupling ripple and stabilizing VIN; placed close to U6 VIN pin.
- **C9: 100nF** (C1525), **C10: 0.1uF** (C1525), **C11: 10uF** (C15525), **C12: 10uF/50V** (C503219), **C13: 220uF/10V** (C747973): Output capacitors for 3.3V rail stability.
- **D15: MBR340FT1G** (Catch diode): Freewheeling diode for inductor current during off-cycle.
- **L6: CDRH104RNP-150NC** (15uH inductor): Core component for buck energy storage; low DCR and EMI.
- **R25: 10K/4** (C25374), **R24: 220/4** (C23079): Feedback and compensation resistors.
- **F4: 1A Fuse** (2C17700142 3) for +12V Fused output; **F5: 1A Fuse** for +3.3V Fused output.
- **C16: 1.5nF** (C1595), **C17: 150pF** (C1594): Additional filtering.

#### Interfaces

- **Inputs**: 12V BAT+ (protected), 12V BAT- (GND).
- **Outputs**: +3.3V (fused), +12V Fused, +3.3V (unfused for internal use).
- **Integration**: Supplies power to all other blocks (e.g., +3.3V to INA226s, +12V to levels monitors).

#### Power Needs

- Input: 9-32V DC, up to 3A (for full load including downstream components).
- Output: 3.3V at up to 3A; +12V fused at 1A.

#### Rationale for Separation

This block is isolated as the primary power entry point, with EMI filtering to protect downstream sensitive circuits (e.g., INA226 measurements). In PCB layout, place near board edge for input terminals, with thick traces (100mil+) for high-current paths.

#### Schematic Instructions

1. **Input Protection**: Fuse F1 and diode D5 on BAT+ line.
2. **Buck Conversion**: U6 VIN from filtered BAT+, output to +3.3V with L6, D15, and caps.
3. **Filtering**: Ferrite beads L1/L4/L7 on paths; multiple bypass caps near IC pins.
4. **Outputs**: Fused branches for +12V and +3.3V.
5. **Additional Conditioning**: TVS diodes (implied for transients); star-grounding.
6. **Firmware**: Monitor via INA226 in other blocks; no direct code here.
7. **Safety**: Fuses for overcurrent; coating for marine use.

### 2. PWM Drivers Block

#### Description

This block provides two independent PWM channels (PWM1 and PWM2) for controlling high-current loads like alternator fields (up to 20A). Each channel isolates the ESP32 GPIO signal via an optocoupler for noise immunity and ground loop prevention, then drives a MOSFET for switching. Designed for marine alternator regulation, with fast switching (e.g., for PID control via software) and protection against back-EMF via diodes. EMI is mitigated with bypass caps and resistors. The redundant PWM2 allows for failover or dual-field control.

#### Key Components

- **U7/U8: PC817C-S** (Optocouplers): Isolate PWM input from GPIO to driver; input LED with 1k resistor (R19/R20).
- **Q1/Q2: IRL60R075CFD7** (N-channel MOSFETs): High-current switching (low Rds(on) for efficiency); gate driven by opto output.
- **D13/D14: MURS360T3G** (Ultrafast diodes): Freewheeling/protection against inductive kickback.
- **R15/R16: 100Ω** (C17408): Gate resistors for MOSFET control.
- **C7/C8: 0.1uF** (C1525): Bypass caps for noise filtering on opto/MOSFET.
- **R4: 100Ω** (not labeled per channel, but implied in path).
- **U3B/U3A: TC4427EPA** (MOSFET drivers, dual non-inverting): Boost signal to drive MOSFET gates; VDD from +3.3V.
- **C3008369: 1** (likely bypass for drivers).

#### Interfaces

- **Inputs**: PWM1/PWM2 (from ESP GPIO), +12V BAT, GND.
- **Outputs**: PWM1_GPIO/PWM2_GPIO (high-current switched outputs).
- **Integration**: Connects to ESP32 GPIOs (e.g., GPIO14/16); outputs to bulkier terminals for field coils.

#### Power Needs

- Logic: +3.3V from Power Block.
- Switching: +12V BAT direct for high-current path (up to 20A post-MOSFET).

#### Rationale for Separation

High-current switching isolated to prevent EMI coupling to analog blocks; thicker traces and thermal vias required in PCB zone.

#### Schematic Instructions

1. **Isolation**: PWM signal through resistor/capacitor to opto input.
2. **Driving**: Opto output to TC4427 INA/INB, OUTA/OUTB to MOSFET gate via resistor.
3. **Protection**: Diode across MOSFET drain-source.
4. **Firmware**: ESP32 PWM generation; duty cycle control via ESPHome.
5. **Safety**: Flyback diodes; overcurrent monitoring via separate INA226.

### 3. Levels Monitor Block

#### Description

This block monitors two pressure-based level sensors (e.g., 4-20mA for 0-1m tanks) using INA226 current/voltage monitors. Each INA226 measures shunt voltage across a resistor in the current loop, powered from +12V. Addresses differentiate devices on shared I2C bus. Includes ESD protection for exposed lines and filtering for marine noise.

#### Key Components

- **U4/U5: INA226** (Current/voltage monitors): Addr 0x41 (U4, A0=0x41) and 0x45 (U5, A1=0x45); measure VBUS (+12V) and shunt (VIN+/VIN-).
- **L2/L3: BLM21PG331SN1D** (Ferrite beads): EMI filtering on +3.3V supply.
- **C5/C6: 0.1uF** (C1525): Bypass caps.
- **D11/D12: ESD52328T1G** (ESD diodes): Protection on LVL1/LVL2 inputs.
- **R13/R14: 17.4kΩ/7** (C13275): Likely pull-ups or dividers.
- **C32733/C32819**: Additional caps/diodes.

#### Interfaces

- **Inputs**: +3.3V, +12V, GND, LVL1/LVL2 (sensor signals), SDA/SCL.
- **Outputs**: SDA_GPIO/SCL_GPIO (I2C to MCU), Alert (optional).
- **Integration**: Shared I2C with other monitors; sensors powered from SPWR (+12V).

#### Power Needs

- +3.3V for logic; +12V for sensor loops.

#### Rationale for Separation

Analog-sensitive; isolate from power zones with ground splits.

#### Schematic Instructions

1. **Monitoring**: VIN+ to sensor shunt, VBUS to +12V.
2. **I2C**: Pull-ups shared; addresses set via A0/A1.
3. **Protection**: ESD diodes on inputs.
4. **Firmware**: Read current via ESPHome; convert to level %.

### 4. Battery Charger Block

#### Description

This block provides LiFePO4 battery charging (up to 500mA) from +5V input (e.g., USB) and a regulated +5V output from +12V. Includes status LED and programming resistor for charge current. Suitable for supplemental charging in marine setups.

#### Key Components

- **U15: MCP73831-2-OT** (Charger IC): Single-cell Li-Ion/Li-Po charger; PROG resistor (R32=2K) sets current.
- **U14: AMS1117-5.0** (LDO regulator): +5V output from +12V input.
- **D22: LED** (C2286): Charge status.
- **R30: 1K** (C11702): LED resistor.
- **C25/C27: 10uF** (C15525): Input/output caps.
- **C28: 0.1uF** (C1525): Bypass.
- **D23/D24: SS14** (C2480): Protection diodes.
- **CN1: JST-GH 1.25mm** (C161690): +5V connector.
- **C30: 10uF** (C5325): Output cap.
- **F6: 1A Fuse** (2C17700142 3): On +5V output.

#### Interfaces

- **Inputs**: +5V (for charging), +12V, GND, VBAT.
- **Outputs**: VBAT (charged battery), +5V Fused.
- **Integration**: Optional for board; connects to battery monitor.

#### Power Needs

- Input: +5V for charger; +12V for regulator.
- Output: +5V at up to 1A (fused).

#### Rationale for Separation

Charging is auxiliary; separate to avoid interference with main power.

#### Schematic Instructions

1. **Charging**: +5V to U15 VIN, VBAT output with caps/diodes.
2. **Regulation**: +12V to U14 VIN, +5V out.
3. **Status**: LED on STAT pin via resistor.
4. **Firmware**: Monitor via INA226 if integrated.

### 5. CAN Interface Block

#### Description

This block provides a CAN bus transceiver for integration with marine networks (e.g., Victron, Raymarine, Yanmar, SignalK). Includes ESD protection, termination jumper, and filtering for reliable communication in noisy environments.

#### Key Components

- **U9: TJA1050T/CM,118** (CAN transceiver): High-speed, connects to ESP32 CAN TX/RX.
- **D16: PESD1CAN** (C2687121): ESD protection on CANH/CANL.
- **L5: DLW21SN900SQ2L** (C97856): Common-mode choke for EMI.
- **R21: 120Ω** (C17437): Termination resistor (via JP1 jumper).
- **C14: 47nF** (C1622): Bypass.
- **C15: 0.1uF** (C1525): Decoupling.
- **C18: 0.1uF**: Additional cap.
- **JP1: Jumper** (C150949): For termination.

#### Interfaces

- **Inputs**: +5V, GND, CAN TX_GPIO, CAN RX_GPIO.
- **Outputs**: CANH, CANL, GND.
- **Integration**: To ESP32 GPIO1/3; optional termination.

#### Power Needs

- +5V from Battery Charger Block.

#### Rationale for Separation

Digital communication; isolate with chokes to prevent noise ingress.

#### Schematic Instructions

1. **Transceiver**: TXD/RXD to GPIO; CANH/CANL to bus.
2. **Protection**: ESD diode across bus.
3. **Filtering**: Choke on bus lines.
4. **Firmware**: CAN protocol via ESP32 TWAI.

### 6. 12V Battery Monitor Block

#### Description

This block monitors the 12V battery voltage and current using an INA226, with direct VBUS measurement (up to 36V) and shunt sense (SH+/SH-) for external high-current shunt (e.g., 400A/75mV). Low-side configuration for accuracy; shared I2C.

#### Key Components

- **U12: INA226** (Current/voltage monitor): Addr 0x40 (A1=0x40); VIN+/VIN- to SH+/SH-, VBUS to +12V.
- **L9: BLM21PG331SN1D** (Ferrite bead): EMI on +3.3V.
- **C21: 0.1uF** (C1525): Bypass cap.

#### Interfaces

- **Inputs**: +3.3V, +12V, GND, SH+, SH-.
- **Outputs**: SDA_GPIO/SCL_GPIO, Alert.
- **Integration**: Shared I2C; shunt external.

#### Power Needs

- +3.3V for logic.

#### Rationale for Separation

Analog monitoring; place near terminals for short sense traces.

#### Schematic Instructions

1. **Monitoring**: VBUS to +12V, VIN to shunt.
2. **I2C**: To MCU.
3. **Firmware**: Read voltage/current; integrate with HA.

### Unused GPIOs and Breakout Recommendations

The ESP32-S3-WROOM-2 exposes a subset of GPIOs (primarily 0–21, with some higher like 35–48 for ADC/touch, per datasheet). Based on the assigned GPIOs (1,3,4,5,9,10,12,14,15,16,17,18,19,21), unused exposed GPIOs include: GPIO0 (boot strapping—avoid for general use), GPIO2, GPIO6, GPIO7, GPIO8, GPIO11, GPIO13, GPIO20, GPIO35–48 (ADC-capable, good for analog expansions).

- **Recommended for Breakout (Solder Pins)**: GPIO2, GPIO6, GPIO7, GPIO8, GPIO11, GPIO13, GPIO20 (general-purpose, no conflicts; add to low-power section as G2O, G6O, etc., for future expansions like extra sensors or debug).
- **Protection**: Add ESD TVS diodes (e.g., ESD5Z3.3T1G) on each; optional pull-up/down (10kΩ) and 0.1μF bypass per pin for noise/ESD immunity.
