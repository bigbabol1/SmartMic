# SmartMic

ESPHome firmware for DIY voice satellites paired with Home Assistant Assist.

## Devices

`smartmic.yaml` is the single fleet firmware — one image flashes to every SmartMic unit.

| Hardware | Wake word | Notes |
|----------|-----------|-------|
| XIAO ESP32-S3 (8 MB flash, octal PSRAM) + 1× I2S MEMS mic (**INMP441** or **ICS-43434**, drop-in pin-compatible, mono on left channel — L/R → GND) + external status LED on GPIO4 (XIAO silkscreen pad **D3**) | `hey_jarvis` (microWakeWord, on-device) | `name_add_mac_suffix: true` → each device auto-names `smartmic-<last6mac>`. Timer ring derives `satellite_entity_id` at runtime from `App.get_name()`, so one .bin works for the whole fleet. TTS via [Mic to MediaPlayer](https://github.com/bigbabol1/HomeAssistant_mic_to_mediaplayer). |

## Pinout (XIAO ESP32-S3)

| GPIO | Silkscreen pad | Function |
|------|----------------|----------|
| GPIO7 | D8 | I2S BCLK |
| GPIO8 | D9 | I2S LRCLK / WS |
| GPIO9 | D10 | I2S DIN (mic data) |
| GPIO4 | **D3** | External status LED (active-low: anode → 3V3 via resistor, cathode → pin) |

> ⚠️ XIAO silkscreen labels (`D0…D10`) are **not** the GPIO numbers — D3 = GPIO4, D4 = GPIO5, etc. Solder to the pad whose silkscreen matches the second column above, not the GPIO number.

## Microphone modules (drop-in compatible)

Both supported MEMS mics share the I2S Philips protocol, 24-bit MSB in 32-bit frame, and identical 6-pin layout. No firmware changes are required when swapping — only update the `mic_module` substitution at the top of the YAML for log clarity.

| Module | SNR (dBA) | Sensitivity | Built-in HPF | Notes |
|--------|-----------|-------------|--------------|-------|
| INMP441 | 61 | -26 dBFS @ 94 dB SPL | No | Original board fitment; small DC offset present in raw stream. |
| ICS-43434 | 65 | -26 dBFS @ 94 dB SPL | Yes (~80 Hz) | Default for new builds; cleaner low-frequency baseline, slightly better wake-word recall in noisy rooms. |

**Wiring:** L/R pad → GND selects the left slot (`channel: left` — what `smartmic.yaml` expects). L/R → VDD selects the right slot; only use that if you also change `channel: right` in the YAML.

## Companion stack

These satellites are designed to work with the full voice stack:

1. **This firmware** — wake word + audio capture, sends audio to HA pipeline.
2. **[Mic to MediaPlayer](https://github.com/bigbabol1/HomeAssistant_mic_to_mediaplayer)** — captures TTS audio from the pipeline and plays it on a configured `media_player` (e.g. AirPlay speaker, TV, Alexa). Also drives the `start_follow_up` ESPHome service for `continue_conversation` follow-up turns without re-triggering the wake word.
3. **[AI Plugin](https://github.com/bigbabol1/AI-plugin)** — LLM-backed Home Assistant conversation entity (Ollama/OpenAI/Anthropic), with tool calling for entity discovery, area control, web search, weather forecast and persistent user-fact memory.

## Tunable runtime parameters

The YAML exposes several `number` entities so you can tune behaviour from the HA UI without re-flashing:

| Entity | Range | Default | Effect |
|--------|-------|---------|--------|
| Noise suppression | 0–4 | 2 | ESPHome built-in noise suppression strength. |
| Auto-Gain (dBFS) | 0–31 | 24 | Target loudness for automatic gain control. |
| Microphone gain | 0.5–12.0 | 2.0 | Linear gain multiplier applied after AGC. |
| Wake Probability Cutoff | 0.30–0.95 | 0.50 | microWakeWord detection threshold. Higher = fewer false wakes, more misses. |
| Wake Sliding Window | 1–20 | 10 | Number of consecutive frames to consider for wake-word voting. **Requires recompile.** |

Two switches let you mute the mic (`Microphone mute`) or disable wake-word detection entirely (`Wake Word Detection`). State is restored across reboots.

## Build / flash

The ESPHome HA add-on cannot compile this firmware: ESP32-S3 + microWakeWord peaks at 2–4 GB RAM, exceeding what HA-OS host containers provide. Two supported paths instead:

### A) Pre-built releases (recommended)

Each tagged release on this repo is built by the GitHub Actions workflow in `.github/workflows/release.yml`, with `firmware.factory.bin`, `firmware.ota.bin` and a `manifest.json` attached. Devices already running v8.1+ poll `manifest.json` on `main` every 30 min and surface available updates as a HA `update.<device>_firmware_update` entity — click **Install** and the device self-flashes.

To cut a new release: `git tag vX.Y.Z && git push origin vX.Y.Z` (or use `workflow_dispatch` with a `version` input).

### B) Local docker compile (development)

```bash
cp secrets.example.yaml secrets.yaml   # fill in real wifi / OTA / AP passwords
docker run --rm -v "$PWD:/config" -w /config ghcr.io/esphome/esphome:latest \
  -s fw_version dev compile smartmic.yaml
# Outputs:
#   .esphome/build/smartmic/.pioenvs/smartmic/firmware.factory.bin   (USB first-flash)
#   .esphome/build/smartmic/.pioenvs/smartmic/firmware.ota.bin       (OTA upload)
```

First flash of a brand-new unit: open [web.esphome.io](https://web.esphome.io), connect via USB-C in bootloader mode (hold BOOT, tap RESET), upload `firmware.factory.bin` to offset `0x0`.

OTA from local: `docker run … upload smartmic.yaml --device <ip>` — needs the device's current `ota_password` from `secrets.yaml`.

`secrets.yaml` and `*.bak-*` backups are intentionally not committed.

## Architecture notes

- **Pattern B satellite** — no local audio sink. The HA pipeline's `continue_conversation` flow needs help: `start_follow_up` API service (defined under `api.services`) is invoked by Mic to MediaPlayer after TTS playback ends, which calls `voice_assistant.start` so the firmware re-enters STT without another wake-word detection.
- **Loudness sensor** — a callback on the I2S mic stream computes a smoothed dBFS RMS estimate, surfaced as `sensor.mikrofon_loudness_dbfs` for diagnostics.
- **State text sensor** — `Ready` / `Listening` / `Processing` / `Responding` / `Muted` / `Off` / `Error` / `Timer running` / `Timer finished` for use in dashboards.
- **Voice timer hooks** — `on_timer_started/finished/cancelled/updated` integrate with the HA built-in voice timer (`HassStartTimer` etc.). On `finished`, the firmware calls `mic_to_mediaplayer.announce` with the timer name; Mic2MP routes the speech to the bound speaker (with `announce: true` so any current music is ducked).
- **Fleet naming** — `name: smartmic` + `name_add_mac_suffix: true` produces `smartmic-<last6mac>` per device. The timer hook resolves `satellite_entity_id` at runtime via a lambda that reads `App.get_name()` and substitutes `-` → `_`, yielding `assist_satellite.<runtime_name>_assist_satellit` (HA German `object_id_base` "Assist-Satellit"). One firmware image, N devices, no per-device recompile.
