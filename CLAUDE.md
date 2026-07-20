# CLAUDE.md — Freematics tracker firmware (fork)

## What this is
Fork `securityguy/Freematics` (upstream `stanleyhuangyc/Freematics`). Firmware for a
**Freematics ONE+ (ESP32) OBD-II / LTE / GPS tracker**.

- **Active firmware:** `firmware_v5/telelogger/` — this is the only tree we work in.
  Other `firmware*/`, `ESPRIT/`, `server/` dirs are upstream baggage (many show as
  deleted in `git status`); ignore them.
- **Release status: RELEASED.** Devices are in the field. Ask before breaking changes
  or on-wire telemetry semantics. Do not preserve unrelated legacy behavior on
  unreleased/local edits, but treat the wire protocol as a contract.

## Data path (important)
The device sends the **Freematics Hub UDP protocol directly to Traccar** — there is NO
intermediate server/bridge. Traccar's built-in `freematics` protocol decoder parses it.
- On-wire format: `devid#PID:value,PID:value,...*<checksumHex>` — hex PID, `:` delimiter,
  `,`-separated (`telestore.cpp`). Every transmitted packet is echoed on Serial as `[DAT] …`
  (and standby reports as `[STBY] …`).
- `config.h` points at a Freematics-Hub-style host/port (`178.156.222.4:5170`, `PROTOCOL_UDP`),
  but in practice that's Traccar's freematics port.

### Traccar `freematics` decoder PID mapping + scaling (verify at traccar/traccar master before relying)
- `0x82` PID_DEVICE_TEMP → `deviceTemp`, **÷10** → send tenths.
- `0x24` PID_BATTERY_VOLTAGE → `battery`, **÷100** → send `volts*100`.
- `0x81` → RSSI. `0x20` → acceleration.
- `0x10C` = `PID_RPM | 0x100` → RPM. (OBD PIDs are sent OR'd with `0x100`.)
- GPS: `0xA` lat, `0xB` lng, `0xC` alt, `0xD` speed (kph, decoder→knots), `0xE` course,
  `0xF` sats, `0x12` hdop, `0x10` time, `0x11` date. No lat/lng ⇒ position marked invalid.
- **Unknown PIDs become opaque `ioNN` attributes.** MEMS temp `0x23` is NOT decoded — do
  not use it; the MEMS temperature goes out on `0x82` instead.

## Architecture gotchas
- **Two FreeRTOS tasks sharing globals:**
  - `loop()` (main) — owns OBD, MEMS, GNSS; runs `process()`; calls `standby()` when
    `!STATE_WORKING`.
  - `telemetry()` task — owns the cell modem; the ONLY place that transmits.
- **Standby (engine off):** `process()` returns early (OBD not answering) → `standby()`
  blocks in `waitMotion(-1)` reading only the accelerometer until motion (then `ESP.restart`,
  `RESET_AFTER_WAKEUP`). Meanwhile the telemetry task wakes every `PING_BACK_INTERVAL` and,
  as of the `enhanced-reports` work, sends a real report via `sendStandbyReport()` (GPS powered
  up per wake, temp/battery/RPM0). Any report that reaches `BUFFER_STATE_FILLED` includes
  device temp; early-return cycles never transmit.
- **Temperature:** `deviceTemp` (int, whole °C) is used by `COOLING_DOWN_TEMP` throttling and
  serial display — keep it whole. `deviceTempX10` (tenths, from the MEMS float) is what goes
  on the wire. MEMS read fills temp via `mems->read(acc, 0, 0, &temp)`.
- **RPM_IGNITION_INDICATOR:** when the ECU responds, RPM 0 is remapped to 1 (ignition ON,
  distinct from the RPM-0 ignition-OFF marker). Keep sending RPM 0 while parked.

## Building
- It's a **PlatformIO** project (`platformio.ini`: `esp-wrover-kit`, `huge_app`,
  `BOARD_HAS_PSRAM` commented out). Use PlatformIO for real builds/flash.
- `pio` is NOT installed locally; `arduino-cli` is, but its only generic ESP32 core is
  **3.3.10, which is too new** for this code — it breaks pre-existing calls (`ledcAttachPin`
  in `FreematicsPlus.cpp`, `esp_spiram_get_size` under `BOARD_HAS_PSRAM`). Neither is our code.
- To sanity-check that `telelogger.ino` compiles, build with a no-PSRAM board and count
  sketch errors (0 = clean):
  `arduino-cli compile --fqbn esp32:esp32:esp32 --libraries ../../libraries . 2>&1 | grep -c "telelogger.ino.*error:"`
  The overall link will still fail on the library `ledcAttachPin` — that's expected here.
- After firmware changes, always still flash + drive/park test on real hardware.

## Key config knobs (`firmware_v5/telelogger/config.h`)
`PING_BACK_INTERVAL` (standby report interval, 1800s), `STANDBY_GPS_TIMEOUT` (90s fix wait),
`RPM_IGNITION_INDICATOR`, `IGNITION_OFF_REPORTS`, `SILENT` (1 = mute buzzer), `GNSS_ALWAYS_ON`,
`DATA_INTERVAL_TABLE`, `COOLING_DOWN_TEMP`.

## Misc
- Buzzer: only source of sound is `beep()` → `sys.buzzer()`. It beeps on each network
  (re)connect (WiFi + cellular "In service"). Gated by `SILENT`.
- **Upstream PRs:** #148 (processOBD out-of-bounds tier index) pulled in & adapted. #159
  (SIMCOM cellular-GNSS recovery) skipped — only relevant to `GNSS_CELLULAR`; we use standalone
  GNSS. Most other upstream PRs are already merged here, cosmetic (OLED), or HTTPS-only.
- Branch `enhanced-reports` holds the standby-report / temp-precision / OBD-fix / SILENT work
  (not merged, no PR by request).
