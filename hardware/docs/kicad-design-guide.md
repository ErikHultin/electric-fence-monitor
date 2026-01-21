# KiCad Design Guide - Electric Fence Monitor Sensor Board

This guide walks you through completing the PCB design in KiCad, from schematic to manufacturing-ready files.

## Prerequisites

- **KiCad 7.0 or later** installed
- Basic familiarity with KiCad interface
- Reference: `sensor-design.md` in project root

## Project Structure

```
hardware/
├── sensor-board.kicad_pro      # Project file (ready to open)
├── sensor-board.kicad_sch      # Schematic (template)
├── sensor-board.kicad_pcb      # PCB layout (to be created)
├── libraries/
│   ├── symbols/sensor-custom.kicad_sym      # RFM95W symbol
│   └── footprints/sensor-custom.pretty/     # Custom footprints
├── bom/BOM.csv                 # Bill of materials with JLCPCB parts
└── docs/                       # Documentation
```

## Part 1: Schematic Design

### Step 1: Open the Project

1. Launch KiCad
2. File → Open Project
3. Navigate to `hardware/sensor-board.kicad_pro`
4. Open the schematic editor (Schematic Layout Editor icon)

### Step 2: Configure Symbol Libraries

1. Preferences → Manage Symbol Libraries
2. Project Specific Libraries tab
3. Add the custom library:
   - Nickname: `sensor-custom`
   - Library Path: `${KIPRJMOD}/libraries/symbols/sensor-custom.kicad_sym`
4. Verify standard libraries are available:
   - `MCU_ST_STM32L0`
   - `Regulator_Linear`
   - `Device` (R, C, L, etc.)
   - `Connector`
   - `Diode`

### Step 3: Create Hierarchical Sheets

For organization, create separate schematic sheets for each major section:

#### Create Sheet 1: Power Supply

1. Place → Hierarchical Sheet
2. Draw a rectangle on the main sheet
3. Name: "Power Supply"
4. File name: "power-supply.kicad_sch"
5. Add hierarchical pins:
   - Input: `BAT+` (power input)
   - Input: `GND` (power input)
   - Output: `+3V3` (power output)
   - Output: `GND` (ground)

6. Double-click sheet to edit, add components:

**J1 - Battery Input:**
- Symbol: `Connector:Screw_Terminal_01x02`
- Pins: 1=BAT+, 2=GND

**TVS1 - Transient Protection:**
- Symbol: `Diode:SMBJ15CA`
- Connects between BAT+ and GND
- JLCPCB Part: C82397

**U1 - 3.3V LDO:**
- Symbol: `Regulator_Linear:MCP1700-3302E_SOT23`
- VIN → BAT+ (through TVS1)
- VOUT → +3V3
- GND → GND
- JLCPCB Part: C39051

**C1 - Input Capacitor:**
- Symbol: `Device:C`
- Value: 1µF
- Between VIN and GND
- Footprint: `Capacitor_SMD:C_0805_2012Metric`
- JLCPCB Part: C28323

**C2 - Output Capacitor:**
- Symbol: `Device:C`
- Value: 1µF
- Between +3V3 and GND
- JLCPCB Part: C28323

**Power Symbols:**
- Place `power:+3V3` symbols on the +3V3 net
- Place `power:GND` symbols on all ground connections
- This creates global nets that connect across sheets

#### Create Sheet 2: Battery Voltage Sensing

Per sensor-design.md lines 66-93:

1. Create hierarchical sheet "Battery Voltage Sensing"
2. Hierarchical pins:
   - Input: `BAT+`
   - Input: `GND`
   - Output: `VBAT_SENSE` (to MCU ADC)

3. Add components:

**R1 - Voltage Divider High:**
- Symbol: `Device:R`
- Value: 120kΩ
- Footprint: `Resistor_SMD:R_0805_2012Metric`
- Tolerance: 1%
- JLCPCB Part: C103934

**R2 - Voltage Divider Low:**
- Value: 27kΩ
- JLCPCB Part: C103931

**C3 - Filter Capacitor:**
- Value: 100nF
- JLCPCB Part: C49678

**Connections:**
```
BAT+ ──┬────[R1:120k]────┬──── VBAT_SENSE (to MCU PA0)
       │                 │
     [TVS]            [R2:27k]
       │                 │
      GND           ┌────┴────┐
                    │  [C3]   │
                    │  100nF  │
                    └────┬────┘
                         │
                        GND
```

**TP1 - Test Point:**
- Symbol: `Connector:TestPoint`
- On VBAT_SENSE net for calibration

#### Create Sheet 3: Pulse Detection

Per sensor-design.md lines 259-316:

1. Create sheet "Pulse Detection"
2. Hierarchical pins:
   - Input: `SENSE_PLATE`
   - Input: `GND`
   - Output: `PULSE_SIGNAL` (to MCU comparator)

3. Add components:

**J2 - Sense Plate Input:**
- Symbol: `Connector:Screw_Terminal_01x02`
- JLCPCB Part: C8465

**R3 - Input Impedance:**
- Value: 1MΩ
- JLCPCB Part: C114757

**R4 - Filter Resistor:**
- Value: 100kΩ
- JLCPCB Part: C149504

**C4 - Bandwidth Limiting:**
- Value: 100pF
- JLCPCB Part: C1804

**D1 - Clamp Diodes:**
- Symbol: `Diode:BAV99`
- Dual diode for clamping to 0-3.3V
- JLCPCB Part: C727116

**R5 - Isolation Resistor:**
- Value: 10kΩ
- JLCPCB Part: C84376

**C5 - Noise Filter:**
- Value: 100nF
- JLCPCB Part: C49678

**Circuit:**
```
SENSE_PLATE ──[R3:1M]──┬──[R4:100k]──┬──[R5:10k]─── PULSE_SIGNAL
                       │             │
                     [C4]          [D1]
                      100pF       BAV99
                       │         clamp
                      GND          │
                                  ┌┴┐
                                  │C5│ 100nF
                                  └┬┘
                                  GND
```

#### Create Sheet 4: Microcontroller

1. Create sheet "MCU"
2. Hierarchical pins:
   - Input: `+3V3`, `GND`
   - Input: `VBAT_SENSE`
   - Input: `PULSE_SIGNAL`
   - Output: SPI signals (MOSI, MISO, SCK, NSS)
   - Output: LoRa control (DIO0, RESET)

3. Add components:

**U2 - STM32L072CBT6:**
- Symbol: `MCU_ST_STM32L0:STM32L072CBTx`
- LQFP-48 package
- JLCPCB Part: C94384

**Key pin connections:**
- VDD pins (5, 22, 28, 50) → +3V3
- VSS pins (6, 21, 27, 49) → GND
- VDDA (13) → +3V3
- VSSA (12) → GND
- PA0 (14) → VBAT_SENSE (ADC input)
- PA1 (15) → PULSE_SIGNAL (COMP1 input)
- PA4 (20) → NSS (SPI chip select)
- PA5 (21) → SCK (SPI clock)
- PA6 (22) → MISO (SPI master in)
- PA7 (23) → MOSI (SPI master out)
- PA8 (29) → LORA_RESET
- PA10 (31) → LORA_DIO0
- PA13 (46) → SWDIO (debug)
- PA14 (49) → SWCLK (debug)

**Decoupling Capacitors (C7-C10):**
- 4× 100nF capacitors
- Place near each VDD pin
- JLCPCB Part: C49678

**C6 - Bulk Capacitor:**
- Value: 4.7µF
- Near main VDD pin
- JLCPCB Part: C1779

**R6 - BOOT0 Pulldown:**
- Value: 10kΩ
- BOOT0 pin to GND
- Ensures normal boot mode

**R7 - NRST Pullup:**
- Value: 10kΩ
- NRST pin to +3V3
- Keeps MCU out of reset

**Y1 - RTC Crystal (Optional):**
- Value: 32.768kHz
- Between PC14/PC15
- JLCPCB Part: C32346
- Add 2× 12pF loading capacitors if used

#### Create Sheet 5: LoRa Module

1. Create sheet "LoRa"
2. Hierarchical pins:
   - Input: `+3V3`, `GND`
   - Input: SPI signals from MCU
   - Input: `LORA_RESET`, `LORA_DIO0`
   - Output: `ANT` (to SMA connector)

3. Add components:

**U3 - RFM95W:**
- Symbol: `sensor-custom:RFM95W` (from custom library)
- Footprint: `sensor-custom:RFM95W`
- Not available from JLCPCB - hand solder or external source

**Pin connections:**
- Pin 1, 8, 10: GND
- Pin 11: +3V3
- Pin 2: MISO
- Pin 3: MOSI
- Pin 4: SCK
- Pin 5: NSS
- Pin 6: RESET
- Pin 16: DIO0
- Pin 9: ANT → to J5 (SMA connector)

**J5 - SMA Connector:**
- Symbol: `Connector:SMA`
- JLCPCB Part: C496548
- Center pin → ANT
- Shield → GND

**Note:** RFM95W typically requires no external matching network, but check datasheet for your specific frequency band.

#### Create Sheet 6: Debug & Programming

1. Create sheet "Debug"
2. Add components:

**J3 - SWD Header:**
- Symbol: `Connector:Conn_01x04`
- Pin header, 2.54mm pitch
- JLCPCB Part: C2337
- Pinout:
  - Pin 1: +3V3
  - Pin 2: SWDIO
  - Pin 3: SWCLK
  - Pin 4: GND

**J4 - UART Header (Optional):**
- Symbol: `Connector:Conn_01x03`
- For serial debug output
- Pinout:
  - Pin 1: TX (PA2 or PA9)
  - Pin 2: RX (PA3 or PA10)
  - Pin 3: GND

**Test Points:**
- Add test points for critical signals:
  - +3V3
  - VBAT_SENSE
  - PULSE_SIGNAL
  - GND

### Step 4: Complete Main Sheet

On the root schematic sheet:
1. Place all 6 hierarchical sheet symbols
2. Connect power nets (+3V3, GND, BAT+)
3. Connect signal nets between sheets
4. Add title block information:
   - Title: "Electric Fence Monitor - Sensor Board"
   - Company: Your name/company
   - Rev: v1.0
   - Date: Current date

### Step 5: Run Electrical Rules Check (ERC)

1. Tools → Electrical Rules Checker
2. Click "Run ERC"
3. Fix any errors (warnings are often acceptable)
4. Common issues:
   - Power pins not driven: Add power flags
   - Unconnected pins: Add "no connect" flags if intentional

### Step 6: Assign Footprints

1. Tools → Assign Footprints
2. For each component, assign the footprint from `bom/BOM.csv`:
   - STM32L072: `Package_QFP:LQFP-48_7x7mm_P0.5mm`
   - MCP1700: `Package_TO_SOT_SMD:SOT-23`
   - 0805 resistors: `Resistor_SMD:R_0805_2012Metric`
   - 0805 capacitors: `Capacitor_SMD:C_0805_2012Metric`
   - BAV99: `Package_TO_SOT_SMD:SOT-23`
   - SMBJ15CA: `Diode_SMD:D_SMB`
   - Screw terminals: `TerminalBlock:TerminalBlock_bornier-2_P5.08mm`
   - RFM95W: `sensor-custom:RFM95W`

3. Click "Apply, Save Schematic & Continue"

---

## Part 2: PCB Layout

### Step 1: Open PCB Editor

1. From schematic editor: Tools → Update PCB from Schematic (F8)
2. Click "Update PCB"
3. All components will appear in the center of the PCB editor

### Step 2: Set up Design Rules

1. File → Board Setup
2. **Board Dimensions:**
   - Width: 50mm
   - Height: 50mm

3. **Design Rules → Constraints:**
   - Minimum clearance: 0.25mm
   - Minimum track width: 0.25mm
   - Minimum via diameter: 0.8mm
   - Minimum via drill: 0.4mm
   - Minimum hole to hole: 0.5mm

4. **Net Classes:**
   - Default: Track width 0.25mm
   - Power: Track width 1.0mm (assign BAT+, +3V3)
   - RF: Track width 0.3mm (assign ANT net)

### Step 3: Draw Board Outline

1. Select Edge.Cuts layer
2. Draw → Rectangle
3. Draw 50mm × 50mm rectangle centered at origin
4. Place mounting holes:
   - Footprint: `MountingHole:MountingHole_3.2mm_M3`
   - Positions (from bottom-left corner):
     - (3, 3)
     - (47, 3)
     - (47, 47)
     - (3, 47)

### Step 4: Component Placement

Follow the layout plan from the implementation document:

#### Top Section (Power Input):
- J1 (Battery terminal) at top-left edge
- TVS1 immediately next to J1
- U1 (LDO) centered, below TVS1
- C1, C2 close to U1 input/output

#### Left Section (Sensing):
- J2 (Sense plate terminal) on left edge, mid-height
- Pulse detection circuit (R3, R4, C4, D1, R5, C5) below J2
- Battery divider (R1, R2, C3) near bottom-left

#### Center (MCU):
- U2 (STM32) in the center
- Decoupling caps (C7-C10) immediately adjacent to VDD pins
- C6 (4.7µF) near pin 5 (main VDD)
- Crystal Y1 (if used) close to PC14/PC15 with ground guard ring

#### Right Section (LoRa):
- U3 (RFM95W) on right side, upper half
- Keep away from MCU crystal to avoid interference
- J5 (SMA) at right edge, centered vertically

#### Bottom (Debug):
- J3 (SWD header) at bottom edge, centered
- J4 (UART) next to J3 if included

**Tips:**
- Group related components together
- Orient components for logical signal flow
- Place decoupling caps on same side as IC
- Keep high-frequency traces short (SPI, RF)

### Step 5: Routing

#### Power Routing:
1. Route BAT+ from J1 → TVS1 → U1 with 1.0mm traces
2. Route +3V3 from U1 → decoupling caps → ICs with 0.5-1.0mm traces
3. Use multiple vias to connect to ground plane

#### Signal Routing:
1. **SPI Bus:** Route MOSI, MISO, SCK, NSS from MCU to RFM95W
   - Keep traces parallel and similar length
   - 0.25mm width is fine for this low-speed SPI

2. **Analog Signals:** Route VBAT_SENSE and PULSE_SIGNAL carefully
   - Avoid running near noisy digital signals
   - Keep traces short
   - Route over solid ground plane

3. **RF Trace:** ANT signal from RFM95W to SMA
   - Width: 0.3mm for 50Ω impedance (on 1.6mm FR4)
   - Keep as short and straight as possible
   - No ground plane directly under trace
   - Ground on both sides (coplanar waveguide)
   - No vias in RF path if possible

#### Ground Plane:
1. Select B.Cu (bottom copper) layer
2. Draw → Filled Zone
3. Net: GND
4. Fill entire board area
5. Clearance: 0.25mm
6. Thermal relief for pads: Yes

7. Add stitching vias around board perimeter (every 5-10mm)
   - These connect top and bottom ground

### Step 6: Silkscreen

1. Add component reference designators (R1, C1, U1, etc.)
2. Add polarity marks for polarized components
3. Label connectors:
   - J1: "BAT+ BAT-"
   - J2: "SENSE GND"
   - J3: "+3V3 DIO CLK GND"
4. Add board info:
   - Top silk: "Electric Fence Monitor v1.0"
   - Add your name/logo if desired

### Step 7: Design Rule Check (DRC)

1. Tools → Design Rules Checker
2. Click "Run DRC"
3. Fix all errors:
   - Clearance violations
   - Track width violations
   - Unconnected nets

4. Warnings to check:
   - Silkscreen over pads (fix these)
   - Courtyards overlap (acceptable if intentional)

### Step 8: 3D View

1. View → 3D Viewer
2. Check component placement visually
3. Verify mounting holes are accessible
4. Check for component clearance issues
5. Verify connectors are at board edges

---

## Part 3: Generate Manufacturing Files

### Gerber Files for JLCPCB

1. File → Fabrication Outputs → Gerbers (.gbr)
2. Output directory: `gerbers/`
3. Layers to include:
   - F.Cu, B.Cu (copper layers)
   - F.Paste, B.Paste (solder paste)
   - F.Silkscreen, B.Silkscreen
   - F.Mask, B.Mask (solder mask)
   - Edge.Cuts (board outline)

4. Settings:
   - Format: 4.6, unit mm
   - Check "Use Protel filename extensions"
   - Check "Generate Gerber job file"

5. Click "Plot"

6. Generate Drill Files:
   - Click "Generate Drill Files"
   - Format: Excellon
   - Units: mm
   - Click "Generate Drill File"

7. **Zip all gerber files** for upload to JLCPCB

### BOM and CPL for Assembly

#### BOM File:
- Already created at `bom/BOM.csv`
- Format for JLCPCB:
  - Designator
  - MPN (Manufacturer Part Number)
  - Quantity
  - LCSC Part number (C##### format)

#### CPL (Pick and Place) File:
1. File → Fabrication Outputs → Component Placement (.pos)
2. Format: CSV
3. Units: mm
4. Include only SMT components (not through-hole)
5. **Important:** JLCPCB requires specific format:
   - Columns: Designator, Mid X, Mid Y, Layer, Rotation
   - You may need to manually edit the file to match JLCPCB format

**CPL Format Example:**
```
Designator,Mid X,Mid Y,Layer,Rotation
C1,10.5,15.2,Top,0
C2,12.5,15.2,Top,0
...
```

---

## Part 4: Ordering from JLCPCB

### Step 1: Upload Gerbers

1. Go to https://jlcpcb.com
2. Click "Add Gerber file"
3. Upload your `gerbers.zip`
4. Wait for automatic parsing

### Step 2: PCB Options

- **Base Material:** FR-4
- **Layers:** 2
- **Dimensions:** 50 × 50 mm (auto-detected)
- **PCB Qty:** 5 (minimum)
- **Thickness:** 1.6mm
- **PCB Color:** Green (cheapest) or your preference
- **Silkscreen:** White
- **Surface Finish:** HASL (cheap) or ENIG (better, more expensive)
- **Copper Weight:** 1 oz
- **Remove Order Number:** Yes (if you don't want JLCPCB's mark)

### Step 3: Enable SMT Assembly

1. Toggle "SMT Assembly" to ON
2. Select side: Top
3. Tooling holes: Added by JLCPCB
4. Confirm

### Step 4: Upload Assembly Files

1. Click "Add BOM File" → upload `BOM.csv`
2. Click "Add CPL File" → upload your `.pos` file
3. Click "Next"

### Step 5: Component Matching

JLCPCB will show a table matching your BOM to their parts database:

1. **Review each line:**
   - Check LCSC part number matches your BOM
   - Verify component is in stock
   - Note "Basic" vs "Extended" parts

2. **Extended parts** incur $3 setup fee per unique part:
   - STM32L072: Extended
   - MCP1700: Extended
   - Most passives: Basic

3. **Components NOT available from JLCPCB:**
   - RFM95W module: Mark as "Do not place"
     - You'll hand-solder this later
   - Through-hole connectors (J1, J2, J3): Do not place
     - Hand-solder these as well

4. Click "Next"

### Step 6: Component Placement Review

1. JLCPCB shows 3D preview of board with components
2. Check:
   - All components are correctly placed
   - Orientations are correct (especially ICs, diodes)
   - No components overlapping

3. If wrong:
   - Go back and fix CPL file
   - Or use JLCPCB's web editor to adjust

4. Click "Next"

### Step 7: Review and Pay

1. Review total cost:
   - PCB fabrication
   - Assembly service
   - Components
   - Shipping

2. Typical cost for 5 boards: $70-120 USD

3. Select shipping (DHL Express recommended for speed)

4. Add to cart and checkout

---

## Part 5: Assembly and Testing

### What You'll Receive from JLCPCB

- 5× PCBs with SMD components assembled
- **Not assembled:** RFM95W module, screw terminals, SMA connector, pin headers

### Manual Assembly Steps

1. **Solder RFM95W module** (if using castellated version):
   - Apply solder paste to pads
   - Place module carefully (check orientation!)
   - Reflow with hot air or on hot plate
   - Or use soldering iron with thin tip

2. **Solder screw terminals:**
   - J1, J2: Through-hole, easy to solder
   - Insert from top, solder from bottom

3. **Solder SMA connector:**
   - Ensure mechanical stability
   - Good solder joints on all pins

4. **Solder pin headers:**
   - J3, J4: Standard 2.54mm pin headers
   - Consider using right-angle for easier access

### Testing

#### Visual Inspection:
- Check for solder bridges
- Verify component orientations
- Check for missing components

#### Continuity Tests:
- BAT+ to U1 input
- U1 output to +3V3 net
- All GND connections continuous

#### Power-On Test:
1. Connect 12V battery to J1
2. Measure voltage at U1 output: Should be 3.3V
3. Check current draw: <100µA in sleep (no firmware)

#### Programming Test:
1. Connect ST-Link or similar programmer to J3
2. Use STM32CubeProgrammer or openocd
3. Try to connect and read device ID
4. If successful, flash a blink LED test firmware

#### Functional Tests:
1. **Battery voltage sense:** Apply known voltage, read ADC
2. **Pulse detection:** Inject test pulse, check comparator output
3. **LoRa:** Load test firmware, attempt transmission

---

## Appendix A: Alternative RFM95W Options

If castellated RFM95W modules are hard to source:

### Option 1: Breakout Board with Headers

1. Purchase RFM95W on a breakout board (Adafruit, SparkFun, etc.)
2. Modify PCB footprint:
   - Replace castellated pads with 2× 8-pin header footprint
   - Use 2.54mm pitch female headers
3. Plug breakout board into headers

**Pros:** Easier to source, easier to solder
**Cons:** Taller, less compact

### Option 2: Hand-Wire Module

1. Leave castellated footprint empty
2. Use thin wire-wrap wire to connect RFM95W to PCB
3. Secure module with hot glue or epoxy

**Pros:** Works with any RFM95W module
**Cons:** Not as robust, looks less professional

---

## Appendix B: Design Checklist

Before ordering, verify:

- [ ] ERC passes with no errors
- [ ] DRC passes with no errors
- [ ] All components have footprints assigned
- [ ] All footprints match component packages
- [ ] BOM matches schematic (correct values, LCSC parts)
- [ ] Power trace widths adequate (1mm for 12V input)
- [ ] Decoupling caps placed close to IC power pins
- [ ] Ground plane continuous, no splits
- [ ] Mounting holes have clearance (3mm from edge)
- [ ] Screw terminals accessible from board edge
- [ ] SWD header accessible for programming
- [ ] RF trace 50Ω impedance (calculated)
- [ ] Silkscreen doesn't overlap pads
- [ ] Polarity markings on diodes, capacitors
- [ ] Board version number on silkscreen
- [ ] 3D view looks correct

---

## Appendix C: Resources

**KiCad Documentation:**
- https://docs.kicad.org

**JLCPCB Assembly Service:**
- https://jlcpcb.com/smt-assembly
- Parts Library: https://jlcpcb.com/parts

**Component Datasheets:**
- STM32L072: https://www.st.com/resource/en/datasheet/stm32l072c8.pdf
- RFM95W: https://www.hoperf.com/data/upload/portal/20190307/RFM95W-V2.0.pdf
- MCP1700: https://ww1.microchip.com/downloads/en/DeviceDoc/20001826D.pdf

**STM32 Programming:**
- STM32CubeMX: https://www.st.com/en/development-tools/stm32cubemx.html
- OpenOCD: https://openocd.org

---

## Need Help?

If you get stuck or have questions:
1. Check KiCad forums: https://forum.kicad.info
2. Review sensor-design.md for circuit details
3. Search for "KiCad [your topic]" tutorials on YouTube
4. Ask in r/PrintedCircuitBoard or r/AskElectronics on Reddit

Good luck with your PCB design!
