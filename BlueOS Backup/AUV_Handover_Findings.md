# What Last Year's AUV Team Actually Built
**Recovered from the Raspberry Pi / BlueOS vehicle, September 2026**
Source: 38 MAVLink telemetry logs (`.tlog`), 2 Dec 2025 – 15 Apr 2026, plus SD-card boot config.

---

## Bottom line

**They wrote no custom software.** Everything on this vehicle is stock Blue Robotics
software that was *configured*, not programmed. There is no codebase to inherit.

What they did build was a Pixhawk-based control system driving four thrusters by joystick.
The vehicle **was operated in water** (confirmed by team video of the v2 red/white vehicle),
but **with no working depth sensor** in any logged session, and with a frame configuration
that did not match the physical thruster layout.

---

## Evidence for "no custom code"

| Check | Result |
|---|---|
| BlueOS Extensions installed | `Cockpit` (stock Blue Robotics UI) and `major_tom` (stock Blue Robotics cloud agent). Nothing custom. |
| Lua scripting (`SCR_ENABLE`) | `0` — disabled. No onboard scripts ever ran. |
| Cockpit stored data | 0.0 kB — not even a saved custom control layout. |
| Boot config (`config.txt`) | Stock BlueOS Navigator block, unmodified. |
| First-boot config (`custom.toml`) | Stock Raspberry Pi Imager defaults: hostname `blueos`, user `pi`. |

---

## The hardware they actually used

- **Autopilot:** Pixhawk 1 (`fmuv2`, serial `003F001C 35335116 39363930`)
  — **not** the Navigator board the SD card was configured for.
- **Firmware:** ArduSub **V4.0.3** (`96882ed0`) — released around 2020, several major
  versions behind current.
- **Frame:** `FRAME_CONFIG = 1` (Vectored) — a **six**-thruster layout.
  ⚠️ The physical vehicle has **four** thrusters (2 side/horizontal, 2 base/vertical).
  The correct frame for that is **`FRAME_CONFIG = 5` (SimpleROV-4)**. This mismatch is
  probably the root cause of the strange servo mapping described below.
- **Camera:** a MAVLink camera component appeared in the logs from 18 March 2026 onward.
- **Depth sensor:** **none detected in any log.** No Bar30 / `SCALED_PRESSURE2` data exists.
- **Battery monitoring:** `BATT_MONITOR = 0` — disabled. No voltage or current was ever logged.

---

## Timeline of work

| Period | What happened |
|---|---|
| 2 Dec 2025 | First session. Thruster mapping correct and standard (Motor 1–6 on outputs 1–6). |
| Dec 2025 – Feb 2026 | Regular bench sessions. Motor tests, joystick control, EKF alignment. |
| 4–6 Mar 2026 | Output 6 changed from Motor 6 to `RCPassThru1`; output 9 set to `ThrottleLeft`. |
| 17–18 Mar 2026 | Thruster mapping substantially rearranged. Camera appears. |
| 14–15 Apr 2026 | Final and heaviest motor-test session. Last activity on the vehicle. |

Roughly 5,500 armed heartbeats were recorded, so the motors were genuinely spun up many times.

---

## ⚠️ Problems they left behind

### 1. The thruster mapping does not match a four-thruster vehicle
Final state of the servo outputs:

| Output | Function | Notes |
|---|---|---|
| SERVO1 | `ThrottleLeft` (73) | A fixed-wing/rover function. Meaningless on a submarine. |
| SERVO2 | Motor 3 | |
| SERVO3 | Motor 4 | |
| SERVO4 | Motor 5 | |
| SERVO5 | Motor 6 | |
| SERVO6 | **Motor 3** | **Duplicate of SERVO2** |
| SERVO7 | Motor 1 | |
| SERVO9 | `ThrottleLeft` (73) | Meaningless on a submarine. |

Read against the four physical thrusters, the intent becomes clear: **Motor 3 and Motor 4**
(two of the Vectored frame's angled horizontal thrusters) plus **Motor 5 and Motor 6**
(its vertical pair) — i.e. they picked four of the six motor slots to match four real
thrusters. `SERVO6` (duplicate Motor 3) and `SERVO7` (Motor 1) look like leftovers.

The problem is that the **Vectored mixer assumes six thrusters, with the horizontal ones
mounted at 45°**. Driving only four of its outputs means the mixing maths is wrong for this
vehicle no matter how the outputs are assigned. Hand-patching servo functions cannot fix a
frame mismatch — the frame itself has to change to SimpleROV-4.

Also note `SERVO1` and `SERVO9` are set to `ThrottleLeft`, which will sit at a fixed output
rather than centre. If anything is wired to those outputs, check it carefully.

### 2. Thruster power limits were narrowed, inconsistently
Default PWM range is 1100–1900. They left it at:

| Output | Min | Max |
|---|---|---|
| SERVO2 | 1302 | 1700 |
| SERVO3 | 1330 | 1677 |
| SERVO4 | 1370 | 1623 |
| SERVO5 | 1349 | 1650 |
| SERVO6 | 1292 | 1709 |
| SERVO7 | 1100 | 1900 (untouched) |

Each thruster is limited to a different fraction of full power — roughly ±12–20%.
This looks like deliberate power limiting for a current-limited bench supply, but it
was never applied consistently.

### 3. Safety checks were switched off
- `ARMING_CHECK = 448` — barometer, compass, GPS, INS and parameter checks all **disabled**.
- `FS_EKF_ACTION = 0` — EKF failsafe off.
- `FS_PRESS_ENABLE = 0`, `FS_TEMP_ENABLE = 0`, `FS_CRASH_CHECK = 0`.
- `FS_LEAK_ENABLE = 1` — leak detection was left on (the one good one).

These are classic "make it arm on the bench" settings. **They must be restored before
the vehicle goes near water.**

### 4. Sensors were never healthy
- `Calibration FAILED` and `Place vehicle level and press any key` appear in the logs —
  an accelerometer calibration was attempted and failed.
- `EKF2 IMU0 forced reset` appears **1,295 times**, plus a ground magnetic anomaly warning.
- Flight mode usage: **62,332 heartbeats in MANUAL** vs only **697 in STABILIZE**.
  Depth hold and autonomous modes were never used — consistent with having no depth sensor.

### 5. No depth sensor in any logged session
Depth reads 0.00 m in every log, and **no external pressure-sensor (Bar30) messages exist
at all** across all 38 sessions. The only barometer present is the Pixhawk's internal one,
reading normal atmospheric pressure (989–1013 mbar).

The vehicle *was* in the water (team video confirms this for the v2 vehicle), so the
conclusion is that **it was piloted without any depth sensing** — or the in-water runs were
not captured by BlueOS logging. Either way, no depth data survives.

### 6. The attitude estimate was unusable — consistent with an unsecured autopilot
Standard deviation of the roll estimate across sessions: **42°, 45°, 103°, 129°**.
Pitch up to 30°, yaw up to 134°. A securely-mounted vehicle does not produce these numbers.
Measured vibration was very low (mean 0.0–0.2 m/s², no accelerometer clipping), which rules
out motor-induced vibration and points instead to the **board physically moving** — the
Pixhawk contains the IMUs, so if it is loose in the enclosure, it reports its own tumbling
as vehicle attitude. This matches the 1,295 `EKF2 IMU0 forced reset` events and the failed
accelerometer calibration.

---

## Recommendations for the 2026–2027 team

1. **Treat the recovered parameter file as history, not a starting point.** Keep it for
   reference, but do not load it onto a vehicle as-is — the motor map is broken and the
   safety checks are disabled.
2. **Set `FRAME_CONFIG = 5` (SimpleROV-4)** to match the real four-thruster layout, then
   assign motors using the frame setup page rather than editing servo functions by hand.
3. **Mount the Pixhawk rigidly** to the frame, in a known orientation, before any further
   tuning. Nothing else can be trusted until the autopilot stops moving independently of
   the vehicle.
4. **Add a depth sensor (Bar30).** An AUV cannot hold depth or run autonomous missions
   without one. This is the single biggest capability gap.
5. **Enable battery monitoring.** Running thrusters with no voltage or current telemetry
   is how packs get over-discharged and thrusters get cooked.
6. **Decide on the autopilot.** The SD card was configured for a Navigator board, but the
   vehicle actually ran a Pixhawk 1. Pick one deliberately.
7. **Upgrade the firmware.** ArduSub 4.0.3 is roughly six years old. Current releases add
   substantially better EKF, depth control and failsafe behaviour.
8. **Redo the thruster setup from scratch** using the frame setup wizard rather than
   patching the existing mapping.
9. **Restore `ARMING_CHECK`** to full checks once the sensors are healthy.

---

## Files recovered

- `ArduSub_recovered_2026-04-15.params` — all 903 parameters in QGroundControl format.
- `ArduSub_recovered_2026-04-15.csv` — the same values as a plain spreadsheet.
- `_logs.zip` — the original 38 telemetry logs (80 MB).
