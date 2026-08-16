# Infinity Clock

ESPHome firmware for an analogue ring clock built from 60 addressable LEDs. The hour
and minute hands are drawn as a colour gradient around the ring, accompanied by a
handful of animation effects (fire, moon phase, dawn).

The whole device — configuration and all C++ animations — lives in a single file,
[`infinity.yaml`](infinity.yaml). No extra headers or external components are needed.

## Hardware

| Part | Value |
| --- | --- |
| Board | ESP32-C3 (`esp32-c3-devkitm-1`) |
| Framework | ESP-IDF |
| LED ring | 60× WS2811, GRB, on `GPIO02`, driven via RMT (`esp32_rmt_led_strip`) |
| Brightness sensor | ADC on `GPIO01` |

The ring needs its own 5 V supply — 60 LEDs at full white draw considerably more than
the board's USB port can deliver.

60 LEDs is not a decorative choice: one pixel per minute means a pixel index *is* the
minute (and second) value, and the hour hand lands on `hour * 5`. All of the effect
and indicator maths assumes that mapping.

Note the framework: no Arduino-only APIs (`String`, `WiFi.*`, `byte`, the `min`/`max`
macros) work in a lambda here. The effects use `std::min` / `std::max` / `std::abs`
for that reason.

## Getting started

1. Clone the repository and create `secrets.yaml` next to `infinity.yaml`:

   ```yaml
   wifi_ssid: "..."
   wifi_password: "..."
   wifi_ap_password: "..."      # fallback access point "infinity"
   ota_password: "..."
   api_encryption_key: "..."    # 32 bytes, base64 — e.g. `openssl rand -base64 32`
   mqtt_broker: "..."
   mqtt_username: "..."
   mqtt_password: "..."
   ```

   The file is listed in `.gitignore` and must never be committed.

2. Build and flash:

   ```sh
   esphome run infinity.yaml
   ```

   Use USB for the first flash; after that OTA works over the network. The web
   interface is then available at `http://infinity.local/`.

3. Calibrate the ambient light sensor — see below.

## Calibration

All tunables live in `substitutions:` at the top of `infinity.yaml`:

| Substitution | Purpose |
| --- | --- |
| `DEFAULT_BRIGHTNESS` | brightness the clock returns to on boot and on switch-on |
| `AMBIENT_DARK` / `AMBIENT_BRIGHT` | Brightness sensor readings in a dark room and in full daylight |
| `MIN_FACE_BRIGHTNESS` | how far the face may dim at night, as a fraction — never `0` |
| `DAWN_DURATION_S` | length of the `Dawn` ramp |

`AMBIENT_DARK` and `AMBIENT_BRIGHT` ship as rough starting values; the divider on
`GPIO01` decides the real range. Read the `Brightness sensor` entity once in the dark
and once in the sun and put those two numbers in. **If the clock dims the wrong way
round, swap the two values** — the mapping is linear and handles a negative span on
its own, so no code change is needed.

## Networking

The node is deliberately configured to keep running on its own:

- **API** with encryption and `reboot_timeout: 0s` — no reboot when Home Assistant is unreachable.
- **MQTT** with `reboot_timeout: 0s` and `discovery: False`; entities have to be added to Home Assistant by hand.
- **WiFi** with `fast_connect`, power saving disabled, plus a fallback access point and captive portal.
- **Web server** (version 3, assets served locally) on port 80.
- **Time** via SNTP against `pool.ntp.org`, timezone `Europe/Berlin`.

## Entities

**Light** — `LED color`: the whole ring, dimmable and colourable, carrying all effects.

**Clock controls**

| Entity | Type | Values |
| --- | --- | --- |
| `Face type` | Select | `Normal`, `Darken to midday` |
| `Indicator` | Select | `None`, `Show midday`, `Show quadrants`, `Show hour marks` |
| `Enable seconds` | Switch | shows the second hand |
| `Color hour` / `Color minute` / `Color second` | Text | hex colour, e.g. `#FF0000` |

**Diagnostics** — ESPHome Version, Firmware Version, Device Uptime, Uptime, SSID,
IP Address, DNS Address, Connection Status, WiFi Signal (dBm), WiFi Signal (%),
Brightness sensor.

**Buttons** — Restart, Restart (Safe Mode), Factory Reset.

## Effects

| Effect | Description |
| --- | --- |
| `Time` | the clock face: hour, minute and optional second hand as a gradient around the ring, in the colours set by the `Color …` entities, dimmed to ambient light |
| `Fire` | fire simulation, mirrored across both halves of the ring |
| `Moon` | current moon phase as a lit segment, from the synodic month against a fixed new-moon epoch |
| `Dawn` | wake-up ramp: opens from midday outwards and warms from red through amber to daylight over `DAWN_DURATION_S`, then holds |
| `Alarm` | red pulse, roughly two seconds per cycle |

Plus the ESPHome built-ins: Rainbow, Color Wipe, Scan, Twinkle, Random Twinkle,
Fireworks and Flicker.

Each effect declares its own `update_interval`, matched to how fast it can actually
change — 500 ms for `Time`, 30 ms for `Fire` and `Alarm`, 1 s for `Dawn` and `Moon`.
Left at the ESPHome default of `0ms` a lambda re-renders all 60 pixels on every
main-loop pass.

`Dawn` does not follow the real sunrise: that needs latitude and longitude via the
`sun` component, which this device does not configure. It is a ramp on its own frame
counter, restarted whenever the effect is selected.

`gamma_correct` is set to `1.0` rather than the ESPHome default of `2.8`. Gamma is
right for a light dimmed by hand, but every effect here computes its own gradient —
applying gamma on top crushes the dark end of the hour/minute blend into black.

### Boot behaviour

The ring briefly runs `Twinkle`, then settles on `Time` at `DEFAULT_BRIGHTNESS` once
all components are up. Switching the light on from Home Assistant always returns to
that same state, whatever effect or colour was set before. The ring is turned off on
shutdown.

## State that survives a reboot

`Enable seconds`, `Face type`, `Indicator` and the three `Color …` entities all
persist. Their `restore_mode` / `restore_value` settings are spelled out in the YAML
rather than left to the schema defaults, which are `ALWAYS_OFF` and `false` — with the
defaults every setting was silently lost on each restart. `initial_value` /
`initial_option` therefore only apply on the very first boot, or after a factory
reset.

The option **order** of `Face type` and `Indicator` is load-bearing: the `Time` effect
switches on `active_index()`, so reordering or inserting an option silently remaps the
face.

## Known limitations

- **The `on_boot` priority 200 block never fires.** All `on_boot` triggers run during
  `setup()`, milliseconds apart — the priorities order them, they do not spread them
  over time — so WiFi has not associated yet and the condition is always false. A real
  "waiting for network" indicator needs a `wifi:` `on_connect` / `on_disconnect`
  trigger. The block is kept, commented, to document the trap.
- **`Dawn` is not tied to the actual sunrise**, see above.

## License

[GNU General Public License v3.0](LICENSE)
