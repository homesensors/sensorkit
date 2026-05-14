# Architecture

High-level overview of how the three `homesensors/*` repos fit together
and the data flow through a running deployment.

## Repo map

```
github.com/homesensors/
├── sensorkit/      ← umbrella (you are here). Pitch, architecture, protocol spec, build guide.
├── firmware/       ← MCU code that runs on the WB09 sensor.
├── hardware/       ← KiCad project, Gerbers, BOM.
└── homeassist/     ← HA add-on (door_ack_daemon) + dev scripts.
```

Each sub-repo is independently versioned. Cross-references between them
are by stable tags, not by commit hashes.

## Component view

```
   ┌─────────────────────────────────────────────────────────────┐
   │                       SENSOR                                │
   │   ┌──────────────┐   ┌────────────┐                         │
   │   │ Hall sensor  │──▶│  STM32WB09  │── PA1 ──▶ status LED   │
   │   │ TCS40DPR PA0 │   │             │                         │
   │   └──────────────┘   │  on-chip    │── PB1 ──▶ battery div  │
   │                      │  BLE radio  │                         │
   │   CR2032 ──────────▶ │             │── matching net ──▶ ANT │
   │                      └─────┬───────┘                         │
   │                            │ 2.4 GHz BLE                    │
   └────────────────────────────┼─────────────────────────────────┘
                                │
              BTHome advert (0xFCD2) ▼    ▲ ACK advert (0xFCD3)
                                │           │
   ┌────────────────────────────┼───────────┼─────────────────────┐
   │                            ▼           │                     │
   │                    ┌──────────────┐    │                     │
   │                    │ BlueZ / hci0 │ ───┘                     │
   │                    │  (Pi 5 BLE)  │                          │
   │                    └──┬───────┬───┘                          │
   │                       │       │                              │
   │                       ▼       ▼                              │
   │            ┌──────────────┐  ┌────────────────────────────┐  │
   │            │ HA Bluetooth │  │ door_ack_daemon (add-on)   │  │
   │            │ integration  │  │                            │  │
   │            └──────┬───────┘  │  poll pid sensor @ 1 Hz    │  │
   │                   │          │  on increment → emit ACK   │  │
   │                   ▼          │  via bluetoothctl          │  │
   │           ┌───────────────┐  └─────────┬──────────────────┘  │
   │           │ BTHome native │            │                     │
   │           │ integration   │            │                     │
   │           │               │            │                     │
   │           │ entities:     │            │                     │
   │           │  - opening    │            │                     │
   │           │  - battery %  │            │                     │
   │           │  - packet_id  │────────────┘                     │
   │           │  - timestamp  │   (Supervisor REST polling)      │
   │           └───────┬───────┘                                  │
   │                   │                                          │
   │                   ▼                                          │
   │             Dashboard / automations / notifications          │
   │                                                              │
   │                       HOME ASSISTANT (HA OS on Pi 5)         │
   └──────────────────────────────────────────────────────────────┘
```

## Power budget (steady state, healthy daemon)

| Line item | Average current |
|---|---|
| MCU sleep (STOP_LS_CLOCK_ON between 100 ms polls) | ~10 µA |
| Boot-LED heartbeat (50 ms / s) | 0 µA after LPM arms |
| TX burst (400 ms × 5.5 mA × N events/day) | ~0.05 mA-h/day @ 20 events |
| RX scan window (~800 ms × 5 mA × N events/day) | ~0.05 mA-h/day @ 20 events |
| Battery-divider quiescent | ~1.5 µA |
| **Total daily** | **~0.4 mAh** |

CR2032 (225 mAh effective) projected lifetime: **~1.5 years**. With LPM
Tier B (full STOP + EXTI wake, not yet implemented), idle drops to
~1.5 µA → lifetime extends to ~4 years.

## Data flow per event

1. Magnet leaves the field of the Hall sensor → PA0 transitions high.
2. 100 ms VTIMER poll task detects the edge.
3. Sensor:
   - Increments `packetId` (mod-256).
   - Reads battery ADC (PB1, VINP1).
   - Captures `event_unix_time` from the LSE-backed wall clock.
   - Pushes `{pid, opening, event_unix_time}` to AckBuf.
   - Builds BTHome service-data payload.
   - Starts non-connectable burst, duration `CFG_BTHOME_ADV_BURST_MS`.
   - Starts/keeps scanner running.
4. HA's Bluetooth integration sees the advert, the BTHome integration
   decodes it, updates entities (`opening`, `battery`, `packet_id`,
   `timestamp`).
5. `door_ack_daemon` (polling at 1 Hz) observes `packet_id` change.
   Spawns `door_ack.py`.
6. `door_ack.py` invokes `bluetoothctl` to register and broadcast an
   ADV_NONCONN_IND with service-data UUID 0xFCD3 for `ack_duration`
   seconds (default 3 s).
7. Sensor's scanner receives the ACK advert, parses it, retires every
   AckBuf entry with `pid ≤ acked_pid` (cumulative, wrap-aware).
8. Sensor's wall-clock anchor is updated from the ACK's Unix-time field.
9. If AckBuf is now empty, scanner stops. CPU returns to LPM at next
   poll tick.

If step 6 fails (HA offline, daemon stopped, BLE collision, etc.):

- AckBuf retains the entry.
- After `BASE_MS × 2^retry_n` (capped at 30 min), the entry retransmits
  with its *original* `event_unix_time` (not current time).
- Repeats up to `MAX_RETRIES` (32) ≈ 12.7 h total. After that, the
  event is dropped from the buffer.

## What lives where, in detail

### Firmware (`homesensors/firmware`)

STM32WB09 application, built with STM32CubeIDE 2.1+. Key modules:

- `Core/Src/hall_sensor.c` — polling + LED state machine.
- `Core/Src/bthome.c` — BTHome v2 payload encoder.
- `Core/Src/ack_buffer.c` — retry ring buffer with mod-256-wrap-aware retire.
- `Core/Src/battery_monitor.c` — ADC + scaling for cell voltage.
- `STM32_BLE/App/app_ble.c` — BLE init, broadcaster + observer roles,
  ACK parser, scanner control, wall-clock anchor.

Detailed design notes in the firmware repo's `HANDOVER.md`.

### Hardware (`homesensors/hardware`)

KiCad 7+ project. Files:

- `ble_door_sensor.kicad_pro/sch/pcb` — current revision.
- `fab/` — Gerber zip, drill files, pick-and-place CSV (when generated).
- `bom/` — BOM with LCSC / Mouser part numbers.

Revisions tagged in git as `pcb-rev-a`, `pcb-rev-b`, etc.

### HA add-on (`homesensors/homeassist`)

HA local add-on packaging the daemon. Two-script structure:

- `door_ack.py` — single-shot ACK emitter via `bluetoothctl`. Callable
  standalone for testing.
- `ha_door_ack_daemon.py` — long-running driver: HA REST polling +
  fire-and-forget ACK emission.

Containerised as `ghcr.io/home-assistant/<arch>-base:3.20` with
`bluez` + `python3`. Talks to HA via `SUPERVISOR_TOKEN` against the
Supervisor proxy at `http://supervisor/core`.

## Roadmap touchpoints

| Item | Scope | Status |
|---|---|---|
| Tier B LPM (full STOP, EXTI wake) | Firmware | Pending ST-LINK V3 MINI E |
| PCB Rev B (VDDA_VCAP cap + battery divider folded in) | Hardware | In progress |
| HACS-installable shim | HA add-on | Not started |
| Multi-sensor mac-hash routing | HA add-on | Daemon already supports — needs UI/docs |
| Zigbee variant | Firmware / Hardware | Distant future |
| LoRa variant | Firmware / Hardware | Distant future |
| Wi-Fi variant (ESP32) | Firmware / Hardware | Distant future |
