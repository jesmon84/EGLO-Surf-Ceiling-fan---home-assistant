# EGLO Surf ceiling fan → ESPHome → Home Assistant

Local control of EGLO Surf DC ceiling fans (fan + CCT light) in Home Assistant, with
no cloud. The stock Tuya WiFi module is replaced by a **Wemos D1 Mini (ESP8266)**
running ESPHome, which talks to the fan's own Tuya MCU over UART using ESPHome's
built-in [`tuya`](https://esphome.io/components/tuya.html) component.

The config here has been running on six fans in one house.

## What you get in Home Assistant

| Entity | Type | Notes |
|---|---|---|
| Ergo Fan | `fan` | On/off, 5 speeds, forward/reverse |
| Ergo Fan Light | `light` | On/off, brightness, colour temperature (warm ↔ cool) |
| Ergo Fan Mode | `select` | Normal / Natural (breeze) |
| Reset Reason, Uptime, WiFi Signal, WiFi BSSID, Heap Free | diagnostics | For tracking down reboots and WiFi problems |

The fan also **restores its previous on/off state and speed after a reboot or power
blip**, so it doesn't just stop if the ESP restarts.

## Repository layout

```
esphome/
  packages/ergo-fan.yaml   # all shared config - edit once, applies to every fan
  example-fan.yaml         # per-fan file: just name, IP, hotspot name, API key
  secrets.example.yaml     # template for your secrets.yaml (never commit the real one)
```

## Tuya datapoints

These were confirmed on EGLO Surf fans. If your model differs, flash the ESP with just
the `uart:` and `tuya:` blocks and the ESPHome log will list every datapoint the MCU
reports at boot.

| DP | Function | Type | Values |
|---|---|---|---|
| 20 | Light on/off | bool | |
| 22 | Light brightness | int | 100–1000 |
| 23 | Light colour temperature | int | 0–1000 (**inverted** – see below) |
| 60 | Fan on/off | bool | |
| 61 | Fan mode | enum | 0 = Normal, 1 = Natural |
| 62 | Fan speed | int | 1–5 |
| 63 | Fan direction | bool/enum | forward / reverse |

## Hardware

- **Wemos D1 Mini (ESP8266)** – recommended.
- The fan's original WiFi module plugs into a socket in the fan's receiver that
  provides **power, TX and RX**. The D1 Mini is wired to that socket in place of it.
- UART: **ESP TX = GPIO5 (D1)**, **ESP RX = GPIO4 (D2)**, 9600 baud, 8N1.
  ESP TX goes to MCU RX and ESP RX to MCU TX.
- ⚠️ **Check the socket pinout and voltage with a multimeter before connecting
  anything.** Don't assume it from photos or someone else's fan – confirm which pin is
  5 V / 3.3 V, GND, TX and RX on yours. Isolate the board with heatshrink or a small
  enclosure; it sits next to mains wiring in the canopy.

### Why ESP8266 and not ESP32?
An ESP32-S3 was tried first (UART on GPIO13/GPIO12) and worked, but some fans'
receivers **couldn't supply enough current** through that socket and the ESP32
browned out and rebooted. The D1 Mini draws much less and has been stable. If you want
to use an ESP32 anyway, power it from a separate supply (sharing GND with the fan's MCU)
or add a decent buck regulator, and just change `esp8266:`/`board:` and the UART pins.

## Setup

1. Copy `esphome/packages/ergo-fan.yaml` into your ESPHome config folder under
   `packages/` (in Home Assistant that's `/config/esphome/packages/`).
2. Copy `example-fan.yaml` once per fan, rename it, and set `name`, `friendly_name`,
   `static_ip`, `ap_ssid` and `api_key`.
3. Add your values to `secrets.yaml` (see `secrets.example.yaml`). Give each fan its
   own API key.
4. If your network isn't `192.168.1.0/24`, override `gateway`/`subnet` in the per-fan
   file.
5. First flash over USB, then install over the air from then on. Home Assistant will
   discover the device via the ESPHome integration.

## Things learned the hard way

- **Colour temperature is reversed** on these fans. `color_temperature_invert: true`
  fixes it so warm is warm and cool is cool in HA.
- **Random turn-offs / unresponsive fans:** the diagnostic sensors (Reset Reason,
  Uptime, WiFi Signal/BSSID) were added to find out why, together with
  `power_save_mode: none`, which is a common fix for ESP8266 WiFi dropouts. The state
  restore means that even if a fan does reboot, it comes back as it was.
- **`api: reboot_timeout: 0s`** stops the fan rebooting (and stopping) every time Home
  Assistant restarts or is down for more than 15 minutes, which is the ESPHome default.
- **`logger: baud_rate: 0`** keeps logging off the hardware serial port.
- The **15 s boot delay** before restoring state gives the Tuya MCU time to finish
  its handshake; restoring earlier was unreliable.
- `flash_write_interval: 5min` limits flash wear from saving the fan state.
- **Use a shared package.** With several fans, putting everything in one package file
  means a single edit (then "Install" on each) keeps them all identical.
- **Network:** these don't need internet. Blocking WAN on your IoT VLAN is fine as long
  as they can still reach Home Assistant; point them at a local NTP server if you do,
  so their clocks don't drift.

## Credits / references

- ESPHome Tuya component: https://esphome.io/components/tuya.html
- ESPHome packages & substitutions: https://esphome.io/components/packages.html

Use at your own risk – this involves opening a mains-powered fitting. If you're not
qualified to work on it, get an electrician to remove and refit the canopy.
