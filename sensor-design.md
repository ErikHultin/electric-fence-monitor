# Electric Fence Monitor - Sensor Design

## Battery Voltage + Capacitive Pulse Detection

This design provides the two most important pieces of information:
- **Is the system alive?** (battery voltage)
- **Is the fence working?** (pulse detection)

Current sensing can be added later if needed.

---

## System Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│                         REMOTE SENSOR UNIT                             │
│                                                                        │
│                                                                        │
│   FENCE PULSE SENSING                      BATTERY MONITORING          │
│   ═══════════════════                      ══════════════════          │
│                                                                        │
│   Fence Wire                               12V Lead-Acid               │
│       ║                                         │                      │
│       ║                                    ┌────┴────┐                 │
│   ┌───╨───┐                                │Resistive│                 │
│   │ Sense │                                │ Divider │                 │
│   │ Plate │                                └────┬────┘                 │
│   └───┬───┘                                     │                      │
│       │                                         │                      │
│       ▼                                         ▼                      │
│   ┌────────┐                               ┌────────┐                  │
│   │ Pulse  │                               │ Buffer │                  │
│   │ Cond.  │                               │ (opt)  │                  │
│   └───┬────┘                               └───┬────┘                  │
│       │                                        │                       │
│       │         ┌──────────────────┐           │                       │
│       └────────▶│       MCU        │◀──────────┘                       │
│                 │                  │                                   │
│                 │  • ADC (voltage) │                                   │
│                 │  • Comparator/   │                                   │
│                 │    GPIO (pulse)  │                                   │
│                 │  • Timer (rate)  │                                   │
│                 └────────┬─────────┘                                   │
│                          │                                             │
│                          ▼                                             │
│                    ┌──────────┐                                        │
│                    │   LoRa   │                                        │
│                    │  Module  │                                        │
│                    └──────────┘                                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Battery Voltage Measurement

### Design Requirements

A 12V lead-acid battery ranges from about 10.5V (dead) to 14.4V (charging). We need to scale this to the MCU's ADC range (typically 0-3.3V) while ensuring headroom for charging voltage and transients.

### Resistive Divider

```
        12V Battery
            │
            │
           ┌┴┐
           │ │ R1 = 120kΩ
           │ │
           └┬┘
            ├──────────────────▶ To ADC (0 - 3.3V)
           ┌┴┐                        │
           │ │ R2 = 27kΩ              │
           │ │                    ┌───┴───┐
           └┬┘                    │ 100nF │ (optional filter)
            │                     └───┬───┘
           GND                       GND
```

**Divider math:**

$$V_{ADC} = V_{BAT} \times \frac{R2}{R1 + R2} = V_{BAT} \times \frac{27k}{147k} = V_{BAT} \times 0.184$$

| Battery Voltage | ADC Voltage |
|-----------------|-------------|
| 10.5V (dead) | 1.93V |
| 12.0V (nominal) | 2.21V |
| 12.7V (full, resting) | 2.34V |
| 14.4V (charging) | 2.65V ✓ |
| 15V (overvoltage) | 2.76V ✓ |

This ratio provides adequate headroom for charging voltage and transients while maintaining good ADC resolution across the operating range.

### Power Consumption of the Divider

This is often overlooked:

$$I_{divider} = \frac{V_{BAT}}{R1 + R2} = \frac{12V}{147k\Omega} = 82\mu A$$

That's 82 µA continuous—not terrible, but it adds up.

#### Option A: Accept it (simple, reliable)

82 µA from a 12V battery = ~1 mW. Over 24 hours = 24 mWh. A 7Ah battery holds 84 Wh, so this is 0.03% per day. **Negligible.**

#### Option B: High-impedance divider (if you're optimizing hard)

```
        12V Battery
            │
           ┌┴┐
           │ │ R1 = 1MΩ
           │ │
           └┬┘
            ├──────────────▶ To ADC
           ┌┴┐                   │
           │ │ R2 = 220kΩ     ┌──┴──┐
           │ │                │100nF│ Critical for high-Z
           └┬┘                └──┬──┘
            │                    │
           GND                  GND
```

**Current:** 12V / 1.22MΩ = ~10 µA

**Tradeoff:** Higher impedance means more susceptibility to noise and ADC input leakage errors. The 100nF capacitor is essential to stabilize the voltage during ADC sampling.

#### Option C: Switched divider (lowest power, most complex)

```
        12V Battery
            │
            │
       ┌────┴────┐
       │ P-MOSFET│◀──── GPIO (active low)
       │(Si2301) │
       └────┬────┘
            │
           ┌┴┐
           │ │ R1 = 100kΩ
           │ │
           └┬┘
            ├──────────────▶ To ADC
           ┌┴┐
           │ │ R2 = 33kΩ
           │ │
           └┬┘
            │
           GND
```

Turn on the MOSFET only during measurement:

```c
float read_battery_voltage(void) {
    // Enable divider
    gpio_write(VBAT_EN_PIN, LOW);  // P-FET on
    delay_ms(1);  // Settling time

    // Sample ADC (oversample for noise reduction)
    uint32_t sum = 0;
    for (int i = 0; i < 16; i++) {
        sum += adc_read(VBAT_ADC_CH);
    }
    uint16_t raw = sum / 16;

    // Disable divider
    gpio_write(VBAT_EN_PIN, HIGH);  // P-FET off

    // Convert to voltage
    float v_adc = (raw / 4096.0f) * 3.3f;  // 12-bit ADC, 3.3V ref
    float v_bat = v_adc / 0.248f;          // Divider ratio

    return v_bat;
}
```

**Average current:** essentially zero between measurements.

### Protection

Add a TVS diode to clamp transients:

```
        12V Battery ──┬──────────────▶ To divider
                      │
                  ┌───┴───┐
                  │ TVS   │ SMBJ15CA (15V clamp)
                  │       │
                  └───┬───┘
                      │
                     GND
```

---

## Part 2: Capacitive Pulse Detection

### How It Works

The fence wire acts as one plate of a capacitor. Your sense plate is the other plate. Air (or plastic enclosure) is the dielectric:

```
     Fence Wire (8kV pulses)
            ║
            ║
            ║        Electric field couples through air
            ║              │
            ║              │
            ║              ▼
     ═══════╬═══════════════════════
            ║         │
            ║    ┌────┴────┐
            ║    │ Sense   │  ← Copper plate, ~25cm² (5×5cm)
            ║    │ Plate   │
            ║    └────┬────┘
            ║         │
                      ▼
                 To signal conditioning
```

**Coupling capacitance:**

For a parallel plate approximation:

$$C = \varepsilon_0 \times \varepsilon_r \times \frac{A}{d}$$

Where:
- ε₀ = 8.85 pF/m (permittivity of free space)
- εᵣ = 1 for air, ~3 for plastic
- A = plate area
- d = distance from wire

For a 5×5 cm plate, 2 cm from the wire:

$$C \approx 8.85 \times 1 \times \frac{0.0025}{0.02} \approx 1.1\text{ pF}$$

In practice, with fringing fields and geometry effects, expect **1-10 pF**.

### Signal Conditioning Circuit

The capacitive coupling produces a tiny current when voltage changes:

$$I = C \times \frac{dV}{dt}$$

For a fence pulse:
- V = 8 kV
- Rise time ≈ 10 µs
- C = 5 pF

$$I = 5\text{pF} \times \frac{8000V}{10\mu s} = 4\text{ mA (peak)}$$

This current flows through our input impedance, creating a voltage we can detect.

### Circuit Design

```
                                           Protection              Comparator
                                          ┌─────────┐            ┌──────────┐
 Sense      R1         R2                 │         │            │          │
 Plate ────/\/\/──┬───/\/\/──┬────────────┤ Clamp   ├────────────┤+         │
           1MΩ    │   100kΩ  │            │ Diodes  │      ┌─────┤-    Out  ├──▶ To MCU
                  │          │            │         │      │     │          │   (GPIO/INT)
              ┌───┴───┐  ┌───┴───┐        └─────────┘    ┌─┴─┐   └──────────┘
              │ Cp    │  │ C1    │                       │Vth│
              │ ~5pF  │  │ 100pF │                       │   │ Threshold
              │(stray)│  │       │                       └─┬─┘ (e.g., 1V)
              └───┬───┘  └───┬───┘                         │
                  │          │                            GND
                 GND        GND
```

**Detailed schematic:**

```
                CAPACITIVE PULSE DETECTOR

Sense    ┌─────────────────────────────────────────────────────────┐
Plate    │                                                         │
  │      │   R1        R2          D1,D2         R3                │
  │      │  1MΩ      100kΩ        BAV99        10kΩ               │
  ○──────┼──/\/\──┬───/\/\──┬──────┼┼────┬──────/\/\──┬──────┐    │
         │        │         │      ││    │            │      │    │
         │     ┌──┴──┐   ┌──┴──┐   ││  ┌─┴─┐       ┌──┴──┐   │    │
         │     │ Cp  │   │100pF│  GND  │Vcc│       │ C2  │   │    │
         │     │~5pF │   │     │   ││  └───┘       │100nF│   │    │
         │     └──┬──┘   └──┬──┘   ┘│              └──┬──┘   │    │
         │        │         │       │                 │      │    │
         │       GND       GND     GND               GND     │    │
         │                                                   │    │
         │                         ┌─────────────────────────┘    │
         │                         │                              │
         │                         │   COMPARATOR                 │
         │                         │   (built into MCU            │
         │                         │    or external)              │
         │                         │                              │
         │                         │      ┌───────────┐           │
         │                         └──────┤+          │           │
         │                                │           │           │
         │                          Vth ──┤-     Out  ├───────────┼──▶ To MCU
         │                                │           │           │    GPIO
         │                                └───────────┘           │
         │                                                        │
         └────────────────────────────────────────────────────────┘

Component values:
 R1 = 1MΩ      - Input impedance (limits current)
 R2 = 100kΩ    - Forms low-pass with C1
 C1 = 100pF    - Bandwidth limiting (~16 kHz)
 D1,D2 = BAV99 - Clamps signal to 0-3.3V
 R3 = 10kΩ     - Isolates comparator input
 C2 = 100nF    - Filters noise at comparator
 Vth = ~1V     - Detection threshold (adjustable)
```

### How the Circuit Works

**Step 1: Coupling**
The fence pulse (8 kV, 100 µs) induces a voltage spike through the parasitic capacitance Cp.

**Step 2: Attenuation**
R1 (1MΩ) limits current and provides high input impedance. Combined with Cp, it forms a differentiator—we see dV/dt.

**Step 3: Bandwidth Limiting**
R2 + C1 form a low-pass filter at ~16 kHz. This passes the fence pulse fundamental but rejects high-frequency noise (RF, switching spikes).

**Step 4: Protection**
The BAV99 dual diode clamps the signal to -0.3V to Vcc+0.3V, protecting the comparator/MCU from the raw fence transient.

**Step 5: Detection**
The comparator fires when the conditioned signal exceeds the threshold (Vth). Output goes to an MCU interrupt pin.

### Setting the Threshold

The threshold voltage determines sensitivity vs. noise immunity:

```
                    │
          Too high ─┼── Might miss weak pulses
                    │
      Vth = 1.0V ───┼── Good starting point
                    │
           Too low ─┼── False triggers from noise
                    │
```

**Make it adjustable during development:**

```
        3.3V
         │
        ┌┴┐
        │ │ 10kΩ
        │ │
        └┬┘
         ├────────▶ Vth (to comparator -)
        ┌┴┐
        │ │ 10kΩ trimmer
        │↓│
        └┬┘
         │
        GND
```

Or set it digitally with a DAC or PWM + filter.

---

## Complete Schematic

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           COMPLETE SENSOR CIRCUIT                           │
│                                                                             │
│                                                                             │
│  BATTERY VOLTAGE                          PULSE DETECTION                   │
│  ═══════════════                          ═══════════════                   │
│                                                                             │
│  12V Battery                              Fence Wire                        │
│      │                                         ║                            │
│      │                                         ║                            │
│  ┌───┴───┐                                ┌────╨────┐                       │
│  │ TVS   │                                │  Sense  │                       │
│  │SMBJ15A│                                │  Plate  │                       │
│  └───┬───┘                                └────┬────┘                       │
│      │                                         │                            │
│      ├───────────────────────┐                 │                            │
│      │                       │            1MΩ  │                            │
│     ┌┴┐                      │          ┌──/\/\/──┐                         │
│     │ │ 120kΩ                │          │         │                         │
│     └┬┘                      │          │      ┌──┴──┐                      │
│      │                       │          │      │100pF│                      │
│      ├───────▶ ADC_VBAT      │          │      └──┬──┘                      │
│      │                       │          │         │                         │
│     ┌┴┐                      │       100kΩ        │                         │
│     │ │ 27kΩ                 │      ┌──/\/\/──────┤                         │
│     └┬┘                      │      │             │                         │
│      │                       │      │         ┌───┴───┐                     │
│      │                       │      │         │ BAV99 │                     │
│     ───                      │      │         └───┬───┘                     │
│     GND                      │      │             │                         │
│                              │      │          10kΩ                         │
│                              │      │        ┌──/\/\/──┐                    │
│                              │      │        │         │                    │
│                              │      │        │      ┌──┴──┐                 │
│                              │      │        │      │100nF│                 │
│  ┌───────────────────────────┴──────┴────────┴──────┴─────┴──────────────┐ │
│  │                                                                        │ │
│  │                              MCU (STM32L0)                             │ │
│  │                                                                        │ │
│  │    PA0 (ADC) ◀── Battery voltage                                      │ │
│  │    PA1 (COMP+) ◀── Pulse signal                                       │ │
│  │    Internal COMP- ← DAC or internal Vref (threshold)                  │ │
│  │    PA2 (EXTI) ◀── Comparator output (interrupt)                       │ │
│  │                                                                        │ │
│  │    SPI ──────────▶ LoRa Module (RFM95W)                               │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## MCU Integration (STM32L0 Example)

The STM32L0 series has built-in comparators, which simplifies the design:

```
                    Internal to STM32L0
                   ┌─────────────────────────────────┐
                   │                                 │
 Pulse signal ────▶│ COMP1_INP ──────┐              │
                   │                  │  ┌────────┐  │
                   │                  ├──┤  COMP1 ├──┼──▶ COMP1_OUT
                   │                  │  └────────┘  │    (to EXTI)
                   │ DAC_OUT1 ───────┘              │
                   │    ▲                           │
                   │    │ (software-set threshold)  │
                   │                                 │
 Battery div. ────▶│ ADC_IN0                        │
                   │                                 │
                   └─────────────────────────────────┘
```

### Firmware

```c
#include "stm32l0xx_hal.h"

// Configuration
#define VBAT_DIVIDER_RATIO    0.184f    // 27k / (120k + 27k)
#define PULSE_THRESHOLD_MV    1000      // 1V threshold
#define PULSE_MIN_INTERVAL_MS 500       // Debounce (fences pulse ~1/sec)
#define EXPECTED_PULSE_RATE   1.0f      // Hz

// State
volatile uint32_t pulse_count = 0;
volatile uint32_t last_pulse_tick = 0;

// Comparator interrupt handler
void COMP1_IRQHandler(void) {
    if (__HAL_COMP_COMP1_EXTI_GET_FLAG()) {
        __HAL_COMP_COMP1_EXTI_CLEAR_FLAG();

        uint32_t now = HAL_GetTick();
        if ((now - last_pulse_tick) > PULSE_MIN_INTERVAL_MS) {
            pulse_count++;
            last_pulse_tick = now;
        }
    }
}

// Initialize comparator for pulse detection
void pulse_detector_init(void) {
    // Enable clocks
    __HAL_RCC_COMP_CLK_ENABLE();
    __HAL_RCC_DAC_CLK_ENABLE();

    // Configure DAC for threshold generation
    DAC_HandleTypeDef hdac;
    hdac.Instance = DAC;
    HAL_DAC_Init(&hdac);

    DAC_ChannelConfTypeDef dac_config = {
        .DAC_Trigger = DAC_TRIGGER_NONE,
        .DAC_OutputBuffer = DAC_OUTPUTBUFFER_ENABLE
    };
    HAL_DAC_ConfigChannel(&hdac, &dac_config, DAC_CHANNEL_1);

    // Set threshold: 1V = (1.0 / 3.3) * 4096 ≈ 1241
    uint32_t threshold_dac = (PULSE_THRESHOLD_MV * 4096) / 3300;
    HAL_DAC_SetValue(&hdac, DAC_CHANNEL_1, DAC_ALIGN_12B_R, threshold_dac);
    HAL_DAC_Start(&hdac, DAC_CHANNEL_1);

    // Configure comparator
    COMP_HandleTypeDef hcomp;
    hcomp.Instance = COMP1;
    hcomp.Init.InvertingInput = COMP_INPUT_MINUS_DAC1_CH1;
    hcomp.Init.NonInvertingInput = COMP_INPUT_PLUS_IO1;  // PA1
    hcomp.Init.OutputPol = COMP_OUTPUTPOL_NONINVERTED;
    hcomp.Init.Mode = COMP_POWERMODE_ULTRALOWPOWER;      // ~0.5µA
    hcomp.Init.WindowMode = COMP_WINDOWMODE_DISABLE;
    hcomp.Init.TriggerMode = COMP_TRIGGERMODE_IT_RISING;

    HAL_COMP_Init(&hcomp);
    HAL_COMP_Start(&hcomp);

    // Enable comparator interrupt
    HAL_NVIC_SetPriority(ADC1_COMP_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(ADC1_COMP_IRQn);
}

// Read battery voltage
float read_battery_voltage(void) {
    ADC_HandleTypeDef hadc;
    hadc.Instance = ADC1;

    HAL_ADC_Start(&hadc);
    HAL_ADC_PollForConversion(&hadc, 10);
    uint32_t raw = HAL_ADC_GetValue(&hadc);
    HAL_ADC_Stop(&hadc);

    float v_adc = (raw / 4096.0f) * 3.3f;
    float v_bat = v_adc / VBAT_DIVIDER_RATIO;

    return v_bat;
}

// Get fence status
typedef enum {
    FENCE_OK,
    FENCE_NO_PULSE,
    FENCE_LOW_RATE
} fence_status_t;

typedef struct {
    float battery_voltage;
    uint32_t pulse_count;
    float pulse_rate_hz;
    fence_status_t status;
} sensor_data_t;

sensor_data_t read_sensors(uint32_t measurement_period_ms) {
    sensor_data_t data;

    // Capture pulse count atomically
    __disable_irq();
    uint32_t count = pulse_count;
    pulse_count = 0;
    __enable_irq();

    // Calculate rate
    data.pulse_count = count;
    data.pulse_rate_hz = (float)count / (measurement_period_ms / 1000.0f);

    // Read battery
    data.battery_voltage = read_battery_voltage();

    // Determine status
    if (count == 0) {
        data.status = FENCE_NO_PULSE;
    } else if (data.pulse_rate_hz < (EXPECTED_PULSE_RATE * 0.8f)) {
        data.status = FENCE_LOW_RATE;
    } else {
        data.status = FENCE_OK;
    }

    return data;
}
```

---

## Power Budget

| Component | Sleep | Active | Duty | Average |
|-----------|-------|--------|------|---------|
| MCU (Stop mode) | 0.3 µA | 3 mA | 0.1% | ~3.3 µA |
| Comparator (ULP mode) | 0.5 µA | 0.5 µA | 100% | 0.5 µA |
| Voltage divider | 82 µA | 82 µA | 100% | 82 µA |
| LoRa (sleep) | 0.2 µA | 100 mA | 0.01% | ~10 µA |
| **Total** | | | | **~96 µA** |

From a 7 Ah battery: **~7 Ah / 0.096 mA ≈ 3000+ days** (theoretical).

In practice, battery self-discharge (~3%/month for lead-acid) will be the limiting factor, not your circuit.

---

## Physical Construction

### Sense Plate Design

```
        ┌─────────────────────────────────────────────┐
        │           SENSOR HEAD ENCLOSURE             │
        │              (IP65 box)                     │
        │                                             │
        │    ┌───────────────────────────────────┐   │
        │    │                                   │   │
        │    │     Copper plate (sense)          │   │
        │    │     5cm × 5cm, 1mm thick          │   │
        │    │     ┌─────────────────────┐       │   │
        │    │     │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│       │   │
        │    │     │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│       │   │
        │    │     │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│       │   │
        │    │     └──────────┬──────────┘       │   │
        │    │                │ Wire to PCB      │   │
        │    └────────────────┼──────────────────┘   │
        │                     │                      │
        │    ┌────────────────┴──────────────────┐   │
        │    │         Signal conditioning       │   │
        │    │              PCB                  │   │
        │    │    (fits in same enclosure)       │   │
        │    └───────────────────────────────────┘   │
        │                                             │
        └───────────────────────────────────────────┬─┘
                                                    │
                              ════════════════════════  Fence wire
                              (plate faces wire,
                               10-30mm gap)
```

### Mounting

```
        SIDE VIEW

        Enclosure
        ┌────────────────┐
        │ ┌────────────┐ │         Fence
        │ │   Plate    │ │          Wire
        │ │            │◀┼── 10-30mm ──║
        │ └────────────┘ │             ║
        │    │           │             ║
        │ ┌──┴───────┐   │             ║
        │ │   PCB    │   │
        │ └──────────┘   │
        │                │
        └────────┬───────┘
                 │
           Mounting bracket
           (clamps to fence post)
```

### Shielding Considerations

To reject interference from other directions, add a grounded shield on the *back* of the sense plate:

```
        TOP VIEW

                    Fence wire
                        ║
                        ║
        ┌───────────────╫───────────────┐
        │               ║               │
        │    ┌──────────╫──────────┐    │
        │    │░░░░░░░░░░║░░░░░░░░░░│    │ ← Shield (grounded)
        │    │░ ┌───────╫───────┐ ░│    │   (copper, connected to GND)
        │    │░ │▓▓▓▓▓▓▓█▓▓▓▓▓▓▓│ ░│    │
        │    │░ │▓▓▓▓▓▓▓█▓▓▓▓▓▓▓│ ░│    │ ← Sense plate
        │    │░ │▓▓▓▓▓▓▓█▓▓▓▓▓▓▓│ ░│    │   (floats, to signal)
        │    │░ └───────╫───────┘ ░│    │
        │    │░░░░░░░░░░║░░░░░░░░░░│    │
        │    └──────────╫──────────┘    │
        │               ║               │
        └───────────────╫───────────────┘
                        ║
                        ▼
            Signal from fence only
            (shielded from sides/back)
```

---

## Bill of Materials

| Component | Value/Part | Qty | Notes |
|-----------|------------|-----|-------|
| MCU | STM32L072CBT6 | 1 | Or STM32L031 for simpler |
| LoRa | RFM95W-868/915 | 1 | Match to your region |
| LDO | MCP1700-3302E | 1 | 2µA quiescent |
| R1 (batt) | 120kΩ 1% | 1 | Voltage divider |
| R2 (batt) | 27kΩ 1% | 1 | Voltage divider |
| R1 (pulse) | 1MΩ | 1 | Input impedance |
| R2 (pulse) | 100kΩ | 1 | Filter |
| R3 (pulse) | 10kΩ | 1 | Comparator isolation |
| C1 (pulse) | 100pF | 1 | Bandwidth limit |
| C2 (pulse) | 100nF | 1 | Noise filter |
| D1 (pulse) | BAV99 | 1 | Clamp diodes |
| TVS | SMBJ15CA | 1 | Battery protection |
| Sense plate | Cu 50×50×1mm | 1 | Copper sheet |
| Enclosure | IP65 junction box | 1 | ~100×68×50mm |

**Estimated cost:** ~$15-20 for the sensor node (excluding battery and solar).
