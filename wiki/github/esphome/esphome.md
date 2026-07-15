# esphome/esphome

> YAML-to-firmware compiler for ESP32/ESP8266-class microcontrollers, built around Home Assistant.

[GitHub repo](https://github.com/esphome/esphome) ·
[Official website](https://esphome.io) ·
[License: MIT / GPL-3.0 (mixed)](https://github.com/esphome/esphome/blob/dev/LICENSE)

## Overview

ESPHome takes a declarative YAML configuration file describing a device — its
board, sensors, switches, network, and automations — and generates, compiles,
and flashes native C++ firmware for microcontrollers in the ESP32, ESP8266,
RP2040, and BK72xx families[^1]. You do not write embedded C; you describe what
the device *is* from a catalog of several hundred components, and ESPHome emits
the `main.cpp` and PlatformIO project that implement it. The result is a small
single-purpose firmware image, not a general-purpose runtime interpreting your
config at boot.

It is, in practice, the semi-official firmware layer of Home Assistant. ESPHome
was created by Otto Winter in 2018 (as `esphomeyaml`/`esphomelib`), joined Nabu
Casa — the company behind Home Assistant — in early 2021, and was transferred to
the non-profit Open Home Foundation in 2024[^2]. Devices expose themselves to
Home Assistant over ESPHome's own encrypted binary "native API" (protobuf over
TCP) or over MQTT, with auto-discovery on both paths. This tight coupling is the
whole value proposition and the defining constraint: ESPHome is superb if your
world is Home Assistant, and comparatively awkward if it is not.

The central tradeoff versus its main rival Tasmota is compile-per-config vs.
flash-once. ESPHome recompiles bespoke firmware for every device and every
config change; Tasmota ships one binary you configure at runtime. That makes
ESPHome infinitely composable and slow to iterate, while Tasmota is instant to
deploy and fixed in capability.

## Getting Started

```bash
pip install esphome        # also ships as a Home Assistant add-on and Docker image
```

```yaml
# living-room.yaml
esphome:
  name: living-room
esp32:
  board: esp32dev
  framework:
    type: esp-idf        # esp-idf preferred on ESP32; arduino still supported

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:
api:                     # native encrypted link to Home Assistant
  encryption:
    key: !secret api_key
ota:
  platform: esphome

sensor:
  - platform: dht
    pin: GPIO4
    temperature:
      name: "Living Room Temperature"
    humidity:
      name: "Living Room Humidity"
```

```bash
esphome run living-room.yaml   # compile → flash (USB first time, OTA after) → tail logs
```

## Architecture / How It Works

The pipeline is codegen, not interpretation:

1. **Load & validate** — the Python core parses YAML (with `!secret`,
   `!lambda`, substitutions, and reusable `packages`) and validates every
   component against a voluptuous schema. Schema errors are reported here, before
   any compilation.
2. **Code generation** — each component contributes C++ via a `to_code`
   function. ESPHome assembles a `main.cpp`, a `platformio.ini`, and pulls the
   needed component sources into a build directory.
3. **Compile** — PlatformIO drives the actual toolchain (Arduino framework or
   ESP-IDF for ESP32; the Arduino core for ESP8266; LibreTiny for BK72xx/RTL87xx).
   First compile downloads a platform toolchain and can take several minutes.
4. **Flash & run** — the image goes over USB serial the first time and over the
   network (OTA) thereafter. At runtime a cooperative main loop polls components;
   there is no RTOS-style scheduler you interact with directly.

Only the components referenced in the YAML are compiled in, so an image contains
exactly its declared capabilities and nothing else — this is why footprints stay
small enough for the ESP8266's constrained flash and RAM. Custom logic escapes
the declarative model through `lambda:` blocks, which are literally inlined C++.

The **native API** is the preferred transport: a persistent, Noise-encrypted
protobuf connection to Home Assistant, lower-latency than MQTT and requiring no
broker. A large real-world deployment class is the **Bluetooth proxy**, where
cheap ESP32s relay BLE traffic to Home Assistant, extending its Bluetooth range
across a house.

## Production Notes

- **Compile time is the tax.** Every config change rebuilds firmware. Clean
  builds run minutes; the first build of a new platform also downloads a
  toolchain. Fleets are best managed with the dashboard's build cache and staged
  rollouts rather than recompiling everything at once.
- **ESP8266 is the tight target.** Limited RAM and flash mean feature-heavy
  configs (TLS, many components, the native API plus MQTT plus web server) will
  not fit. New designs should prefer ESP32; treat ESP8266 as legacy.
- **Arduino vs. ESP-IDF.** On ESP32, ESP-IDF is the recommended framework and
  some components only exist on one framework. Switching framework is a full
  rebuild and can change flash layout — plan a USB reflash, not an OTA.
- **OTA is a footgun.** A bad config flashed over the air can leave a device
  unreachable, requiring physical USB access to recover. Keep devices reachable,
  validate configs (`esphome config`), and stage changes on one unit first.
- **Version coupling.** The Home Assistant ESPHome integration expects device
  firmware within a compatible range. Upgrading Home Assistant can prompt you to
  re-flash devices; a stale device may drop off the API until rebuilt.
- **CalVer breaking changes.** Releases are monthly (`YYYY.MM.patch`) and each
  changelog lists breaking changes — renamed keys, removed platforms, tightened
  validation. Read release notes before mass-upgrading a fleet; pin the ESPHome
  version in CI if you build firmware there.
- **Secrets and the API key.** The native API encryption key and Wi-Fi
  credentials live in `secrets.yaml`. That file, not the per-device config, is
  the thing to guard and back up.

## When to Use / When Not

**Use when:**
- Your home automation hub is Home Assistant and you want first-class,
  auto-discovered devices.
- You are building custom sensor/actuator hardware from ESP32/ESP8266 modules.
- You want a Bluetooth or other proxy fleet feeding Home Assistant.
- You prefer declarative config in version control over hand-written embedded C.

**Avoid when:**
- You are not on Home Assistant and don't want to run one — MQTT works but you
  lose most of the ecosystem benefit.
- You need instant reconfiguration of already-deployed devices without a rebuild.
- You are shipping a commercial product needing a controlled, signed, over-the-air
  update pipeline and a custom cloud — build on ESP-IDF directly.
- Your target is outside the supported chip families (ESP, RP2040, BK72xx).

## Alternatives

- arendst/Tasmota — precompiled firmware configured at runtime via web UI/MQTT; use instead when you want flash-once devices and no per-change recompile.
- Aircoookie/WLED — purpose-built addressable-LED firmware; use instead when the device is only a light strip.
- xoseperez/espurna — Tasmota-style multi-purpose firmware for ESP8266/ESP32; use instead when you want runtime config outside the Home Assistant orbit.
- espressif/esp-idf — the native SDK; use instead when you need full low-level control, custom OTA, or a commercial product firmware.
- platformio/platformio-core — the build system ESPHome sits on; use instead when you want to write the C++ yourself but keep the toolchain automation.

## History

| Version | Date | Notes |
|---------|------|-------|
| esphomeyaml/esphomelib | 2018-04 | Otto Winter's original projects; repo created[^1]. |
| ESPHome (rename) | ~2019 | `esphomeyaml` + `esphomelib` unified under the ESPHome name. |
| Nabu Casa | 2021-01 | ESPHome joins Nabu Casa; Otto Winter joins the team[^2]. |
| 2021.8.0 | 2021-08 | Switched to CalVer date-based versioning (`YYYY.MM.patch`). |
| Open Home Foundation | 2024-04 | ESPHome (with Home Assistant) transferred to the non-profit OHF[^3]. |
| 2026.x | 2026 | Active monthly releases on the `dev` branch; ESP32/8266/RP2040/BK72xx support[^1]. |

## References

[^1]: ESPHome documentation and repository README. https://esphome.io and https://github.com/esphome/esphome
[^2]: Nabu Casa, "ESPHome joins Nabu Casa" — 2021. https://www.nabucasa.com/blog/
[^3]: Open Home Foundation. https://www.openhomefoundation.org/

## Tags

cpp, python, yaml, iot, esp32, esp8266, home-automation, home-assistant, firmware, embedded, mqtt, platformio
