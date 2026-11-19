# GLaDOS — Hands-free Home Assistant Voice Assistant

A fully local, wake-word-driven voice assistant for Home Assistant running on a
MacBook Pro (glados). Say **"GLaDOS"** + your question (e.g. *"GLaDOS, what time
is it?"*) and get a spoken answer in a GLaDOS voice — no Assist panel, no
push-to-talk, no cloud STT/LLM round-trips.

## Architecture at a glance

```
┌─────────────┐   mic audio    ┌──────────────────┐  forward   ┌──────────────────────┐
│ always_     │ ─────────────► │ Home Assistant   │ ─────────► │ wake_word_satellite  │
│ listening.py│                │ Assist pipeline  │            │ (wyoming, :10203)    │
│ (client)    │ ◄───────────── │ "GLaDOS"         │ ◄───────── │ oww "glados" model   │
└──────┬──────┘   TTS audio    └──────────────────┘  detect    │ + whisper verify     │
       │ play                                   STT/LLM/TTS    └──────────────────────┘
       ▼                                                        ┌──────────────────────┐
   speakers / BT         stt: faster-whisper :10201            │ OmniVoice TTS :10205 │
                         llm: qwen3.5:4b via LM Studio         │ (GLaDOS voice clone) │
                         tts: omnivoice “glados”               │ Piper fallback       │
                         (fast when omnivoice is off)          └──────────────────────┘
                                                              └──────────────────────┘
```

## Documentation

| Doc | Contents |
|-----|----------|
| [docs/architecture.md](docs/architecture.md) | Components, data flow, and how a request travels end-to-end |
| [docs/wake-word.md](docs/wake-word.md) | Custom "glados" model, three-gate verification design, bugs fixed |
| [docs/voice-tts.md](docs/voice-tts.md) | OmniVoice GLaDOS-clone TTS (current) + Piper fallback, history |
| [docs/setup.md](docs/setup.md) | launchd services, restart/update commands, file layout |
| [docs/networking.md](docs/networking.md) | UniFi VLANs, why glados lives on the IOT segment, IOT devices (Govee/Levoit) |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Known issues, symptoms, and fixes |

## Status

- ✅ Wake word detection + 3-gate verification (custom openWakeWord model, two-stage
  whisper, near-miss gate — the 2026-11-14 fix for over-triggering)
- ✅ STT / LLM / TTS all local (faster-whisper, qwen3.5:4b via LM Studio, OmniVoice)
- ✅ Auto-restart services via launchd; client reconnects on its own
- ✅ Relocated to the new UniFi network; IOT devices discovered (5 Govee lights) —
  see [networking](docs/networking.md)
- ⚠️ Levoit purifier: vSync (VeSync) cloud login still to be completed in the HA UI
- ⚠️ Bluetooth headset audio is flaky — see [troubleshooting](docs/troubleshooting.md#bluetooth-headset-flakiness)
- ⚠️ UniFi gateway has intermittent full-network outages — see [networking](docs/networking.md#unifi-gateway-flakiness)
