# Troubleshooting

## "The wake word isn't working" — checklist

1. **Is the satellite up?** `nc -z 127.0.0.1 10203` and check
   `~/homeassistant/voice/logs/wake.err.log` for `Wake word satellite listening`.
2. **Are candidates even firing?** The log prints a throttled line per handler:
   `oww handler=… best=glados score=… (threshold 0.50)`. If scores stay ~0.0 while you
   speak, the model isn't seeing your audio (wrong mic? client not streaming?).
3. **Are candidates being verified?** Look for `wake-word verification PASS/REJECT …`
   lines with transcripts. REJECTs mean whisper didn't hear "GLaDOS" — try speaking a
   bit louder/clearer; quiet attempts are the known weak spot (see two-stage
   verification in [wake-word.md](wake-word.md)).
4. **Leaked connections** — if `lsof -iTCP:10203` shows hundreds of ESTABLISHED
   sockets, a stuck client retry loop is saturating the event loop. Fix:
   `launchctl kickstart -k gui/$(id -u)/wyoming-openwakeword-wake`.

## Bluetooth headset flakiness

Symptoms (seen with a Logitech G435 / "BlackShark V3 Pro BT"):

- **Hard failure:** PortAudio error `-9986` opening the output stream → client logs
  `output device … failed; trying next` and falls back to another output. Works as designed.
- **"Connected but silent":** macOS reports the headset connected (even at 100% battery)
  while delivering no audio — playback "succeeds" into a dead end, so **no fallback
  triggers**. This is the nasty one: responses are generated and "played" but inaudible.
- **Silent mic:** input on the headset goes quiet; client aborts after ~5 s of silence
  (`audio stream aborted: … went silent`) and falls back to the built-in mic, costing a
  few seconds per session restart.

Mitigations already in place: fallback device chains for both directions + dead-device
TTL (1 h). If you're at the desk, set macOS default output/input to **MacBook Pro
Speakers / Microphone** (System Settings → Sound) and the problem disappears entirely.

## Slow responses

- **TTS engine:** OmniVoice GLaDOS clone takes ~25 s/response on this CPU; Piper is
  ~0.2 s. If responses are slow, check which TTS the pipeline uses (see [setup.md](setup.md)).
- **Verification queue:** fixed by non-blocking verification — if you see long gaps
  again, look for a chain of `verification REJECT` lines in `wake.err.log` (TV noise
  storm) and confirm the satellite is running the patched code.
- **LLM:** qwen3.5:4b via LM Studio is usually a few seconds; check LM Studio isn't
  busy with another request.

## HA log spam: "Received binary message for non-existing handler"

The client keeps streaming mic audio after a run ends (it's always listening); frames
arriving at a torn-down binary handler are logged as errors by HA and dropped. Harmless,
but it fills `ha.stderr.log` fast — filter with:

```bash
grep -v "non-existing handler" ~/homeassistant/logs/ha.stderr.log | tail
```

## Testing the satellite directly

Stream a wav through the Wyoming protocol (note: step by **bytes**, 2560 per chunk for
16-bit mono @ 16 kHz — see [wake-word.md](wake-word.md#testing-gotcha)):

```python
import asyncio, wave
from wyoming.client import AsyncTcpClient
from wyoming.event import Event

DATA = {"rate": 16000, "width": 2, "channels": 1}

async def main():
    w = wave.open("/tmp/glados_test_16k.wav")
    audio = w.readframes(w.getnframes())
    c = AsyncTcpClient("127.0.0.1", 10203)
    await c.connect()
    await c.write_event(Event(type="audio-start", data=DATA))
    for i in range(0, len(audio), 2560):
        await c.write_event(Event(type="audio-chunk", data=DATA, payload=audio[i:i+2560]))
    ev = await asyncio.wait_for(c.read_event(), timeout=30)
    print(ev.type, ev.data)   # expect: detection {'name': 'glados', ...}
    await c.disconnect()

asyncio.run(main())
```

## Log locations (quick reference)

| What | File |
|------|------|
| Client (wake word detected / playing response / device fallbacks) | `~/homeassistant/voice/logs/always.err.log` |
| Satellite (scores, verification PASS/REJECT + transcripts) | `~/homeassistant/voice/logs/wake.err.log` |
| Home Assistant (pipeline errors, TTS proxy) | `~/homeassistant/logs/ha.stderr.log` |
| Piper / OmniVoice TTS servers | `~/homeassistant/voice/logs/piper.err.log`, `omnivoice.err.log` |
