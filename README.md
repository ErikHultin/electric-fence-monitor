# Electric Fence Monitor

A low-power IoT sensor system for monitoring electric fence operation via battery voltage monitoring and non-contact capacitive pulse detection, with LoRa wireless communication.

## Project Overview

This project provides a circuit design for a remote electric fence monitoring sensor. The system can:

- Monitor 12V battery voltage (detect low battery conditions)
- Detect electric fence pulses using non-contact capacitive coupling (no direct connection to high voltage)
- Communicate status via LoRa (long-range, low-power wireless)
- Operate for months on a small battery

**Key Features:**
- Non-invasive sensing (no electrical connection to fence wire)
- Ultra-low power consumption (~96µA average)
- Long-range wireless (LoRa: several km line-of-sight)

## Repository Structure

```
electric-fence-monitor/
├── README.md                    # This file
├── sensor-design.md             # Detailed circuit design document
├── firmware/                    # (To be added: STM32 firmware)
└── gateway/                     # (To be added: LoRa gateway code)
```

## Getting Started

### 1. Review the Design

Start with [`sensor-design.md`](sensor-design.md) to understand the circuit design:
- Battery voltage monitoring circuit
- Capacitive pulse detection principle
- Component selection and calculations
- Power budget analysis

### 2. Develop Firmware

(Coming soon: STM32 firmware using STM32Cube)

**Firmware options:**
- **Option A - Proprietary LoRa:** Simple point-to-point communication (~20KB flash)
- **Option B - LoRaWAN/TTN:** Full network stack using I-CUBE-LRWAN (~100KB flash)

**Firmware will handle:**
- Battery voltage ADC sampling
- Pulse detection via comparator interrupt
- LoRa/LoRaWAN transmission at regular intervals
- Deep sleep power management

### 3. Set Up Gateway

**Option A - Custom Gateway:**
- Raspberry Pi + RFM95W LoRa module
- Python software for data collection
- Local database and web interface

**Option B - The Things Network (TTN):**
- Use existing TTN infrastructure (if available in your area)
- Free cloud platform for LoRaWAN
- Built-in integrations and APIs
- Requires LoRaWAN firmware (Option B above)

**Gateway will:**
- Receive sensor data via LoRa/LoRaWAN
- Log to database or cloud service
- Provide web interface for monitoring
- Send alerts for low battery or fence failure

## Technical Specifications

### Electrical

- **Input voltage:** 10.5-15V DC (12V lead-acid battery)
- **Operating current:** ~96µA average (with periodic LoRa transmissions)
- **Battery life:** ~3000 days theoretical (self-discharge limiting factor)
- **ADC resolution:** 12-bit (battery voltage sensing)
- **Pulse detection:** Capacitive coupling, ~8kV pulse detection
- **Communication:** LoRa 868/915 MHz (regional variant)

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

### Why STM32L072CZ?

- Ultra-low-power sleep modes (<1µA)
- Built-in comparator (no external comparator needed)
- Built-in DAC (for adjustable threshold)
- **192KB flash** (CZ variant) - sufficient for LoRaWAN/TTN compatibility
  - Simple proprietary LoRa: ~20KB flash needed
  - LoRaWAN stack (I-CUBE-LRWAN): ~100KB flash needed
  - The 128KB CB variant would be too tight for LoRaWAN
- Plenty of peripherals for future expansion
- Well-supported toolchain
- Same package/pinout as CB variant (LQFP-48)

### Why LoRa?

- Long range (several km line-of-sight)
- Very low power consumption
- No cellular subscription required
- Works in remote areas without infrastructure
- **Two deployment options:**
  - Simple proprietary protocol (custom gateway)
  - LoRaWAN (TTN or private network)

## Development Status

- [x] Circuit design complete
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

This is an open project. Contributions welcome:
- Firmware development
- Gateway software
- Documentation improvements
- Field testing and feedback

## License

**Software:** (To be determined when firmware is added - likely MIT or GPL)
**Documentation:** CC-BY-SA-4.0

## Support

For questions or issues:
1. Review circuit details: `sensor-design.md`
2. Open an issue on GitHub

## Acknowledgments

- Circuit design inspired by capacitive sensing techniques
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

**Project Status:** Circuit design complete, firmware development pending

Last updated: 2026-01-21
