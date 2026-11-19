# Voice / TTS

## Current: OmniVoice GLaDOS-clone (re-enabled 2026-11-01)

The pipeline uses the cloned GLaDOS voice (see experiment history below) — the
assistant sounds like GLaDOS:

- Server: `wyoming-piper --backend omnivoice` on **tcp://127.0.0.1:10205**,
  `--omnivoice-ref-dir ~/homeassistant/voice/models/glados_ref`
  (reference audio of the GLaDOS voice), `--omnivoice-steps 5` (launchd agent
  `wyoming-omnivoice-tts`).
- Pipeline config: `tts_engine=tts.omnivoice`, `tts_language=en`, `tts_voice=glados`.
- Cost: **~20–30 s per response** on this CPU (diffusion inference + reference
  reload per connection). This is the price of the voice; if responses feel too
  slow, switch the pipeline to Piper (below) and relaunch the agent later.

## Fallback: Piper (en_GB-alba-medium)

The plain Piper server keeps running in parallel:

- Server: `wyoming-piper` on **tcp://127.0.0.1:10200** (launchd agent
  `wyoming-piper-tts`).
- Voices in `~/homeassistant/voice/models/piper/`:
  - `en_GB-alba-medium` ← the fallback voice
  - `en_GB-cori-high`, `en_GB-jenny_dioco-medium` (available)
- Switching the pipeline to `tts_engine=tts.piper`, `tts_voice=en_GB-alba-medium`
  restores the ~0.2 s-synthesis baseline.

## Switching between them

1. Stop HA → edit `~/homeassistant/config/.storage/assist_pipeline.pipelines`
   (`tts_engine`/`tts_voice`) → start HA (no websocket command exists for this).
   Previous states are preserved in `*.bak-omnivoice` / `*.bak-piper`.
2. Optionally stop the unused server:
   `launchctl bootout gui/$(id -u)/wyoming-omnivoice-tts` (or `wyoming-piper-tts`),
   and re-`bootstrap` it when needed.
   (Both agents come back at every login/reboot — they are installed launchd
   agents; bootout only stops them for the current session.)

## Experiment history: OmniVoice GLaDOS voice clone

Goal: make the assistant sound like GLaDOS (speech patterns + timbre).

| Aspect | Finding |
|--------|---------|
| Intelligibility | ✅ Good — whisper transcribed responses cleanly ("Oh, hello. I suppose you're here to test my capabilities.") |
| Latency | ❌ ~25 s per response (CPU diffusion inference + reference reload per connection) vs 0.2 s for Piper |
| Stability | ❌ Hung repeatedly the first week (accepted TCP connections but never answered events); HA coordinator logged `Unable to get info from 127.0.0.1:10205` all day; process had burned 150+ min CPU |

**History:** first deployed late October → rolled back to Piper (2026-10-31; the
omnivoice state preserved as `*.bak-omnivoice`) → **re-enabled 2026-11-01 22:12**
(the piper state preserved as `*.bak-piper`) and kept since — the latency trade-off
was accepted for the voice. The stability issues have not recurred (agent running
cleanly, HA log quiet on 10205).

## LLM "personality"

The GLaDOS *speech patterns* come from the LLM itself — qwen3.5:4b via LM Studio
(`localhost:1234`, litellm integration) — not from any prompt stored in HA. No custom
system prompt is configured in Home Assistant; the persona emerges from the model's
responses to voice questions.