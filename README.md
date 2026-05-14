# HomeSensors — `sensorkit`

[![Docs License: CC BY-SA 4.0](https://img.shields.io/badge/Docs%20License-CC%20BY--SA%204.0-lightgrey.svg)](LICENSE)
[![Firmware: MIT](https://img.shields.io/badge/Firmware-MIT-yellow.svg)](https://github.com/homesensors/firmware/blob/main/LICENSE)
[![Hardware: CERN--OHL--P--2.0](https://img.shields.io/badge/Hardware-CERN--OHL--P--2.0-blue.svg)](https://github.com/homesensors/hardware/blob/main/LICENSE)
[![HA add-on: MIT](https://img.shields.io/badge/HA%20add--on-MIT-yellow.svg)](https://github.com/homesensors/homeassist/blob/main/LICENSE)

> Open-hardware, open-firmware DIY smart-home sensor kit. Pre-configured for
> Home Assistant. Native BTHome v2 over BLE today; Zigbee / LoRa / Wi-Fi
> variants planned. Goal: a workable smart home in minutes, without
> professional installation.

<!-- TODO: hero image of the soldered board + CR2032 + HA dashboard -->
<!-- ![sensorkit hero](docs/images/hero.jpg) -->

## What's in this repo

This is the **umbrella project**. The actual code, hardware, and HA
integration live in separate repos so each can be developed and consumed
independently:

| Repo | What | License |
|---|---|---|
| [`homesensors/firmware`](https://github.com/homesensors/firmware) | STM32WB09 firmware. BTHome v2 broadcaster + Tier 1.5 fault-tolerant ACK protocol. ~2-year CR2032 battery life. | MIT |
| [`homesensors/hardware`](https://github.com/homesensors/hardware) | KiCad project for the BLE door sensor (Rev A fabricated). Ready-to-fab Gerbers + BOM. | CERN-OHL-P-2.0 |
| [`homesensors/homeassist`](https://github.com/homesensors/homeassist) | Home Assistant add-on implementing the HA side of the ACK protocol (`door_ack_daemon`) + dev/test scripts. | MIT |
| [`homesensors/sensorkit`](https://github.com/homesensors/sensorkit) (this repo) | Project pitch, architecture, BLE protocol spec, build guide, issue triage. | CC-BY-SA 4.0 |

## Why another door sensor?

There are plenty of Zigbee door sensors on Aliexpress for $8. What `sensorkit`
adds:

- **No proprietary bridge.** BTHome v2 is a native HA protocol that any
  HA install with a Bluetooth adapter speaks out of the box. No Zigbee
  dongle, no proprietary hub, no cloud roundtrip.
- **Fault-tolerant delivery.** The Tier 1.5 ACK protocol guarantees event
  delivery even through hour-long HA / Wi-Fi / power outages. Events
  retransmit on exponential back-off (5 s → 30 min cap, ~12 h retention)
  and are timestamped with the original Hall-edge moment so HA's history
  shows the event at the correct time, not at the retransmit time. See
  [`docs/ble-protocol.md`](docs/ble-protocol.md) for the spec.
- **Open all the way down.** Schematic, PCB layout, BOM, firmware,
  HA integration — every layer is open and forkable.
- **Long battery life.** 2+ years on a single CR2032 in steady state.

## Quick start

1. **Build / buy a unit.** See [`homesensors/hardware`](https://github.com/homesensors/hardware) for the
   Gerbers + BOM (sourcable from JLCPCB / LCSC). If you'd prefer not to
   solder, see the *Buy* section below.
2. **Flash firmware.** STM32CubeIDE 2.1+ → see [`homesensors/firmware`](https://github.com/homesensors/firmware) README.
3. **Install the HA add-on.** Add this repo as a local add-on source in
   HA → Add-ons → ⋮ → Repositories → paste
   `https://github.com/homesensors/homeassist`. Start the
   `Door ACK Daemon` add-on. Done.

The BTHome integration auto-discovers the device on first event. The
add-on emits ACKs over BLE so the device retires delivered events from
its retransmit buffer.

## Architecture overview

```
   ┌────────────────────┐                  ┌──────────────────────┐
   │ Custom PCB / WB09  │                  │ HA OS on Pi 5 (or    │
   │ + Hall sensor      │── BTHome adv ──▶ │ any BT-capable host) │
   │ + CR2032           │   (UUID 0xFCD2)  │                      │
   │                    │                  │  ┌────────────────┐  │
   │  Broadcaster role: │                  │  │ BTHome native  │  │
   │   Hall edge →      │                  │  │ integration    │  │
   │     burst + push   │                  │  └──────┬─────────┘  │
   │     to AckBuf      │                  │         │            │
   │                    │                  │  ┌──────▼─────────┐  │
   │  Observer role:    │ ◀── ACK adv ─────│  │ door_ack_daemon│  │
   │   listen for ACK   │    (UUID 0xFCD3) │  │ (this repo,    │  │
   │   retire AckBuf    │                  │  │  add-on)       │  │
   │                    │                  │  └────────────────┘  │
   │  Retransmit:       │                  │                      │
   │   exp backoff      │                  │                      │
   │   ~12h retention   │                  │                      │
   └────────────────────┘                  └──────────────────────┘
```

See [`docs/architecture.md`](docs/architecture.md) for the long form.

## Roadmap

- **v0.x — door sensor (BLE / BTHome v2)** — current. PCB Rev A fabricated;
  Rev B in progress (folds in the VDDA_VCAP decoupling + battery-divider
  rework that's hand-soldered on the prototypes).
- **v1.x — kit form** — case design, pre-paired HA configuration, multi-unit
  branding.
- **Future PHYs** — Zigbee, LoRa, Wi-Fi variants planned for sensors where
  range or topology demands them. BTHome stays the default for short-range
  battery-powered.
- **Future sensors** — temperature/humidity, motion, leak, vibration. Same
  protocol design, same kit framing.

## Buy

<!-- TODO: link to Tindie listing when available -->
<!-- TODO: link to PCBWay Shared Project when available -->

Not yet — the project is in open-beta. If you want to be notified when
units / kits become available, ⭐ star this repo and watch for the
v0.1.0 release tag.

## Documentation index

- [`docs/architecture.md`](docs/architecture.md) — block diagram, data flow, power budget.
- [`docs/ble-protocol.md`](docs/ble-protocol.md) — wire format for BTHome adverts + Tier 1.5 ACK
  protocol; complete enough to write a compatible client from scratch.
- [`docs/build-guide.md`](docs/build-guide.md) — end-to-end build guide (KiCad → PCB fab →
  assembly → firmware flash → HA install). *Coming soon.*

## Licensing

| Component | License |
|---|---|
| Firmware (`homesensors/firmware`) | MIT |
| Hardware (`homesensors/hardware`) | CERN-OHL-P-2.0 (permissive open hardware) |
| HA add-on (`homesensors/homeassist`) | MIT |
| Docs in this repo | CC-BY-SA 4.0 |

See [`LICENSING.md`](LICENSING.md) for full text references.

## Contributing

Issues, PRs, and design questions all welcome — please file them on the
repo that owns the relevant code:

- Firmware bugs / questions → `homesensors/firmware`
- Schematic / PCB / fab questions → `homesensors/hardware`
- HA integration questions → `homesensors/homeassist`
- Cross-cutting questions, architecture, protocol → this repo

## Status

🟡 **Open beta.** Working end-to-end on the maintainer's bench (90+ events/day,
> 2-year battery-life trajectory verified). Not yet shipped as a polished
product. PCB Rev B in progress.
