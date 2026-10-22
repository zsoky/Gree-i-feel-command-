# Gree "I Feel" temperature sender (ESP32-C3 + ESPHome)

[Magyar verzió](README.md)

A small ESP32-C3 device that takes a room temperature from Home Assistant and sends it to a Gree air conditioner over infrared, **exactly the way the remote control's "I Feel" function does**. The AC then regulates by a sensor of your choice (for example one mounted low, in the middle of the room) instead of the one built into the indoor unit.

Why bother?
- An inverter AC stops cycling on and off because of a reading skewed by its own warm outlet air.
- Smaller temperature swings, better comfort, and less ticking and popping of the indoor unit's plastic housing.
- The temperature can be a weighted average of several rooms (see below).

> **Note:** this project was tested with the Gree **YBEF1FB6** remote and two Gree ACs (Amber Royale 3.5 kW, Smart One 5.3 kW). The protocol was worked out by capturing the remote's frames, not from official documentation. Other models may differ. Use at your own risk.

## How it works

In "I Feel" mode the Gree remote sends a short, **separate IR frame** containing the measured temperature after every button press, and on its own every 10 minutes. The AC accepts this frame on its own and uses it instead of its internal sensor.

Frame layout:

| Part | Value |
|---|---|
| Header | ~6000 µs mark / ~3000 µs space |
| Data | 16 bits, **LSB first**: `T` `0xA5` |
| `T` | temperature as an integer in °C (e.g. 24 → `0x18`) |
| `0xA5` | constant byte |
| Bits | mark 630 µs; 0: space 575 µs, 1: space 1675 µs |
| Stop mark | 630 µs |
| Carrier | 38 kHz |

The ESP rounds the sum of the received temperature and the offset to an integer and sends it every 2 minutes. The AC only accepts whole degrees, so the decimals are lost in rounding.

## Hardware

- ESP32-C3 Super Mini (the YAML targets the `esp32c3` variant)
- IR LED (e.g. TSAL6400 or any 940 nm emitter)
- NPN transistor to drive the LED (e.g. BC337)
- Resistors: 1 kΩ (GPIO → transistor base), 33 Ω (series resistor for the LED)

Wiring:

```
GPIO5 ──[1 kΩ]── base (BC337)
                 collector ── LED cathode
3V3 ──[33 Ω]── LED anode
                 emitter ── GND
```

(Size the resistors for your own LED.)

You can check the LED with a phone camera: IR light shows up as a purplish-white flash. If you see nothing, test the LED directly between 3V3 and GND through a series resistor, since cheap LEDs are sometimes defective.

Point the LED at the AC. Two ACs need two separate devices, each facing its own AC, unless one LED can see both.

## Installation

1. Install [ESPHome](https://esphome.io) (dashboard or command line).
2. Copy `secrets.yaml.example` to `secrets.yaml` and fill it in (WiFi, API key, AP password). **Never commit `secrets.yaml`**; the `.gitignore` excludes it.
3. Edit the `substitutions` block at the top of `esp32-c3-ifeel.yaml`:
   - `name` and `friendly_name`: the device name,
   - `source_sensor`: the Home Assistant entity whose value is sent to the AC.
4. Flash the ESP (USB the first time, OTA afterwards).
5. The ESPHome integration in Home Assistant discovers the device automatically.

For more than one AC, copy the YAML and change the `substitutions` block in each copy. If you share one `secrets.yaml`, give the `api_key` a separate name per device (e.g. `api_key_1`, `api_key_2`) and change the `api:` section accordingly.

## Usage

First turn on "I Feel" **on the remote itself**, since the AC only pays attention to the frame in that mode (that is how the project was tested). Then **take the batteries out of the remote**, otherwise it overwrites the ESP's value with its own temperature every 10 minutes.

The device exposes these entities in Home Assistant. The entity names in the YAML are in Hungarian; rename them if you like (the English meaning is given here):

| Entity (name in YAML) | What it does |
|---|---|
| `I Feel offset` | Added to the measured value, −10 to +10 °C in 0.5 steps. A positive value makes the AC think the room is warmer. |
| `I Feel utoljára küldött` (last sent) | The last (integer) temperature sent to the AC. |
| `I Feel eltelt idő` (elapsed time) | Minutes since the last frame was sent. |
| `I Feel azonnali küldés` (send now) | Button: sends the current value immediately. |

The ESP first sends 90 seconds after boot and every 2 minutes after that. In the log you will see: `I Feel keret elküldve: 25 C` ("I Feel frame sent").

## Averaging several rooms

If the AC heats several connected rooms, it helps to send a **weighted average** of the sensors. `homeassistant-templates.yaml` contains a ready example of a Home Assistant template sensor that:
- skips unavailable sensors,
- redistributes the weights over the remaining ones,
- returns `unavailable` if no sensor has a numeric value (the ESP then sends no frame).

Weighting tips: give the sensor in the AC's own room the largest weight, leave out small closed-off rooms (toilet, pantry), and do not use a sensor close to the AC, because it shows the same distortion as the AC's own sensor.

## Troubleshooting

- **`Auth Expired` on WiFi:** `output_power: 8.5dB` fixed this on an ESP32-C3 Super Mini. Try other values if yours behaves differently.
- **The AC does not react:** check the LED with a phone camera and make sure "I Feel" is switched on at the remote.
- **No frame is sent and the log shows an "invalid" warning:** the source sensor is `unavailable`, or below 5 °C or above 45 °C.
- **ESPHome's built-in `climate: gree` cannot be used for this:** it does not support the "I Feel" frame, and on my ACs its output made the unit restart.

## Known limitations and open questions

- The AC takes whole degrees; I did not measure whether the remote rounds or truncates. The ESP rounds.
- I do not know how long the AC keeps the last value if frames stop (the remote sends every 10 minutes, the ESP every 2).
- The AC did not accept 30 °C (the highest value seen on the remote was 29), so the code clamps to 5–45 °C.
- The project is built on the frames of the Gree YBEF1FB6 remote; other models may use different frames.

## Files

| File | Contents |
|---|---|
| `esp32-c3-ifeel.yaml` | The ESPHome configuration |
| `secrets.yaml.example` | Template for the secret values (WiFi, API key, AP password) |
| `homeassistant-templates.yaml` | Example Home Assistant template sensor for the weighted average |
| `.gitignore` | Excludes `secrets.yaml` and build files |
| `README.md` | The Hungarian description |
