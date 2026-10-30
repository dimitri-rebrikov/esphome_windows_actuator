# ESPHome based window actuator

ESPHome firmware migration of the [Tasmota based window actuator](../tasmota_window_actuator/).

The **hardware is unchanged** — same WeMos D1 mini, 4-channel relay module, two
push buttons and 8× WS2812b strip. Only the firmware changes: the device now
runs [ESPHome](https://esphome.io) instead of Tasmota.

## Communication

The device talks to home automation **only** over the ESPHome native API.

- `api:` enabled — Home Assistant auto-discovers the device.
- `mqtt:` **absent** — the device never speaks MQTT.
- Web server **absent**.

There is no MQTT broker, no HTTP control endpoint and no Tasmota console. The
only non-API channels are device-management/recovery mechanisms, never used for
home automation:

- **OTA** — flashing new firmware (encrypted with the same key as the API).
- **Captive portal** — only serves a config page while the fallback hotspot is
  active (Wi-Fi recovery); it is inert when the device is joined to the network.

## Hardware

Identical to the Tasmota project. Circuit diagram and assembly photos live in
[`../tasmota_window_actuator/`](../tasmota_window_actuator/).

### GPIO mapping

| D1 mini pin | GPIO   | Function                                             |
|-------------|--------|------------------------------------------------------|
| D4          | GPIO2  | WS2812b data in                                      |
| D2          | GPIO4  | Power relay + motor on/off (Tasmota Relay1) — "run"  |
| D1          | GPIO5  | Direction/polarity relay (Tasmota Relay2)            |
| D6          | GPIO12 | OPEN button (active-low, external 4.7 kΩ pull-up)    |
| D7          | GPIO13 | CLOSE button (active-low, external 4.7 kΩ pull-up)   |

The relay module is **active-low** (Tasmota `Relay_i`): driving a GPIO **low**
energises the corresponding relay.

### Motor topology (circuit-safe)

Four relays, driven as two groups:

| Group  | GPIO | Relays | Purpose                                   |
|--------|------|--------|-------------------------------------------|
| run       | GPIO4 | Tasmota Relay1 | power enable + motor on/off          |
| direction | GPIO5 | Tasmota Relay2 | polarity inversion (open vs close)   |

This is Tasmota *ShutterMode 2* ("circuit safe"): the direction relay is set
first, then the run relay is energised. The direction relays are wired so both
switch together, which makes an H-bridge short circuit impossible.

## What the firmware does

- **Cover** (`Window`, `device_class: window`) — time-based position tracking,
  open `4 s`, close `6 s` (read from the device's Tasmota config).
- **Buttons** — OPEN/CLOSE drive the cover; pressing either while the window is
  moving **stops** it (same behaviour as the Tasmota `ShutterButton` setup).
- **Status LEDs** — internal indicator:
  - fully closed → LED 1 red
  - fully open → LED 8 green
  - in between → LEDs 2–7 amber, lit proportionally to position

Only the `Window` cover is exposed to Home Assistant. The relays, buttons and
LED strip are `internal` (device-side only), so Home Assistant sees a single
clean cover entity.

## Getting started

### 1. Secrets

```bash
cp secrets.yaml.example secrets.yaml
```

Edit `secrets.yaml` and fill in your Wi-Fi credentials. Generate the API
encryption key (one key protects both the API connection and OTA):

```bash
openssl rand -base64 32
```

### 2. Flash

The board currently runs Tasmota. There are two ways to install ESPHome; pick
either.

#### Option A — from Tasmota over the network (no cable)

This is the officially supported route (ESPHome's
[Migrating from Tasmota](https://esphome.io/guides/migrate_sonoff_tasmota) guide).

1. Build the binary:
   - CLI: `esphome compile window-actuator.yaml` → the image is
     `.esphome/build/window-actuator/.pioenvs/window-actuator/firmware.bin`.
   - Dashboard: device menu → *Install → Manual download → factory format*.
2. On ESP8266, gzip it first (recommended, smaller upload):
   `gzip -kf firmware.bin` → `firmware.bin.gz`.
3. Tasmota web UI → **Firmware Upgrade** → *Upgrade by file upload* → choose the
   image → *Start upgrade*.
4. On "Upload Successful" the device reboots into ESPHome and joins `wifi_ssid`.

If Tasmota rejects the image: run `SetOption78 1` in the Tasmota console, restart,
and retry. If it fails because the ESP8266 image is too large: upload
`tasmota-minimal.bin` (from the Tasmota repo) first, then the ESPHome image.

#### Option B — USB serial (guaranteed fallback)

Only USB should power the board while flashing — **unplug the 12 V/mains supply**
so the external 5 V rail does not back-feed the USB port. The relays default to
off, so the motor cannot move during flashing.

1. Connect the D1 mini with a **data** USB cable (not charge-only).
2. Identify the serial port:
   - Windows: *Device Manager → Ports (COM & LPT)* → e.g. `COM3`
     (CH340 clones need the CH340 driver if no port appears).
   - Linux/macOS: `/dev/ttyUSB0` or `/dev/cu.usbserial-*`.
3. Build and upload:

   ```bash
   esphome run window-actuator.yaml
   ```

   Pick the port when prompted (or `--device COM3`). This compiles the full
   firmware first — that is where the lambdas are checked — then writes it with
   esptool. It fully replaces Tasmota; no manual erase is needed.

If the upload cannot connect, hold the board's RESET button while the port is
opened (esptool auto-resets via DTR/RTS on a healthy D1 mini).

Note: an interrupted OTA can leave the board needing a USB reflash — keep power
stable, and keep USB as your recovery path.

### 3. First boot

The firmware joins `wifi_ssid`. If it cannot, it starts the fallback hotspot
**"Window Actuator Fallback Hotspot"** (`fallback_password`); connect to it and
use the captive portal to fix the Wi-Fi settings. After that, every update is
**OTA** (encrypted with `api_encryption_key` — no cable):

```bash
esphome run window-actuator.yaml      # choose "Over the air"
```

### 4. Home Assistant

Once the device is online it is auto-discovered via the native API. Approve it
in *Settings → Devices & Services* using the `api_encryption_key` from
`secrets.yaml`.

## Verify the direction

If the window moves in the **wrong** direction (OPEN closes it and vice versa),
swap the `direction_relay` state between `open_action` and `close_action` in
`window-actuator.yaml` — i.e. change `output.turn_off: direction_relay` to
`output.turn_on: direction_relay` in `open_action`, and the reverse in
`close_action`.

This is a one-time hardware calibration (motor polarity / mechanical mounting);
it cannot be derived from the schematic.

## Differences from the Tasmota project

| Aspect | Tasmota | ESPHome |
|--------|---------|---------|
| Firmware | Tasmota 13.2 | ESPHome |
| HA transport | MQTT (+ broker) | native API, auto-discovery |
| Shutter | `ShutterMode 2` + rules | `time_based` cover |
| LEDs | `Rule1` position rules | `interval` + lambda |
| Buttons | `ShutterButton1` | `binary_sensor` + lambda |

## Files

- `window-actuator.yaml` — the ESPHome configuration.
- `secrets.yaml.example` — template for credentials.
- `.gitignore` — keeps `secrets.yaml` and build artifacts out of git.
- `LICENSE` — MIT.
