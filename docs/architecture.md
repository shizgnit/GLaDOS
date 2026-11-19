# Architecture

Everything runs locally on a MacBook Pro (glados, Apple Silicon). Home Assistant
2026.10.x is the orchestrator; a handful of helper processes do the heavy lifting.
For where the box sits on the network and the IOT devices it sees, see
[networking.md](networking.md).

## Components

### 1. `always_listening.py` — always-on client

A Python client that keeps one Home Assistant Assist pipeline session open at all
times:

- Opens a websocket to HA (`/api/websocket`) and starts an assist **run** with the
  "GLaDOS" pipeline.
- Streams mic audio (16 kHz mono int16, ~320-byte frames) into the run's binary
  handler while waiting for events.
- On `wake-word-detected` → HA runs STT/LLM/TTS; on `tts-end` the client fetches
  the MP3 from HA's TTS proxy and plays it with `sounddevice`.
- Mutes its own mic input during playback (+ tail) so the speaker output is never
  re-captured.
- On `run-end` (or error) it closes the session, waits a short backoff, and starts
  a fresh run — this is what makes it "always listening".

Device handling: tries the **system default** input/output first; if a device hard-fails
(e.g. disconnected Bluetooth headset → PortAudio `-9986`) it falls back to other working
devices. Devices found dead are skipped for 1 hour (`DEAD_DEVICE_TTL_S`).

### 2. Home Assistant Assist pipeline ("GLaDOS")

| Stage | Engine | Notes |
|-------|--------|-------|
| Wake word | `wake_word.openwakeword` (model: **glados**) | Served by the satellite below, not HA's built-in oww |
| STT | `stt.faster_whisper` (`en`) | faster-whisper small.en, int8 on CPU |
| LLM | `conversation.qwen_qwen3_5_4b` | qwen3.5:4b served by LM Studio at `localhost:1234`, via the litellm integration |
| TTS | `tts.omnivoice` (`glados`) | wyoming-piper with the OmniVoice GLaDOS-clone backend on port 10205 — see [voice-tts.md](voice-tts.md). Plain Piper (`tts.piper`, port 10200) is the fast fallback |

### 3. `wake_word_satellite.py` — custom wake word server

A Wyoming satellite (TCP `127.0.0.1:10203`) that HA's wyoming integration points at
for the wake-word stage. It wraps openWakeWord with project-specific additions:

- **Per-handler model instances** — each client connection gets its own oww `Model`
  (see [wake-word.md](wake-word.md#shared-model-contamination) for why).
- **Whisper verification gatekeeper** — every candidate above threshold is checked
  against a 3-second audio window with faster-whisper before HA is told "detected".
  This is what makes the system usable despite the model firing on TV noise.
  As of 2026-11-14 the verification is a **three-gate** check (near-miss gate
  included) — see [wake-word.md](wake-word.md).

### 4. TTS servers (both run; pipeline picks one)

- `wyoming-piper --backend omnivoice` on **port 10205** — the OmniVoice GLaDOS-clone
  voice (`glados_ref` reference audio). This is the **active** pipeline TTS since
  2026-11-01. Slow (~20–30 s/response on this CPU) but it *sounds* like GLaDOS.
- `wyoming-piper` on **port 10200** — plain Piper with local voices in
  `~/homeassistant/voice/models/piper/` (en_GB-alba-medium, …). Fast (~0.2 s).
  Used when the pipeline is pointed at `tts.piper`.

See [voice-tts.md](voice-tts.md) for the experiment history and how to switch.

## Request lifecycle

```
user says "GLaDOS, turn on the lights"
        │
        ▼
client streams mic frames → HA run (binary handler)
        │
        ▼
HA forwards audio to wake_word_satellite (2 concurrent connections)
        │  oww scores each chunk; candidate ≥ 0.50?
        ▼
satellite verifies last 3 s with faster-whisper (three gates, non-blocking task)
        │  transcript matches "glados"? → Detection event
        ▼
HA: STT (faster-whisper) on buffered audio → LLM (qwen via LM Studio)
        │
        ▼
HA: TTS (omnivoice "glados" voice) → tts-end with proxy URL
        │
        ▼
client fetches MP3, plays it (default output device), mic muted during playback
        │
        ▼
run-end → client starts a fresh run → listening again
```

Typical end-to-end latency when everything is healthy: **~5–10 s** with Piper;
add the OmniVoice diffusion time (~20–30 s) while the GLaDOS-clone voice is
selected.