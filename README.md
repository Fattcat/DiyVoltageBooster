# Adjustable DC-DC Boost Converter (0–30 V, up to 6 A)

A DIY, PCB-designed adjustable boost converter with dual 4-digit 7-segment displays (voltage + current), RGB load-level LEDs, a potentiometer for output voltage, and a settable current limit that protects the connected device.

> **Status:** design phase — schematic + BOM defined, not yet built/tested. Designed for capture in EasyEDA or KiCad.

## Table of Contents

- [Design assumptions](#design-assumptions--read-this-first)
- [Architecture](#architecture)
- [Power stage](#power-stage)
- [Output voltage setting — POT1](#output-voltage-setting--pot1)
- [Current & voltage sensing — INA226](#current--voltage-sensing--ina226)
- [Overload protection](#overload-protection)
- [Display — 2× 4-digit 7-segment via one MAX7219](#display--2-4-digit-7-segment-via-one-max7219)
- [MCU and pinout](#mcu-and-pinout)
- [Bill of Materials](#bill-of-materials)
- [Full connection list (pin-to-pin)](#full-connection-list-pin-to-pin)
- [PCB layout notes](#pcb-layout-notes)
- [Building the schematic (EasyEDA / KiCad)](#building-the-schematic-easyeda--kicad)
- [Safety & first power-on](#safety--first-power-on)
- [License](#license)

## Design assumptions — read this first

- **Vin = 9–24 V DC** (recommended range). Below ~9 V, achievable power at high Vout drops sharply — at Vin=5V with the full 30V/6A corner, input current would exceed 40A, which isn't reasonable for a hobby PCB. If your real source is 5V only, reduce the max output current (to ~1–2A) or rework the power-stage sizing.
- A boost converter **physically cannot regulate below Vin**. The real adjustable output range is roughly (Vin + 1–2V) up to 30V — not literally from 0V. True 0V regulation needs an additional linear/buck post-stage.
- The power stage is sized for a realistic design point around **150W** (e.g. 6A at ~25V, or 4A at 30V) with short-term headroom to 180W (6A/30V simultaneously). If you genuinely need 6A AND 30V as a continuous state, the inductor and MOSFET need to go up a class (bigger, pricier) — the BOM below assumes the 150W design point.
- Ammeter resolution with a single shunt range degrades at very low currents (roughly tens-of-mA accuracy below ~50mA). True mA-accurate readings across the full 6A range would need an auto-ranging/switchable shunt — not included here.
- Exact compensation-network values for the control loop (RC network on the controller's COMP pin) are normally tuned on the bench against real ripple — that's standard practice for home-built switchers, not a design flaw.

## Architecture

**Power path:** Input → boost converter (L, MOSFET, diode) → protection MOSFET switch → output/load. POT1 sets the output voltage directly in the boost converter's feedback divider (analog, independent of the MCU).

![Power path block diagram](images/power_path.svg)

**Control path:** INA226 (output V/I sensing) and POT2 (Imax setting) feed the MCU, which drives the displays, LED indicators, and the protection MOSFET.

![Control and measurement block diagram](images/control_path.svg)

## Power stage

At ~150–180W and a boost ratio up to 1:3, an off-the-shelf integrated module (XL6009/MT3608-class) isn't enough — its internal switch is only good for tens of watts. This is a discrete boost design with an external MOSFET.

- **Controller IC:** current-mode boost controller, LM3478-class (Vin 2.97–40V, external N-MOSFET, switching frequency set via RT resistor).
- **Switching MOSFET Q1:** N-channel, Vds ≥ 60V, Id ≥ 25A, RDS(on) as low as possible (ideally < 10mΩ at Vgs=10V), TO-220/TO-247 package (hand-solderable + heatsink screw mount).
- **Rectifier diode D1:** Schottky, Vr ≥ 60V, If(avg) ≥ 10A, IFSM ≥ 30A.
- **Inductor L1:** ≈15µH, Isat ≥ 20A (25A recommended headroom), DCR as low as possible (< 5mΩ). The largest and most expensive part in the design — search EasyEDA/LCSC's parametric search for a "shielded power inductor" matching these specs.
- **Input caps:** 2× 100µF/35V low-ESR electrolytic + 100nF ceramic.
- **Output caps:** 3× 100µF/50V low-ESR electrolytic + 100nF ceramic, optionally +1× 10µF ceramic for fast transients.

(Sizing basis: Vin=9–12V, Vout=30V, design power ~150W, f≈150kHz, ~30% Iin ripple → ~12µH calculated, rounded up to the standard 15µH value.)

### Auxiliary supply rails
- LM3478 needs its own VCC, and the gate driver benefits from a higher rail for fully enhancing the MOSFET. Add a **small auxiliary 12V regulator/converter** fed from Vin, powering U1's VCC and its internal gate driver — this gives reliable gate drive regardless of whether Vin is 9V or 24V.
- A second auxiliary **5V regulator** powers the MCU, displays, and INA226.

## Output voltage setting — POT1

LM3478's FB pin reference is ~1.26V:

**Vout = 1.26V × (1 + R_top / R_bottom)**

Instead of a fixed R_bottom, use a **10kΩ 10-turn panel potentiometer** (fine adjustment) in series with a small fixed resistor (e.g. 1kΩ) so it can't be turned all the way to 0Ω (which would cause an uncontrolled high Vout). Choose R_top so that POT1 at maximum gives Vout ≈ 30–32V; fine-tune the exact values on the bench with a multimeter (starting point: R_top ≈ 39–47kΩ).

![Boost converter power loop schematic](images/schematic_power_loop.svg)

*U1 (the controller IC) isn't drawn — GATE and FB are shown as labeled stubs pointing to its pins. The 1kΩ series resistor between POT1 and GND, and U1's own VCC/GND/COMP/RT/CS pins, are omitted here for clarity (see the full connection list below).*

## Current & voltage sensing — INA226

- **R_shunt = 10mΩ (0.01Ω), ≥1W** — at 6A it dissipates ~0.36W, so 1W gives margin. Place it in series with the output, after the protection MOSFET, right before the load terminal block.
- INA226 senses both the shunt voltage (current) and bus voltage (Vout, up to 36V — covers our 0–30V range), so no separate voltage divider is needed for the voltmeter.
- Firmware calibration (standard INA226 procedure):
  - Current_LSB = 6.000A / 2^15 ≈ 183µA
  - CAL = 0.00512 / (Current_LSB × R_shunt) ≈ 2796
  - Power_LSB = 25 × Current_LSB

## Overload protection

- **POT2** (linear 10kΩ, 0–5V divider) is read via the MCU's ADC and linearly mapped to 0–6.000A → the configured Imax.
- **Protection element:** P-channel MOSFET Q2, high-side switch in series with the output + line, driven through an NPN level-shift transistor Q3 (see the full connection list). Logic: GPIO HIGH = switch closed (normal operation), GPIO LOW = disconnected (fault).
- **Firmware loop (runs roughly every 20ms):**
  1. Read I, V from INA226; read the Imax setting from POT2.
  2. If I > Imax for N consecutive samples (debounce ~100ms) → open the switch, latch the red LED solid, enter a latched fault state.
  3. The fault state can only be cleared with the RESET button — never automatically (to avoid repeatedly re-energizing into a short).
  4. Outside a fault: LEDs show load % relative to Imax — green < 50%, orange 50–85%, red > 85% (unlatched, warning only).
  5. Update both displays.

## Display — 2× 4-digit 7-segment via one MAX7219

- 8 individual common-cathode 7-segment digits (e.g. 0.56", 5161AS-type) wired as a single 8-digit multiplex through one MAX7219 (DIN, CLK, CS/LOAD).
- Segments a–g + DP of all 8 digits are wired in parallel to the MAX7219's SEG outputs (one shared RSET resistor sets brightness for all digits at once).
- Cathodes: **DIG0–DIG3 = voltage** (format "XX.XX" V), **DIG4–DIG7 = current** (format "X.XXX" A — gives 1mA display resolution).
- MAX7219 and the MCU (below) both run on 5V logic, so no level shifter is needed between them.

## MCU and pinout

**Arduino Nano (ATmega328P, 5V/16MHz)** is recommended — it runs natively at 5V, so INA226 and MAX7219 (both need ~4V+ logic) connect directly without level shifters. (An ESP32 would also work, but since it's 3.3V logic it would need a level shifter to MAX7219 — unnecessary complexity when WiFi/BT isn't needed here.)

| Signal | Pin (Arduino Nano) | Note |
|---|---|---|
| INA226 SDA | A4 | I2C |
| INA226 SCL | A5 | I2C |
| MAX7219 DIN | D11 (MOSI) | SPI |
| MAX7219 CLK | D13 (SCK) | SPI |
| MAX7219 CS/LOAD | D10 | |
| POT2 (Imax) | A0 | 0–5V analog input |
| Green LED | D2 | via 330Ω |
| Orange LED | D3 | via 330Ω |
| Red LED | D4 | via 330Ω |
| Protection MOSFET (Q3 base) | D5 | via 4.7kΩ |
| RESET button | D6 | INPUT_PULLUP, button to GND |

**POT1 is not connected to the MCU** — it goes directly into the boost converter's FB divider, purely analog. It keeps working even if the MCU hangs or resets.

**Protection MOSFET Q2 (high-side, P-channel) wiring:** Source Q2 = Vout(+), Drain Q2 = to load. R1 (10kΩ) from Gate to Source (pull-up, default OFF). Q3 collector → Gate Q2; Q3 emitter → GND; Q3 base → 4.7kΩ → MCU pin D5. MCU HIGH → Q3 saturates → Gate Q2 pulled toward GND → Vgs strongly negative → Q2 turns ON. MCU LOW → Q3 off → R1 pulls Gate up to Vout → Vgs≈0 → Q2 OFF.

## Bill of Materials

| # | Part | Value / type | Package | Note |
|---|---|---|---|---|
| 1 | Boost controller | LM3478 or equivalent current-mode boost controller | SO-8 | verify current LCSC availability |
| 2 | Switching MOSFET Q1 | N-channel, 60V+, RDS(on)<10mΩ | TO-220/TO-247 | parametric search in EasyEDA |
| 3 | Rectifier diode D1 | Schottky, 60V, 10A | TO-220/DPAK | |
| 4 | Inductor L1 | 15µH, Isat≥20A, low DCR | shielded, large | priciest part — check footprint dimensions before layout |
| 5 | Protection MOSFET Q2 | P-channel, 60V, RDS(on)<30mΩ (e.g. IRF4905-class — verify current stock) | TO-220 | high-side switch |
| 6 | NPN driver Q3 | 2N3904 or equivalent | TO-92 | level-shift for Q2 |
| 7 | Shunt resistor | 10mΩ, 1W, 1% tolerance | SMD 2512 or THT | for INA226, Kelvin-connected |
| 8 | Current/voltage sensor | INA226 | SOT-23-8 / MSOP | I2C |
| 9 | Display driver | MAX7219 | DIP-24 / SOIC-24 | |
| 10 | 7-seg displays | 0.56" common cathode ×8 | THT | 4× voltage + 4× current |
| 11 | MCU | Arduino Nano (ATmega328P) | module | or bare ATmega328P + 16MHz crystal |
| 12 | POT1 | 10kΩ, 10-turn, panel mount | THT | output voltage setting |
| 13 | POT2 | 10kΩ linear, panel mount | THT | Imax setting |
| 14 | LEDs | 5mm — green, orange, red | THT | |
| 15 | Aux 12V regulator | small buck/boost module or LM2596-ADJ | module/SMD | gate drive for LM3478 |
| 16 | Aux 5V regulator | LM7805 or small buck module | TO-220/SOT | logic (MCU, displays, INA226) |
| 17 | Input capacitors | 100µF/35V ×2 + 100nF ×2 | electrolytic/ceramic | |
| 18 | Output capacitors | 100µF/50V ×3 + 100nF ×2 | electrolytic/ceramic | |
| 19 | Input fuse | fast-blow, sized to max Iin (e.g. 20A automotive) | holder + fuse | physical backup on top of the electronic protection |
| 20 | Connectors | screw terminals or XT60-class, sized to current | THT | not thin pin headers for the power section! |
| 21 | Reset button | tactile button | THT | |
| 22 | Misc R/C | per schematic (FB divider, RT, compensation, pull-ups) | SMD/THT | fine-tune during build |

*Verify exact part numbers in EasyEDA via Search Parts → LCSC against the specs above — availability shifts, so no specific catalog number is pinned per part.*

## Full connection list (pin-to-pin)

A direct text form of the schematic — enough to wire up directly in EasyEDA or KiCad regardless of which exact symbols you use per part.

**Power**
- Vin+ → fuse F1 → C_in1(+), C_in2(+) → L1 pin1 (boost input) → aux 12V regulator U3 input → aux 5V regulator U4 input
- Vin−/GND → common GND star point at C_in1 → C_in1(−), C_in2(−), U3 GND, U4 GND
- U3 OUT (12V) → U1 (LM3478) VCC
- U4 OUT (5V) → Arduino Nano 5V, INA226 (U2) VS, MAX7219 (U5) VCC

**Boost converter (L1, Q1, D1, U1=LM3478)**
- L1 pin2 (SW node) → Q1 drain, D1 anode
- Q1: source → power GND; gate → U1 GATE pin
- D1: cathode → Vout(+) (followed immediately by C_out1..3 + 100nF, all on Vout(+) and GND)
- U1 pins: VIN←Vin, VCC←U3 OUT, GND←common GND, GATE→Q1 gate, CS→sense point at Q1 source (the controller's internal current limit — not the output ammeter), FB←FB-divider/POT1 node, COMP→RC compensation network to GND, RT→resistor to GND, SS→capacitor to GND (if the IC has a soft-start pin)

**FB divider + POT1 (output voltage setting)**
- Vout(+) → R_top (~39–47kΩ) → "FB" node
- FB node → U1 FB pin
- FB node → POT1 top terminal; POT1 wiper tied to the bottom terminal (rheostat wiring) → R_series (1kΩ) → GND

**Protection MOSFET Q2 + driver Q3**
- Vout(+) → Q2 source
- Q2 drain → "Vload+" node (continues to the shunt)
- Q2 gate → R1 (10kΩ) → Q2 source (pull-up, default OFF)
- Q2 gate → Q3 collector
- Q3 emitter → GND
- Q3 base → R2 (4.7kΩ) → MCU pin D5

**Shunt + INA226 (U2, ammeter/voltmeter)**
- Vload+ → R_shunt (10mΩ) → "Vout_final" node → output terminal block J2 pin1
- INA226 VIN+ = shunt side closer to Q2 (Kelvin connection directly on the resistor pad)
- INA226 VIN− = shunt side closer to the terminal block (Kelvin connection)
- INA226 VS←U4 OUT (5V), GND←common GND
- INA226 A0, A1 (address pins) → GND (address 0x40, default)
- INA226 SDA→MCU A4, SCL→MCU A5 (add 4.7kΩ pull-ups from SDA and SCL to 5V if your INA226 breakout doesn't already include them)
- INA226 ALERT pin → left unconnected (firmware polls V/I, no interrupt used)

**Output terminal block J2**
- J2 pin1 ← Vout_final (after the shunt)
- J2 pin2 ← GND

**MCU (Arduino Nano)**
- 5V←U4 OUT, GND←common GND
- A4↔INA226 SDA, A5↔INA226 SCL
- D11 (MOSI)→U5 DIN, D13 (SCK)→U5 CLK, D10→U5 CS/LOAD
- A0←POT2 wiper (POT2 top terminal→5V, bottom terminal→GND — wired as a voltage divider)
- D2→R (330Ω)→green LED anode; green LED cathode→GND
- D3→R (330Ω)→orange LED anode; cathode→GND
- D4→R (330Ω)→red LED anode; cathode→GND
- D5→R2 (4.7kΩ)→Q3 base
- D6→RESET button→GND (firmware: INPUT_PULLUP)

**MAX7219 (U5) + 8× display**
- VCC←U4 OUT (5V), GND←common GND
- DIN←MCU D11, CLK←MCU D13, CS/LOAD←MCU D10
- DOUT → unconnected (no cascade needed, one chip handles 8 digits)
- ISET→RSET resistor (10–27kΩ for desired brightness)→GND
- SEG A,B,C,D,E,F,G,DP → in parallel to the matching segments of all 8 displays
- DIG0–DIG3 → cathodes of displays 1–4 (voltage, "XX.XX" format)
- DIG4–DIG7 → cathodes of displays 5–8 (current, "X.XXX" format)

## PCB layout notes

- **Trace widths:** the power path (Vin, inductor, MOSFET, diode, Vout) carries roughly 10–20A peak. On 1oz copper at 10°C rise that's roughly 5mm+ trace width — use **2oz copper** and/or a wide copper pour instead of a thin trace, plus via stitching if it crosses layers.
- Keep the critical loop (MOSFET–diode–output cap) **as short and tight as possible** — it's the most RF-noisy part of the circuit.
- Connect the shunt resistor to INA226 with **Kelvin (4-point) connections** directly on the resistor, not through the shared power copper pour — otherwise copper IR drop will skew the current reading.
- Join analog GND (MCU, INA226, displays) and power GND (boost stage, MOSFETs) at **one star point** near the input capacitor, not as a flood everywhere — this limits switching noise coupling into the displays.
- Q1 and Q2 will dissipate roughly 1–3W each at full load — leave a copper pour with thermal vias under them, or add a small bolt-on heatsink.
- Keep the displays on a separate area of the board, away from the inductor, to avoid switching EMI.

## Building the schematic (EasyEDA / KiCad)

1. Create a new schematic project.
2. For each BOM part, use **Search Parts** (EasyEDA) to pull it from the LCSC library by part number or parametric search (ready-made footprints + 3D models, directly orderable via JLCPCB/LCSC). In KiCad, passives/discretes are in the standard **Device** library (`Device:R`, `Device:C`, `Device:D_Schottky`, `Device:Q_NMOS_GDS`, `Device:Q_PMOS_GDS`, `Device:Q_NPN_BCE`); for LM3478/INA226/MAX7219, check `Regulator_Switching`, `Sensor_Current`, and `Display_7Segment`, or pull a symbol from SnapEDA/Ultra Librarian by exact part number if KiCad's built-ins don't match.
3. Draw the schematic following the blocks above: power section → boost converter with FB divider and POT1 → protection MOSFET → output terminal block; INA226 in parallel on the output; MCU per the pin table; MAX7219 + 8 displays; LEDs + resistors; POT2 on the ADC.
4. Run DRC, then move to PCB layout (EasyEDA: "Convert to PCB").
5. Place the power components (L1, Q1, D1, caps) as tightly as possible first, route the thick power traces with a wider "power" net class, then route the thin signal/I2C/SPI traces last.
6. Add mounting holes for heatsinks/screws, and don't skip silkscreen (terminal labels, polarity markers).

## Safety & first power-on

- Before first power-up: check electrolytic polarity, diode and MOSFET orientation with a multimeter (diode test mode), confirm POT1 is at minimum.
- Do the first power-up **through a bench supply with an adjustable current limit** (e.g. 0.5A), not straight to full power — watch that Vout rises smoothly with POT1.
- Test the protection: set POT2 to a low Imax (e.g. 0.5A), use a resistive dummy load, confirm the switch actually opens on overcurrent and the red LED stays lit until RESET is pressed.
- The input fuse is an extra backup — the electronic protection guards the connected load, the fuse guards the booster itself against an internal fault.

## License
