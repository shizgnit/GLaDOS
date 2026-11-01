# GLaDOS — Hands-free Home Assistant Voice Assistant

A fully local, wake-word-driven voice assistant for Home Assistant running on a
MacBook Pro. Say **"GLaDOS"** + your question (e.g. *"GLaDOS, what time is it?"*)
and get a spoken answer — no Assist panel, no push-to-talk, no cloud STT/TTS/LLM
round-trips required.

## Architecture at a glance

```
┌─────────────┐   mic audio    ┌──────────────────┐  forward   ┌──────────────────────┐
│ always_     │ ─────────────► │ Home Assistant   │ ─────────► │ wake_word_satellite  │
│ listening.py│                │ Assist pipeline  │            │ (wyoming, :10203)    │
│ (client)    │ ◄───────────── │ "GLaDOS"         │ ◄───────── │ oww "glados" model   │
└──────┬──────┘   TTS audio    └──────────────────┘  detect    │ + whisper verify     │
       │ play                                   STT/LLM/TTS    └──────────────────────┘
       ▼                                                        ┌──────────────────────┐
   speakers / BT         stt: faster-whisper (local)           │ wyoming-piper :10200 │
                         llm: qwen3.5:4b via LM Studio         │ en_GB-alba-medium    │
                         tts: piper en_GB-alba-medium          └──────────────────────┘
```

## Documentation

| Doc | Contents |
|-----|----------|
| [docs/architecture.md](docs/architecture.md) | Components, data flow, and how a request travels end-to-end |
| [docs/wake-word.md](docs/wake-word.md) | Custom "glados" model, two-stage verification design, bugs fixed |
| [docs/voice-tts.md](docs/voice-tts.md) | Current Piper voice + the OmniVoice GLaDOS-clone experiment (and why it was rolled back) |
| [docs/setup.md](docs/setup.md) | launchd services, restart commands, file layout |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Known issues, symptoms, and fixes |

## Status

- ✅ Wake word detection + verification (custom openWakeWord model + whisper gatekeeper)
- ✅ STT / LLM / TTS all local (faster-whisper, qwen3.5:4b via LM Studio, Piper)
- ✅ Auto-restart services via launchd; client reconnects on its own
- ⚠️ Bluetooth headset audio is flaky — see [troubleshooting](docs/troubleshooting.md#bluetooth-headset-flakiness)
