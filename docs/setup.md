# Setup & Services

All services are macOS **launchd** agents in `~/Library/LaunchAgents/` (auto-start at
login, auto-restart on crash).

## Service inventory

| Label | What it runs | Port | Status |
|-------|--------------|------|--------|
| `home-assistant` | Home Assistant (`hass --config ~/homeassistant/config`) | 8123 | ✅ running |
| `wyoming-openwakeword-wake` | `wake_word_satellite.py` (custom wake word + verification) | 10203 | ✅ running |
| `wyoming-faster-whisper-stt` | wyoming-faster-whisper STT server | 10201 | ✅ running |
| `wyoming-piper-tts` | wyoming-piper TTS (en_GB voices) | 10200 | ✅ running |
| `ha-always-listening` | `always_listening.py --source mic` (the client) | — | ✅ running |
| `wyoming-omnivoice-tts` | wyoming-piper OmniVoice GLaDOS clone | 10205 | ⏸ stopped (rolled back, see [voice-tts.md](voice-tts.md)) |

## File layout

```
~/homeassistant/
├── config/                     # HA configuration + .storage (pipeline config lives here)
├── logs/ha.stderr.log          # Home Assistant log
└── voice/
    ├── always_listening.py     # client: mic → HA, TTS playback
    ├── wake_word_satellite.py  # custom wyoming wake-word server
    ├── glados_model/           # trained openWakeWord model (glados.onnx + .onnx.data)
    ├── models/
    │   ├── piper/              # Piper voices (en_GB-alba-medium, …)
    │   ├── whisper/            # faster-whisper small.en weights
    │   └── glados_ref/         # OmniVoice reference audio (GLaDOS voice clone)
    ├── logs/                   # always.{log,err.log}, wake.{log,err.log}, piper.err.log, omnivoice.err.log
    └── venv/                   # Python 3.11 venv for all voice services

~/Library/LaunchAgents/*.plist  # service definitions (see inventory above)
```

## Useful commands

```bash
U=$(id -u)

# Restart a single service (kill + relaunch):
launchctl kickstart -k gui/$U/wyoming-openwakeword-wake   # wake word satellite
launchctl kickstart -k gui/$U/ha-always-listening         # client
launchctl kickstart -k gui/$U/home-assistant              # Home Assistant

# Stop / start (e.g. the rolled-back omnivoice TTS):
launchctl bootout    gui/$U/wyoming-omnivoice-tts
launchctl bootstrap  gui/$U ~/Library/LaunchAgents/wyoming-omnivoice-tts.plist

# Check status:
launchctl print gui/$U/ha-always-listening | head -20

# Quick port checks:
nc -z 127.0.0.1 8123   # HA
nc -z 127.0.0.1 10203  # wake satellite
nc -z 127.0.0.1 10200  # piper TTS
```

## Pipeline configuration

The Assist pipeline ("GLaDOS") is stored in HA storage, not YAML:
`~/homeassistant/config/.storage/assist_pipeline.pipelines`. Current stage config:

- `wake_word_entity = wake_word.openwakeword`, `wake_word_id = glados`
- `stt_engine = stt.faster_whisper` (`en`)
- `conversation_engine = conversation.qwen_qwen3_5_4b` (qwen3.5:4b via LM Studio)
- `tts_engine = tts.piper`, `tts_voice = en_GB-alba-medium`

Backups of previous states sit next to it (`*.bak-*`). To change the pipeline while HA
is running, stop HA → edit the JSON → start HA (this HA version exposes no websocket
command for pipeline updates).
