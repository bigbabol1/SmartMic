# SmartMic

ESPHome firmware for DIY voice satellites paired with Home Assistant Assist.

## Devices

| File | Hardware | Wake word | Notes |
|------|----------|-----------|-------|
| `seeed-mic1.yaml` | XIAO ESP32-S3 (8 MB flash, octal PSRAM) + INMP441 I2S mic (mono, right channel) + onboard LED on GPIO4 | `hey_jarvis` (microWakeWord, on-device) | No onboard speaker — TTS routed through [Mic to MediaPlayer](https://github.com/bigbabol1/HomeAssistant_mic_to_mediaplayer) to an external `media_player`. |

## Pinout (XIAO ESP32-S3)

| Pin | Function |
|-----|----------|
| GPIO7 | I2S BCLK |
| GPIO8 | I2S LRCLK / WS |
| GPIO9 | I2S DIN (mic data) |
| GPIO4 | Status LED (active low) |

## Companion stack

These satellites are designed to work with the full voice stack:

1. **This firmware** — wake word + audio capture, sends audio to HA pipeline.
2. **[Mic to MediaPlayer](https://github.com/bigbabol1/HomeAssistant_mic_to_mediaplayer)** — captures TTS audio from the pipeline and plays it on a configured `media_player` (e.g. AirPlay speaker, TV, Alexa). Also drives the `start_follow_up` ESPHome service for `continue_conversation` follow-up turns without re-triggering the wake word.
3. **[AI Plugin](https://github.com/bigbabol1/AI-plugin)** — LLM-backed Home Assistant conversation entity (Ollama/OpenAI/Anthropic), with tool calling for entity discovery, area control, web search, weather forecast and persistent user-fact memory.

## Tunable runtime parameters

The YAML exposes several `number` entities so you can tune behaviour from the HA UI without re-flashing:

| Entity | Range | Default | Effect |
|--------|-------|---------|--------|
| Geräuschunterdrückung (Noise Suppression) | 0–4 | 2 | ESPHome built-in noise suppression strength. |
| Auto-Gain (dBFS) | 0–31 | 24 | Target loudness for automatic gain control. |
| Mikrofon-Verstärkung (Volume Multiplier) | 0.5–12.0 | 2.0 | Linear gain multiplier applied after AGC. |
| Wake Probability Cutoff | 0.30–0.95 | 0.50 | microWakeWord detection threshold. Higher = fewer false wakes, more misses. |
| Wake Sliding Window | 1–20 | 10 | Number of consecutive frames to consider for wake-word voting. **Requires recompile.** |

Two switches let you mute the mic (`Mikrofon stumm`) or disable wake-word detection entirely (`Wake Word Detection`). State is restored across reboots.

## Build / flash

The YAML is intended for the ESPHome Home Assistant add-on:

1. Place `seeed-mic1.yaml` in `/config/esphome/`.
2. Provide `wifi_ssid` and `wifi_password` in `/config/esphome/secrets.yaml`.
3. ESPHome dashboard → seeed-mic1 → **Install** (USB for first flash, OTA afterwards).

`secrets.yaml` and `*.bak-*` backups are intentionally not committed.

## Architecture notes

- **Pattern B satellite** — no local audio sink. The HA pipeline's `continue_conversation` flow needs help: `start_follow_up` API service (defined under `api.services`) is invoked by Mic to MediaPlayer after TTS playback ends, which calls `voice_assistant.start` so the firmware re-enters STT without another wake-word detection.
- **Loudness sensor** — a callback on the I2S mic stream computes a smoothed dBFS RMS estimate, surfaced as `sensor.mikrofon_loudness_dbfs` for diagnostics.
- **State text sensor** — `Bereit` / `Hört zu` / `Verarbeite` / `Antwortet` / `Stumm` / `Aus` / `Fehler` for use in dashboards.
