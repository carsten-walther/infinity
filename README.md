# Infinity Clock

ESPHome firmware for an analogue ring clock built from 60 addressable LEDs. The hour
and minute hands are drawn as a colour gradient around the ring, accompanied by a
handful of animation effects (fire, moon phase, sun).

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
minute (and second) value, and the hour hand belongs at `(hour % 12) * 5` — plus
`minute / 12`, so it creeps through the hour rather than jumping on it. All of the
effect and indicator maths assumes that mapping.

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

   **Build from a local disk.** This config uses the ESP-IDF framework, and IDF's
   CMake writes compiler probe binaries and reads them straight back. Over SMB or
   NFS that round-trip is unreliable and the build dies during ABI detection with
   `CMakeDetermineCompilerABI_CXX.bin cannot be read` / `CMAKE_C_COMPILER not set`.
   Nothing is wrong with the config — move the checkout to local storage. Building
   through the Home Assistant ESPHome add-on is unaffected, since that compiles on
   the Home Assistant host's own filesystem.

3. Calibrate the ambient light sensor — see below.

## Calibration

All tunables live in `substitutions:` at the top of `infinity.yaml`:

| Substitution | Purpose |
| --- | --- |
| `DEFAULT_BRIGHTNESS` | brightness the clock returns to on boot and on switch-on |
| `IDLE_RED` / `IDLE_GREEN` / `IDLE_BLUE` | the colour the ring carries underneath the effects |
| `AMBIENT_DARK` / `AMBIENT_BRIGHT` | LDR divider voltage in a dark room and in full daylight |
| `MIN_FACE_BRIGHTNESS` | how far the face may dim at night, as a fraction — never `0` |
| `DEFAULT_LATITUDE` / `DEFAULT_LONGITUDE` | seed values for the `Latitude` / `Longitude` entities on first boot |
| `SUN_WHITE_ELEVATION` | solar elevation at which `Sun` reads as plain daylight |
| `FIRE_COOLING` / `FIRE_SPARKING` | how short and how active the `Fire` flames are |
| `FIRE_MAX_HEAT` | caps the fire palette short of white — raise towards `255` for white tips |
| `ALARM_RED` / `ALARM_GREEN` / `ALARM_BLUE` | peak colour of the `Alarm` pulse |
| `GRADIENT_DIM` | how far the `Gradient` face dims its ramps, so the hands still stand out |
| `NUM_LEDS` | ring size — see the note above, this is not a free parameter |
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
- **WiFi** with `fast_connect`, light power saving (see below), plus a fallback access point and captive portal.
- **Web server** (version 3, assets served locally) on port 80.
- **Time** via SNTP against `pool.ntp.org`, timezone `Europe/Berlin`.

### Power and heat

The die ran at 67 °C with `power_save_mode: NONE` and the CPU at its default 160 MHz —
warm, though well inside the C3's 125 °C limit. With the LED ring on its own supply,
none of that came from the strip. Two settings brought it down to **38–41 °C**
(measured 2026-08-16, ten minutes after boot):

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
| `Face type` | Select | `Normal`, `Simple`, `Gradient` |
| `Indicator` | Select | `Off`, `Show midday`, `Show quadrants`, `Show hour marks` |
| `Enable seconds` | Switch | shows the second hand |
| `Color hour` / `Color minute` / `Color second` | Text | hex colour, e.g. `#FF0000` |
| `Latitude` / `Longitude` | Text | decimal degrees, north and east positive — drives the `Sun` and `Moon` effects |
| `Alarm Time` | Datetime | daily alarm, switches the ring to the `Alarm` effect |
| `Alarm Enabled` | Switch | arms `Alarm Time` without losing the setting |
| `Alarm Duration` | Number | minutes the alarm pulses before standing down, 1–60 |

The alarm switches the ring on, selects the `Alarm` effect, and stands down by itself
after `Alarm Duration` minutes. There is deliberately no snooze.

Standing down restores what was there before: if the ring had been off, it goes back
off; if it had been showing the clock, it returns to the clock. It also checks that the
alarm is still what is showing — someone who silenced it early by picking another
effect has already dealt with it, and changing the display out from under them minutes
later would be worse than doing nothing.

The script runs in `mode: restart`, so an alarm firing while a previous one is still
running restarts the timer rather than queueing a second stand-down.

The light's `on_turn_on` handler only normalises a switch-on that did not ask for a
specific effect — otherwise it would override the alarm the instant it fired.

Effects are set on the `LED color` light entity itself — Home Assistant lists them in
its more-info dialog, and automations use `light.turn_on` with `effect:`. There is
deliberately no separate `Effect` select: it duplicated a native capability, and its
option list was a hand-maintained copy of the effect names, which is the most
drift-prone thing a config like this can carry.

**Diagnostics** — ESPHome Version, Firmware Version, Device Uptime, Uptime,
Reset Reason, Reset Count, Internal Temperature, Sun Elevation, Moon Altitude,
Moon Illumination, Moon Phase, SSID, IP Address, DNS Address, Connection Status,
WiFi Signal (dBm), WiFi Signal (%), Brightness sensor.

`Sun Elevation`, `Moon Altitude`, `Moon Illumination` and `Moon Phase` come free: the
`Sun` and `Moon` effects need those numbers anyway, so exposing them turns the clock
into a usable source for automations without adding an integration for it. They are
pushed by the interval that computes them rather than polled, so they can never report
something the ring is not showing.

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
| `Moon` | the actual moon: the ring lights the illuminated fraction of the disc while the moon is above the horizon, and is dark when it has set |
| `Sun` | the real sun: lights at 12 o'clock as it clears the horizon, opens outwards as it climbs, closes the ring at solar noon, in the sun's own colour |
| `Alarm` | red pulse, roughly two seconds per cycle |

Plus the ESPHome built-ins: Rainbow, Color Wipe, Scan, Twinkle, Random Twinkle,
Fireworks and Flicker.

Each effect declares its own `update_interval`, matched to how fast it can actually
change — 500 ms for `Time`, 30 ms for `Fire` and `Alarm`, 10 s for `Sun` and `Moon`.
Left at the ESPHome default of `0ms` a lambda re-renders all 60 pixels on every
main-loop pass.

`Sun` simulates the sun in real time. It computes the solar elevation for the
device's own coordinates and shows it directly: dark below the horizon, one pixel at
12 o'clock the moment the sun rises, opening outwards as it climbs, the full circle at
solar noon, then closing again towards sunset. The colour follows the sun too — deep
red at the horizon through orange and yellow to daylight white above
`SUN_WHITE_ELEVATION`.

Berlin, 16 August, times UTC:

| UTC | Elevation | Pixels lit | Colour |
| --- | --- | --- | --- |
| 03:00 | −7.5° | 0 | dark |
| 04:00 | +0.5° | 3 | `(142, 2, 0)` deep red |
| 06:00 | 18.3° | 23 | `(205, 193, 5)` yellow |
| 11:00 | 51.1° | 60 | `(255, 255, 255)` daylight |
| 17:00 | 12.3° | 17 | `(142, 62, 0)` orange |
| 19:00 | −4.9° | 0 | dark |

The arc is normalised against the highest the sun gets *that day*
(`90° − |latitude − declination|`), so the ring closes at local noon in December as
well as in June.

The position comes from the low-precision solar algorithm in the Astronomical Almanac,
accurate to well under a tenth of a degree — a hundred times finer than one pixel of
this ring. It reads UTC from the clock, so the `Europe/Berlin` timezone affects only
the clock face, not this.

### Where the coordinates come from

`Latitude` and `Longitude` are text entities, and they are what the `Sun` and `Moon`
effects read. Two
`internal` `homeassistant` sensors import `zone.home`'s `latitude` and `longitude`
attributes and write them into those entities, so a device added through the ESPHome
integration is correct without anyone typing anything.

The `Color …`, `Latitude` and `Longitude` entities all sit in Home Assistant's
**configuration** category rather than among the controls — they are set once and then
left alone.

**Home Assistant wins.** Editing the entities by hand works, but the next time HA
connects or `zone.home` changes it overwrites them. Delete the two `homeassistant`
sensors if the device should keep its own coordinates.

Without Home Assistant — MQTT only, or standalone — the imports never publish and the
text entities simply keep their stored value; with no stored value the effect falls
back to `DEFAULT_LATITUDE` / `DEFAULT_LONGITUDE`.

The import compares the formatted value before writing. The text entities use
`restore_value`, so every write reaches flash, and `zone.home` republishes unchanged on
each reconnect.

`Moon` follows the real moon, and only shows it when it is actually up.

The lit arc is the illuminated fraction of the disc, taken from the true elongation
between moon and sun, so it grows slowly around new and full and quickly around the
quarters. Below the horizon the ring is dark.

| Phase | Ring |
| --- | --- |
| new | dark |
| day 3 | 7 of 60 pixels |
| first quarter | 33 of 60 |
| full | all 60 |

The position comes from the moon's orbital elements plus the twelve largest
perturbation terms in longitude and five in latitude. They are not optional: the
evection term alone is 1.27°, which is several minutes of moonrise. Topocentric
parallax is applied too — the moon is near enough that where you stand moves it by
about a degree. Accuracy is a few hundredths of a degree, and moonrise lands within a
minute or two of an almanac. The threshold is the altitude of the moon's *centre*, so
it differs slightly from the almanac definition, which uses the upper limb and allows
for refraction.

The lit limb faces the sun: right while waxing in the northern hemisphere, mirrored
south of the equator — taken from the `Latitude` entity, so this is correct wherever
the device is.

`MOON_BRIGHTNESS` sets the peak level.

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

The three faces:

| Face | What it draws |
| --- | --- |
| `Normal` | the two arcs between the hands, each in a flat hand colour |
| `Simple` | the hands as bare dots on a dark ring |
| `Gradient` | both arcs as colour ramps — hour colour to minute colour one way round, minute back to hour the other |

`Gradient` dims its ramps to `GRADIENT_DIM` and paints the hands over them at full
strength. Without that the hands would vanish: a gradient is by definition almost the
same colour either side of any point on it.

`Indicator` and `Enable seconds` apply to every face, so hour marks and a second dot
work on `Simple` and `Gradient` too.

The option **order** of `Face type` and `Indicator` is load-bearing: the `Time` effect
switches on `active_index()`, so reordering or inserting an option silently remaps the
face.

## Known limitations

- **The `on_boot` priority 200 block never fires.** All `on_boot` triggers run during
  `setup()`, milliseconds apart — the priorities order them, they do not spread them
  over time — so WiFi has not associated yet and the condition is always false. A real
  "waiting for network" indicator needs a `wifi:` `on_connect` / `on_disconnect`
  trigger. The block is kept, commented, to document the trap.

## License

[GNU General Public License v3.0](LICENSE)

## Architecture

Two pieces of shared state are computed once and read everywhere, rather than
recalculated per effect:

- **`interval: 10s`** computes solar and lunar position into globals. Both the `Sun`
  and `Moon` effects need the sun — `Moon` for the elongation that gives the phase and
  for sidereal time — and the four sky diagnostics need both. Three copies of the same
  trigonometry would be three chances to fix a bug in only two of them. The effect
  lambdas read globals and draw; no double-precision astronomy sits on a render path.
- **`global_ambient_dim`** holds the volts-to-brightness mapping, computed on the
  `Brightness sensor`. `Time`, `Fire`, `Sun` and `Moon` each scale their finished frame
  by it. `Alarm` deliberately does not — an alarm that turns itself down in a dark room
  fails at the one moment it is needed.

The two solar algorithms were cross-checked before being merged: the Astronomical
Almanac's low-precision formula and Schlyter's orbital elements agree to 0.003° over a
full year, so one set of elements now serves both bodies.

### Boot and network

`on_boot` lights `Twinkle`, and nothing switches to the clock face until WiFi
`on_connect` fires — so the busy indicator is real. It used to be a boot trigger at
priority 200 testing `wifi.connected`, which could never be true: every `on_boot`
trigger runs inside `setup()`, milliseconds apart, long before the radio associates.

There is deliberately no `on_disconnect` counterpart, and `on_connect` only acts if the
ring is still on `Twinkle`. The clock runs off the system clock, not off the network,
so losing WiFi is no reason to stop showing the time — and a reconnect should not undo
whatever was selected in Home Assistant.

The system clock takes two sources: SNTP, and Home Assistant. Whichever sets it first
makes every time component valid, which covers the case where the internet is down but
the LAN is fine.

An OTA update parks the light for its duration — the effect lambdas and RMT transfers
otherwise compete with the transfer for CPU and for the WiFi stack, which matters more
now that the radio sleeps between beacons.
