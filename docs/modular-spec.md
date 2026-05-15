# Modular architecture specification

> **Status: DRAFT 0.2.** Proposed architecture, not yet binding.
> Intended to be redlined and refined into a `modular-spec-v1.0` tag
> before any hardware committed to fab follows it. All design decisions
> below are subject to revision based on bring-up evidence.
>
> *Changes since DRAFT 0.1:* ADR-1 power topology now supports
> multi-cell LiIon (1S–6S) batteries up to a 24 V VIN_RAW ceiling,
> implemented as two regulator footprints with per-SKU DNP. §3.4 open
> questions closed (extension owns its power management). ADR-5 added
> for the implementation-language choice (C++17 across the framework).
> Sensor categories expanded from two to three (DISCRETE / BATCH /
> STREAMING). SensorObject interface extended with `read_frame()` for
> streaming data movement. §7.3 streaming-flow specified concretely.

This document specifies the modular evolution of `sensorkit` from a
single fixed-purpose door sensor into a kit-style platform supporting
heterogeneous sensors over a small number of carrier board variants
(BLE / Wi-Fi / Zigbee / LoRa). It defines the hardware contract
(connector, power, signalling) between *carrier board* and *extension*,
and the firmware framework that makes a single source tree support
many sensor types.

The companion documents are
[`architecture.md`](architecture.md) (current single-board design) and
[`ble-protocol.md`](ble-protocol.md) (the BLE upstream protocol).
Cross-references between them are by stable section anchors.

## 1. Scope

### 1.1 In scope

- A **carrier board** PCB family, one PCB per PHY (BLE,
  Wi-Fi, Zigbee, LoRa). All carrier board variants share the same physical
  extension connector, the same firmware framework, and the same
  upstream protocol logic.
- An **extension** PCB family, one per sensor type (or sensor
  cluster — multiple sensors on one extension is allowed).
  Extensions are PHY-agnostic — the same door-contact extension works
  on a BLE carrier board or a Zigbee carrier board without modification.
- A **firmware framework** that abstracts sensor-specific drivers from
  PHY-specific stacks, so a new sensor type only requires a new driver
  + a one-line registration, not a full firmware fork.
- An **upstream protocol matrix** that routes discrete events
  (door, motion), batched samples (temperature, humidity), and
  streaming data (audio, video, vibration FFT) onto the appropriate
  transport for each carrier board's PHY (BTHome v2 for BLE-friendly payloads,
  MQTT for everything else).

### 1.2 Out of scope

- **Programmable interconnect** between MCU pins and extension pins
  via on-board mux/switch. The connector pinout is fixed and pins
  retain their mikroBUS roles across all carrier board variants. Different
  extensions plug into the same pins; pin function is statically
  bound to extension type via firmware config (see §5 ADR-3).
- **Code generation tooling.** Firmware is hand-written C++ (see
  ADR-5 in §8) with framework support; no GUI / IDE plugin generates
  the firmware for a given assembly.
- **Multiple extensions per carrier board.** Each carrier board has a
  single extension socket in v1.0. An **expansion hub** (a passive
  fan-out board that plugs into one carrier-board socket and exposes
  multiple downstream sockets for extensions) is a possible v2
  addition but not specified here. The "expansion hub" name avoids
  reusing "carrier board" for a different role.
- **Self-describing extensions over a service protocol.** Discovery is
  a single ADC reading (§5 ADR-3), not an I2C handshake or EEPROM
  read. Bidirectional service-channel protocols are a possible v2
  upgrade.

### 1.3 Design principles

1. **Wireless first.** Wired uplink (USB/Ethernet) is supported only
   for stationary, mains-powered carrier board configurations as a secondary use
   case. The BLE carrier board is the canonical reference design.
2. **Single firmware tree.** All PHY variants and all sensor types
   build from one Git repository. PHY-specific code lives below a HAL
   line; sensor-specific code lives above a SensorObject interface
   line; the bulk of the framework lives between.
3. **Static configuration is the default.** Dynamic discovery exists
   only to detect *wrong* extensions (red-LED behaviour), not to
   support truly heterogeneous deployments without firmware rebuild.
4. **Reuse established standards.** mikroBUS for connector,
   BTHome v2 + MQTT for upstream protocols, JST-SH for the compact
   form factor. No invented connectors, no invented wire formats, no
   proprietary handshakes unless absolutely necessary.

## 2. System block diagram

```
   ┌──────────────────────────────────────────────────────────────┐
   │                     CARRIER BOARD                             │
   │                                                               │
   │   Power input options (one populated per SKU):                │
   │   - CR2032 (2.0-3.0 V)         ──┐                            │
   │   - 1S LiIon (3.0-4.2 V)       ──┤                            │
   │   - USB-C (5 V)                ──┤   ┌────────────────────┐   │
   │   - 2S-6S LiIon (6.0-24 V)     ──┼──▶│ Regulator (one of 2│   │
   │   - DC barrel jack (5-24 V)    ──┘   │ footprints, DNP'd):│   │
   │                                       │  A) low-V boost    │   │
   │                            VIN_RAW ──▶│     (≤5.5V in)     │   │
   │                                       │  B) wide buck-boost│   │
   │                                       │     (≤30V in)      │   │
   │                                       └───────┬────────────┘   │
   │                                               │ 3V3_BUS       │
   │                                               ▼               │
   │   ┌───────────────────┐  ┌──────────────────────────────────┐  │
   │   │  PHY MCU          │  │ Status LED (heartbeat/state/err) │  │
   │   │  - STM32WB09 BLE  │  └──────────────────────────────────┘  │
   │   │  - ESP32 WiFi     │                                        │
   │   │  - EFR32MG24 Zig  │                                        │
   │   │  - STM32WLE5 LoRa │                                        │
   │   │                   │                                        │
   │   │  GPIO/I2C/SPI ────┤◀───── EXTENSION CONNECTOR (mikroBUS) ─│
   │   │                   │       3V3 │ VIN_RAW │ GND │ INT │ …   │
   │   │  ADC ────────────►│        AN│RST│CS│SCK│MISO│MOSI│SDA│… │
   │   │  (ID-resistor)    │                                       │
   │   └───────────────────┘                                       │
   └──────────────────────────────────────────────────────────────┘
                              │
                              │ (mikroBUS 16-pin)
                              ▼
   ┌──────────────────────────────────────────────────────────────┐
   │                     EXTENSION (one of N)                      │
   │                                                               │
   │   Power: 3V3 and/or VIN_RAW from carrier board. Extension owns any      │
   │   further regulation (LDO, boost, buck) it needs.             │
   │                                                               │
   │   ┌──────────────────────────────────────────────┐            │
   │   │ Simple extension (door, magnet, temperature):│            │
   │   │   sensor IC → SDA/SCL or GPIO → connector    │            │
   │   │   ID resistor (1) → AN pin                   │            │
   │   └──────────────────────────────────────────────┘            │
   │                       OR                                      │
   │   ┌──────────────────────────────────────────────┐            │
   │   │ Smart extension (audio, video, FFT, NN):     │            │
   │   │   companion MCU + multiple sensors           │            │
   │   │   companion MCU ↔ carrier board over SPI or UART       │            │
   │   │   edge processing on companion MCU           │            │
   │   │   carrier board only sees pre-processed frames         │            │
   │   └──────────────────────────────────────────────┘            │
   └──────────────────────────────────────────────────────────────┘
```

## 3. ADR-1: Power topology

### 3.1 Decision

The carrier board supports **any power input from 2.0 V to 24 V** on the
`VIN_RAW` rail, via **two regulator footprints on the same PCB** —
exactly one is populated per assembled SKU:

| Footprint | Input range | Class | Example part | Iq | Populated for |
|---|---|---|---|---|---|
| **A** — Low-V boost | 0.7 – 5.5 V | Synchronous boost | TI TPS63802 family | ~100 nA | CR2032 / 1S LiIon — battery-only SKUs |
| **B** — Wide buck-boost | 4.5 – 30 V | Buck-boost | LT8390, LM5176, similar | ~25–50 µA | USB / DC barrel / 2S–6S LiIon — mains or multi-cell SKUs |

Both footprints sit on the same PCB tracks; the unpopulated footprint's
pads are simply left bare on the assembly. Output of either regulator
ties into the same `3V3_BUS` net. This way **the carrier board PCB is a single
SKU at fab time**; the battery vs. mains distinction is solely a
stuffing-list choice.

`VIN_RAW` accepts up to **24 V nominal** input. This range covers:

| Source class | Voltage range | Suitable footprint |
|---|---|---|
| CR2032 coin cell | 2.0 – 3.0 V | A |
| 1S LiIon (single cell) | 3.0 – 4.2 V | A |
| USB-C VBUS | 5 V | A or B (both work; B for headroom) |
| 2S LiIon | 6.0 – 8.4 V | B |
| 3S LiIon | 9.0 – 12.6 V | B |
| 4S LiIon | 12.0 – 16.8 V | B |
| 5S LiIon | 15.0 – 21.0 V | B |
| 6S LiIon | 18.0 – 25.2 V * | B |
| DC barrel jack | 5 – 24 V | A (≤5 V) or B (>5 V) |

\* 6S full-charge voltage is 25.2 V, marginally above the 24 V nominal
spec. The wide buck-boost footprint must tolerate brief transients up
to 30 V (and we should pick parts accordingly), but extensions plugged
into `VIN_RAW` should be designed assuming 24 V worst-case typical.

Only one input option is populated per assembled carrier board SKU
(through-hole CR2032 socket, USB-C connector, or DC barrel jack —
mutually exclusive). The selected input feeds `VIN_RAW` through an
ideal-OR diode controller (or a simple Schottky OR for cost-down
variants) so that, if multiple inputs were stuffed, the highest one
would win.

`VIN_RAW` is exposed on the extension connector alongside `3V3`. The
extension can draw from either rail, depending on what its sensors
need.

### 3.2 Rationale

Requirement #6 ("low-energy autonomous or high-consumption mains
configuration with the *same exact* main board") rules out any
fixed-topology single-regulator design. A practical analysis of
available silicon shows that **no single buck-boost part covers the
full 0.7 V → 24 V input range** with quiescent current low enough for
battery operation:

- Sub-5.5 V buck-boost parts (TPS63802 class) achieve ~100 nA Iq but
  cannot accept the multi-cell LiIon or DC-barrel inputs.
- Wide-input parts (LT8390, LM5176 class) accept up to 30 V or more
  but quiescent jumps to 25–50 µA — acceptable for mains-powered
  builds, fatal for CR2032 battery builds.

The two-footprint, one-PCB approach gives the best of both: shared
PCB tooling and BOM management, while letting each SKU pick the
regulator silicon optimal for its power-source class.

### 3.3 Extension power contract

| Pin | Voltage | Max current draw by extension | Notes |
|---|---|---|---|
| `3V3` | 3.3 V ±2 % | 100 mA continuous, 300 mA peak | Regulated. Quiet enough for ADC reference. |
| `VIN_RAW` | 2.0–24 V (depends on carrier board input population) | 500 mA continuous | Unregulated. Extension owns its further regulation if it needs 5 V or other. |
| `5V` (mikroBUS pin 10) | **Tied to VIN_RAW**, NOT a separate 5 V rail | — | Standard mikroBUS extensions expecting clean 5 V will work *only* on USB-powered carrier board assemblies. Documented per-extension. |

The "5V" label on mikroBUS pin 10 is preserved for ecosystem
compatibility with off-the-shelf Click boards designed for USB-host
carrier boards, but documented to track VIN_RAW on homesensors carrier boards.

### 3.4 Extension power management

**The extension owns its power management.** The carrier board does not gate the
extension's supply at any sub-second granularity — there is no
load-switch on the `3V3` or `VIN_RAW` rails leaving the carrier board. If an
extension needs to power down sub-components between samples (e.g.
the analog front-end of a battery-budget temperature sensor), the
extension provides its own load switch driven by its own logic.

**Compile-time validation:** the firmware framework knows the
configured power class of the carrier board (`battery_LP`, `battery_LiIon`,
`mains_USB`, `mains_DC`) and the worst-case current draw declared by
each extension driver. Combinations that violate the budget — e.g.
configuring a video-stream extension on a CR2032 carrier board — cause a
compile-time error, not a silent runtime failure. The error is
explicit ("extension `video_stream` declares 80 mA continuous, exceeds
`battery_LP` budget of 0.5 mA").

This pushes the "is this combination safe?" question to build time,
where it can be answered statically, rather than to a runtime check
that the deployed device must perform on every boot.

## 4. ADR-2: Physical connector

### 4.1 Decision

The carrier board exposes a **mikroBUS-compatible** female socket
(2×8 pin, 2.54 mm pitch). Extensions present a 2×8 male header in the
same physical configuration. Pinout follows the mikroBUS Standard
v2.00 exactly:

```
   ┌─────────────────────┐
   │  1  AN          PWM 16 │      ← top of board / mounting side
   │  2  RST         INT 15 │
   │  3  CS          RX  14 │
   │  4  SCK         TX  13 │
   │  5  MISO        SCL 12 │
   │  6  MOSI        SDA 11 │
   │  7  3V3         5V  10 │
   │  8  GND         GND  9 │
   └─────────────────────┘
```

For size-constrained deployments (compact door-magnet retrofits where
the 28×25 mm Click-board footprint is too large), a **secondary
compact form factor** is defined: a 1×8 JST-SH 1 mm-pitch connector
carrying a strict subset of mikroBUS pins, in this order:

| JST-SH pin | mikroBUS equivalent | Function |
|---|---|---|
| 1 | 7 | 3V3 |
| 2 | 8 | GND |
| 3 | 11 | SDA |
| 4 | 12 | SCL |
| 5 | 15 | INT |
| 6 | 2 | RST |
| 7 | 1 | AN (ID resistor) |
| 8 | 10 | VIN_RAW (alias of mikroBUS 5V) |

The compact form factor loses the SPI bus and the UART, so it cannot
host smart extensions with high-bandwidth data (audio, video). It is
intended for I2C-only or single-GPIO-only extensions.

**Firmware does not distinguish between the two physical connectors** —
the same pin numbers (`SDA`, `INT`, etc.) map to the same MCU pins
on both. Only the mechanical bring-out differs. A carrier board SKU exposes
exactly one of the two connectors; both are not co-populated.

### 4.2 Rationale

- mikroBUS solves the problem completely. Its 16-pin layout exposes
  all four common bus types (I2C, SPI, UART, analog) simultaneously
  plus two interrupt lines and two power rails. Different extensions
  populate different pins without renegotiating the contract.
- The ecosystem benefit is significant: ~1500 existing MikroElektronika
  Click boards will physically plug into homesensors carrier board sockets and
  often work with minor firmware support, even outside our intentional
  extension catalogue. This dramatically lowers the bar for sensor
  experimentation.
- The compact JST-SH form factor is offered only because requirement #1
  explicitly lists "door/window contact" as a primary use case, and
  the mikroBUS socket physical envelope would dominate that
  enclosure's interior. JST-SH preserves mikroBUS pin semantics where
  possible.
- We deliberately **do not** adopt Qwiic / STEMMA QT despite their
  popularity, because they are I2C-only by spec. Requirement #1's
  audio/video streaming entry needs SPI or UART, which Qwiic cannot
  carry.

### 4.3 Pin reassignment rules

Within the mikroBUS pinout, *which* pins a given extension actually
uses is its own concern. The firmware framework treats each pin role
(`SDA`, `MISO`, `INT`, etc.) as an opaque resource that an extension
driver requests via the framework API. Specific examples:

| Extension type | Pins used | Pins floating |
|---|---|---|
| Door contact (Hall + I2C) | 3V3, GND, SDA, SCL, INT, AN | RST, CS, SCK, MISO, MOSI, RX, TX, PWM |
| Temperature/humidity (I2C) | 3V3, GND, SDA, SCL, AN | rest |
| Vibration FFT (companion MCU over SPI) | 3V3, VIN, GND, CS, SCK, MISO, MOSI, INT, AN, RST | SDA, SCL, RX, TX, PWM |
| Audio stream (companion MCU over UART) | VIN, GND, RX, TX, INT, AN | SDA, SCL, SPI bus |

Unused pins on an extension MUST be left high-impedance (no internal
pull-up/down). Floating GPIO from one extension MUST NOT interfere
with another extension type using those pins.

### 4.4 Mechanical orientation

Per mikroBUS spec §3.3: the carrier board mounts the **female** socket
on the top side; the extension carries the **male** header on the
bottom side; extensions mount with components facing up (away from
carrier board).

For low-profile assemblies, surface-mount mezzanine sockets such as
the Würth Elektronik **WR-MM** family (e.g. 692121710002 and related
part numbers) are acceptable substitutes for through-hole. The
WR-MM family is mikroBUS-compatible (2.54 mm pitch, 2×8 layout) and
available in multiple mating-height variants. The specific stacking
height, mounting type (THT vs SMT), and current rating must be
verified against the manufacturer's datasheet before BOM-locking a
particular variant — Würth lists the data sheet on
[`https://www.we-online.com/en/components`](https://www.we-online.com/en/components)
under part-number search.

Functionally-equivalent alternatives include Samtec SSW-108-01-T-D
(standard through-hole, ~8 mm mated) and Mill-Max 851-43-016-10-001000
(SMT, low profile).

## 5. ADR-3: Identification

### 5.1 Decision

Each extension carries **a single resistor** (1 %) from the AN pin to
GND. The carrier board has a fixed pull-up resistor from AN to 3V3.
On boot, the carrier board ADC reads the AN voltage and decodes the extension
type from a small lookup table.

```
        3V3
         │
       [R_pu] ← on carrier board, fixed 33 kΩ
         │
         ├──── AN pin ──── ADC input
         │
       [R_id] ← on extension, one of 16 documented values
         │
        GND
```

The lookup table is published as part of this spec (see §5.4) and is
the binding source of truth for `R_id` values.

### 5.2 Rationale

The combination "default static config + cheap auto-discovery" is the
right MVP. Reasons:

- Static config alone fails if the user plugs in the wrong extension —
  the carrier board silently malfunctions, support burden is high.
- Full bidirectional protocol (I2C EEPROM with descriptor, GATT-style
  capability exchange, etc.) is an order of magnitude more firmware
  effort and adds ~$0.10 BOM per extension for the EEPROM IC.
- A single resistor adds ~$0.005 BOM, zero firmware overhead beyond a
  single ADC sample at init, and solves the actual user-facing
  failure mode ("did I plug in the right one?").

The mechanism also generalises: identical extensions can be
manufactured with different ID resistors to distinguish hardware
revisions, region variants (24V vs 240V mains for power-monitoring
extensions, e.g.), or marketing SKUs without firmware fork.

### 5.3 Boot-time behaviour

Pseudocode (canonical reference in firmware framework; expressed here
in C, but the framework reference implementation uses C++ per ADR-5):

```c
extension_id_t extension_detect(void)
{
    uint16_t adc_mv = adc_read_blocking(ADC_CHANNEL_AN_PIN);
    for (int i = 0; i < N_EXTENSION_TYPES; ++i) {
        if (adc_mv >= extension_table[i].mv_min &&
            adc_mv <= extension_table[i].mv_max) {
            return extension_table[i].id;
        }
    }
    return EXTENSION_UNKNOWN;
}

void boot(void) {
    extension_id_t id = extension_detect();
    if (id == EXTENSION_UNKNOWN) {
        led_set_pattern(LED_RED_SLOW_BLINK);
        // Continue running with no sensor — carrier board reports
        // "unknown extension" upstream so HA shows a clear error.
    } else if (id != configured_extension_id) {
        led_set_pattern(LED_RED_FAST_BLINK);
        // Wrong extension for this firmware build.
    } else {
        sensor_driver_register(extension_driver_for(id));
    }
}
```

### 5.4 ID-resistor lookup table (v0.1)

Bands chosen to give ≥100 mV margin between adjacent values at 12-bit
ADC resolution with 1 % tolerance resistors and 33 kΩ pull-up. Values
are 5 % E24 series for sourcing simplicity. Category abbreviations:
**D** = DISCRETE, **B** = BATCH, **S** = STREAMING (see §7.4).

| ID slot | `R_id` (Ω) | Mid-band V (V) | Category | Reserved for |
|---|---|---|---|---|
| 0 | open (no R) | ~3.30 | — | UNKNOWN / unplugged |
| 1 | 100 k | 2.49 | D | Door/window (Hall, I2C) — *current sensorkit door* |
| 2 | 56 k | 2.07 | D | Magnet field (3-axis) |
| 3 | 33 k | 1.65 | D | PIR motion |
| 4 | 22 k | 1.32 | B | Temperature/humidity (I2C) |
| 5 | 15 k | 1.03 | D | Leak (resistive probe) |
| 6 | 10 k | 0.77 | D | Fire/smoke |
| 7 | 6.8 k | 0.56 | S | Audio stream (companion MCU, UART) |
| 8 | 4.7 k | 0.41 | S | Video stream (companion MCU, SPI) |
| 9 | 3.3 k | 0.30 | B | Vibration / FFT summary (companion MCU, SPI) |
| 10 | 2.2 k | 0.21 | B | Power monitoring (voltage/current ADC) |
| 11 | 1.5 k | 0.14 | — | reserved |
| 12 | 1.0 k | 0.097 | — | reserved |
| 13 | 680 | 0.067 | — | reserved |
| 14 | 470 | 0.046 | — | reserved |
| 15 | 0 (short) | 0.000 | — | TEST / development |

Slots 11–14 are unassigned and available for future extension types.
Slot 15 (short to GND) is reserved for development boards and bench
test fixtures.

### 5.5 Future upgrade path

When/if extensions become genuinely heterogeneous (multiple
calibrations per type, factory-stored serial numbers, etc.), an I2C
EEPROM at a documented address (`0x50` by convention, matching the
SparkFun / Adafruit STEMMA detect convention) may be added. The
resistor remains as the *fallback* identification (no I2C handshake
required), with the EEPROM holding the richer data.

## 6. ADR-4: Wake-up and reset signalling

### 6.1 Decision

The mikroBUS `INT` pin (pin 15) is the canonical **extension → carrier board
wake-up channel**, edge-triggered (configurable rising or falling per
extension). The carrier board configures EXTI on this pin to wake from STOP mode
on the relevant edge.

The mikroBUS `RST` pin (pin 2) is the canonical **carrier board → extension
reset signal**, active-low. The carrier board drives this pin low for ≥10 ms to
reset the extension's companion MCU (if any) or to reset the
extension's analog front-end on boot.

`RST` is NOT a wake-up channel for the extension. Extensions with
companion MCUs that need their own power-down sleep between events
should implement that sleep internally (consistent with §3.4 —
extension owns its power management).

### 6.2 Rationale

- Most homesensors use cases are event-driven from the sensor side:
  door opens → wake carrier board → emit event. `INT` is exactly the right
  pin for this in the mikroBUS standard.
- Bidirectional wake-up (carrier board waking extension) is rarely needed
  because extensions with companion MCUs run in their own LPM and
  wake themselves on their own internal sensors. They communicate
  with the carrier board only when they have data to push.

### 6.3 EXTI configuration

The carrier board firmware framework MUST register the INT pin as an
EXTI-capable wake-up source whenever an extension driver is loaded.
Drivers declare their preferred edge as part of their `SensorObject`
registration (see §7.2).

## 7. Firmware framework

### 7.1 Layer cake

```
   ┌────────────────────────────────────────────────────────────┐
   │  Application                                                │
   │  - main loop, LPM gate, system housekeeping                 │
   ├────────────────────────────────────────────────────────────┤
   │  Upstream protocol adapter                                  │
   │  - routes events per category to BTHome / MQTT / Zigbee     │
   │  - Tier 1.5 ACK overlay for BTHome (current homesensors)    │
   ├────────────────────────────────────────────────────────────┤
   │  Sensor framework — `SensorObject` interface                │
   │  - generic lifecycle: init / poll / on_int / read_frame /   │
   │    publish / shutdown                                       │
   │  - generic data types: Event, Sample, Frame                 │
   │  - extension-driver registration table                      │
   ├────────────────────────────────────────────────────────────┤
   │  Extension drivers (one per ID slot)                        │
   │  - door_contact, motion_pir, temp_humidity, audio_stream … │
   ├────────────────────────────────────────────────────────────┤
   │  Hardware abstraction layer (HAL)                           │
   │  - mikroBUS pin services: I2C, SPI, UART, GPIO, ADC, EXTI   │
   │  - PHY-specific: BLE / WiFi / Zigbee / LoRa radio drivers   │
   ├────────────────────────────────────────────────────────────┤
   │  Vendor HAL (ST CubeWB, ESP-IDF, Silabs Gecko SDK, ...)     │
   └────────────────────────────────────────────────────────────┘
```

Anything above the Sensor framework line is reusable across PHY
variants. Anything below the Sensor framework line is PHY-specific.
The framework itself is the load-bearing abstraction.

### 7.2 SensorObject interface

A SensorObject is what an extension driver implements. The interface
is deliberately small — enough to express the lifecycle of a sensor
event but not so large that adding a new sensor type requires
understanding the whole framework.

The protocol-level interface is shown here as a C struct for spec
clarity; the v0.2 reference implementation expresses this as a C++
abstract base class with virtual methods, per ADR-5 (§8).

```c
typedef struct sensor_object_s {
    /* Identity (filled by framework from ID-resistor lookup) */
    extension_id_t       id;
    const char          *name;
    sensor_category_t    category;     /* DISCRETE | BATCH | STREAMING */

    /* Lifecycle */
    void   (*init)(struct sensor_object_s *self);
    void   (*poll)(struct sensor_object_s *self);     /* may be NULL */
    void   (*on_int)(struct sensor_object_s *self);   /* may be NULL */
    void   (*shutdown)(struct sensor_object_s *self);

    /* Discrete + Batch publish path — driver calls into framework */
    void   (*publish)(struct sensor_object_s *self, sensor_event_t *evt);

    /* Streaming + Batch frame-fetch path — framework calls into driver.
     * NULL for DISCRETE-only drivers. Returns bytes written, 0 if no
     * frame ready. Typically initiates a DMA transfer over SPI/UART. */
    size_t (*read_frame)(struct sensor_object_s *self,
                         uint8_t *buf, size_t buf_max);

    /* Maximum frame size this sensor can produce. Framework allocates
     * one buffer of this size per active stream. 0 for DISCRETE. */
    size_t  max_frame_bytes;

    /* INT-pin edge preference. */
    int_edge_t int_edge;   /* EXTI_RISING | EXTI_FALLING | EXTI_BOTH */

    /* Driver-private state — opaque to framework. */
    void   *priv;
} sensor_object_t;
```

Three lifecycle modes are supported, mapping onto the three sensor
categories (see §7.4):

1. **Polled** (DISCRETE samples, BATCH summaries): framework calls
   `poll()` every N ms. Driver decides what to publish.
2. **Interrupt-event** (DISCRETE events): framework calls `on_int()`
   when the INT pin fires. Driver reads the sensor, emits an event.
3. **Interrupt-frame** (BATCH frames, STREAMING): framework calls
   `on_int()` to *acknowledge a frame is ready*, then later calls
   `read_frame()` from the framework's stream thread to fetch it.
   Drives the data flow specified in §7.3.

A driver may use multiple modes (e.g. a door sensor uses
interrupt-event for door state + polled for battery percent every
minute).

### 7.3 Streaming and batched data flow

For extensions producing frame-structured data (BATCH and STREAMING
categories), the flow is:

```
   1. Companion MCU on extension prepares a frame in its local buffer.
   2. Companion MCU asserts INT pin.
   3. Framework EXTI handler dispatches driver's on_int() ── ISR context
      └─ driver's on_int() flags "frame ready", returns immediately.
   4. Framework's stream thread (preemptible, lower priority) sees the
      flag, allocates a buffer of size sensor.max_frame_bytes from a
      pre-reserved pool, and calls driver's read_frame(buf, buf_max).
   5. read_frame() initiates the SPI or UART DMA transfer that pulls
      the frame from the companion MCU into the supplied buffer.
      Returns when DMA completes; returns the byte count written.
   6. Framework hands the buffer to the upstream protocol adapter via
      publish_frame(buf, len). The adapter chooses BTHome (for small
      BATCH frames on BLE) or MQTT (everything else) and emits.
   7. Framework returns the buffer to the pool.
```

This separation has three important properties:

- `on_int()` is cheap (ISR-time). LPM wake latency stays low.
- Buffer lifecycle is framework-owned (pool allocation, not malloc).
  Predictable for embedded RTOS / bare-metal.
- The upstream adapter is the only place that knows which transport
  to use. Drivers are PHY-agnostic.

For polled-streaming (rare — only when the companion MCU lacks an
INT line), the framework's polled tick simply calls `read_frame()`
directly with a NULL `on_int()`. Same downstream flow.

### 7.4 Sensor categories

Three categories cover the full range of payload sizes and rates:

| Category | Payload size | Rate | Examples | Buffer model |
|---|---|---|---|---|
| **DISCRETE** | 1–32 bytes | sporadic (event-driven) or slow-periodic (≤1 Hz) | Door, motion, leak, fire | Stack-local `sensor_event_t`. No frame buffer. |
| **BATCH** | 32–512 bytes | periodic (1–10 Hz typically) | Temperature/humidity summary, vibration FFT, power-monitor sample | Single pool buffer per stream, sized to `max_frame_bytes`. |
| **STREAMING** | 512 bytes – 16 kB | continuous (10 Hz – 100 kHz frame rate) | Audio PCM, low-res video, raw IMU stream | Ring of pool buffers, multi-frame in flight under flow control. |

Driver declares its category at registration time. The framework's
upstream adapter uses the category to choose transport (§9). The
carrier board's PHY may statically refuse certain categories at compile time
(e.g. BLE carrier board rejects STREAMING — see §9).

DISCRETE and BATCH can both publish via the BTHome transport on BLE
carrier boards. BATCH adverts are larger (a full advert per sample, up to the
31-byte legacy cap) and fire less often than the rate suggests — the
framework collects multiple values into one advert where possible.

## 8. ADR-5: Implementation language

### 8.1 Decision

The firmware framework, extension drivers, upstream-protocol adapters,
and application layer are written in **C++17**, using a conservative
embedded-C++ subset:

- **`-fno-exceptions`** — no exception unwinding overhead.
- **`-fno-rtti`** — no runtime type information; no `dynamic_cast`,
  no `typeid`.
- **`-fno-threadsafe-statics`** — function-local statics are
  initialised once at startup in controlled order, not under
  thread-safety locks.
- No standard library use of `<iostream>`, `<thread>`, dynamic
  `<memory>` (smart-pointers that allocate). Use **ETL** (Embedded
  Template Library, `https://www.etlcpp.com/`) for containers and
  fixed-size collections.
- No runtime heap allocation after `setup()` completes. All allocation
  is pool-based and capped at boot.
- `constexpr` used liberally for build-time computation (lookup
  tables, calibration constants, pin maps).
- Templates used for type-safe HAL wrappers (e.g. `Gpio<Port::B, 1>`)
  but kept finite — no recursive templates that explode object code.

Vendor HAL adapters (the thin layer that wraps STM32Cube, ESP-IDF,
Gecko SDK, etc.) remain in C, exposed to C++ via `extern "C"`
headers. The framework layer is the first C++ layer above them.

### 8.2 Rationale

The protocol-level SensorObject interface defined in §7.2 is, in
substance, a vtable. Writing it manually as a C struct-of-function-
pointers (as shown for spec clarity) doubles the boilerplate of every
new driver. Expressing it as a C++ class with virtual methods is
shorter, type-safe, and produces *equivalent* machine code at `-Os`.

The toolchain support is universal across our target MCU families:

| PHY MCU | Vendor SDK | C++ toolchain | C++17 support |
|---|---|---|---|
| STM32WB09 (BLE) | STM32CubeWB0 | `arm-none-eabi-g++` 14.x | ✅ Full |
| ESP32 (Wi-Fi) | ESP-IDF 5.x | `xtensa-esp32-elf-g++` | ✅ Full (IDF uses C++ internally) |
| EFR32MG24 (Zigbee) | Simplicity SDK | `arm-none-eabi-g++` | ✅ Full |
| STM32WLE5 (LoRa) | STM32CubeWL | `arm-none-eabi-g++` 14.x | ✅ Full |

No target on the roadmap is C-only. Should that change in the future
(e.g. cost-down to an AVR or PIC for a single-button extension), that
extension's local driver can stay in C and link against the C++
framework via `extern "C"` — interop is one-directional but
straightforward.

### 8.3 Coding-standard implications

- **Class hierarchy**: shallow. One abstract base (`SensorObject`),
  one concrete subclass per extension driver. No deeper hierarchies
  unless an extension explicitly shares partial implementation with
  another (rare; usually compose rather than inherit).
- **Memory**: every dynamic-sized resource has a documented maximum.
  Pool allocators with compile-time-fixed pool sizes preferred over
  any runtime allocator.
- **Error handling**: return values, not exceptions. Use
  `etl::expected<T, error_t>` or a similar Rust-style sum type for
  fallible operations.
- **Logging**: lightweight, format-string compile-time-checked
  (e.g. `fmt::format` or a similar header-only library) so that
  unused log statements compile away cleanly.

### 8.4 Migration of existing code

The current single-purpose `homesensors/firmware` is C. It works and
will not be rewritten as part of this modular framework — it stays
on the v0.x "monolithic firmware" track for the existing door sensor.

The new modular framework is a parallel codebase under a new firmware
repo (likely `homesensors/framework` or kept in `homesensors/firmware`
under a `framework/` subdirectory, decided at v1.0 cutover). The
existing door-sensor logic will be ported to a `door_contact` driver
class in the new framework as part of the first integration test.

## 9. Upstream protocol matrix

| Sensor category | Examples | Protocol over BLE carrier board | Protocol over Wi-Fi carrier board | Over Zigbee carrier board | Over LoRa carrier board |
|---|---|---|---|---|---|
| DISCRETE event | Door, motion, leak, fire | BTHome v2 advert + Tier 1.5 ACK | MQTT JSON or BTHome v2 | Zigbee cluster | LoRaWAN unconfirmed uplink |
| DISCRETE / BATCH sample (small) | Temp/humidity 1-byte readings | BTHome v2 advert | MQTT JSON | Zigbee cluster | LoRaWAN unconfirmed |
| BATCH frame (medium) | FFT 16-bin summary, power-meter sample, multi-channel temp | BTHome v2 advert (if ≤ 21 byte payload) or *unsupported* | MQTT binary | Zigbee binary frame cluster | LoRaWAN (rate-limited per regional duty cycle) |
| STREAMING | Audio PCM, video, raw IMU | *unsupported* — extension rejected at boot | MQTT binary frame | *unsupported* | *unsupported* |

The "*unsupported*" cells are enforced **statically at compile time**:
the BLE carrier board firmware build does not link the audio_stream /
video_stream drivers at all, so an extension declaring those ID slots
hits the EXTENSION_UNKNOWN branch at boot and signals an error.
Static refusal beats runtime detection — the deployed firmware can't
encounter a sensor it doesn't have driver code for.

Region-specific rate limiting for LoRa (EU 868 1 % duty cycle, US
915 dwell-time limits, etc.) is enforced by the LoRa upstream adapter,
not by the SensorObject driver. Drivers publish at their natural rate;
the adapter back-pressures or coalesces as needed and may drop frames
if the cap is reached. Drivers will see a return value from
`publish()` indicating drop count for monitoring.

## 10. Versioning and change control

- This spec starts at **v0.1 (DRAFT)**, currently advancing to
  DRAFT 0.2. No backward-compatibility guarantees while pre-1.0.
- The first hardware fab matching this spec will pin a `v1.0` tag.
  After v1.0:
  - Connector pinout MUST NOT change in a way that breaks existing
    extensions (mikroBUS is itself committed to a stable pinout).
  - ID-resistor lookup table (§5.4) MUST be append-only. Existing
    slots MUST NOT be reassigned.
  - SensorObject interface (§7.2) may have additive changes
    (new optional fields) within v1.x; removing fields requires v2.0.
- Breaking changes (renumbering ID slots, changing connector,
  re-defining wake-up direction) require a major version bump and
  carry an explicit migration note.

## 11. Open questions

Items resolved during the DRAFT 0.1 → 0.2 redline pass have been
moved into the body of the spec. Remaining open items:

1. **Buck-boost-B part selection.** Specific PN for footprint B
   (wide-input buck-boost) — bench-compare LT8390, LM5176,
   MAX17242 etc. against a real PHY-MCU + extension load profile
   for the worst-case 6S → 3.3 V scenario before BOM-locking.
2. **Compact form factor bench validation.** JST-SH 1×8 is
   specified (§4.1) but not yet bench-prototyped. Build one
   door-magnet extension on JST-SH alongside the mikroBUS variant
   before committing both form factors to fab.
3. **Multi-extension carrier** (v2 deferral). Capture any pinout
   decisions now that would foreclose a future 2-socket carrier
   board. Currently we don't see any, but worth one explicit review
   before v1.0 lock.
4. **Provisioning UX.** When a user plugs in an extension, how does
   HA learn what it is without firmware rebuild? The static-config
   default means firmware rebuild is required to support a new
   extension type. An OTA-flashable "extension table" in flash
   could decouple this — out of scope for v1.0 but flagged for v1.x.

## 12. Glossary

| Term | Meaning |
|---|---|
| Carrier board | The PCB carrying the PHY MCU, power input, and extension socket. One PCB per PHY family. |
| Extension | The PCB carrying one or more sensors (or a companion MCU + sensors). Plugs into the carrier board's socket. |
| PHY | The radio standard the carrier board speaks upstream: BLE / Wi-Fi / Zigbee / LoRa. |
| SensorObject | Firmware-side abstraction implemented by each extension driver. §7.2. |
| DISCRETE | Sensor category: small, sporadic, event-driven (door, leak). |
| BATCH | Sensor category: medium-size periodic frames (FFT summary, multi-channel sample). |
| STREAMING | Sensor category: continuous frame series at high rate (audio, video). |
| Static configuration | Firmware build is locked to a specific extension type at compile time; the resistor ID is a defence-in-depth check. |
| ETL | Embedded Template Library (`www.etlcpp.com`) — header-only embedded-friendly STL replacement used by the framework. |
