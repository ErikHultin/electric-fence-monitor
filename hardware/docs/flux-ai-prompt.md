# Flux.ai Prompt for Electric Fence Monitor

Copy and paste this prompt into Flux.ai (https://flux.ai) to generate the PCB design.

---

## Primary Prompt

```
Design a 50mm × 50mm electric fence monitoring sensor PCB with the following specifications:

BOARD REQUIREMENTS:
- 2-layer PCB, 50×50mm square
- 4× M3 mounting holes in corners (3.2mm diameter, 3mm from edges)
- Target manufacturer: JLCPCB
- All SMD components: 0805 for passives, hand-solderable

MAIN COMPONENTS:

1. MICROCONTROLLER:
   - STM32L072CBT6 (LQFP-48 package)
   - Place in center of board
   - Decoupling: 4× 100nF capacitors (one per VDD pin) + 1× 4.7µF bulk capacitor
   - 10kΩ pulldown on BOOT0 pin
   - 10kΩ pullup on NRST pin
   - Optional: 32.768kHz crystal on PC14/PC15 for RTC

2. POWER SUPPLY:
   - Input: 12V battery via 2-pin 5mm screw terminal (J1)
   - TVS diode: SMBJ15CA (15V clamp) on battery input for transient protection
   - LDO regulator: MCP1700-3302E/TO (SOT-23) - 12V input to 3.3V output
   - Input capacitor: 1µF 0805 on LDO input
   - Output capacitor: 1µF 0805 on LDO output
   - Place in top-left area of board

3. BATTERY VOLTAGE SENSING:
   - Resistive divider: R1=120kΩ → R2=27kΩ to ground (both 0805, 1% tolerance)
   - Junction connects to MCU ADC (PA0)
   - 100nF filter capacitor to ground at ADC input
   - Test point at ADC input node

4. CAPACITIVE PULSE DETECTION CIRCUIT:
   - Input: 2-pin 5mm screw terminal (J2) for sense plate connection
   - R3: 1MΩ 0805 (input impedance from sense plate)
   - R4: 100kΩ 0805 (series filter resistor)
   - C4: 100pF 0805 (bandwidth limiting, parallel to ground after R3)
   - D1: BAV99 dual diode (SOT-23) for voltage clamping (clamp to 0-3.3V)
   - R5: 10kΩ 0805 (isolation resistor)
   - C5: 100nF 0805 (noise filter to ground)
   - Output connects to MCU comparator input (PA1)
   - Place on left side of board

5. LORA MODULE:
   - RFM95W-868 or RFM95W-915 module (16×16mm castellated pads)
   - 16-pin configuration with 2mm pad spacing
   - SMA edge-mount connector (J5) for antenna
   - SPI connections to MCU:
     * NSS → PA4
     * SCK → PA5
     * MISO → PA6
     * MOSI → PA7
     * DIO0 → PA10 (interrupt)
     * RESET → PA8
   - Place on right side of board
   - SMA connector at right edge

6. DEBUG/PROGRAMMING:
   - 4-pin SWD header (2.54mm pitch): +3V3, SWDIO (PA13), SWCLK (PA14), GND
   - Place at bottom edge, centered
   - Optional: 3-pin UART header (TX, RX, GND)

PIN ASSIGNMENTS:
- PA0: VBAT_SENSE (ADC input from battery divider)
- PA1: PULSE_SIGNAL (comparator input from pulse detection circuit)
- PA4-PA7: SPI (to LoRa module)
- PA8: LORA_RESET
- PA10: LORA_DIO0
- PA13: SWDIO
- PA14: SWCLK

CIRCUIT CONNECTIONS:

Power Supply Circuit:
J1 (12V+) → SMBJ15CA TVS (to GND) → MCP1700 VIN (with 1µF cap to GND) → MCP1700 VOUT (+3V3, with 1µF cap to GND) → distribute to all ICs

Battery Voltage Divider:
12V+ → R1 (120kΩ) → junction → R2 (27kΩ) → GND
Junction → 100nF cap to GND
Junction → PA0 (ADC input)

Pulse Detection:
J2 sense plate input → R3 (1MΩ) → node1
Node1 → C4 (100pF) → GND
Node1 → R4 (100kΩ) → node2
Node2 → BAV99 clamp diodes (between +3V3 and GND)
Node2 → R5 (10kΩ) → node3
Node3 → C5 (100nF) → GND
Node3 → PA1 (comparator input)

LAYOUT REQUIREMENTS:

Component Placement:
- Top-left: Power section (J1 screw terminal, TVS, LDO, capacitors)
- Left side: Sensing circuits (J2 screw terminal, pulse detection components, battery divider)
- Center: STM32L072 with decoupling capacitors clustered around power pins
- Right side: RFM95W module with SMA connector at edge
- Bottom center: SWD debug header

Routing Requirements:
- Power traces (12V, +3V3): 1.0mm width minimum
- Signal traces: 0.25mm width
- RF trace (RFM95W antenna pin to SMA): 0.3mm width, 50Ω impedance, keep short and straight
- Ground plane on bottom layer (solid pour)
- Via stitching around board perimeter
- Keep analog traces (VBAT_SENSE, PULSE_SIGNAL) away from noisy digital signals
- Place all decoupling capacitors within 5mm of IC power pins

SILKSCREEN:
- Label J1: "BAT+ BAT-"
- Label J2: "SENSE GND"
- Label SWD header: "+3V3 DIO CLK GND"
- Board title: "Electric Fence Monitor v1.0"
- Component reference designators

DESIGN RULES:
- Minimum trace width: 0.25mm
- Minimum spacing: 0.25mm
- Via diameter: 0.8mm, drill: 0.4mm
- Target: JLCPCB 2-layer PCB manufacturing

NET CLASSES:
- Power nets (BAT+, +3V3): 1.0mm traces
- RF net (ANT): 0.3mm traces
- Default: 0.25mm traces
```

---

## If Flux.ai Asks for Clarification

### Component Details:

**If it can't find STM32L072CBT6:**
- Try: STM32L072CB or STM32L0 series
- Alternative: STM32L051C8T6 (similar, slightly less flash)

**If it can't find RFM95W:**
- Search for: "RFM95W LoRa module" or "Hope RF RFM95"
- It's a 16×16mm module with castellated edges
- If unavailable, suggest: "Use generic 16-pin castellated module footprint with 2mm pitch"

**If it can't find SMBJ15CA:**
- Alternative: "15V TVS diode, bidirectional, SMB/DO-214AA package"
- Or: SMBJ15A

**If it asks about component values:**
- All resistors: 1% tolerance, 0805 package, 1/8W
- All capacitors: 0805 package, 50V rating (except where noted)
- Electrolytic not needed - all ceramic

### Iterative Refinements:

After it generates the first version, you can refine with:

**"Move the RFM95W module closer to the right edge for shorter antenna trace"**

**"Add more vias connecting the ground planes between top and bottom layers"**

**"Rotate the STM32 to minimize SPI trace lengths to the LoRa module"**

**"Add test points on these nets: +3V3, VBAT_SENSE, PULSE_SIGNAL, GND"**

**"Increase clearance around the mounting holes to 3mm"**

---

## Export Instructions

Once satisfied with the design in Flux.ai:

1. **Export Gerbers:**
   - File → Export → Gerber files
   - Download the ZIP file

2. **Export to KiCad (if supported):**
   - File → Export → KiCad
   - This gives you .kicad_pcb and .kicad_sch files
   - Import into your local KiCad project

3. **Generate BOM:**
   - Flux should generate a BOM automatically
   - Cross-reference with `hardware/bom/BOM.csv` to add JLCPCB part numbers

4. **Generate Pick-and-Place:**
   - Export CPL/POS file for JLCPCB assembly

---

## Comparison with Our BOM

After Flux generates the design, compare components against our BOM (`hardware/bom/BOM.csv`) which includes verified JLCPCB part numbers:

- STM32L072CBT6: LCSC #C94384
- MCP1700-3302E: LCSC #C39051
- RFM95W: Hand solder or external source
- All resistors and capacitors: Specific LCSC parts listed

Replace Flux's generic part numbers with our LCSC-specific ones for guaranteed JLCPCB availability.

---

## Troubleshooting

**If Flux can't generate the complete design:**
Try breaking it into steps:

1. First prompt: "Create 50×50mm board outline with mounting holes"
2. Second: "Add STM32L072 microcontroller with decoupling"
3. Third: "Add MCP1700 power supply circuit"
4. Fourth: "Add battery voltage divider"
5. Fifth: "Add pulse detection circuit"
6. Sixth: "Add RFM95W LoRa module with SMA"

**If component placement is wrong:**
- Use Flux's interactive editor to drag components
- Or give specific XY coordinates: "Place U1 at position X=25mm, Y=25mm"

**If routing looks messy:**
- Ask: "Re-route with priority on: 1) RF trace straight, 2) short power traces, 3) minimize vias"
- Or: "Use more of the bottom layer for signal routing"

---

## Expected Result

Flux.ai should generate:
- Complete schematic with all connections
- PCB layout with components placed
- Partial or complete routing
- Ground planes on both layers
- Silkscreen labels

You can then:
1. Review and tweak in Flux's editor
2. Export to KiCad for final verification
3. Generate manufacturing files
4. Order from JLCPCB

---

## Backup Plan

If Flux.ai struggles or doesn't have the right components, let me know and I'll:
- Generate more complete KiCad files directly
- Provide alternative prompts
- Help you complete the design manually with better guidance

Good luck! This should save you several hours of manual work.
