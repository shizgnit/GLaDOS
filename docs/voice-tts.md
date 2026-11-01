# Voice / TTS

## Current: Piper (en_GB-alba-medium)

- Server: `wyoming-piper` on **tcp://127.0.0.1:10200** (launchd agent).
- Voices in `~/homeassistant/voice/models/piper/`:
  - `en_GB-alba-medium` ← used by the pipeline
  - `en_GB-cori-high`, `en_GB-jenny_dioco-medium` (available)
- Pipeline config: `tts_engine=tts.piper`, `tts_language=en_GB`,
  `tts_voice=en_GB-alba-medium`.
- Performance: synthesis is fast on CPU — measured **fetch ≈ 0.2 s** for a ~6–9 s
  response, playback at real time. This is the "somewhat working" baseline the project
  settled on.

## Experimented: OmniVoice GLaDOS voice clone (rolled back)

Goal: make the assistant sound like GLaDOS (speech patterns + timbre).

- Server: `wyoming-piper --backend omnivoice` on **tcp://127.0.0.1:10205**, with
  `--omnivoice-ref-dir ~/homeassistant/voice/models/glados_ref` (reference audio of the
  GLaDOS voice) and `--omnivoice-steps 5`.
- Pipeline config was switched to `tts_engine=tts.omnivoice`, `tts_voice=glados`.

Results:

| Aspect | Finding |
|--------|---------|
| Intelligibility | ✅ Good — whisper transcribed responses cleanly ("Oh, hello. I suppose you're here to test my capabilities.") |
| Latency | ❌ ~25 s per response (CPU diffusion inference + reference reload per connection) vs 0.2 s for Piper |
| Stability | ❌ Service hung repeatedly (accepted TCP connections but never answered events); HA coordinator logged `Unable to get info from 127.0.0.1:10205` all day; process had burned 150+ min CPU |

**Decision:** rolled back to Piper for reliability and latency. The omnivoice launchd
agent was stopped (`launchctl bootout gui/$(id -u)/wyoming-omnivoice-tts`).

### Re-enabling (if you want to try again)

1. `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/wyoming-omnivoice-tts.plist`
2. Set the pipeline back to `tts.omnivoice` / `glados` (previous state is preserved in
   `~/homeassistant/config/.storage/assist_pipeline.pipelines.bak-omnivoice`).
3. Expect ~20–30 s per response on this hardware; consider raising `--omnivoice-steps`
   only if quality demands it (it gets slower).

## LLM "personality"

The GLaDOS *speech patterns* come from the LLM itself — qwen3.5:4b via LM Studio
(`localhost:1234`, litellm integration) — not from any prompt stored in HA. No custom
system prompt is configured in Home Assistant; the persona emerges from the model's
responses to voice questions.
