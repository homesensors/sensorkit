# Modular architecture specification

> **Status: DRAFT 0.1.** Proposed architecture, not yet binding.
> Intended to be redlined and refined into a `modular-spec-v1.0` tag
> before any hardware committed to fab follows it. All design decisions
> below are subject to revision based on bring-up evidence.

This document specifies the modular evolution of `sensorkit` from a
single fixed-purpose door sensor into a kit-style platform supporting
heterogeneous sensors over a small number of motherboard variants
(BLE / Wi-Fi / Zigbee / LoRa). It defines the hardware contract
(connector, power, signalling) between *motherboard* and *extension*,
and the firmware framework that makes a single source tree support
many sensor types.

The companion documents are
[`architecture.md`](architecture.md) (current single-board design) and
[`ble-protocol.md`](ble-protocol.md) (the BLE upstream protocol).
Cross-references between them are by stable section anchors.

## 1. Scope

### 1.1 In scope

- A **motherboard (m/b)** PCB family, one PCB per PHY (BLE,
  Wi-Fi, Zigbee, LoRa). All m/b variants share the same physical
  extension connector, the same firmware framework, and the same
  upstream protocol logic.
- An **extension** PCB family, one per sensor type (or sensor
  cluster — multiple sensors on one extension is allowed).
  Extensions are PHY-agnostic — the same door-contact extension works
  on a BLE m/b or a Zigbee m/b without modification.
- A **firmware framework** that abstracts sensor-specific drivers from
  PHY-specific stacks, so a new sensor type only requires a new driver
  + a one-line registration, not a full firmware fork.
- An **upstream protocol matrix** that routes discrete events
  (door, motion, temperature) onto BTHome-like channels and streaming
  data (audio, video, vibration FFT) onto MQTT-like channels.

### 1.2 Out of scope

- **Programmable interconnect** between MCU pins and extension pins
  via on-board mux/switch. The connector pinout is fixed and pins
  retain their mikroBUS roles across all m/b variants. Different
  extensions plug into the same pins; pin function is statically
  bound to extension type via firmware config (see §5 ADR-3).
- **Code generation tooling.** Firmware is hand-written C with
  framework support; no GUI / IDE plugin generates the firmware for a
  given assembly.
- **Multiple extensions per motherboard.** Each m/b carries a single
  extension socket in v1.0. A "carrier board" that fans one m/b
  connector into multiple extension sockets is a possible v2 addition
  but not specified here.
- **Self-describing extensions over a service protocol.** Discovery is
  a single ADC reading (§5 ADR-3), not an I2C handshake or EEPROM
  read. Bidirectional service-channel protocols are a possible v2
  upgrade.

### 1.3 Design principles

1. **Wireless first.** Wired uplink (USB/Ethernet) is supported only
   for stationary, mains-powered m/b configurations as a secondary use
   case. The BLE m/b is the canonical reference design.
2. **Single firmware tree.** All PHY variants and all sensor types
   build from one Git repository. PHY-specific code lives below a HAL
   line; sensor-specific code lives above a SensorObject interface
   line; the bulk of the framework lives between.
3. **Static configuration is the default.** Dynamic discovery exists
   only to detect *wrong* extensions (red-LED behaviour), not to
   support truly heterogeneous deployments without firmware rebuild.
4. **Reuse established standards.** mikroBUS for connector,
   BTHome v2 + MQTT for upstream protocols, JST-MX / JST-SH for
   compact form factors. No invented connectors, no invented wire
   formats, no proprietary handshakes unless absolutely necessary.

## 2. System block diagram

```
   ┌─────────────────────────────────────────────────────────────┐
   │                     MOTHERBOARD                              │
   │                                                              │
   │   Power input                                                │
   │   options:                                                   │
   │   - CR2032 (DNP) ──┐                                         │
   │   - USB-C 5V  (DNP)├─▶ wide-input ──▶ 3V3_BUS                │
   │   - DC 5-24V  (DNP)│   buck-boost     ▲                      │
   │                    │   regulator      │                      │
   │            VIN_RAW ┴──────────────────┼────┐                 │
   │                                       │    │                 │
   │   ┌───────────────────┐  ┌────────────┴───┐│                 │
   │   │  PHY MCU          │  │ Status LED     ││                 │
   │   │  - STM32WB09 BLE  │  │ (heartbeat,    ││                 │
   │   │  - ESP32 WiFi     │  │  state, error) ││                 │
   │   │  - EFR32MG24 Zig  │  └────────────────┘│                 │
   │   │  - STM32WLE5 LoRa │                    │                 │
   │   │                   │                    │                 │
   │   │  GPIO/I2C/SPI ────┤◀───── EXTENSION CONNECTOR (mikroBUS)│
   │   │                   │       3V3 │ VIN_RAW │ GND │ INT │ … │
   │   │  ADC ────────────►│         AN│ RST│ CS│ SCK│MISO│ MOSI│
   │   │  (ID-resistor)    │          │ SDA│SCL│ TX │ RX │ PWM │ │
   │   └───────────────────┘                                     │
   └─────────────────────────────────────────────────────────────┘
                              │
                              │ (mikroBUS 16-pin)
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │                     EXTENSION (one of N)                     │
   │                                                              │
   │   Power: 3V3 and/or VIN_RAW from m/b                         │
   │                                                              │
   │   ┌──────────────────────────────────────────────┐           │
   │   │ Simple extension (door, magnet, temperature):│           │
   │   │   sensor IC → SDA/SCL or GPIO → connector    │           │
   │   │   ID resistor (1) → AN pin                   │           │
   │   └──────────────────────────────────────────────┘           │
   │                       OR                                     │
   │   ┌──────────────────────────────────────────────┐           │
   │   │ Smart extension (audio, video, FFT, NN):     │           │
   │   │   companion MCU + multiple sensors           │           │
   │   │   companion MCU ↔ m/b over SPI or UART       │           │
   │   │   edge processing on companion MCU           │           │
   │   │   m/b only sees pre-processed data           │           │
   │   └──────────────────────────────────────────────┘           │
   └─────────────────────────────────────────────────────────────┘
```

## 3. ADR-1: Power topology

### 3.1 Decision

The motherboard uses a **wide-input buck-boost switching regulator**
(e.g. TI TPS63802 family, 0.7–5.5 V input → fixed 3.3 V output, ~100 nA
quiescent). The same regulator and same PCB serve both battery-powered
and mains-powered assemblies via DNP-controlled population of the three
input options:

| Input | Voltage range | Use case | Populated for |
|---|---|---|---|
| CR2032 socket | ~2.0–3.0 V | Battery, autonomous | Door, window, magnetic, leak |
| USB-C connector | 5 V | Mains, mounted | Audio, video, persistent monitors |
| DC barrel jack | 5–24 V | Industrial / vehicle | Voltage/current meters, fire panels |

Only one input is populated per assembled motherboard SKU. The
regulator selects automatically because all three feed `VIN_RAW`
through an ideal-OR diode controller (or a simple Schottky OR for
cost-down variants).

`VIN_RAW` is exposed on the extension connector alongside `3V3`. The
extension can draw from either rail, depending on what its sensors
need.

### 3.2 Rationale

Requirement #6 ("low-energy autonomous or high-consumption mains
configuration with the *same exact* main board") rules out fixed-
topology LDO designs. A wide-input switching regulator with ~100 nA
quiescent is the only architecture that hits both extremes from one
silicon part:

- Battery side: ~2.4 V cell input → 3.3 V output. Boost mode. LDO
  cannot drop "up", so an LDO-only design would force a per-variant
  m/b SKU.
- Mains side: 5–24 V → 3.3 V. Buck mode. An LDO would waste up to
  90 % as heat at high input.

The cost is ~30–50 µA worst-case quiescent vs. ~1.5 µA for an LDO,
which is a worse battery-life number for battery variants but still
yields a ~1-year CR2032 lifetime with current homesensors firmware
power budget. Tier B LPM (full STOP between events) recovers most of
that gap.

### 3.3 Extension power contract

| Pin | Voltage | Max current draw by extension | Notes |
|---|---|---|---|
| `3V3` | 3.3 V ±2 % | 100 mA continuous, 300 mA peak | Regulated. Quiet enough for ADC reference. |
| `VIN_RAW` | 2.0–24 V (depends on m/b input population) | 500 mA continuous | Unregulated. Extension owns its further regulation if it needs 5 V or other. |
| `5V` (mikroBUS pin 10) | **Tied to VIN_RAW**, NOT a separate 5 V rail | — | Standard mikroBUS extensions expecting clean 5 V will work *only* on USB-powered m/b assemblies. Documented per-extension. |

The "5V" label on mikroBUS pin 10 is preserved for ecosystem
compatibility with off-the-shelf Click boards designed for USB-host
m/bs, but documented to track VIN_RAW on homesensors m/bs.

### 3.4 Open questions for v1.0

- **Load switching on the extension supply.** Should the m/b be able
  to power the extension down between samples (for ultra-low-power
  configurations with infrequent polling)? Adds one load-switch IC
  + one GPIO. Decide once we have a worst-case extension power
  budget.
- **Buck-boost vs. boost-only at battery side.** A boost-only part
  is cheaper and lower-quiescent but can't accept >3.3 V input on
  the battery rail. Acceptable if CR2032 is the only battery option.

## 4. ADR-2: Physical connector

### 4.1 Decision

The motherboard carries a **mikroBUS-compatible** female socket
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
on both. Only the mechanical bring-out differs. A m/b SKU exposes
exactly one of the two connectors; both are not co-populated.

### 4.2 Rationale

- mikroBUS solves the problem completely. Its 16-pin layout exposes
  all four common bus types (I2C, SPI, UART, analog) simultaneously
  plus two interrupt lines and two power rails. Different extensions
  populate different pins without renegotiating the contract.
- The ecosystem benefit is significant: ~1500 existing MikroElektronika
  Click boards will physically plug into homesensors m/b sockets and
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

Per mikroBUS spec §3.3: the motherboard carries the **female** socket
on the top side; the extension carries the **male** header on the
bottom side; extensions mount with components facing up (away from
m/b). For low-profile assemblies, surface-mount sockets such as Würth
692121710002 (1.8 mm seated height) are acceptable substitutes for
through-hole.

## 5. ADR-3: Identification

### 5.1 Decision

Each extension carries **a single resistor** (1 %) from the AN pin to
GND. The motherboard carries a fixed pull-up resistor from AN to 3V3.
On boot, the m/b ADC reads the AN voltage and decodes the extension
type from a small lookup table.

```
        3V3
         │
       [R_pu] ← on motherboard, fixed 33 kΩ
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
  the m/b silently malfunctions, support burden is high.
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

Pseudocode (canonical reference in firmware framework):

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
        // Continue running with no sensor — m/b reports
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
are 5 % E24 series for sourcing simplicity.

| ID slot | `R_id` (Ω) | Mid-band V (V) | Reserved for |
|---|---|---|---|
| 0 | open (no R) | ~3.30 | UNKNOWN / unplugged |
| 1 | 100 k | 2.49 | Door/window (Hall, I2C) — *current sensorkit door* |
| 2 | 56 k | 2.07 | Magnet field (3-axis) |
| 3 | 33 k | 1.65 | PIR motion |
| 4 | 22 k | 1.32 | Temperature/humidity (I2C) |
| 5 | 15 k | 1.03 | Leak (resistive probe) |
| 6 | 10 k | 0.77 | Fire/smoke |
| 7 | 6.8 k | 0.56 | Audio stream (companion MCU, UART) |
| 8 | 4.7 k | 0.41 | Video stream (companion MCU, SPI) |
| 9 | 3.3 k | 0.30 | Vibration / FFT (companion MCU, SPI) |
| 10 | 2.2 k | 0.21 | Power monitoring (voltage/current ADC) |
| 11 | 1.5 k | 0.14 | reserved |
| 12 | 1.0 k | 0.097 | reserved |
| 13 | 680 | 0.067 | reserved |
| 14 | 470 | 0.046 | reserved |
| 15 | 0 (short) | 0.000 | TEST / development |

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

The mikroBUS `INT` pin (pin 15) is the canonical **extension → m/b
wake-up channel**, edge-triggered (configurable rising or falling per
extension). The m/b configures EXTI on this pin to wake from STOP mode
on the relevant edge.

The mikroBUS `RST` pin (pin 2) is the canonical **m/b → extension
reset signal**, active-low. The m/b drives this pin low for ≥10 ms to
reset the extension's companion MCU (if any) or to reset the
extension's analog front-end on boot.

`RST` is NOT a wake-up channel for the extension. Extensions with
companion MCUs that need to be powered-up-then-down by the m/b should
use the load-switch mechanism flagged as a v1.0 open question in §3.4.

### 6.2 Rationale

- Most homesensors use cases are event-driven from the sensor side:
  door opens → wake m/b → emit event. `INT` is exactly the right
  pin for this in the mikroBUS standard.
- Bidirectional wake-up (m/b waking extension) is rarely needed
  because extensions with companion MCUs run in their own LPM and
  wake themselves on their own internal sensors. They communicate
  with the m/b only when they have data to push.

### 6.3 EXTI configuration

The m/b firmware framework MUST register the INT pin as an
EXTI-capable wake-up source whenever an extension driver is loaded.
Drivers declare their preferred edge:

```c
typedef struct {
    extension_id_t id;
    int_edge_t int_edge;  // EXTI_RISING, EXTI_FALLING, EXTI_BOTH
    sensor_init_fn init;
    sensor_event_fn on_event;
    /* ... */
} sensor_driver_t;
```

## 7. Firmware framework

### 7.1 Layer cake

```
   ┌────────────────────────────────────────────────────────────┐
   │  Application                                                │
   │  - main loop, LPM gate, system housekeeping                 │
   ├────────────────────────────────────────────────────────────┤
   │  Upstream protocol adapter                                  │
   │  - routes events to BTHome (discrete) or MQTT (streaming)   │
   │  - Tier 1.5 ACK overlay for BTHome (current homesensors)    │
   ├────────────────────────────────────────────────────────────┤
   │  Sensor framework — `SensorObject` interface                │
   │  - generic lifecycle: register / init / poll / on_int / shutdown
   │  - generic data types: Event, Sample, Stream                │
   │  - extension-driver registration table                      │
   ├────────────────────────────────────────────────────────────┤
   │  Extension drivers (one per ID slot)                        │
   │  - door_contact.c, motion_pir.c, temp_humidity.c, ...       │
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

```c
typedef struct sensor_object_s {
    /* Identity (filled by framework from ID-resistor lookup) */
    extension_id_t  id;
    const char     *name;

    /* Lifecycle */
    void   (*init)(struct sensor_object_s *self);
    void   (*poll)(struct sensor_object_s *self);     /* may be NULL */
    void   (*on_int)(struct sensor_object_s *self);   /* may be NULL */
    void   (*shutdown)(struct sensor_object_s *self);

    /* Upstream presentation */
    sensor_category_t  category;  /* DISCRETE or STREAMING */
    void  (*publish)(struct sensor_object_s *self, sensor_event_t *evt);

    /* Driver-private state — opaque to framework */
    void   *priv;
} sensor_object_t;
```

Two lifecycle modes are supported:

1. **Polled mode**: framework calls `poll()` every N ms (configurable
   per driver). Driver decides whether to publish based on what it
   reads. Suitable for slow-changing sensors (temperature, leak).
2. **Interrupt mode**: framework calls `on_int()` when the INT pin
   fires. Driver reads sensor over I2C/SPI, decides whether to
   publish. Suitable for event-driven sensors (door, motion).

Both modes can co-exist within one driver (e.g. door sensor uses INT
for events + polls battery percent every minute).

### 7.3 Streaming extension support

For extensions with companion MCUs producing continuous data (audio,
video, vibration FFT):

- The companion MCU runs the sensor-specific DSP locally and presents
  a *summary* or a *frame* to the m/b over the high-speed bus
  (typically SPI, optionally UART for narrow-bandwidth).
- The m/b's SensorObject driver consumes the framed data and publishes
  it via the **streaming** path (MQTT or analogous protocol —
  PHY-specific).
- BTHome is NOT used for streaming data. BTHome's per-advert payload
  cap is too tight for continuous data.

The category split (`DISCRETE` vs `STREAMING`) in the SensorObject
interface lets the upstream protocol adapter route data to the right
channel automatically. The extension driver doesn't know or care
which PHY is below it.

## 8. Upstream protocol matrix

| Sensor category | Examples | Protocol over BLE m/b | Protocol over Wi-Fi/Zigbee m/b |
|---|---|---|---|
| DISCRETE event | Door, motion, leak | BTHome v2 advert + Tier 1.5 ACK | BTHome v2 / Zigbee cluster |
| DISCRETE sample (slow-changing) | Temp, humidity | BTHome v2 advert | MQTT JSON / Zigbee cluster |
| STREAMING | Audio, video, vibration FFT | *Not supported* (BLE bandwidth too low) | MQTT binary frame |

The BLE m/b SKU does **not** support streaming categories at all in
v1.0; an extension declaring `STREAMING` category boots into "wrong
PHY for this extension" error state with a clear LED + HA notification.
This is enforced statically at ID-resistor parse time — the BLE m/b's
firmware build does not even include drivers for streaming extension
IDs (slots 7–9 in §5.4).

The Wi-Fi m/b SKU supports both DISCRETE and STREAMING categories. The
Zigbee m/b SKU supports DISCRETE only (Zigbee bandwidth makes
streaming infeasible). LoRa m/b SKU supports DISCRETE only and only
the slow-changing subset (event-rate cap imposed by LoRa duty cycle
regulations).

## 9. Versioning and change control

- This spec starts at **v0.1 (DRAFT)**. No backward-compatibility
  guarantees while pre-1.0.
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

## 10. Open questions for v0.2 → v1.0

1. **Extension power gating.** Should the m/b have a load switch on
   the extension power rail to cut quiescent draw for ultra-low-power
   deployments? §3.4.
2. **Buck-boost part selection.** TPS63802 vs TPS63900 vs an
   alternative — pick after first PHY-MCU power budget is measured.
3. **Compact form factor confirmation.** JST-SH 1×8 is the proposal;
   bench-prototype one door-magnet extension on both mikroBUS and
   JST-SH before committing.
4. **Sensor category enum granularity.** `DISCRETE` vs `STREAMING`
   may need a third category for low-bandwidth periodic streams
   (e.g. once-per-second temperature) that don't fit either model
   cleanly. Bring-up will tell.
5. **Multi-extension carrier.** Defer to v2.0, but capture any
   pinout decisions now that would foreclose a future 2-socket
   carrier board.
6. **Provisioning UX.** When the user plugs in an extension, how does
   HA learn what it is without firmware rebuild? The static-config
   default means firmware rebuild is required. An OTA-flashable
   "extension table" stored in flash could decouple this — out of
   scope for v1.0 but flagged for v1.x.

## 11. Glossary

| Term | Meaning |
|---|---|
| Motherboard (m/b) | The PCB carrying the PHY MCU, power input, and extension socket. One PCB per PHY family. |
| Extension | The PCB carrying one or more sensors (or a companion MCU + sensors). Plugs into the m/b's socket. |
| PHY | The radio standard the m/b speaks upstream: BLE / Wi-Fi / Zigbee / LoRa. |
| SensorObject | Firmware-side abstraction implemented by each extension driver. §7.2. |
| Discrete event | A state-change observation (door opens, leak detected). Small payload, low rate. |
| Streaming | Continuous time-series or frame-series data. High bandwidth, requires SPI or UART link. |
| Static configuration | Firmware build is locked to a specific extension type at compile time; the resistor ID is a defence-in-depth check. |
