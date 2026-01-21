# Electric Fence Monitor

A low-power IoT sensor system for monitoring electric fence operation via battery voltage monitoring and non-contact capacitive pulse detection, with LoRa wireless communication.

## Project Overview

This project provides a complete hardware design for a remote electric fence monitoring sensor. The system can:

- Monitor 12V battery voltage (detect low battery conditions)
- Detect electric fence pulses using non-contact capacitive coupling (no direct connection to high voltage)
- Communicate status via LoRa (long-range, low-power wireless)
- Operate for months on a small battery

**Key Features:**
- Non-invasive sensing (no electrical connection to fence wire)
- Ultra-low power consumption (~96µA average)
- Long-range wireless (LoRa: several km line-of-sight)
- Compact 50×50mm PCB design
- Ready for JLCPCB manufacture and assembly

## Repository Structure

```
electric-fence-monitor/
├── README.md                    # This file
├── sensor-design.md             # Detailed circuit design document
├── hardware/                    # PCB design files
│   ├── sensor-board.kicad_pro  # KiCad project
│   ├── sensor-board.kicad_sch  # Schematic (template)
│   ├── sensor-board.kicad_pcb  # PCB layout (to be completed)
│   ├── libraries/              # Custom symbols & footprints
│   │   ├── symbols/
│   │   └── footprints/
│   ├── bom/                    # Bill of materials
│   │   └── BOM.csv             # Component list with JLCPCB parts
│   ├── gerbers/                # Manufacturing files (generated)
│   └── docs/
│       └── kicad-design-guide.md  # Step-by-step PCB design guide
├── firmware/                   # (To be added: STM32 firmware)
└── gateway/                    # (To be added: LoRa gateway code)
```

## Getting Started

### 1. Review the Design

Start with [`sensor-design.md`](sensor-design.md) to understand the circuit design:
- Battery voltage monitoring circuit
- Capacitive pulse detection principle
- Component selection and calculations
- Power budget analysis

### 2. Complete the PCB Design

Follow the comprehensive guide in [`hardware/docs/kicad-design-guide.md`](hardware/docs/kicad-design-guide.md):

**You'll need:**
- KiCad 7.0 or later
- 2-4 hours for schematic entry
- 4-6 hours for PCB layout (first time)

**The guide covers:**
- Opening and setting up the KiCad project
- Creating hierarchical schematic sheets
- Component placement and routing strategies
- Generating Gerber files for manufacturing
- Preparing assembly files for JLCPCB

### 3. Order PCBs

**Option A: Fully Assembled (Recommended for beginners)**
- Upload gerbers, BOM, and CPL to JLCPCB
- Select PCB Assembly service
- Cost: ~$70-120 for 5 boards
- You hand-solder: RFM95W module, connectors
- Turnaround: ~2 weeks delivered

**Option B: PCB Only (Cheaper, more work)**
- Order bare PCBs from JLCPCB (~$5 for 5)
- Source components from DigiKey/Mouser
- Hand-solder all components yourself
- Cost: ~$60-90 total
- Turnaround: ~1 week for PCBs

See [`hardware/docs/kicad-design-guide.md`](hardware/docs/kicad-design-guide.md) Part 4 for detailed ordering instructions.

### 4. Assemble and Test

After receiving boards:
1. Solder remaining components (RFM95W, connectors)
2. Visual inspection
3. Power-on test (verify 3.3V output)
4. Program via SWD interface

### 5. Develop Firmware

(Coming soon: STM32 firmware using STM32Cube)

**Firmware will handle:**
- Battery voltage ADC sampling
- Pulse detection via comparator interrupt
- LoRa transmission at regular intervals
- Deep sleep power management

### 6. Set Up Gateway

(Coming soon: Raspberry Pi + LoRa gateway)

**Gateway will:**
- Receive sensor data via LoRa
- Log to database or cloud service
- Provide web interface for monitoring
- Send alerts for low battery or fence failure

## Bill of Materials

Complete BOM with JLCPCB part numbers: [`hardware/bom/BOM.csv`](hardware/bom/BOM.csv)

**Key components:**
- **MCU:** STM32L072CBT6 (ultra-low-power ARM Cortex-M0+)
- **LoRa:** RFM95W-868/915 module
- **LDO:** MCP1700-3302E (ultra-low quiescent current)
- **Passives:** 0805 size resistors and capacitors
- **Connectors:** 5mm screw terminals, SMA for antenna

**Estimated cost per board:**
- Components: $15-20
- PCB (qty 5): $2-5 total = $0.40-1 each
- Assembly (JLCPCB): ~$15-25 per board when ordering 5

**Total per sensor:** ~$30-45 (including assembled PCB)

## Technical Specifications

### Electrical

- **Input voltage:** 10.5-15V DC (12V lead-acid battery)
- **Operating current:** ~96µA average (with periodic LoRa transmissions)
- **Battery life:** ~3000 days theoretical (self-discharge limiting factor)
- **ADC resolution:** 12-bit (battery voltage sensing)
- **Pulse detection:** Capacitive coupling, ~8kV pulse detection
- **Communication:** LoRa 868/915 MHz (regional variant)

### Physical

- **PCB dimensions:** 50 × 50 mm
- **Mounting:** 4× M3 holes in corners
- **Enclosure:** Standard IP65 junction box
- **Sense plate:** 50 × 50 mm copper plate (can be PCB itself or separate)

### Environmental

- **Operating temperature:** -20°C to +60°C (component-limited)
- **Humidity:** Design for outdoor installation (IP65 enclosure required)

## Design Decisions

### Why Capacitive Sensing?

Non-contact sensing means:
- No electrical connection to high-voltage fence
- No risk of sensor damage from fence pulses
- Safer installation
- No impact on fence operation

Trade-off: More sensitive to noise, requires careful circuit design

### Why STM32L072?

- Ultra-low-power sleep modes (<1µA)
- Built-in comparator (no external comparator needed)
- Built-in DAC (for adjustable threshold)
- Plenty of peripherals for future expansion
- Well-supported toolchain

### Why LoRa?

- Long range (several km line-of-sight)
- Very low power
- No cellular subscription required
- Works in remote areas without infrastructure

## Development Status

- [x] Circuit design complete
- [x] KiCad project structure created
- [x] Custom symbol library (RFM95W)
- [x] Custom footprint library (RFM95W)
- [x] BOM with JLCPCB part numbers
- [x] Comprehensive design guide
- [ ] Complete schematic entry (in progress - follow guide)
- [ ] Complete PCB layout (in progress - follow guide)
- [ ] Generate manufacturing files
- [ ] Order prototype PCBs
- [ ] Develop firmware
- [ ] Build gateway
- [ ] Field testing

## Future Enhancements

Potential additions for v2.0:
- Solar panel input for perpetual operation
- Current sensing (detect fence load, shorts)
- GPS module (for mobile fence monitoring)
- Temperature/humidity sensors
- Multiple sense inputs (monitor several fence sections)
- OLED display for local status
- BLE configuration interface
- LoRaWAN support (vs. proprietary protocol)

## Contributing

This is an open hardware project. Contributions welcome:
- PCB layout improvements
- Firmware development
- Gateway software
- Documentation improvements
- Field testing and feedback

## License

**Hardware:** CERN-OHL-S-2.0 (Strongly Reciprocal Open Hardware License)
**Software:** (To be determined when firmware is added - likely MIT or GPL)
**Documentation:** CC-BY-SA-4.0

## Support

For questions or issues:
1. Check the design guide: `hardware/docs/kicad-design-guide.md`
2. Review circuit details: `sensor-design.md`
3. Open an issue on GitHub

## Acknowledgments

- Circuit design inspired by capacitive sensing techniques
- KiCad for excellent open-source PCB design tools
- STMicroelectronics for ultra-low-power MCUs
- Hope RF for affordable LoRa modules

## Safety Notice

**High Voltage Warning:** Electric fences operate at high voltage (typically 8-10kV pulses). Although this design uses non-contact sensing, always follow proper safety procedures when working near electric fences:

- Turn off fence before installing sensor hardware
- Maintain proper clearance from fence wire
- Use appropriate enclosures and insulation
- Follow local electrical codes and regulations
- Test installation thoroughly before field deployment

**Battery Safety:** 12V lead-acid batteries can deliver high current. Use appropriate fuses and wire gauge.

---

**Project Status:** Hardware design complete, ready for PCB fabrication

Last updated: 2026-01-21
