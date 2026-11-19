# Setup & Services

All services are macOS **launchd** agents in `~/Library/LaunchAgents/` (auto-start at
login, auto-restart on crash).

## Service inventory

| Label | What it runs | Port | Status |
|-------|--------------|------|--------|
| `home-assistant` | Home Assistant (`venv/bin/hass --config ~/homeassistant/config`) | 8123 | ✅ running |
| `wyoming-openwakeword-wake` | `wake_word_satellite.py` (custom wake word + verification) | 10203 | ✅ running |
| `wyoming-faster-whisper-stt` | wyoming-faster-whisper STT server | 10201 | ✅ running |
| `wyoming-piper-tts` | wyoming-piper, plain Piper voices (fast TTS fallback) | 10200 | ✅ running |
| `wyoming-omnivoice-tts` | wyoming-piper `--backend omnivoice` (GLaDOS-clone voice) | 10205 | ✅ running — active pipeline TTS (re-enabled 2026-11-01) |
| `ha-always-listening` | `always_listening.py --source mic` (the client) | — | ✅ running |
| `homebrew.mxcl.nginx` | nginx static file host (`html/`, `/projects` autoindex) | 8080 | ✅ running |

Not a launchd agent: **LM Studio** (macOS app, `localhost:1234`) serves the
qwen3.5:4b model to the litellm integration — start it manually or via Login
Items; if it isn't running, the LLM stage fails and responses go silent.

## File layout

```
~/homeassistant/
├── config/                     # HA configuration + .storage (pipeline, entries, registries)
├── venv/                       # HA runtime: uv-managed CPython 3.14 (no pip script; use `python -m pip`)
├── logs/                       # launchd capture: ha.stderr.log, ha.stdout.log
├── glados                      # long-lived access token (valid — used for API checks)
├── ha_token                    # stale old token (401) — keep for reference only
├── wire_ha.py                  # helper that reads the glados token
├── update-ha.sh                # updater: pip -U homeassistant + launchctl kickstart
├── ha-backup-*.tar.gz          # pre-cleanup config backup (2026-11-14)
└── voice/
    ├── always_listening.py     # client: mic → HA, TTS playback
    ├── wake_word_satellite.py  # custom wyoming wake-word server
    ├── wake_word_satellite.py.bak-20261114   # pre near-miss-gate state
    ├── glados_model/           # trained openWakeWord model (glados.onnx + .onnx.data)
    ├── models/
    │   ├── piper/              # Piper voices (en_GB-alba-medium, …)
    │   ├── whisper/            # faster-whisper small.en weights
    │   └── glados_ref/         # OmniVoice reference audio (GLaDOS voice clone)
    ├── oww-repo/               # openWakeWord trainer checkout (clone of the repo)
    ├── train-venv/             # venv for the oww training toolchain
    ├── mit_rirs/               # MIT-DEC room impulse responses (training reverb)
    ├── samples/                # generated wake-word training samples
    ├── background_clips/       # negative/background audio for training
    ├── piper-sample-generator/ # tooling for the Piper/TTS experiments
    ├── run_train.py, glados_train.yml, download_training_data.sh
    ├── train_glados.log, eval_*.log   # training/eval runs
    ├── logs/                   # always.{log,err.log}, wake.{log,err.log}, piper.*, omnivoice.*, whisper.*
    └── venv/                   # Python 3.11 venv for all runtime voice services
```

## Useful commands

```bash
U=$(id -u)

# Restart a single service (kill + relaunch):
launchctl kickstart -k gui/$U/wyoming-openwakeword-wake   # wake word satellite
launchctl kickstart -k gui/$U/ha-always-listening         # client
launchctl kickstart -k gui/$U/home-assistant              # Home Assistant

# Stop / start (e.g. the omnivoice TTS):
launchctl bootout    gui/$U/wyoming-omnivoice-tts
launchctl bootstrap  gui/$U ~/Library/LaunchAgents/wyoming-omnivoice-tts.plist

# Update Home Assistant (UI self-update is broken on this install):
~/homeassistant/update-ha.sh          # pip -U homeassistant, then kickstart the agent

# Check status:
launchctl print gui/$U/ha-always-listening | head -20

# Quick port checks:
nc -z 127.0.0.1 8123    # HA
nc -z 127.0.0.1 10203   # wake satellite
nc -z 127.0.0.1 10201   # faster-whisper STT
nc -z 127.0.0.1 10205   # omnivoice TTS
nc -z 127.0.0.1 10200   # piper TTS
nc -z 127.0.0.1 1234    # LM Studio (LLM)
```

## Pipeline configuration

The Assist pipeline ("GLaDOS") is stored in HA storage, not YAML:
`~/homeassistant/config/.storage/assist_pipeline.pipelines`. Current stage config:

- `wake_word_entity = wake_word.openwakeword`, `wake_word_id = glados`
- `stt_engine = stt.faster_whisper` (`en`)
- `conversation_engine = conversation.qwen_qwen3_5_4b` (qwen3.5:4b via LM Studio)
- `tts_engine = tts.omnivoice`, `tts_voice = glados` (the GLaDOS-clone voice;
  `tts.piper` / `en_GB-alba-medium` is the fast fallback)

Backups of previous states sit next to it (`*.bak-0059`, `*.bak-omnivoice`,
`*.bak-piper`). To change the pipeline while HA is running, stop HA → edit the
JSON → start HA (this HA version exposes no websocket command for pipeline
updates).

## Updating Home Assistant

`~/homeassistant/update-ha.sh`:

```bash
venv/bin/python -m pip install -U homeassistant
launchctl kickstart -k gui/$(id -u)/home-assistant
```

(UI-initiated core updates don't work on this install — the venv is uv-managed
with no `pip` shim, which the UI update flow expects.)

## Home Assistant configuration (post-relocation)

After the move to the new home (2026-11-14 cleanup, see
[networking.md](networking.md)):

- Location: Boise, `America/Boise` timezone; `internal_url = http://glados.aperture:8123`
- Recorder DB purged of old-home history (states/events/statistics), vacuumed
- TTS cache cleared, old areas/registries pruned, stale old-home entries removed
  (elgato ×2, thread, cast)
- Full pre-cleanup backup: `~/homeassistant/ha-backup-20261114-1003.tar.gz`