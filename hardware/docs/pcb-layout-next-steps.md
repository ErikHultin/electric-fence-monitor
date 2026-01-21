# PCB Layout - Next Steps

The starter PCB file (`sensor-board.kicad_pcb`) has been created with the foundation for your design. Here's what's included and what you need to do next.

## What's Already Done

✅ **Board outline:** 50×50mm rectangle
✅ **Mounting holes:** 4× M3 holes (3.2mm) at corners (3mm clearance from edges)
✅ **Ground planes:** Defined on both top (F.Cu) and bottom (B.Cu) layers
✅ **Net definitions:** All 15 key nets defined (power, signals, SPI, etc.)
✅ **Silkscreen labels:** Connector labels for BAT+/-, SENSE/GND, and SWD pinout
✅ **Design rules:** Set in kicad_pro file (0.25mm clearance, JLCPCB compatible)

## Next Steps in KiCad

### 1. Open the PCB File

```bash
cd hardware
# Open sensor-board.kicad_pcb in KiCad PCB Editor
```

### 2. Import Components from Schematic

**Before you can do this, you must complete the schematic first!**

Once your schematic is complete:
1. In schematic editor: Tools → Update PCB from Schematic (F8)
2. Click "Update PCB"
3. All components will appear in the center of the PCB editor
4. Click outside the component cluster to deselect

### 3. Place Components

Refer to the layout plan from the design guide. Here's the recommended placement:

```
┌─────────────────────────────────────────────┐
│ H1          BAT+ BAT- SENSE GND         H2  │
│  ○           [J1]      [J2]              ○  │
│                                              │
│         [TVS1]  [U1]                         │
│         [C1,C2]                              │
│                              [U3 RFM95W]     │
│    [Pulse Detection]                         │
│    R3,R4,C4,D1,R5,C5                  [J5]   │
│                                        SMA   │
│    [Battery Divider]      [U2 STM32]        │
│    R1,R2,C3               + C6-C10           │
│                                              │
│  ○          [J3 SWD]                     ○  │
│ H4                                        H3 │
└─────────────────────────────────────────────┘
```

**Placement tips:**
- Keep decoupling caps (C7-C10) very close to U2 power pins
- U3 (RFM95W) should be near the board edge for antenna connector J5
- Keep analog circuitry (pulse detection, battery divider) away from digital/RF
- Orient components for logical signal flow

### 4. Route Traces

#### Power Traces (1.0mm width):
1. Select top toolbar → Track width → 1.0mm
2. Route BAT+ from J1 → TVS1 → U1 input
3. Route +3V3 from U1 output → all ICs (via decoupling caps)

#### Signal Traces (0.25mm width):
1. Select Track width → 0.25mm
2. Route SPI bus: MCU → RFM95W (MOSI, MISO, SCK, NSS)
3. Route control signals: LORA_DIO0, LORA_RESET
4. Route analog signals: VBAT_SENSE, PULSE_SIGNAL

#### RF Trace (0.3mm width, critical!):
1. **This is the most critical trace on the board**
2. Select Track width → 0.3mm (for 50Ω impedance on 1.6mm FR4)
3. Route ANT from U3 pin 9 to J5 center pin
4. **Keep as short and straight as possible**
5. **No vias if possible**
6. Use coplanar waveguide: ground pour on both sides, NOT underneath

#### Ground Connections:
- Use vias to connect component ground pads to bottom ground plane
- Add stitching vias around board perimeter (every 5-10mm)
- Don't split the ground plane

### 5. Fill Ground Zones

1. Select each zone (click on the dotted outline)
2. Press 'B' to fill the zone
3. Verify zones fill correctly without isolated islands

### 6. Add Silkscreen Details

Already added:
- Board title at top
- Connector labels

You may want to add:
- Component reference designators (R1, C1, U1, etc.)
- Polarity marks on polarized components
- Pin 1 indicators on ICs

### 7. Run Design Rule Check (DRC)

1. Inspect → Design Rules Checker
2. Click "Run DRC"
3. Review errors and warnings
4. Fix all errors (warnings are often acceptable)

**Common issues to fix:**
- Clearance violations (traces too close)
- Unconnected nets (forgot to route something)
- Silkscreen over pads (move text)

### 8. View in 3D

1. View → 3D Viewer
2. Check:
   - Component clearances
   - Mounting holes accessible
   - Connectors at board edges
   - No obvious placement issues

## Component Placement Reference

Based on your BOM, here's what you're placing:

**Power Section (Top-Left):**
- J1: Screw terminal (battery input)
- TVS1: SMBJ15CA (DO-214AA/SMB package)
- U1: MCP1700 (SOT-23)
- C1, C2: 1µF 0805 capacitors

**Sensing Section (Left):**
- J2: Screw terminal (sense plate input)
- R3: 1MΩ 0805
- R4: 100kΩ 0805
- C4: 100pF 0805
- D1: BAV99 (SOT-23)
- R5: 10kΩ 0805
- C5: 100nF 0805
- R1: 120kΩ 0805 (battery divider)
- R2: 27kΩ 0805
- C3: 100nF 0805

**MCU Section (Center):**
- U2: STM32L072CBT6 (LQFP-48)
- C6: 4.7µF 0805 (bulk cap)
- C7-C10: 100nF 0805 (decoupling, one per VDD pin)
- R6, R7: 10kΩ 0805 (pullup/pulldown)
- Y1: 32.768kHz crystal (optional)

**LoRa Section (Right):**
- U3: RFM95W module (16×16mm castellated)
- J5: SMA connector (edge mount)

**Debug Section (Bottom):**
- J3: 4-pin header 2.54mm (SWD)
- J4: 3-pin header 2.54mm (UART, optional)

## Routing Priority

Route in this order:

1. **Critical analog traces:**
   - VBAT_SENSE (R1/R2 divider → MCU ADC)
   - PULSE_SIGNAL (pulse circuit → MCU comparator)

2. **RF trace:**
   - ANT (RFM95W → SMA)

3. **Power traces:**
   - BAT+, +3V3, GND connections

4. **SPI bus:**
   - MOSI, MISO, SCK, NSS (MCU → RFM95W)

5. **Control signals:**
   - LORA_DIO0, LORA_RESET

6. **Debug signals:**
   - SWDIO, SWCLK

## Verification Checklist

Before generating Gerbers:

- [ ] All nets routed (0 unconnected in DRC)
- [ ] DRC passes with 0 errors
- [ ] Ground planes filled completely
- [ ] Stitching vias placed around perimeter
- [ ] Decoupling caps within 5mm of IC power pins
- [ ] RF trace is 50Ω impedance (0.3mm width)
- [ ] Mounting holes have 3mm clearance from board edge
- [ ] Silkscreen doesn't overlap pads
- [ ] Connector labels clearly visible
- [ ] 3D view looks correct
- [ ] Power trace widths are 1.0mm
- [ ] Board version "v1.0" on silkscreen

## Time Estimate

- Component placement: 1-2 hours
- Trace routing: 3-5 hours (first time)
- DRC fixes and cleanup: 1 hour
- **Total: 5-8 hours for a complete, routed board**

## Need Help?

- **Stuck on placement?** Check the component placement diagram in the design guide
- **Routing issues?** Try auto-router for initial routing, then clean up manually
- **DRC errors?** Most common fixes:
  - Increase clearance by moving traces apart
  - Use vias to route on different layer
  - Adjust component positions slightly

## After Routing is Complete

Follow "Part 3: Generate Manufacturing Files" in the main design guide:
- Export Gerbers
- Generate BOM and CPL files
- Zip for JLCPCB upload

Good luck! The hardest part is getting started - once you place a few components and see how it comes together, it becomes much more intuitive.
