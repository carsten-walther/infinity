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
| LED ring | 60× WS2811, GRB, on `GPIO10`, driven via RMT (`esp32_rmt_led_strip`) |
| Brightness sensor | LDR divider on `GPIO01` (ADC1, 12 dB attenuation) |

**The ring needs its own 5 V supply.** 60 LEDs at full white draw around 3.6 A, and
even the normal clock face at 70 % brightness lights all 60 pixels for something in
the region of 1–2 A. Feeding that through the DevKit's `5V` pin routes the whole
current across the USB connector, the board traces and the ground return — far beyond
what a devkit is built for, and the usual reason one of these boards "runs hot". Power
the ring directly from the supply and join only the grounds.

60 LEDs is not a decorative choice: one pixel per minute means a pixel index *is* the
minute (and second) value, and the hour hand lands on `hour * 5`. All of the effect
and indicator maths assumes that mapping.

Note the framework: no Arduino-only APIs (`String`, `WiFi.*`, `byte`, the `min`/`max`
macros) work in a lambda here. The effects use `std::min` / `std::max` / `std::abs`
for that reason.

### Choosing pins on the ESP32-C3

Both pins are set as substitutions at the top of `infinity.yaml`. If you rewire, these
are the constraints on this board:

| GPIO | Usable for |
| --- | --- |
| `0`, `1`, `3`, `4` | analogue in (ADC1) or digital |
| `5` | digital only — ADC2, which cannot be read while WiFi is up |
| `6`, `7`, `10` | digital, no second function |
| `2`, `8`, `9` | **strapping pins** — avoid; `8` also drives the on-board RGB LED, `9` is the BOOT button |
| `11`–`17` | SPI flash, unavailable |
| `18`, `19` | USB Serial/JTAG |
| `20`, `21` | UART0 console |

The LED ring sat on `GPIO02` originally. That boots, because the pin is
high-impedance at reset and a WS2811 data input does not pull it down — but a level
shifter, a pulldown or a long noisy cable turns it into a board that occasionally
comes up in download mode. RMT reaches any pin through the GPIO matrix, so there is
no reason to spend a strapping pin on it.

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
| `IDLE_RED` / `IDLE_GREEN` / `IDLE_BLUE` | the colour the ring carries underneath the effects |
| `AMBIENT_DARK` / `AMBIENT_BRIGHT` | LDR divider voltage in a dark room and in full daylight |
| `MIN_FACE_BRIGHTNESS` | how far the face may dim at night, as a fraction — never `0` |
| `DAWN_DURATION_S` | length of the `Dawn` ramp |
| `FIRE_COOLING` / `FIRE_SPARKING` | how short and how active the `Fire` flames are |
| `FIRE_MAX_HEAT` | caps the fire palette short of white — raise towards `255` for white tips |
| `PIN_LED_RING` / `PIN_BRIGHTNESS` | see the pin table above before changing |

`AMBIENT_DARK` and `AMBIENT_BRIGHT` are in **volts**, matching what the
`Brightness sensor` entity publishes. Measured on the built device:

| Condition | Reading |
| --- | --- |
| LDR covered | 0.09 V |
| normal room light | 0.93 V |
| torch on the sensor | 2.96 V |

The shipped values (`0.10` / `1.20`) are deliberately **not** those extremes. An LDR
divider is heavily non-linear and most of its swing sits above anything a living room
reaches — mapping the full 0.09–2.96 V span would leave the clock at ~40 % brightness
in a normally lit room. With the indoor window instead, a dark room gives the
`MIN_FACE_BRIGHTNESS` floor of 15 %, normal room light ~79 %, and anything from
1.20 V up clamps to full. Widen `AMBIENT_BRIGHT` if the clock stays too bright in the
evening.

**If the clock dims the wrong way round, swap the two values** — the mapping is linear
and handles a negative span on its own, so no code change is needed. The measured
polarity on this build is dark = low, bright = high, so no swap is needed.

The ADC runs at `attenuation: 12db` (≈ 0–3.1 V) rather than the ESPHome default of
`0db`, which caps at ≈ 1.1 V. With the default, every reading from a half-lit room
upwards clipped to the same flat maximum — precisely the end of the scale the dimming
has to resolve.

It is also sampled once a second over 8 hardware reads and published every fifth, so
the value is a 20-second moving average at the same 0.2 Hz as a plain reading. The
`Time` effect reads the sensor's state, so the face only changes brightness when the
sensor publishes — a single raw sample every 5 s made that step visibly, from ADC
noise as much as from a passing cloud. The trade-off is latency: switching a lamp on
takes about 20 s to fully register.

## Networking

The node is deliberately configured to keep running on its own:

- **API** with encryption and `reboot_timeout: 0s` — no reboot when Home Assistant is unreachable.
- **MQTT** with `reboot_timeout: 0s` and `discovery: False`; entities have to be added to Home Assistant by hand.
- **WiFi** with `fast_connect`, power saving disabled, plus a fallback access point and captive portal.
- **Web server** (version 3, assets served locally) on port 80.
- **Time** via SNTP against `pool.ntp.org`, timezone `Europe/Berlin`.

### Power and heat

The die ran at 67 °C with `power_save_mode: NONE` and the CPU at its default 160 MHz —
warm, though well inside the C3's 125 °C limit. With the LED ring on its own supply,
none of that came from the strip, so two settings address it:

- `power_save_mode: LIGHT` (`WIFI_PS_MIN_MODEM`) — sleeps between DTIM beacons instead
  of keeping the radio powered permanently. Only **inbound** traffic is delayed, by up
  to one beacon interval (typically 100–300 ms); outbound publishes and the clock
  itself are unaffected. If OTA updates start failing, this is the first thing to put
  back to `NONE`.
- `cpu_frequency: 80MHZ` — half the default. Nothing here is compute-bound; raise it
  back to `160MHZ` if an effect ever stutters.

`output_power: 12dB` is a reduction from the ~20 dBm default, but it barely matters —
this node transmits well under 0.1 % of the time. Do not go lower: at the measured
RSSI of −63 to −65 dBm a weaker uplink causes retransmits, which cost more airtime
than the lower power saves.

Note the ESP32-WROOM-32 would run *hotter*, not cooler: two Xtensa cores, a 240 MHz
default clock, higher receive current — and no working die sensor to measure it with.

## Entities

**Light** — `LED color`: the whole ring, dimmable and colourable, carrying all effects.

**Clock controls**

| Entity | Type | Values |
| --- | --- | --- |
| `Effect` | Select | any effect from the table below, or `None` to switch the ring off |
| `Face type` | Select | `Normal`, `Darken to midday` |
| `Indicator` | Select | `None`, `Show midday`, `Show quadrants`, `Show hour marks` |
| `Enable seconds` | Switch | shows the second hand |
| `Color hour` / `Color minute` / `Color second` | Text | hex colour, e.g. `#FF0000` |

`Effect` is a convenience wrapper — the `LED color` light entity already carries the
effect list natively in its more-info dialog. The select exists so the effect can sit
on a dashboard as its own control and be set from an automation without a
`light.turn_on` service call. It syncs both ways: changing the effect on the light
updates the select.

Its `None` entry switches the ring **off**. In ESPHome, "None" means "no effect, show
the light's plain colour" — and that colour is white here, because `on_boot` and
`on_turn_on` both set red, green and blue to 100 %. Picking it therefore lit the whole
ring white, which is not what a `None` entry suggests. The option keeps its ESPHome
name so the reverse sync still matches: `get_effect_name()` reports `None` whenever no
effect runs, the light being off included.

**Diagnostics** — ESPHome Version, Firmware Version, Device Uptime, Uptime,
Reset Reason, Reset Count, Internal Temperature, SSID, IP Address, DNS Address,
Connection Status, WiFi Signal (dBm), WiFi Signal (%), Brightness sensor.

`Internal Temperature` is the SoC die sensor, not the enclosure. The C3's absolute
maximum is 125 °C and anything up to ~60 °C under load is unremarkable for a QFN part
with no heatsink. If the device runs hot, switch the light off and watch this for ten
minutes: a clear drop means the heat comes from the LED supply path, not the ESP32 —
see below.

`Reset Reason` and `Reset Count` belong together: the reason comes straight from
`esp_reset_reason()` and tells you *why* the device last restarted, the count tells
you *that* it restarted while nobody was watching. Read both against `Device Uptime` —
a rising count at a low uptime is a device rebooting in a loop. The count survives
restarts and is only cleared by a factory reset.

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
