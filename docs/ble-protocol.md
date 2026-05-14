# BLE protocol specification

This document fully specifies the on-air wire format for the
`sensorkit` BLE sensors and the Tier 1.5 fault-tolerant ACK protocol
between sensor and Home Assistant. It is detailed enough to write a
compatible sensor or HA-side daemon from scratch in any language.

## Scope and design goals

- **Native HA discovery.** Sensor adverts must be valid BTHome v2
  (https://bthome.io/format/) so that HA's built-in BTHome integration
  picks the device up with zero configuration.
- **Connectionless.** No GATT, no pairing, no connection-state retention.
  Every event is a self-contained broadcast.
- **Guaranteed delivery.** Events survive HA / Wi-Fi / power outages of
  ~12 h via per-event retransmits on exponential back-off, with
  delivery confirmed via a custom ACK advert.
- **History-correct.** Each event carries the Unix timestamp of the
  original Hall-edge moment, so a retransmit arriving hours after the
  event still lands in HA's history at the correct time.
- **Battery-friendly.** Sensor average current ~10 µA in steady state.
  The ACK protocol only runs the BLE scanner while there are unacked
  events in the buffer.

## Reference link layer

- **PHY**: BLE 1M (legacy), uncoded.
- **Advertising channels**: 37 / 38 / 39 (full primary scan set).
- **Advertising type**: ADV_NONCONN_IND (non-connectable, non-scannable,
  undirected). Both outbound BTHome and inbound ACK adverts use this
  type — neither side ever forms a connection.
- **Address type**: static-random (top two bits of MSB = `11`).

## Outbound — sensor → HA (BTHome v2)

### Advertising data structure

| AD type | Bytes | Value | Notes |
|---|---|---|---|
| Flags (0x01) | 3 | `02 01 04` | LE-only, BR/EDR not supported. `0x04` is coherent with `GAP_MODE_NON_DISCOVERABLE` per Bluetooth Core §10.7.1 (non-connectable/non-scannable adverts). |
| Service Data — 16-bit UUID (0x16) | 16 | `0F 16 D2 FC <12-byte payload>` | UUID 0xFCD2 little-endian (BTHome). 12-byte BTHome v2 payload. |

Total ADV PDU payload: **19 bytes**, well inside the 31-byte legacy cap.

### Service-data payload (12 bytes, post clock-sync)

| Offset | Length | Field | Encoding |
|---|---|---|---|
| 0 | 1 | BTHome device info | `0x44` = v2, unencrypted, trigger-based |
| 1 | 1 | Object ID | `0x00` (packet id) |
| 2 | 1 | Packet ID | uint8, mod-256, monotonically increasing per event |
| 3 | 1 | Object ID | `0x01` (battery percent) |
| 4 | 1 | Battery | uint8, 0..100 % |
| 5 | 1 | Object ID | `0x2D` (opening / binary sensor "opening") |
| 6 | 1 | Opening state | `0` = closed (door + magnet aligned), `1` = open |
| 7 | 1 | Object ID | `0x50` (timestamp) |
| 8..11 | 4 | Unix epoch seconds | uint32, **little-endian**. Original Hall-edge moment, NOT current time. Preserved across retransmits. |

### Pre-sync payload (7 bytes)

Before the sensor has received its first ACK, its wall clock is unsynced
and the `0x50` timestamp object is omitted. The payload is then 7 bytes:

| Offset | Length | Field |
|---|---|---|
| 0 | 1 | `0x44` |
| 1..2 | 2 | `0x00 PID` |
| 3..4 | 2 | `0x01 BAT` |
| 5..6 | 2 | `0x2D OPN` |

After the first ACK lands and anchors the wall clock, all subsequent
adverts (including retransmits of pre-sync events that were buffered)
use the 12-byte timestamped form.

HA's BTHome integration treats `0x50`-tagged events as having occurred
at the object's Unix time and writes that into the entity's
`last_changed` field. This is what allows retransmits hours after the
fact to appear in history at the correct moment.

### Burst behaviour

On every accepted Hall edge:
1. Increment `packetId` (mod 256).
2. Build the service-data payload above (with current Unix time from
   the sensor's clock anchor).
3. Push `{packetId, opening, unix_time}` to the ACK retry buffer (see
   below).
4. Enable advertising for `CFG_BTHOME_ADV_BURST_MS` (default 400 ms).
   The controller stops the burst automatically via the Duration field.
5. Open the ACK scan window (see below).

At the default 100–150 ms ADV interval, a 400 ms burst yields ~3 ADV
PDUs per channel × 3 channels = ~9 PDUs total per event. HA's
BTHome scan duty cycle is high enough to catch one of these reliably.

## Inbound — HA → sensor (Tier 1.5 ACK)

### Advertising data structure

| AD type | Bytes | Value | Notes |
|---|---|---|---|
| Flags (0x01) | 3 | `02 01 06` | LE general discoverable + BR/EDR not supported. Standard for non-targeted adverts. |
| Service Data — 16-bit UUID (0x16) | 13 | `0C 16 D3 FC <9-byte payload>` | UUID 0xFCD3 little-endian (custom — chosen to be adjacent to 0xFCD2 BTHome). |

Total: 16 bytes. Emitted by `door_ack_daemon` for `ack_duration`
seconds (default 3 s) every time HA observes the BTHome packet-id
entity advance.

### ACK payload (9 bytes)

| Offset | Length | Field | Encoding |
|---|---|---|---|
| 0..3 | 4 | Target MAC hash | First 4 bytes of the sensor's static-random BD address, **in display order** (i.e. the high-order half of the MAC as printed). Per-device filter so multiple sensors on one HA can coexist. |
| 4 | 1 | Last acknowledged packet id | uint8, mod-256. The sensor retires *every* buffered entry with `pid ≤ acked_pid` (cumulative ACK, wrap-aware). |
| 5..8 | 4 | Daemon's current Unix time | uint32, **little-endian**. Re-anchors the sensor's wall clock on every ACK. |

### Sensor-side ACK handling

1. Scanner runs (passive, 200 ms interval / 100 ms window) for
   `CFG_ACK_SCAN_WINDOW_MS` (default 800 ms) after each advert burst,
   and stays running as long as the retry buffer has unacked entries.
2. On `HCI_LE_ADVERTISING_REPORT`:
   - Parse AD structures looking for Service Data UUID 0xFCD3.
   - Compare bytes 0..3 of the payload against `s_own_mac4` (BD address
     bytes [5..2], high half in display order). Reject if mismatch.
   - Extract byte 4 as `last_acked_pid`.
   - Extract bytes 5..8 as `daemon_unix`.
3. Call `AckBuf_RetireUpTo(last_acked_pid)`:
   - Walk every in-use entry.
   - Compute `signed_delta = (last_acked_pid − entry.pid) mod 256`,
     interpreted as a signed 8-bit value.
   - If `signed_delta >= 0`, mark entry retired.
4. Update the wall-clock anchor: store `(daemon_unix, current sysTime)`
   as the new anchor. `tier15_now_unix()` then computes wall time as
   `daemon_unix + (sysTime_now − anchor_sysTime) / 1000`.

### Retransmit schedule

Each unacked entry in the retry buffer carries a retry counter and a
`next_retry_tick`. The schedule is exponential back-off:

```
retry_n_delay = min(CFG_ACK_RETRY_BASE_MS << n, CFG_ACK_RETRY_MAX_MS)
```

Defaults:
- `BASE_MS` = 5 000
- `MAX_MS`  = 1 800 000 (30 min)
- `MAX_RETRIES` = 32

Schedule:
```
   n=0:    5 s
   n=1:   10 s
   n=2:   20 s
   n=3:   40 s
   n=4:   80 s
   n=5:  160 s
   n=6:  320 s
   n=7:  640 s
   n=8: 1280 s → capped to 1800 s
   n=9..31: 1800 s each (24 × 30 min)
```

Cumulative time to drop: approximately **12.7 hours**. After
`MAX_RETRIES`, the entry is dropped from the buffer and considered
permanently lost.

When a retransmit fires, the burst carries the *original*
`event_unix_time` captured at the Hall edge, **not** the current time —
this is what lets HA reconstruct history correctly.

### Buffer management

- Buffer size: `CFG_ACK_BUFFER_SIZE` (default 32 slots).
- Overflow policy: drop oldest. Each magnet wave generates 2 events
  (close → open → close), so 32 slots ≈ 16 waves of buffering.
- Cumulative retire: an ACK with `pid = 17` retires entries 5, 8, 10,
  17 in one shot (all `≤ 17` mod-256).
- Mod-256 wrap: handled by signed-delta comparison so an ACK for `pid =
  2` after the sensor has wrapped from 255 → 0 → 2 correctly retires
  255 and 0 too.

## HA-side reference implementation

The `door_ack_daemon` add-on
([`homesensors/homeassist`](https://github.com/homesensors/homeassist)) is the
reference implementation of the HA side. Behaviour summary:

1. Poll `sensor.<door>_packet_id` (configurable entity) at 1 Hz via the
   Supervisor proxy `http://supervisor/core/api/states/<entity>`.
2. On every observed pid increment, spawn `door_ack.py` which invokes
   `bluetoothctl` to construct + emit an ADV_NONCONN_IND advert with the
   structure above. ACK emission duration is `ack_duration` seconds
   (default 3 s).
3. Concurrent ACKs are fine — `bluetoothctl` handles them as separate
   advertisement objects.

Authentication uses the auto-injected `SUPERVISOR_TOKEN` against the
add-on proxy — no user-managed long-lived tokens.

## Compatibility notes

- **BTHome version**: this protocol uses BTHome v2 (device-info byte
  `0x44`). BTHome v1 had a different object encoding and is not
  supported.
- **HA Bluetooth integration**: any HA Bluetooth adapter works (USB
  dongle, on-board hci0 on Pi 5, ESPHome Bluetooth Proxy). The ACK
  emitter requires direct BlueZ access on the HA host, so it works
  with HA OS on Pi or HA Supervised on a Linux host with the Bluetooth
  integration enabled. Pure Bluetooth Proxy setups (where the proxy
  forwards adverts to a remote HA) currently can't run the daemon —
  the proxy doesn't expose an ACK-emit channel.
- **UUID 0xFCD3**: the ACK UUID is locally allocated. Not registered
  with the Bluetooth SIG, but in the
  "member-services-data" range that's commonly used for
  vendor-specific protocols. Verify no collision with other devices
  on your home network if you have other 0xFCD3 emitters; collision
  is harmless (MAC-hash filter rejects unrelated adverts).

## Versioning

This is **protocol version 1.0**. Future revisions will be signalled
by additional BTHome objects (e.g. version object) and by ACK payload
extensions. Backwards compatibility:

- New BTHome objects appended to the outbound payload: HA's BTHome
  integration silently ignores unknown objects. Compatible with older
  HA installs.
- New ACK payload bytes: sensor parser must check AD length before
  accessing higher offsets. Reference implementation does this.

## Reference values summary

| Constant | Default | File |
|---|---|---|
| BTHome service UUID | 0xFCD2 | `bthome.h` |
| BTHome device-info | 0x44 | `bthome.h` |
| BTHome flags | 0x04 | `app_ble.c` |
| Advert burst duration | 400 ms | `app_conf.h` |
| ACK service UUID | 0xFCD3 | `app_conf.h` |
| ACK scan window | 800 ms | `app_conf.h` |
| ACK scan interval / window | 200 ms / 100 ms | `app_ble.c` |
| Retry buffer size | 32 slots | `app_conf.h` |
| Retry base | 5 s | `app_conf.h` |
| Retry cap | 30 min | `app_conf.h` |
| Max retries | 32 | `app_conf.h` |
| Total retention | ~12.7 h | derived |
