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
| LED ring | 60× WS2812, GRB, on `GPIO10`, driven via RMT (`esp32_rmt_led_strip`) |
| Brightness sensor | LDR divider on `GPIO01` (ADC1, 12 dB attenuation) |
| RTC | DS1307 ("Tiny RTC") on i²c, address `0x68`, `GPIO06` = SDA, `GPIO07` = SCL, **powered from 3V3** |
| Supply | 12 V power supply → MP1584EN step-down set to 5 V → ring and board in parallel |

**The ring's current must never cross the DevKit.** 60 LEDs at full white draw around
3.6 A, and even the normal clock face at 70 % brightness lights all 60 pixels for
something in the region of 1–2 A. Feeding that through the DevKit's `5V` pin routes the
whole current across the USB connector, the board traces and the ground return — far
beyond what a devkit is built for, and the usual reason one of these boards "runs hot".
Ring and board therefore hang off the supply as two separate pairs of wires that meet
only at the terminals they both come from — see [The 5 V supply](#the-5-v-supply).

60 LEDs is not a decorative choice: one pixel per minute means a pixel index *is* the
minute (and second) value, and the hour hand belongs at `(hour % 12) * 5` — plus
`minute / 12`, so it creeps through the hour rather than jumping on it. All of the
effect and indicator maths assumes that mapping.

Note the framework: no Arduino-only APIs (`String`, `WiFi.*`, `byte`, the `min`/`max`
macros) work in a lambda here. The effects use `std::min` / `std::max` / `std::abs`
for that reason.

### The 5 V supply

A 12 V power supply feeds an **MP1584EN** adjustable step-down module set to 5 V, and
the ring and the board hang off its output in parallel:

```
12 V PSU ──▶ IN+ ─┐                ┌─ OUT+ ──┬──▶ ESP32-C3  5V
                  │   MP1584EN     │         └──▶ LED ring  +
         ──▶ IN− ─┘   12 V → 5 V   └─ OUT− ──┬──▶ ESP32-C3  GND
                                             └──▶ LED ring  −
```

Photos: [`docs/Step-Down-MP1584-001.jpg`](docs/Step-Down-MP1584-001.jpg) (top),
[`docs/Step-Down-MP1584-002.jpg`](docs/Step-Down-MP1584-002.jpg) (bottom). The bottom
silkscreen is the entire pinout — `IN+`/`IN−` at one end, `OUT+`/`OUT−` at the other, an
arrow pointing from in to out. On top sit the MP1584EN in SOIC-8, a 4.7 µH inductor
(`4R7`), an `SS34` Schottky and the trimmer that sets the output voltage. The MP1584 is
a **non-synchronous** buck: it integrates the high-side switch only, which is why the
catch diode is a discrete part on the board instead of a second MOSFET inside the chip.
That matters for where the heat ends up — see below.

**Set the output before the load is connected.** These modules ship at whatever the
trimmer was last left at, the trimmer is multiturn with no marked direction, and the
first thing a wrong setting reaches is 60 LEDs and an ESP32. Meter on `OUT`, nothing
else attached. **Measured here: 5.0 V.**

#### The current budget

12 V in and 5 V out is a real conversion ratio, so the converter trades voltage for
current and the input side is comfortable. Reckoning with roughly 85 % efficiency:

| Ring state | Ring current at 5 V | Power | Drawn from the 12 V supply |
| --- | --- | --- | --- |
| Clock face at 70 % (normal operation) | 1–2 A | 5–10 W | 0.5–1.0 A |
| Full white at 100 % | 3.6 A | 18 W | ≈ 1.8 A |

A 12 V/2 A brick therefore covers even full white. **The converter does not.** The
MP1584 is a 3 A part, and 3 A is the die rating rather than what a module this size
dissipates without a heatsink or a copper pour — so full white asks 3.6 A of a component
that runs out somewhere below 3 A.

Nothing in `infinity.yaml` prevents asking: there is no `max_power`, so the light can be
set to 100 % white. At the default 70 % on the clock face the ring stays near 1–2 A and
everything in the chain is inside its rating with margin, which is where this is meant
to run.

Because the converter is non-synchronous, the losses land mostly in the **`SS34`**, not
in the chip. At a duty cycle of (5 + 0.5)/(12 + 0.5) ≈ 0.44 the diode carries the output
current for the other 56 % of every cycle, at ~0.5 V forward:

| Ring current | `SS34` | MP1584 switch (≈ 100 mΩ) |
| --- | --- | --- |
| 2 A | ≈ 0.56 W | ≈ 0.18 W |
| 3.6 A | ≈ 1.0 W | ≈ 0.57 W |

The `SS34` is an SMA-package part on a small board. Around half a watt it is warm; near
a watt it is the component that decides how long this runs. If the module ever needs to
deliver more, that diode is the thing to fix — a synchronous module has no catch diode
at all — and the answer is a different converter rather than a bigger heatsink.

#### One thing that is marginal on paper

A WS2812's specified input threshold is 0.7 × V_DD. The rail here measures 5.0 V, which
puts that threshold at **3.5 V**, and an ESP32-C3 pin drives 3.3 V — so the data line is
nominally *below* spec. This is the standard reason a 5 V pixel ring on a 3.3 V
controller works "mostly", with the occasional wrong pixel and no obvious cause. It is
not a rounding-error miss: 5.0 V is the worst case for this margin, and any converter
setting below it would improve matters.

It works here, and the flicker this device used to show turned out to be a timing
mismatch rather than a level problem (see [Power and heat](#power-and-heat)). Two
details make the margin less alarming than the numbers suggest: real WS2812Bs switch
well below the specified threshold, and only the **first** pixel ever sees the 3.3 V
signal — every pixel after it is driven by its predecessor's reshaped 5 V output. So a
level failure would not look like scattered wrong pixels; it would take out the whole
ring at once, because pixel 1 misread the frame everyone else is fed from.

Worth knowing rather than worth fixing. If that symptom ever appears, the cheapest
remedy is the trimmer: dropping the rail to 4.5 V puts the threshold at 3.15 V and buys
real margin, at the cost of a little headroom on the blue die. Below roughly 4 V blue
runs out of forward voltage before red and green do, and flat white drifts warm.

### Choosing pins on the ESP32-C3

Board photo and full pinout: [`docs/ESP32-C3.jpg`](docs/ESP32-C3.jpg),
[`docs/ESP32-C3-Pinout.jpg`](docs/ESP32-C3-Pinout.jpg).

All pins are set as substitutions at the top of `infinity.yaml`. If you rewire, these
are the constraints on this board:

| GPIO | Usable for | Used here |
| --- | --- | --- |
| `0`, `1`, `3`, `4` | analogue in (ADC1) or digital | `1` — LDR |
| `5` | digital only — ADC2, which cannot be read while WiFi is up | |
| `6`, `7`, `10` | digital, no second function | `6`/`7` — i²c, `10` — LED ring |
| `2`, `8`, `9` | **strapping pins** — avoid; `8` also drives the on-board RGB LED, `9` is the BOOT button | |
| `11`–`17` | SPI flash, unavailable | |
| `18`, `19` | USB Serial/JTAG | |
| `20`, `21` | UART0 console | |

All four pins in use are outside the strapping set. `6`, `7` and `10` are free of a
second function because the `ESP32-C3-MINI-1` module carries its flash in-package on
`11`–`17`; on a bare C3 with external flash those pins are not necessarily spare.

Keeping the i²c bus off the strapping pins matters more than it does for the other
peripherals. An idle i²c line rests **high** on its pull-ups, and those pull-ups live
on the RTC module — so an unpowered, unplugged or miswired module leaves both lines
low, which a strapping pin cannot tell from a deliberate pull-down. On `GPIO8`/`GPIO9`
that is a board that boots into the ROM bootloader; on `GPIO6`/`GPIO7` it is a bus
error and a clock that falls back to SNTP.

The LED ring sat on `GPIO02` originally. That boots, because the pin is
high-impedance at reset and a WS2812 data input does not pull it down — but a level
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
| `HAND_FADE_MS` | how long a hand takes to step to its next pixel |
| `HAND_TAIL` | length of the trail behind a hand, in pixels |
| `DEFAULT_LATITUDE` / `DEFAULT_LONGITUDE` | seed values for the `Latitude` / `Longitude` entities on first boot |
| `SUN_WHITE_ELEVATION` | solar elevation at which `Sun` reads as plain daylight |
| `FIRE_COOLING` / `FIRE_SPARKING` | how short and how active the `Fire` flames are |
| `FIRE_MAX_HEAT` | caps the fire palette short of white — raise towards `255` for white tips |
| `ALARM_RED` / `ALARM_GREEN` / `ALARM_BLUE` | peak colour of the `Alarm` pulse, scaled by its own 0–255 ramp |
| `GRADIENT_DIM` | how far the `Gradient` face dims its ramps, so the hands still stand out |
| `FACE_BLEND` | how wide the `Normal` face's two ramps are, `0` = two flat arcs, `1` = both arcs start on the same colour |
| `NUM_LEDS` | ring size — see the note above, this is not a free parameter |
| `PIN_LED_RING` / `PIN_BRIGHTNESS` | see the pin table above before changing |

`AMBIENT_DARK` and `AMBIENT_BRIGHT` are in **volts**, matching what the
`Brightness` entity publishes. Measured on the built device:

| Condition | Reading |
| --- | --- |
| LDR covered | 0.09 V |
| normal room light | 0.93 V |
| torch on the sensor | 2.96 V |

The shipped values (`0.10` / `1.20`) are deliberately **not** those extremes. An LDR
divider is heavily non-linear and most of its swing sits above anything a living room
reaches — mapping the full 0.09–2.96 V span would leave the clock at ~40 % brightness
in a normally lit room. With the indoor window instead, a dark room gives the
`MIN_FACE_BRIGHTNESS` floor of 10 %, normal room light ~78 %, and anything from
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
- **MQTT** is **off at boot**, with `reboot_timeout: 0s` and `discovery: False`. Everything the device does works without a broker — the clock runs from SNTP and Home Assistant reaches it over the native API — so it comes up in the cheapest state and MQTT is switched on when wanted, from the `MQTT` switch.
- **WiFi** with light power saving (see below), plus a fallback access point and captive portal. `fast_connect` is **off**: it skips the scan and takes the first access point that answers, which is right for a hidden SSID and wrong everywhere else — with several APs on one SSID the device would ignore the strongest. Turn it on only if this network is hidden.
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
- `cpu_frequency: 80MHZ` — half the default. **This one has since been reverted**, see
  below; the 38–41 °C figure was measured with it in place and so understates what the
  power save mode is worth on its own.

`cpu_frequency` is back at the variant default of `160MHZ`. Halving the clock looked
free — nothing here is compute-bound — but it also halves the margin on **interrupt
latency**, and the LED ring depends on that. `esp32_rmt_led_strip` drives the strip
from a 96-symbol RMT buffer (the C3 maximum: two 48-symbol blocks, so there is no
headroom to configure), which at WS2812 timing holds about 134 µs of data. The refill
ISR therefore has to run every ~67 µs for the whole ~2 ms a frame takes. Miss it once
and every LED past that point takes shifted bits — a handful of pixels in the wrong
colour for one frame, on a chip that is also servicing WiFi, the API, MQTT and a web
server. If the die runs hotter than wanted, take it out of `power_save_mode` or the
brightness setting rather than out of the clock.

That is an argument for keeping margin, not a diagnosis. This device *did* show brief
wrong-coloured pixels for a while, and an RMT underrun was the standing theory — but the
cause turned out to be simpler: the strip is WS2812 and `chipset:` was set to `WS2811`.
The bit periods are near enough (1.39 vs 1.40 µs) that it worked, but `WS2811` sends a
300 ns `T0H` against a WS2812B's 0.45 µs upper bound for a zero, leaving almost no
margin at the bottom — and a bit that tips there looks exactly like a stray pixel.
Correcting the chipset fixed it. The underrun budget above is real and still the reason
the clock stays at 160 MHz; it just was not what was happening here.

`output_power: 12dB` is a reduction from the ~20 dBm default, but it barely matters —
this node transmits well under 0.1 % of the time. Do not go lower: a weaker uplink
causes retransmits, which cost more airtime than the lower power saves. RSSI here sits
around **−55 dBm**, though it is not steady — −40 and −65 have both been read on this
node — and that spread is also why `power_save_mode` stays at `LIGHT` rather than
`HIGH`.

Note the ESP32-WROOM-32 would run *hotter*, not cooler: two Xtensa cores, a 240 MHz
default clock, higher receive current — and no working die sensor to measure it with.

## Entities

**Light** — `LED color`: the whole ring, dimmable and colourable, carrying all effects.

**Clock controls**

| Entity | Type | Values |
| --- | --- | --- |
| `Face type` | Select | `Normal`, `Simple`, `Gradient` |
| `Indicator` | Select | `Off`, `12 o'clock`, `Quarters`, `Hour marks` |
| `Enable seconds` | Switch | shows the second hand |
| `Smooth hands` | Switch | hands slide as comets (on) or occupy one pixel each (off) |
| `Color hour` / `Color minute` / `Color second` | Text | hex colour, e.g. `#FF0000` |
| `Latitude` / `Longitude` | Text | decimal degrees, north and east positive — drives the `Sun` and `Moon` effects |
| `Alarm Time` | Datetime | daily alarm, switches the ring to the `Alarm` effect |
| `Set Clock` | Datetime | sets the clock by hand when nothing else can — see below |
| `Alarm Enabled` | Switch | arms `Alarm Time` without losing the setting |
| `Alarm Duration` | Number | minutes the alarm pulses before standing down, 1–10 |
| `Hourly Effect` | Select | `Rainbow`, `Color Wipe`, `Fireworks`, `Twinkle` (default), `Random Twinkle` |
| `Hourly Enabled` | Switch | arms the full-hour animation |
| `MQTT` | Switch | brings the broker connection up or down — off after every boot |

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

### The full-hour animation

With `Hourly Enabled` on, the ring runs `Hourly Effect` for **three seconds** at the top
of every hour and then returns to the clock face. The trigger is a cron on the SNTP
component — `0 0 * * * *`, i.e. second 0 of minute 0 — but it reads the *system* clock,
so it works on a network-less boot where the DS1307 was what set that clock. `CronTrigger`
checks `is_valid()` before firing, so an unsynced clock produces no burst at midnight.

Three conditions have to hold, and all three exist to avoid surprising anyone:

| Condition | Why |
| --- | --- |
| `Hourly Enabled` is on | the point of the feature |
| the ring is on | a ring someone switched off stays off — it must not light the room once an hour, all night |
| the clock face is showing | only `Time` gets interrupted |

The last one does the most work. Anyone watching `Fire`, or a running alarm, or
something picked by hand has said what they want on the ring, and taking it away for
three seconds every hour would be a bug rather than a chime. It also removes any need to
remember what to restore: **the only thing this can interrupt is the thing it returns
to.**

Unlike the alarm, the script sets neither brightness nor colour. The alarm goes to 100 %
and then puts `DEFAULT_BRIGHTNESS` back, which is right for an alarm and would quietly
undo a dimmed ring every hour on the hour here.

Standing down uses the same guard the alarm does: it returns to the clock only if the
animation is still what is showing, compared against the current `Hourly Effect`. Someone
who reached for Home Assistant during those three seconds and picked something else has
taken over.

Three seconds is `HOURLY_DURATION` in the substitutions. Note that no transition length
belongs anywhere near it: a light call carrying both an effect and a transition has the
transition stripped and logs `effect cannot be used with transition/flash`, and a call
carrying an effect never gets `default_transition_length` applied in the first place
(`light_call.cpp`) — so the window really is the full three seconds, start to finish.

Effects are set on the `LED color` light entity itself — Home Assistant lists them in
its more-info dialog, and automations use `light.turn_on` with `effect:`. There is
deliberately no separate `Effect` select: it duplicated a native capability, and its
option list was a hand-maintained copy of the effect names, which is the most
drift-prone thing a config like this can carry.

**Diagnostics** — ESPHome Version, Firmware Version, Device Uptime,
Reset Reason, Reset Count, Sun Elevation, Moon Altitude,
Moon Illumination, Moon Phase, SSID, IP Address, DNS Address, Connection Status,
WiFi Signal (dBm), WiFi Signal (%), Brightness.

`Sun Elevation`, `Moon Altitude`, `Moon Illumination` and `Moon Phase` come free: the
`Sun` and `Moon` effects need those numbers anyway, so exposing them turns the clock
into a usable source for automations without adding an integration for it. They are
pushed rather than polled, so they can never report something the ring is not showing.
The push runs on a 60 s interval while the maths behind it runs every 10 s — the ring
follows the sky at the faster rate, Home Assistant hears about it at the slower one,
because `publish_state` does not deduplicate and at one decimal the moon's
illumination changes essentially never.

**There is no `Internal Temperature` entity, deliberately.** On this ESP32-C3 the SoC
die sensor stops the fallback access point from beaconing, exactly like the ESPHome
`debug` component does — both were removed for that reason, and `Reset Reason` is read
from `esp_reset_reason()` directly instead. That trade matters here because a
configured `ap:` block also disables ESPHome's reboot-on-no-WiFi (`WiFiComponent`
guards it with `!has_ap()`), so a mute SoftAP would leave no way back in but a serial
reflash. If you need die temperatures for thermal work, add the sensor back
temporarily, read it over USB, and take it out again before the clock goes on a wall.

`Reset Reason` and `Reset Count` belong together: the reason comes straight from
`esp_reset_reason()` and tells you *why* the device last restarted, the count tells
you *that* it restarted while nobody was watching. Read both against `Device Uptime` —
a rising count at a low uptime is a device rebooting in a loop. The count survives
restarts and is only cleared by a factory reset.

The raw `Uptime` sensor behind `Device Uptime` is `internal: True` and never reaches
Home Assistant: it exists only to be formatted into the human-readable string, and
publishing both would be the same number twice.

**Buttons** — Restart, Restart (Safe Mode), Factory Reset.

## Effects

| Effect | Description |
| --- | --- |
| `Time` | the clock face: hour, minute and optional second hand as a gradient around the ring, in the colours set by the `Color …` entities, dimmed to ambient light |
| `Fire` | fire simulation, mirrored across both halves of the ring |
| `Moon` | the actual moon: the ring lights the illuminated fraction of the disc while the moon is above the horizon, and is dark when it has set |
| `Sun` | the real sun: lights at 12 o'clock as it clears the horizon, opens outwards as it climbs, closes the ring at solar noon, in the sun's own colour |
| `Alarm` | red pulse, roughly two seconds per cycle |

Plus five ESPHome built-ins: Rainbow, Color Wipe, Fireworks, Twinkle and Random
Twinkle. Two things separate them from the five lambdas above. They do **not** follow
the room light — `global_ambient_dim` is applied inside four of those five lambdas
(`Alarm` opts out on purpose) and a built-in has no lambda to put it in, so Rainbow at
night stays as bright as the brightness setting says. And `Twinkle`
doubles as the boot indicator: the handover to the clock face tests for it by name, so
selecting `Twinkle` by hand means the next reconnect or SNTP sync will quietly replace
it with `Time`.

Each effect declares its own `update_interval`, matched to how fast it can actually
change — 50 ms for `Time`, 30 ms for `Fire` and `Alarm`, 10 s for `Sun` and `Moon`.
Left at the ESPHome default of `0ms` a lambda re-renders all 60 pixels on every
main-loop pass.

`Time` runs at 50 ms, down from 500 ms originally, for two reasons. The render period
is exactly how late a hand can be — the display only moves on the first frame after the
clock changes. And it is the step size of the hand fade below: at the 999 ms fade
shipped here, 50 ms gives twenty steps, where the old 500 ms render period would have
given two.

### Hand movement

With `Smooth hands` on — the default — hands do not snap. Each one is drawn as a short
comet sliding from its previous pixel to the current one over `HAND_FADE_MS`: the new
pixel fades in, the old fades out, and a tail of dimming pixels follows behind. A second
stepping from 10 to 11 looks like this, at the shipped 999 ms fade:

| t | pixel 8 | 9 | 10 | 11 |
| --- | --- | --- | --- | --- |
| 0 ms | 0.33 | 0.67 | 1.00 | 0.00 |
| 500 ms | 0.17 | 0.50 | 0.83 | 0.50 |
| 999 ms | 0.00 | 0.33 | 0.67 | 1.00 |

The levels depend only on how far through the fade you are, not on `HAND_FADE_MS`
itself — shortening it compresses the same three rows into less time.

The trail is drawn *blended into* whatever is already on the ring rather than
overwriting it: on `Simple` the background is black, so it reads as a pure comet; on
`Normal` and `Gradient` it is a highlight that melts back into the ramp.

Note that `HAND_TAIL` leaves a trail at rest as well as during the step — at `3.0` a
stationary hand still has two dimmer pixels behind it. Set it to `1.0` for a pure
crossfade with no trail between steps.

At the shipped `HAND_FADE_MS` of 999 the second hand is never at rest: the fade fills
the whole second and reaches its pixel a millisecond before the next second moves it
on, so it sweeps rather than ticks. That is the point of the value. Drop it towards
200 for a hand that visibly arrives and sits still, which also makes the trail note
above apply to the second hand as well as to the other two.

**1000 is a hard ceiling.** The fade divides an age that resets every second by
`HAND_FADE_MS`, so anything above 1000 never reaches full and leaves the second hand
trailing its true pixel permanently.

**Switching `Smooth hands` off** replaces all of it with one pixel per hand. That is the
real trade the comet makes: at `HAND_TAIL: 3.0` a hand occupies three or four LEDs at
once, which reads as motion but costs the ability to say precisely which minute the hand
is on. With the switch off a hand is exactly its own pixel and steps cleanly, at the
price of looking like a different display rather than a moving one — worth trying both
before deciding which this clock is.

The two branches agree on where a settled hand sits: the comet's head at full level
blends to exactly the colour the single-pixel branch assigns, so the switch only ever
adds or removes what trails behind. The step bookkeeping keeps running while the switch
is off (three comparisons and three stores per frame), so the comets resume mid-stride
when it goes back on instead of every hand lurching because its timestamp went stale.

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

The position comes from Schlyter's orbital elements, cross-checked against the
Astronomical Almanac's low-precision solar formula (see *Architecture*) and accurate to
well under a tenth of a degree — a hundred times finer than one pixel of this ring. It
reads UTC from the clock, so the `Europe/Berlin` timezone affects only the clock face,
not this. The elevation itself is computed by the shared `interval: 10s`, not by the
effect; the effect only draws the arc.

### Where the coordinates come from

`Latitude` and `Longitude` are text entities, and they are what the `Sun` and `Moon`
effects read. Two
`internal` `homeassistant` sensors import `zone.home`'s `latitude` and `longitude`
attributes and write them into those entities, so a device added through the ESPHome
integration is correct without anyone typing anything.

`entity_category` splits the entities by how often they are touched. The `Color …`
entities, `Latitude`, `Longitude`, `Alarm Duration` and `MQTT` sit in Home Assistant's
**configuration** category — set once and then left alone. `Face type`, `Indicator`,
`Enable seconds`, `Smooth hands`, `Alarm Time`, `Alarm Enabled`, `Hourly Effect` and
`Hourly Enabled` are plain **controls**: they are
what you actually reach for, and burying them behind the configuration fold made the
clock harder to use than it needed to be. `Brightness` is a **diagnostic** — it reports
the room, nothing sets it.

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

The ring runs `Twinkle` from the moment the LEDs are addressable and stays there until
two things are true: the radio has associated **and** SNTP has delivered a valid time.
Then it settles on `Time` at `DEFAULT_BRIGHTNESS`. Both conditions matter — an access
point without a route to the internet gives no time, and the clock face blanks the ring
when the clock is invalid, so handing over on association alone would trade a visible
"waiting" indicator for a black ring.

Switching the light on from Home Assistant always returns to that same state, whatever
effect or colour was set before.

The ring is **not** darkened on shutdown, and cannot be: WS2812s latch their last frame
and ESPHome's restart path gives the light no chance to write another one. It keeps
showing whatever it was showing until the new firmware boots. See *Known limitations*.

## State that survives a reboot

Fifteen values persist: the three `Color …` entities, `Latitude` and `Longitude`,
`Alarm Time`, `Alarm Duration` and `Alarm Enabled`, `Hourly Effect` and
`Hourly Enabled`, `Face type` and `Indicator`, `Enable seconds`, `Smooth hands`, and the
`Reset Count`. The `MQTT` switch is deliberately **not**
among them — `ALWAYS_OFF` neither reads nor writes a preference.

Their `restore_mode` / `restore_value` settings are spelled out in the YAML rather than
left to the schema defaults, which are `ALWAYS_OFF` and `false` — with the defaults
every setting was silently lost on each restart. `initial_value` / `initial_option`
therefore only apply on the very first boot, or after a factory reset.

`Smooth hands` is the one switch on `RESTORE_DEFAULT_ON` rather than
`RESTORE_DEFAULT_OFF`. Comet hands are how this clock has always looked, so a device
that comes up without a stored preference has to come back looking the way it did.

All fifteen land in the same flash sector, which is why `preferences:` sets
`flash_write_interval: 5min` instead of the default 60 s: it folds a burst of
slider-dragging in Home Assistant into one erase cycle. The cost either way is that a
change made less than the interval before a power cut is lost.

The three faces:

| Face | What it draws |
| --- | --- |
| `Normal` | the two arcs between the hands, each ramping from a blend of both hand colours up to its own hand colour — width set by `FACE_BLEND` |
| `Simple` | the hands as bare dots on a dark ring |
| `Gradient` | both arcs as colour ramps — hour colour to minute colour one way round, minute back to hour the other |

`Gradient` dims its ramps to `GRADIENT_DIM` and paints the hands over them at full
strength. Without that the hands would vanish: a gradient is by definition almost the
same colour either side of any point on it.

`Indicator` and `Enable seconds` apply to every face, so hour marks and a second dot
work on `Simple` and `Gradient` too.

`Indicator` marks positions on the dial — one mark at the top, four at 12/3/6/9, or all
twelve hours. The names describe positions rather than times of day: pixel 0 is where
both noon and midnight land, so the old `Show midday` was wrong for half of every day,
and `Show quadrants` named the sectors when what it draws are the marks between them.

A mark is the **inverse of the luminance** of the face around it: a neutral grey, bright
over a dark surround and dark over a bright one. It used to invert each colour channel
instead, which on a saturated face produced the complement — a cyan mark on a red arc, a
magenta one on green — and it was computed after the hands were drawn, so every mark
also shifted as a hand slid past it. The marks are now drawn with the face, before the
hands, and carry no hue of their own.

The option **order** of `Face type` and `Indicator` is load-bearing (renaming is safe,
`restore_value` stores the index too): the `Time` effect
switches on `active_index()`, so reordering or inserting an option silently remaps the
face.

`Hourly Effect` is the exact opposite, and the two rules are easy to mix up. Its
**strings** are load-bearing and its order does not matter: the script passes the
selected option straight to `light.turn_on`'s `effect:`, so an option that does not name
a real effect is a silent no-op — `LightCall` logs `invalid effect index`, clears the
flag and carries on, and the ring simply keeps showing the clock at every full hour with
nothing to explain it. Rename an effect in the `light:` block and this select has to be
renamed with it.

## Known limitations

- **The ring cannot be darkened on shutdown.** `Application::reboot()` calls
  `on_shutdown()` on every component and then restarts immediately, while the strip is
  only ever written from `LightState::loop()` — and neither `light_state` nor
  `esp32_rmt_led_strip` implements `on_shutdown` or `teardown`, so that loop never runs
  again. A `light.turn_off` there sets `next_write_` and nothing consumes it. WS2812s
  latch, so the last frame stays lit through the restart. The config used to carry such
  a block; it was removed rather than left looking as if it worked.
- **No die temperature.** `internal_temperature` and the ESPHome `debug` component both
  stop the fallback access point from beaconing on this ESP32-C3. Since a configured
  `ap:` block also disables ESPHome's reboot-on-no-WiFi, a mute SoftAP would leave no
  way back in short of a serial reflash. Add the sensor back only for a measurement
  session over USB.
- **The built-in effects ignore the light sensor.** Ambient dimming lives inside each
  `addressable_lambda`; Rainbow, Color Wipe, Fireworks, Twinkle and Random Twinkle have
  no lambda to put it in and stay at the set brightness all night.
- **`Alarm` only stands itself down if it is still showing.** Silencing it by picking
  another effect, or by switching the ring off, leaves the timer to expire quietly —
  which is the point, but it also means an `Alarm` selected by hand from the effect
  list never ends on its own.

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
  A second `interval: 60s` pushes the four sky sensors — the ring follows the sky at
  10 s, Home Assistant hears about it at 60 s, because `publish_state` does not
  deduplicate and republishing an unchanged moon phase six times a minute is traffic
  for nothing.
- **`global_ambient_dim`** holds the volts-to-brightness mapping, computed on the
  `Brightness` sensor. `Time`, `Fire`, `Sun` and `Moon` each scale their finished frame
  by it. `Alarm` deliberately does not — an alarm that turns itself down in a dark room
  fails at the one moment it is needed.

The two solar algorithms were cross-checked before being merged: the Astronomical
Almanac's low-precision formula and Schlyter's orbital elements agree to 0.003° over a
full year, so one set of elements now serves both bodies.

### Boot and network

`on_boot` lights `Twinkle`, and nothing switches to the clock face until the
`show_clock` script says so — so the busy indicator is real. Since the RTC was fitted
it indicates "no valid time yet" rather than "no network yet"; the two stopped being
the same thing. It used to be a boot
trigger at priority 200 testing `wifi.connected`, which could never be true: every
`on_boot` trigger runs inside `setup()`, milliseconds apart, long before the radio
associates.

`show_clock` is called from three places that each know only part of what is needed —
`wifi:` `on_connect` knows the radio associated, `time:` `on_time_sync` knows a network
sync landed, and the late `on_boot` block knows the DS1307 has been read — so the
guards live in one script instead of being part-checked three times. It acts only if
the ring is still on `Twinkle` **and** the clock is valid.

That third caller is what makes the RTC worth fitting: with a good battery the clock
face is up before the radio has associated, and a clock that sat on the boot indicator
waiting for an access point would be an RTC bought for nothing. It runs from the
`-100` block rather than from the priority-800 one because the guard tests the *active
effect*, and `Twinkle` is not selected until priority 600. No extra guard is needed for
a missing or halted RTC: `read_time()` then leaves the system clock alone, the
`is_valid()` test fails, and `Twinkle` stays up exactly as before.

There is deliberately no `on_disconnect` counterpart. The clock runs off the system
clock, not off the network, so losing WiFi is no reason to stop showing the time — and
a reconnect should not undo whatever was selected in Home Assistant, which is what the
`Twinkle` guard prevents.

The `MQTT` switch reports what was *asked for*, not what the network is doing — the
only getter available is `is_connected()`, so mirroring the real state would make the
switch snap back to off whenever the broker was briefly unreachable and look broken.

Its `restore_mode` has to stay in step with `enable_on_boot` in the `mqtt:` block. The
coupling is not cosmetic: `TemplateSwitch::setup()` acts on the restored state rather
than just publishing it — it calls `turn_on()` or `turn_off()` outright, firing the
actions — so a mismatch silently overrides `enable_on_boot` a moment after boot, and
the config validates happily while doing it. Both currently say off.

`ALWAYS_OFF` rather than `RESTORE_DEFAULT_OFF`, deliberately: MQTT off is the intended
state after *every* boot, not merely the first one. Persisting an "on" across a restart
would quietly undo that.

**`esphome logs` cannot find the device over MQTT while it is off** — that discovery
asks the broker on `esphome/discover/infinity`. Use the address instead:

```sh
esphome logs infinity.yaml --device infinity.local
```

The system clock has two sources: SNTP over the network, and the battery-backed
DS1307 for everything else. A `time: platform: homeassistant` fallback was tried and
removed: the API carries time as a uint32 epoch in **seconds**, so every 15-minute sync
discarded the sub-second alignment SNTP had established and stepped the clock by up to
a second — backwards, usually, since truncation always rounds down. On a device whose
job is showing seconds that was a visible jump four times an hour. The fallback it
bought was real — time without a route to the internet — and the RTC now supplies that
without the jumping.

### The RTC

The DS1307 is read **once**, from `on_boot` at priority 800, and never polled again
(`update_interval: never`). Re-reading it would only pull the accurate SNTP time back
towards the drift of a DS1307, which is the wrong direction. The write goes the other
way: every successful SNTP sync is written back to the RTC, so the battery-backed time
is never more than one sync interval stale.

`read_time()` sets the **system** clock rather than just its own component's, so the
`Time` effect, the `show_clock` guard and the astronomy interval all see a valid time
from priority 800 onwards with no network involved.

**A new module is dead on its first boot, and that is normal.** A DS1307 ships with
`CH` — the clock-halt bit in register 0 — set, oscillator stopped, registers reading
`2000-01-01 00:00:00`. ESPHome refuses to sync from that (`RTC halted, not syncing to
system clock`), so the first boot behaves exactly like a device with no RTC: `Twinkle`
stays up until SNTP lands. The write-back on that first sync clears `CH` and starts the
oscillator — from the **second** boot onwards the clock face is up before the radio has
associated. A fitted-but-never-set module and a dead one therefore look identical for
exactly one boot; the i²c scan and the `CH` flag in the `ds1307` DEBUG line tell them
apart.

Priority 800 sits between the i²c bus (a `BUS`-priority component at 1000, so already
up) and the DS1307's own `setup()` at `DATA` (600, so not yet run) — which is fine,
because the action needs only the bus pointer and the address, and both are assigned
before `App.setup()` is entered.

#### Setting the clock by hand

Every automatic path needs something the device may not have: SNTP needs a route to the
internet, the RTC needs to have been set once already, and Home Assistant needs the
API. With none of the three the ring sits on `Twinkle` indefinitely — the boot handover
is honest about not knowing the time, which is right, but on its own it left no way to
*tell* it.

`Set Clock` is that way, and it needs no more than the LAN: `web_server:` runs on port
80 with digest auth, so the entity is reachable from a browser with no Home Assistant,
no broker and no route out. Setting it validates the value, sets the system clock,
copies it into the DS1307, and calls `show_clock` so the ring leaves the boot indicator
immediately. Because the write reaches the RTC, the setting also survives the next
power cut — and it clears the `CH` bit, so **setting the time by hand is what
commissions a factory-fresh module** that has never seen SNTP.

Nothing is pinned by this: the next SNTP sync overwrites both the system clock and the
RTC, as it should. Network time wins when there is any.

Two details are deliberate and easy to get wrong:

- **`set_action`, not `on_value`.** `publish_state()` calls `state_callback_`, and
  `setup()` publishes — so an `on_value` automation fires once on *every* boot and would
  slam the clock back to whatever was stored or compiled in, undoing the RTC read at
  priority 800. `set_trigger_` is reached only from `control()`, i.e. only when someone
  actually sets the entity.
- **No `initial_value`, and `restore_value` left at its default of `false`.** With
  neither, `publish_state()` sees `year == 0` and reports no state at all, so the field
  reads empty until it is used. A setter that came back after a reboot showing the
  moment it was last used would look like a clock and be wrong by exactly the downtime.

This is one `DATETIME` entity rather than the six `number` helpers
`dew-point-ventilation` uses for the same job — those exist because that device drives
them from an encoder menu on an LCD, and a web form wants one field, not six.

#### Why the module runs on 3V3, not 5V

A DS1307 is specified for 4.5–5.5 V, so this looks wrong, and on most boards carrying
one it *would* be. Two details of this particular module decide it — see
[`docs/TinyRTC-Diagram.jpg`](docs/TinyRTC-Diagram.jpg).

**The access threshold.** A DS1307 stops answering on the bus whenever V_CC drops below
1.25 × the voltage on its `VBAT` pin. Wire a 3.0 V cell straight to `VBAT`, as most
modules do, and the threshold is 3.75 V: on 3V3 the clock keeps perfect time and simply
never replies — the single most common "my DS1307 is broken" report there is.

This board puts a divider in the way. `R6` (470 kΩ) sits in series between the cell and
`VBAT`, `R4` (1.5 MΩ) runs from `VBAT` to ground, so the pin sees 1.5/1.97 = **0.76** of
the cell:

| Cell | `VBAT` pin | Access threshold (1.25 ×) | Answers on 3V3? |
| --- | --- | --- | --- |
| CR2032, 3.0 V | 2.28 V | 2.86 V | yes — **confirmed on hardware** |
| CR2032, 2.8 V | 2.13 V | 2.66 V | yes |
| LIR2032, 3.6 V | 2.74 V | 3.43 V | marginal |
| *no divider* (`R6` shorted) | 3.00 V | 3.75 V | **no** |

That last row is a warning, not a suggestion: shorting `R6` to "fix" the low backup
voltage is exactly what breaks the bus.

**The charging circuit.** `R5` (200 Ω) and `D1` (1N4148) run from V_CC to the cell — the
module is built for a rechargeable LIR2032. On 5 V that pushes **≈ 6.5 mA** into
whatever is in the holder; into a non-rechargeable CR2032 that means heat, outgassing
and eventually a vented cell. On 3V3 the diode never forward-biases (3.3 − 0.7 = 2.6 V,
below any usable cell), so the path is inert.

**So: 3V3 with a CR2032.** A LIR2032 is the wrong cell at this voltage — it would never
be recharged and would drop out of the DS1307's 2.0 V `VBAT` minimum within months. The
CR2032 is not generous either: at 500 nA through the divider's 358 kΩ source impedance
the pin sits at ≈ 2.11 V against that 2.0 V minimum, so the module will lose the time
while the cell still measures close to 3 V. That is survivable — SNTP re-syncs and
writes back — and it is why the boot handover never assumes the RTC answered.

Powering the module from 3V3 also keeps the bus pull-ups (`R2`/`R3`, 3.3 kΩ to V_CC) at
3V3. On 5 V they would idle the bus at 5 V, past the absolute maximum of a C3 GPIO and
back-feeding the 3V3 rail through the pin clamps.

Photos of the module as fitted: [`docs/TinyRTC-001.jpg`](docs/TinyRTC-001.jpg),
[`docs/TinyRTC-002.jpg`](docs/TinyRTC-002.jpg).

#### What else is on the module

An **AT24C32** EEPROM at `0x50`–`0x57` (set by the `A0`–`A2` jumpers) shares the bus,
unused here. The schematic also shows a **DS18B20** temperature sensor on a 1-Wire
header pin — but **that footprint is unpopulated on this build**, so there is nothing
to read there.

The EEPROM is worth knowing about for one reason: it is a 2.5–5.5 V part on the same
two wires as the RTC, so an i²c scan listing `0x50` but not `0x68` isolates the DS1307
from any wiring doubt.

An OTA update parks the light for its duration — the effect lambdas and RMT transfers
otherwise compete with the transfer for CPU and for the WiFi stack, which matters more
now that the radio sleeps between beacons.
