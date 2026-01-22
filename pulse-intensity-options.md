# Pulse Intensity Measurement Options

This document explores sensor options for measuring electric fence pulse intensity (amplitude), rather than just presence/absence detection. Key considerations include measurement accuracy and robustness to lightning-induced surges.

## Current Design Limitation

The existing capacitive coupling design uses a threshold-based comparator that detects pulse presence but does not measure intensity. To track fence health over time (detecting degradation or improvement), we need to quantify the pulse amplitude.

## Operational Mode: Scheduled Wake Cycle

The sensor operates on a **scheduled wake cycle**, not continuous monitoring:

1. **Sleep** (~30 minutes): MCU in deep sleep, minimal power draw
2. **Wake**: Timer triggers wake-up
3. **Measure**: Capture one or more fence pulses and measure intensity
4. **Transmit**: Send data via LoRa
5. **Sleep**: Return to deep sleep

### Implications for Pulse Measurement

**Pulse timing:** Electric fence energizers typically pulse at ~1Hz (once per second). When the sensor wakes, it must wait for the next pulse - up to 1 second in the worst case.

**Measurement window:** The sensor should stay awake for 1-2 seconds to guarantee capturing at least one pulse. At ~3mA active current, a 2-second wake window adds ~1.7µAh per 30-minute cycle - negligible.

**Peak detector behavior:** The peak detector capacitor must hold its value for the duration between pulse arrival and ADC sampling (milliseconds), not for 30 minutes. The capacitor is reset at the start of each wake cycle, captures the pulse peak, and is read before returning to sleep.

**Comparator role:** Even with scheduled wake, the comparator serves as a "pulse arrival" detector to trigger ADC sampling at the right moment. It can also detect if no pulse arrives within the expected window (energizer failure).

---

## Overview: Understanding the Approaches

The five options fall into two fundamental categories based on what physical quantity they measure:

### Voltage-Based Sensing (Options 1-2)

These approaches detect the **electric field** created by the high-voltage pulse on the fence wire. The capacitive plate acts as one plate of a capacitor, with the fence wire as the other plate and air as the dielectric.

**What you're measuring:** The rate of voltage change (dV/dt) on the fence wire, which creates a displacement current through the air gap to your sense plate.

**Key characteristic:** Truly non-contact - the sensor never touches the fence wire. However, the signal amplitude depends heavily on the distance between the plate and wire, making calibration and consistent mounting critical.

**Best for:** Situations where you cannot or don't want to modify the fence installation, or where multiple sense points are needed along the fence line.

### Current-Based Sensing (Options 3-5)

These approaches detect the **magnetic field** created by current flowing through the fence wire. When the energizer fires, current flows through the wire (and returns through ground), creating a magnetic field that can be sensed.

**What you're measuring:** The actual current magnitude flowing through the fence at that point. This directly correlates to the energy being delivered.

**Key characteristic:** Requires proximity to (or installation around) the fence wire, but provides a more accurate measure of fence "strength" since current × voltage = power delivered.

**Best for:** Situations where you want accurate energy measurement, need galvanic isolation for lightning protection, or plan to add load/fault detection features.

### Quick Selection Guide

| If you want... | Consider... |
|----------------|-------------|
| Minimal hardware changes | Option 2 (Direct ADC) |
| Non-contact installation | Options 1-2 (Voltage-based) |
| Best measurement accuracy | Option 3 (CT) or Option 4 (Rogowski) |
| Best lightning protection | Option 3 (CT) or Option 4 (Rogowski) |
| Lowest cost | Option 2 (Direct ADC) |
| Future expandability (load sensing) | Option 3 (CT) |

---

## Option 1: Peak Detector with ADC Sampling

**Category:** Voltage-based sensing (enhancement to existing capacitive design)

### Concept

Add a peak-hold circuit that captures the maximum voltage from the coupled signal, then sample with ADC at leisure.

The fundamental challenge with measuring fast pulses is timing - the fence pulse is only ~100µs wide, but a typical low-power MCU ADC takes 10-20µs per conversion. A peak detector solves this by using analog circuitry to "remember" the highest voltage seen, allowing the ADC to read it at any time afterward.

### How It Works

1. **Wake & reset:** MCU wakes from scheduled timer, resets peak detector (discharge capacitor via MOSFET)
2. **Arm & wait:** Release reset, enable comparator, wait for pulse (up to ~1.5 seconds)
3. **Pulse arrives:** The capacitively-coupled signal rises rapidly
4. **Diode conducts:** The Schottky diode (BAT54) allows current to flow into the hold capacitor only when the input exceeds the capacitor voltage
5. **Peak captured:** The capacitor charges to the peak voltage and holds it (diode blocks discharge)
6. **Comparator triggers:** Signals that a pulse occurred (can also trigger timestamp capture)
7. **ADC reads:** The MCU reads the held voltage through a high-impedance buffer
8. **Sleep:** Transmit data, return to deep sleep for ~30 minutes

### Scheduled Wake Considerations

- **Hold time required:** Only milliseconds (pulse to ADC read), not minutes - capacitor leakage is not a concern
- **Multiple pulse option:** Could capture several pulses and average, or take the maximum
- **Timeout handling:** If no pulse detected within 2 seconds, report energizer failure
- **Power impact:** Op-amp can remain powered during the brief wake window (~2s × 100µA = negligible)

### When to Choose This Option

- You want to keep the existing non-contact capacitive sensing approach
- Installation simplicity is important (no fence wire modification)
- You need reliable peak capture regardless of MCU timing
- Moderate accuracy is acceptable (±10-20% typical)

```
Sense Plate → Input Stage → Peak Detector → ADC
                              ↓
                           Reset (GPIO)
```

### Circuit Components

- Fast Schottky diode (BAT54) charges a hold capacitor
- High-impedance buffer (MCP6001 or similar rail-to-rail op-amp)
- GPIO-controlled discharge through MOSFET for reset after reading
- Hold capacitor: 10-100nF (tradeoff between droop and response)

### Pros

- Captures true peak even with fast pulses (~100µs)
- ADC can sample at leisure (no timing criticality)
- Relatively simple addition to existing design
- Works with existing capacitive coupling approach

### Cons

- Adds ~4 components (op-amp, capacitor, diode, MOSFET)
- Hold capacitor leakage affects accuracy over long sample windows
- Op-amp adds ~10-100µA quiescent current (can be power-switched)
- Requires calibration to relate peak voltage to actual fence energy

### Lightning Robustness

**Moderate** - Diode clamp protects op-amp input, but very fast transients can punch through Schottky diode before clamp reacts. Recommend adding series resistance before peak detector.

### Estimated Additional Cost

~$1.50 (op-amp $0.50, passives $0.30, MOSFET $0.20, diode $0.15, misc $0.35)

---

## Option 2: Direct ADC Sampling with Interrupt-Triggered Capture

**Category:** Voltage-based sensing (firmware-only modification)

### Concept

Use the existing comparator to trigger an interrupt, then immediately start fast ADC capture to catch the pulse peak.

This is the simplest approach from a hardware perspective - it uses the existing circuit exactly as-is and relies on fast firmware response to capture the pulse waveform. The comparator acts as a "wake-up" trigger, and the ADC captures multiple samples during the pulse to find the peak in software.

### How It Works

1. **MCU sleeps:** Comparator is active in ultra-low-power mode, ADC is off
2. **Pulse arrives:** Comparator threshold exceeded, triggers interrupt
3. **Fast wake:** MCU wakes from stop mode (~5-10µs wake time on STM32L0)
4. **Burst capture:** ADC runs in continuous mode with DMA, capturing samples as fast as possible
5. **Software processing:** After pulse ends, firmware scans the buffer to find the maximum value

### The Timing Challenge

The STM32L072's ADC can achieve ~1Msps in fast mode, giving ~1µs per sample. However:
- Wake-from-sleep latency: ~5-10µs
- Comparator-to-interrupt latency: ~1-2µs
- First ADC conversion startup: ~2µs

**Total delay before first sample: ~10-15µs**

With a ~100µs pulse, you'd capture the latter ~85µs - likely including the peak, but not guaranteed if the peak is early in the waveform.

### Scheduled Wake Considerations

With the 30-minute wake cycle, this approach works as follows:

1. **Wake from timer:** MCU wakes, enables comparator
2. **Wait for pulse:** MCU stays in light sleep (stop mode with comparator active) waiting for pulse
3. **Comparator fires:** Full wake, immediate ADC burst capture
4. **Process & sleep:** Find peak in buffer, transmit, return to deep sleep

**Advantage:** No additional hardware - the comparator is already in the design for pulse detection.

**Challenge:** The MCU must respond very quickly to the comparator interrupt to catch the pulse. This means either:
- Stay fully awake while waiting (higher power, but only for ~1-2 seconds)
- Use stop mode with comparator as wake source (lower power, but adds wake latency)

**Power comparison:**
- Fully awake for 2s every 30min: 2s × 3mA = 6mAs per cycle → ~3.3µA average
- Stop mode + fast wake: 2s × 1µA + 100µs × 3mA = ~0.07µA average

The stop-mode approach is viable but adds ~5-10µs latency, which may cost you the pulse peak.

### When to Choose This Option

- You want zero additional hardware cost
- The existing capacitive circuit is already built/working
- You're comfortable with firmware complexity
- "Good enough" accuracy is acceptable for your use case
- You want to prototype quickly before committing to hardware changes

### Implementation

1. Comparator detects pulse rising edge
2. ISR triggers ADC in fast continuous mode
3. DMA captures multiple samples over ~100µs pulse window
4. Firmware finds peak value in captured buffer

### Pros

- Minimal hardware changes
- Uses existing STM32L072 peripherals (comparator + DMA + ADC)
- No additional power consumption
- No additional cost

### Cons

- STM32L072 ADC conversion time ~12µs - may miss sharp peaks
- Current 16kHz RC filter (R2/C1) smooths the signal, reducing peak amplitude
- CPU must wake from sleep and respond quickly (~10µs latency)
- Interrupt jitter affects measurement consistency
- May need to increase filter bandwidth (reduces noise immunity)

### Lightning Robustness

**Good** - Existing BAV99 clamp diodes handle protection. No additional sensitive components exposed.

### Estimated Additional Cost

$0 (firmware changes only)

### Required Firmware Changes

```c
// Pseudo-code for interrupt-triggered ADC capture
void COMP_IRQHandler(void) {
    // Start DMA-driven ADC burst capture
    HAL_ADC_Start_DMA(&hadc, adc_buffer, SAMPLE_COUNT);
    // Process in main loop after DMA complete
}
```

---

## Option 3: Current Transformer (CT)

**Category:** Current-based sensing (replaces capacitive sensing entirely)

### Concept

Install a small toroidal current transformer around the fence wire to measure pulse current directly. The CT provides galvanic isolation and outputs a voltage proportional to the instantaneous current.

A current transformer works on the same principle as a power transformer, but instead of transforming voltage, it transforms current. The fence wire passes through the center of a toroidal (donut-shaped) core and acts as a single-turn primary winding. The secondary winding (many turns of fine wire) outputs a proportionally reduced current that's safe to measure.

### How It Works

1. **Primary current:** Fence pulse creates current flow through the wire (typically 1-10A peak for a few hundred microseconds)
2. **Magnetic coupling:** Current creates a magnetic field in the toroidal core
3. **Secondary current:** The changing magnetic field induces a current in the secondary winding, reduced by the turns ratio (e.g., 1:100)
4. **Burden resistor:** Secondary current flows through a resistor, creating a measurable voltage (e.g., 10A ÷ 100 × 10Ω = 1V)
5. **Peak detection:** Same peak detector circuit as Option 1 captures the maximum

### Why Current Measurement is Valuable

Electric fence effectiveness depends on the **energy** delivered to an animal that touches it:

```
Energy = Voltage × Current × Time
```

- **Voltage alone** (capacitive sensing) tells you the energizer is producing pulses, but not how much energy is available
- **Current measurement** tells you how much energy is actually flowing through the fence system
- A fence with high voltage but low current (due to poor grounding or wire breaks) won't be effective

### Installation Consideration

The fence wire must pass through the CT's center hole. Options:
- **During initial installation:** Route wire through CT, then connect to fence
- **Existing installation:** Temporarily disconnect wire, thread through CT, reconnect
- **Split-core CT:** More expensive, but clamps around wire without disconnection

### Scheduled Wake Considerations

The CT approach integrates well with the scheduled wake cycle:

1. **Wake from timer:** MCU wakes, resets peak detector
2. **Passive sensing:** CT is always "listening" - no power required
3. **Wait for pulse:** Peak detector captures pulse peak automatically
4. **Read & sleep:** ADC samples held peak, transmit, return to deep sleep

**Key advantage:** The CT + peak detector combination works entirely passively during the wait period. No active circuitry is required until the ADC read - the CT generates a signal from the pulse current, and the peak detector captures it without MCU involvement.

**Power profile:** Only the peak detector op-amp draws current during the wake window. If power-switched, it can be enabled just before the wait period and disabled after reading.

### When to Choose This Option

- You want the most accurate measure of fence energy delivery
- Lightning protection is a high priority (CT provides galvanic isolation)
- You're willing to install the CT around the fence wire
- You want to enable future features like load monitoring or fault detection
- You prefer a well-established, reliable sensing technology

### Signal Chain

```
Fence Wire (primary) → CT (1:100) → Burden Resistor → Protection → Peak Detector → ADC
```

### Suitable Components

| Part | Ratio | Max Current | Package | Price |
|------|-------|-------------|---------|-------|
| Talema AS-103 | 1:100 | 10A | Toroid 13mm | ~$3.00 |
| Talema AC-1005 | 1:500 | 5A | Toroid 10mm | ~$2.50 |
| Custom wound | Variable | Variable | Ferrite toroid | ~$1.00 |

### Pros

- Measures actual current, not voltage derivative
- Galvanic isolation inherent in transformer design
- Can detect fence load/shorts (planned v2.0 feature)
- Linear response to current magnitude
- Well-understood technology

### Cons

- Requires physical installation around fence wire (not truly non-contact)
- Core saturation possible with very high currents (lightning)
- CT output is proportional to di/dt, may need integrator for true current
- Calibration required for absolute measurements
- Adds installation complexity

### Lightning Robustness

**Excellent** - CT provides galvanic isolation between fence wire and electronics. Secondary winding can have TVS protection. Core may saturate during extreme events but will recover.

### Estimated Additional Cost

~$4.00-6.00 (CT $3.00, burden resistor $0.10, protection $0.50, peak detector $1.50)

### CT vs Capacitive Plate: Do You Need Both?

**No.** If you choose the CT approach, the capacitive plate is not needed. The CT can fully replace the capacitive sensing for pulse detection while adding intensity measurement.

#### What Each Sensor Measures

| Sensor | Measures | What It Tells You |
|--------|----------|-------------------|
| Capacitive plate | Voltage (dV/dt) | "A pulse occurred" |
| Current transformer | Current (I or di/dt) | "A pulse occurred AND how much current flowed" |

The CT provides strictly more information - both pulse presence AND intensity in a single sensor.

#### Subtle Difference in Detection

- **CT**: Measures current flowing *through* that point in the fence. Low current could mean a break downstream, high vegetation load, or weak energizer.
- **Capacitive plate**: Measures voltage *at* that point. Shows the energizer is producing voltage regardless of downstream load.

#### Practical Implication

If you install the CT near the energizer output, you get a good picture of what the energizer is delivering. If installed far down the fence line, low readings could indicate either a weak pulse OR a fault between the energizer and sensor location.

#### Components Removed (if using CT only)

When switching from capacitive to CT sensing, you can remove:
- Sense plate (50×50mm copper sheet)
- 1MΩ input resistor (R1)
- 100kΩ + 100pF filter (R2/C1)
- BAV99 clamp diodes
- Comparator threshold circuit (if using peak detector with ADC instead)

#### Components Added

- CT (toroid installed around fence wire)
- Burden resistor (sets output voltage scale)
- TVS on CT secondary (lightning protection)
- Peak detector circuit (op-amp, hold cap, reset MOSFET)

**Net result:** Similar component count, but CT approach gives better measurement accuracy and inherent galvanic isolation.

---

## Option 4: Rogowski Coil

**Category:** Current-based sensing (advanced alternative to CT)

### Concept

An air-core coil placed around the fence wire. Unlike iron-core CTs, Rogowski coils cannot saturate and have excellent transient response.

A Rogowski coil is essentially a CT without an iron core - just a coil of wire wound on a flexible or rigid non-magnetic former. This seemingly simple change has profound implications for performance, especially with transient signals like fence pulses.

### How It Works

1. **Coil placement:** The flexible coil wraps around the fence wire (can be clip-on)
2. **Magnetic sensing:** Changing current creates a changing magnetic field through the coil loops
3. **Voltage output:** The coil outputs a voltage proportional to di/dt (rate of current change)
4. **Integration:** An analog integrator circuit converts di/dt back to a signal proportional to current
5. **Peak detection:** Same as other options

### Why No Core Matters

**Iron-core CT limitations:**
- Core can **saturate** at high currents, causing output to clip and lose accuracy
- Core has **frequency-dependent** permeability, affecting response
- Core **retains magnetization**, potentially causing offset errors

**Air-core Rogowski advantages:**
- **Cannot saturate** - works from milliamps to megaamps
- **Linear at all frequencies** - ideal for fast transients
- **No remanence** - always returns to true zero
- **Lightweight and flexible** - easier installation

### The Integrator Requirement

The Rogowski coil outputs voltage proportional to di/dt, not current. For a fence pulse:

```
Pulse current: 0 → 10A → 0 over 100µs
Rogowski output: Positive spike → Negative spike (derivative waveform)
```

An analog integrator (op-amp + capacitor) converts this back to a signal proportional to instantaneous current. This adds:
- One op-amp (e.g., MCP6001)
- Integration capacitor and reset circuit
- Gain-setting resistors

### Scheduled Wake Considerations

The Rogowski coil has a complication with scheduled wake: the integrator.

**Problem:** The integrator op-amp must be running to convert di/dt to current. If it's powered off during sleep, it can't capture pulses.

**Solutions:**
1. **Power the integrator during wake window:** ~2s × 100µA = negligible impact, but adds complexity
2. **Use a resettable integrator:** Reset at wake, let it integrate the first pulse, read result
3. **Skip integration, measure di/dt directly:** Capture the derivative waveform with ADC, integrate in firmware

Option 3 (firmware integration) is interesting - you capture the raw Rogowski output with fast ADC sampling, then numerically integrate. This avoids the analog integrator entirely but requires the same fast-response approach as Option 2.

**Recommendation:** If considering Rogowski, the added complexity of the integrator may not be worth it for this application. A CT is simpler and provides similar benefits.

### When to Choose This Option

- You need measurement accuracy at very high currents (lightning-induced surges)
- The fence energizer produces unusually high-current pulses
- You want the most "future-proof" sensing for any fence system
- Cost and complexity are secondary to performance
- You're comfortable with more complex analog signal conditioning

### Signal Chain

```
Fence Wire → Rogowski Coil → Integrator (op-amp) → Peak Detector → ADC
```

### Pros

- No magnetic saturation (air core) - handles any current level
- Excellent high-frequency/transient response
- Very linear across wide dynamic range (mA to kA)
- Can be flexible/clip-on design for easy installation
- No core losses

### Cons

- Output is di/dt (voltage proportional to rate of change), requires analog integrator
- Lower sensitivity than iron-core CT (needs more turns or amplification)
- Integrator op-amp adds power consumption and complexity
- More expensive than CT approach
- Integrator can drift, needs periodic reset

### Lightning Robustness

**Excellent** - Inherently isolated from fence wire. Cannot saturate regardless of current level. Integrator circuit needs standard protection (TVS, clamp diodes).

### Estimated Additional Cost

~$8.00-12.00 (coil $4.00, integrator op-amp $1.00, peak detector $1.50, passives $1.50, protection $1.00)

---

## Option 5: Hall Effect Sensor

**Category:** Current-based sensing (non-contact magnetic field detection)

### Concept

Place a Hall effect sensor near the fence wire to measure the magnetic field generated by the pulse current.

Hall effect sensors detect magnetic fields using the Hall effect - when current flows through a conductor in the presence of a magnetic field, a voltage appears perpendicular to both the current and field. Modern Hall ICs integrate this sensing element with amplification and temperature compensation.

### How It Works

1. **Current flows:** Fence pulse creates current through the wire
2. **Magnetic field:** Current creates a circular magnetic field around the wire (right-hand rule)
3. **Hall sensing:** The IC detects the field strength and outputs a proportional voltage
4. **Direct ADC reading:** Output connects directly to MCU ADC - no additional conditioning needed

### The Distance Problem

Magnetic field strength from a wire falls off as 1/r (inverse of distance):

```
B = μ₀ × I / (2π × r)

For 10A at 10mm: B ≈ 0.2 mT
For 10A at 20mm: B ≈ 0.1 mT (half the signal)
For 10A at 50mm: B ≈ 0.04 mT (barely detectable)
```

This means:
- Sensor must be positioned very close to the wire (5-15mm ideally)
- Small positioning variations cause large measurement variations
- External magnetic fields (earth's field, nearby equipment) can interfere

### The Timing Problem

Most Hall sensors are designed for DC or low-frequency AC measurement. A 100µs fence pulse is challenging:
- Sensor bandwidth must exceed 10kHz (most are 20-100kHz, so usually OK)
- But the pulse is a single transient, not repetitive - averaging won't help
- Internal filtering on some Hall ICs may smooth out the pulse

### Scheduled Wake Considerations

Hall sensors have a significant power penalty with scheduled wake:

**Problem:** Hall sensors draw continuous current (2-10mA typical) whenever powered. They cannot capture a pulse that occurs while powered off.

**Options:**
1. **Power on during wake window:** 2s × 5mA = 10mAs per cycle → ~5.5µA average. Acceptable, but higher than other options.
2. **Use Hall + peak detector:** Power the Hall sensor, let peak detector capture the pulse, read with ADC. Still pays the power cost during wake window.
3. **Use Hall + comparator trigger:** Power Hall sensor, use its output to trigger when pulse detected, then sample with ADC.

**Startup time:** Most Hall sensors have 10-50µs startup time after power-on. This is fine for the scheduled wake approach since you're waiting ~1-2s anyway.

**Key disadvantage:** Unlike passive options (CT, capacitive plate), the Hall sensor must be powered and active to detect anything. This makes it fundamentally less power-efficient for low-duty-cycle operation.

### When to Choose This Option

- You want simple, integrated sensing with minimal external components
- The sensor can be positioned very close to the fence wire (within 10-15mm)
- Power consumption is less critical (Hall sensors draw mA, not µA)
- You're monitoring near the energizer where current is highest
- Rough indication of pulse strength is sufficient (not precision measurement)

**Honest assessment:** This is probably the weakest option for fence monitoring due to sensitivity, positioning challenges, and power consumption. It's included for completeness, but the other options are generally preferable.

### Suitable Components

| Part | Type | Sensitivity | Power | Price |
|------|------|-------------|-------|-------|
| Allegro ACS712 | Invasive (series) | 185mV/A | 10mA | $2.00 |
| Melexis MLX91220 | Contactless | 25mV/mT | 5mA | $4.00 |
| TI DRV5055 | Ratiometric | 100mV/mT | 2.5mA | $1.50 |
| Honeywell SS49E | Linear | 1.4mV/G | 6mA | $2.00 |

### Pros

- Can be truly non-contact with correct positioning
- DC-capable (can detect if energizer is stuck-on or has DC offset)
- Simple interface (analog voltage output, directly to ADC)
- Some variants have digital output with threshold

### Cons

- Sensitivity issues - fence pulse is brief (~100µs) and wire current may be low
- Positioning extremely critical for consistent readings (field falls off as 1/r)
- Susceptible to external magnetic interference (nearby motors, other wires)
- Higher power consumption than other options (mA vs µA)
- May need signal averaging across multiple pulses for reliability

### Lightning Robustness

**Poor to Moderate** - Hall effect ICs contain sensitive semiconductor elements. While the sensing is non-contact, induced voltages on power/signal lines can damage the IC. Requires robust power supply filtering and signal line protection.

### Estimated Additional Cost

~$3.00-5.00 (Hall sensor $2.00, decoupling $0.30, protection $1.00)

---

## Lightning and Surge Protection Strategies

Since robustness to induced surges (not direct strikes) is a key requirement, a layered protection approach is recommended.

### Why Electric Fences are Lightning Magnets

Electric fences present a worst-case scenario for lightning vulnerability:

1. **Long conductor runs:** Fences can be kilometers long, acting as an antenna for electromagnetic pulses
2. **Elevated installation:** Fence wires are typically 0.5-1.5m above ground - not the highest point, but exposed
3. **Poor grounding:** While energizers have ground rods, the fence wire itself is isolated from ground (by design)
4. **Rural locations:** Often in open fields with few taller structures for protection

A lightning strike within 1-2km can induce thousands of volts on the fence wire through electromagnetic coupling, even without a direct hit. The energizer itself usually has surge protection, but your sensor is downstream and exposed.

### Protection Philosophy

**Goal:** Survive induced surges from nearby strikes (within ~500m), accept that direct strikes will cause damage.

**Realistic expectations:**
- Direct strike to fence wire: ~200kA, ~1GJ - no practical protection possible
- Nearby strike (100m): ~10kV induced, ~10kA surge - survivable with proper design
- Distant strike (500m): ~1kV induced, ~1kA surge - easily survivable

**Design principle:** Multiple protection layers, each handling different energy levels and response speeds. Assume any single layer can fail; the system should survive with defense in depth.

### Protection Layers

#### Layer 1: Gas Discharge Tube (GDT)

```
Input Terminal → GDT (90V) → Rest of circuit
                  ↓
                 GND
```

- **Function:** Crowbar device that short-circuits when voltage exceeds threshold
- **Handles:** High-energy transients (kA range, joules of energy)
- **Response time:** ~1µs (slow, needs faster backup)
- **Example part:** Bourns 2049-09-SM (90V breakdown, 5kA surge)
- **Cost:** ~$0.50
- **Placement:** At input terminals, before any semiconductors

#### Layer 2: TVS Diode (existing, upgrade recommended)

```
After GDT → TVS (bidirectional) → Next stage
              ↓
             GND
```

- **Function:** Clamps voltage by avalanche breakdown
- **Response time:** <1ns (very fast)
- **Handles:** Medium energy after GDT absorbs initial surge
- **Current part:** SMBJ15CA (15V clamp)
- **Upgrade option:** Automotive-grade (AEC-Q101) for higher reliability
- **Placement:** After GDT, before active components

#### Layer 3: Series Resistance (existing)

```
→ Current Limiting R (≥1kΩ) →
```

- **Function:** Limits current during transient clamping
- **Current design:** 1MΩ input resistor (excellent protection)
- **Consideration:** Higher R = better protection but slower response
- **Tradeoff:** For intensity measurement, may need lower R for accuracy

#### Layer 4: Clamp Diodes (existing)

```
Signal → BAV99 → To comparator/ADC
          ↓↑
      GND  Vcc
```

- **Function:** Final protection, clamps signal to supply rails
- **Current part:** BAV99 dual diode
- **Upgrade option:** ESD-rated TVS diodes (e.g., PESD5V0S1BL)
- **Placement:** Immediately before IC input pins

### Additional Hardening Measures

#### PCB Design

- **Spark gaps:** 0.3mm trace gaps on input to ground plane (arcs at ~1kV in air)
- **Creepage distance:** Minimum 2mm between high-voltage and low-voltage sections
- **Ground plane strategy:** Isolated sense ground with single-point connection to main ground

#### Physical Protection

- **Metal enclosure:** Provides Faraday cage effect, reduces induced EMI
- **Conformal coating:** Silicone or acrylic coating prevents surface tracking during high-voltage events
- **Cable shielding:** Shielded cable from sensor to main board, shield grounded at one end only

#### Component Selection

- **Automotive-grade parts:** Higher reliability, tested for transients (AEC-Q100/Q101)
- **Voltage derating:** Use components rated for 2x expected maximum voltage
- **Redundant protection:** Multiple protection stages, assume any single stage can fail

---

## Comparison Matrix

| Approach | Accuracy | Hardware Complexity | Power Impact | Lightning Robustness | Installation | Scheduled Wake | Cost |
|----------|----------|---------------------|--------------|---------------------|--------------|----------------|------|
| Peak Detector + ADC | Good | Moderate (+4 parts) | +50-100µA | Moderate | Non-contact | Excellent | +$1.50 |
| Direct ADC Sampling | Fair | Minimal (firmware) | None | Good | Non-contact | Good* | $0 |
| Current Transformer | Excellent | Moderate (+5 parts) | +10-50µA | Excellent | On-wire | Excellent | +$5.00 |
| Rogowski Coil | Excellent | High (+8 parts) | +100-200µA | Excellent | On-wire | Fair** | +$10.00 |
| Hall Effect | Fair | Low (+2 parts) | +5-10mA | Poor | Near-wire | Poor*** | +$4.00 |

*Direct ADC requires fast MCU response to catch pulse peak - may miss peak if using deep sleep during wait
**Rogowski requires integrator to be powered during wait window, adds complexity
***Hall sensor must be continuously powered during wait window, highest power cost

---

## Recommendations

The "best" option depends on your priorities. Here's how to think about the decision:

### Decision Framework

**Prioritize ease of installation?** → Options 1 or 2 (voltage-based, non-contact)
- No fence wire modification required
- Can add sensors anywhere along the fence line
- Trade-off: Less accurate, more sensitive to mounting position

**Prioritize measurement accuracy?** → Option 3 (CT)
- True current measurement correlates to fence effectiveness
- Galvanic isolation protects against surges
- Trade-off: Must install CT around fence wire

**Prioritize lightning robustness?** → Option 3 (CT) or Option 4 (Rogowski)
- Inherent galvanic isolation
- No direct electrical connection to fence
- Trade-off: Installation complexity, higher cost

**Prioritize minimal changes?** → Option 2 (Direct ADC)
- Firmware only, no hardware changes
- Trade-off: Lowest accuracy, timing-dependent

### Primary Recommendation: Peak Detector + ADC (Option 1)

Best balance of capability and simplicity for non-contact installation.

**Rationale:**
- Builds on existing capacitive coupling design
- Maintains non-contact installation (no fence wire modification)
- Moderate complexity increase
- Acceptable power impact (can power-switch op-amp)
- Add GDT + upgraded TVS for lightning robustness

**Enhanced protection additions:**
1. GDT (90V) at sense plate input
2. Automotive-grade TVS after input resistor
3. ESD-rated clamp diodes before op-amp

### Alternative: Current Transformer

Best choice if willing to install around the fence wire.

**Rationale:**
- True current measurement (better correlation to fence energy)
- Inherent galvanic isolation (excellent lightning protection)
- Enables future features (load detection, short circuit detection)
- Well-proven technology in power monitoring

**Trade-off:** Requires routing fence wire through CT during installation.

---

## Next Steps

1. Select preferred approach based on installation constraints
2. Design detailed circuit schematic
3. Prototype on breadboard for validation
4. Characterize response against known fence pulses
5. Integrate into main PCB design
6. Update firmware for intensity measurement and reporting

---

## TL;DR Summary

Given the **scheduled wake operation** (wake every ~30 min, measure, sleep):

**If you can install around the fence wire:** Use a **Current Transformer (Option 3)**. It gives you accurate current measurement, inherent lightning protection through galvanic isolation, and works perfectly with scheduled wake - the CT is passive during the wait-for-pulse period. It's the most robust and accurate solution.

**If you need non-contact sensing:** Use **Peak Detector + ADC (Option 1)**. It builds on the existing capacitive design, captures pulse amplitude reliably, and the peak detector handles the pulse timing automatically - no fast MCU response needed. Add a GDT and upgraded TVS for lightning protection.

**If you want zero hardware changes:** Try **Direct ADC Sampling (Option 2)** first. It's free to implement in firmware and will give you a baseline. Caveat: requires the MCU to respond quickly to the comparator interrupt, so you'll need to stay in a lighter sleep mode during the ~2s wait window. If accuracy is insufficient, upgrade to Option 1.

**Skip Hall Effect (Option 5)** - it requires continuous power during the wait window, defeating much of the power savings from scheduled wake.

**Skip Rogowski Coil (Option 4)** - the integrator requirement adds complexity that doesn't fit well with the scheduled wake model. Use a CT instead for similar benefits with less complexity.
