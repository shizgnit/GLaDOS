# Wake word: "GLaDOS"

## The custom model

- Trained with **openWakeWord** on the phrase `glados`.
- Files: `~/homeassistant/voice/glados_model/glados.onnx` (+ `glados.onnx.data`
  external-data sidecar, ~215 KB total). Loads in ~45 ms per instance.
- Served by `wake_word_satellite.py` (Wyoming satellite on port 10203) with
  `--models-dir ~/homeassistant/voice/glados_model`.

### Model characteristics (important!)

The model is **overfit to this Mac's microphone signature**: it fires at scores of
0.75–0.99 on live room noise and TV audio, while quiet synthetic speech can score ~0.0.
Consequences:

- The threshold (0.50) cannot be raised meaningfully — real utterances and TV noise
  overlap in score space (measured distribution of candidates: 222 × ≥0.95, most of
  them false positives).
- **Whisper verification is the real gatekeeper.** Detection without it produces a
  response for every podcast sentence.

## Two-stage whisper verification

Problem: faster-whisper mishears quiet "GLaDOS" utterances ("G laddles", "Alexa, …").
Using `initial_prompt="GLaDOS"` fixes transcription — but makes whisper **hallucinate**
"GLaDOS" on pure noise (verified across multiple noise samples).

Solution implemented in `_verify_sync`:

1. **Stage 1** — transcribe the last 3 s with no prompt; accept if it matches.
2. **Stage 2** — only if stage 1 rejected *and* produced ≥ 3 words (i.e. real speech
   was present), re-transcribe with `initial_prompt="GLaDOS"` and accept a match.

The word-count gate blocks the pure-noise hallucination path while recovering quiet,
misheard wake-word attempts. Both transcripts are logged on stage-2 PASS for debugging.

## Non-blocking verification (queue fix)

Symptom: responses took 30–60 s intermittently. Root cause: `await self._verify(...)`
blocked each connection's event loop for the whole 2–8 s transcription, so with TV
noise constantly producing candidates, verifications chained back-to-back and a real
"GLaDOS" had to wait behind the backlog (audio also piled up in the socket buffer).

Fix: verification runs as an `asyncio` background task per handler; while one is in
flight, new candidates are **skipped** (the next burst re-checks with fresh audio)
instead of queuing. Audio/model processing never stalls, and HA's two concurrent
handlers can verify independently.

## Shared-model contamination (fixed)

openWakeWord's `Model.predict()` keeps a 16-frame (~1.28 s) context window **per
instance**. The satellite originally shared one `Model` across all handlers; interleaved
audio from different connections cross-contaminated predictions — measured: real
"glados" scored 0.002 with two active handlers vs 0.999 alone, while noise scored ~1.0.

Fix: the handler factory creates a **per-connection `Model` instance** (~45 ms load).

## Tuning constants (`wake_word_satellite.py`)

| Constant | Value | Purpose |
|----------|-------|---------|
| threshold | 0.50 | oww candidate cutoff (kept low; verification is the gate) |
| `VERIFY_WINDOW_S` | 3.0 s | audio window handed to whisper |
| `VERIFY_COOLDOWN_S` | 2.0 s | min gap between verifications per handler (same-burst guard) |

## Testing gotcha

When streaming a wav through the Wyoming protocol, step by **bytes** (chunk length),
not samples: with 16-bit mono audio and 1280-sample chunks, the step is 2560 bytes.
Stepping by sample count sends overlapping/duplicated audio and silently kills
detection — this cost a long debugging session that looked exactly like "the wake word
isn't working".
