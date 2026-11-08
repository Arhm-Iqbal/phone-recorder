# Native iOS client specification (100 Hz)

Context for continuing this work on a Mac. The server, receiver, storage, CSV export and dashboard
all stay as they are. Only the **phone client** is replaced.

## Why

Safari caps `DeviceMotionEvent` at 60 Hz. The value is fixed in WebKit
(`kMotionUpdateInterval = 1.0f / 60.0f`) and there is no web API to raise it — Safari does not
implement the Generic Sensor API, which is the only web interface with a `frequency` option.
The iPhone IMU itself runs at 100 Hz; only the browser delivery rate is limited.

`README.md` records that the Python classifier expects a calibrated 100 Hz-or-higher dataset and
that Safari recordings below its minimum rate are not directly compatible. Interpolating 60 Hz up
to 100 Hz is explicitly ruled out there. A native CoreMotion client is the only way to reach the
real rate.

Secondary benefit, arguably larger: CoreMotion provides a true per-sample sensor timestamp.
`README.md` currently notes that the browser exposes one shared event timestamp for both sensors
and that independent native sensor timestamps are unavailable. That limitation disappears.

## Target

- `CMMotionManager.deviceMotionUpdateInterval = 0.01` (100 Hz)
- Same WebSocket protocol as `public/phone.js`, so no server change is required
- Two devices: `phone_1` and `phone_2`

## Endpoint and auth

    wss://phone-recorder-eewc.onrender.com/ws

The pairing key is the `IMU_TOKEN` environment variable on the Render service. It is sent in the
`hello` message. Treat it as a password; do not commit it.

## Clock

The web client uses `performance.now()/1000` (monotonic seconds). The Swift equivalent is
`CACurrentMediaTime()`. CoreMotion's `CMLogItem.timestamp` is seconds since boot and lives in the
same monotonic domain, which is what makes native timestamps usable directly.

Synchronization, implemented in `lib/protocol.mjs` as `estimateClock`:

- Server sends `sync_probe {id, h1}`
- Phone captures `p2` **on receipt, before parsing**, and `p3` **immediately before sending**
- Phone replies `sync_reply {id, p2, p3}`
- Server captures `h4` on receipt
- `offset = ((p2 - h1) + (p3 - h4)) / 2`, `rtt = (h4 - h1) - (p3 - p2)`
- Exchanges are valid only when `rtt >= 0`, `rtt < 1`, `p3 >= p2`, all finite
- At least 10 valid exchanges required; the median offset of the 5 lowest-RTT exchanges wins
- 25 probes initially, then every 15 seconds
- **Computer time = phone time − offset**

Capture `p2` before JSON parsing. Parsing cost otherwise leaks into the offset estimate.

## Messages

Phone sends:

| Message | Fields |
|---|---|
| `hello` | `protocol_version:1`, `role:'phone'`, `token`, `device_id`, `client_id`, `boot_id` |
| `sync_reply` | `id`, `p2`, `p3` |
| `status` | `sensors`, `visible`, `sync_id`, `error`, `warning`, `buffered` |
| `armed` | `session_id` |
| `samples` | see envelope below |
| `preview` | `samples` (throttled, every 80 ms) |
| `stop_request` | — |
| `ended` | `session_id`, `last_sequence` |

Server sends: `welcome`, `sync_probe`, `sync_model`, `prepare`, `stop`, `ack`, `end_ack`, `error`.

On `prepare`, refuse to arm unless sensors are live, the app is foregrounded, storage is healthy
and a clock model exists — then reply `armed`.

## Sample packet envelope

Validated by `validatePacket` in `lib/protocol.mjs`; a malformed packet is rejected outright.

```
type: 'samples'
protocol_version: 1
device_id: 'phone_1' | 'phone_2'
device_label, client_id, boot_id, session_id
packet_sequence_number: safe integer >= 0, per session, starts at 0
accel_units: 'm/s^2'
gyro_units: 'deg/s'
samples: 1..100 entries
```

Every sample requires these finite numbers, plus `sync_id` as a safe integer:

    phone_timestamp, original_device_timestamp_ms, handler_timestamp,
    estimated_computer_timestamp, ax, ay, az, gx, gy, gz,
    rotation_alpha, rotation_beta, rotation_gamma

`estimated_computer_timestamp = phone_timestamp − model.offset`, or `0` when no model yet.
Optional: `event_index` (safe integer), `timestamp_order` (`increasing` | `duplicate` |
`out_of_order`), `browser_interval_ms`, `timestamp_kind`.

## Units and axes — verify, do not assume

This is the highest-risk part of the port. The web and native APIs do not agree, and
`README.md` states the classifier's unit, mounting-axis, gravity and calibration checks must stay
intact.

- Web `accelerationIncludingGravity` is **m/s²**. CoreMotion reports `gravity` and
  `userAcceleration` in **G units** — multiply their sum by ~9.81.
- Web `rotationRate` is **degrees/second**. CoreMotion `rotationRate` is **radians/second** —
  multiply by 180/π.
- The web client maps `gx = beta`, `gy = gamma`, `gz = alpha`. CoreMotion's axis and sign
  conventions differ from the W3C ones.

Do not hardcode an axis mapping from documentation. Derive it empirically: record the same physical
motion through the Safari client and the native client simultaneously and compare. The tap-alignment
test in `VALIDATION.md` (step 5) is designed for exactly this.

## Transport reliability

Mirror `pump()` in `public/phone.js`:

- At most 4 unacknowledged packets in flight
- Retry a packet if unacknowledged after 2 seconds
- Stop sending when the socket's buffered amount exceeds 128 KB
- Buffer unsent packets locally and never evict unacknowledged data
- `ack {session_id, sequence}` clears one packet; `end_ack` clears a finished session
- Batch up to 10 samples, flushed every 80 ms

## Setup on the Mac

```bash
git clone https://github.com/Arhm-Iqbal/phone-recorder.git
cd phone-recorder
```

Xcode is a 10 GB+ download. For signing, note that a **free Apple ID issues 7-day provisioning
profiles** — the app stops launching after a week and must be reinstalled. The paid Apple Developer
Program ($99/year) issues year-long profiles, which matters for a project running across a term.

Both phones must be provisioned, and both must stay foregrounded and unlocked during recording;
iOS suspends motion updates otherwise unless background modes are configured.

## Definition of done

Run `VALIDATION.md` against the native client, not just a smoke test:

1. Measured rate ≥ 95 Hz sustained on both phones, with interval SD and max interval recorded
2. Shared start boundary honoured after the three-second countdown
3. Tap-alignment test confirms axis and sign agreement with the previous Safari recordings
4. Ten-minute session with periodic clock updates logged in `sync.jsonl`
5. CSVs download and carry correct units
