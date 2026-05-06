# OpenClaw Telegram Voice Note STT Setup

_Last updated: 2026-05-06_

This note summarizes a working OpenClaw Telegram voice-note setup: a Telegram bot receives voice/audio messages, OpenClaw routes the downloaded media through local Whisper transcription, and the transcribed turn can be handled by the configured OpenClaw agent.

This is a sanitized/public version. Remove real bot tokens, chat IDs, usernames, hostnames, absolute user paths, logs containing secrets, and environment-specific identifiers before publishing.

## Current Status

- Telegram bot channel: working.
- Telegram text messages: working.
- Telegram voice-note transcription: configured for local Whisper STT.
- STT: local whisper.cpp.
- Model: `ggml-base.en.bin`.
- FFmpeg: installed and available.
- TTS voice replies: not enabled in this setup. Recommended first milestone is voice-note input → text/agent reply.

## Why This Setup

Goals:

- Let the user send quick voice notes to the assistant from Telegram.
- Avoid paid speech-to-text API usage.
- Keep the setup lightweight and auditable.
- Use local Whisper for transcription.
- Keep Telegram as a practical mobile capture channel.

Important distinction:

- This is **not** live Telegram voice-chat/call participation.
- Telegram bots do not cleanly join live Telegram calls/voice chats through the normal Bot API like Discord bots can.
- This setup is for **Telegram voice notes/audio files** sent to the bot.

## Components

Typical components:

```text
~/.openclaw/tools/whisper.cpp
~/.openclaw/tools/whisper.cpp/build/bin/whisper-cli
~/.openclaw/tools/whisper.cpp/models/ggml-base.en.bin
```

Sanitize or generalize these paths before sharing if your local username or organization name appears in them.

## Prerequisites

- OpenClaw installed and gateway running.
- Telegram channel configured and connected.
- Git installed.
- CMake installed.
- FFmpeg installed.
- Enough local disk space for whisper.cpp build artifacts and model files.

On macOS with Homebrew:

```bash
brew install cmake ffmpeg
```

## Check Telegram Channel Health

```bash
openclaw channels status telegram --probe
```

Expected high-level result:

```text
Gateway reachable.
Telegram default: enabled, configured, running, connected, mode:polling, works
```

Do not publish your real bot username, token source, chat IDs, or channel metadata if present in your output.

## Install whisper.cpp Locally

```bash
mkdir -p ~/.openclaw/tools
cd ~/.openclaw/tools

git clone --depth 1 https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp

cmake -B build -DWHISPER_BUILD_TESTS=OFF -DWHISPER_BUILD_EXAMPLES=ON
cmake --build build --config Release -j2
```

## Download Whisper Model

```bash
cd ~/.openclaw/tools/whisper.cpp
mkdir -p models
bash ./models/download-ggml-model.sh base.en
```

Expected model:

```text
~/.openclaw/tools/whisper.cpp/models/ggml-base.en.bin
```

The `base.en` model is a good first default for English voice notes because it is reasonably accurate without being too slow.

## Verify Whisper CLI

```bash
~/.openclaw/tools/whisper.cpp/build/bin/whisper-cli --help | head
ls -lh ~/.openclaw/tools/whisper.cpp/models/ggml-base.en.bin
```

Optional sample test if you have a local audio file:

```bash
~/.openclaw/tools/whisper.cpp/build/bin/whisper-cli \
  -m ~/.openclaw/tools/whisper.cpp/models/ggml-base.en.bin \
  -f /path/to/test-audio.wav \
  -nt -np -l en
```

## Enable OpenClaw Audio Transcription

Relevant OpenClaw config path:

```text
tools.media.audio
```

Example config:

```json
{
  "enabled": true,
  "scope": {
    "default": "allow"
  },
  "maxBytes": 20971520,
  "models": [
    {
      "type": "cli",
      "command": "~/.openclaw/tools/whisper.cpp/build/bin/whisper-cli",
      "args": [
        "-m",
        "~/.openclaw/tools/whisper.cpp/models/ggml-base.en.bin",
        "-f",
        "{{MediaPath}}",
        "-nt",
        "-np",
        "-l",
        "en"
      ],
      "timeoutSeconds": 120
    }
  ]
}
```

Depending on your OpenClaw config loader, absolute expanded paths may be safer than `~` paths. Use whichever your environment supports.

## Backup Config Before Editing

Before editing OpenClaw config:

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak-telegram-voice-$(date +%Y%m%d-%H%M%S)
```

Then edit:

```bash
nano ~/.openclaw/openclaw.json
```

or use a JSON-aware editor/script.

## Restart Gateway

```bash
openclaw gateway restart
openclaw channels status telegram --probe
```

## Verify Config

```bash
python3 - <<'PY'
import json, os
config_path = os.path.expanduser('~/.openclaw/openclaw.json')
j = json.load(open(config_path))
a = j.get('tools', {}).get('media', {}).get('audio', {})
print('audio enabled:', a.get('enabled'))
print('model count:', len(a.get('models', [])))
if a.get('models'):
    m = a['models'][0]
    print('type:', m.get('type'))
    print('command exists:', os.path.exists(os.path.expanduser(m.get('command', ''))))
PY
```

## Test from Telegram

1. Open the Telegram chat with your OpenClaw bot.
2. Send a short voice note, for example:

```text
Nox, this is a voice-note transcription test. Add a note that Telegram voice capture is working.
```

3. Confirm the bot replies appropriately.
4. If it does not, inspect logs.

## Useful Troubleshooting Commands

Check gateway:

```bash
openclaw gateway status
```

Check Telegram channel:

```bash
openclaw channels status telegram --probe
```

Restart gateway:

```bash
openclaw gateway restart
```

Inspect current day logs:

```bash
tail -n 200 /tmp/openclaw/openclaw-$(date +%F).log
```

Search for Telegram/audio/STT logs:

```bash
grep -iE 'telegram|voice|audio|media|whisper|transcri|ffmpeg|stt' \
  /tmp/openclaw/openclaw-$(date +%F).log | tail -120
```

Verify Whisper exists:

```bash
~/.openclaw/tools/whisper.cpp/build/bin/whisper-cli --help | head
ls -lh ~/.openclaw/tools/whisper.cpp/models/ggml-base.en.bin
```

Verify FFmpeg:

```bash
ffmpeg -version | head
```

## Troubleshooting Checklist

### Telegram text works but voice notes do not transcribe

- Confirm `tools.media.audio.enabled = true`.
- Confirm the bot can receive media/audio files.
- Confirm the voice note file size is below `maxBytes`.
- Confirm OpenClaw downloaded the Telegram media locally.
- Check logs for media/audio/transcription errors.

### Whisper command not found

- Confirm whisper.cpp built successfully.
- Confirm `whisper-cli` path in config matches the actual path.
- Prefer absolute paths if `~` is not expanded by the config loader.

### Whisper model missing

- Confirm `ggml-base.en.bin` downloaded successfully.
- Confirm config path matches the actual model location.

### Transcription times out

- Increase `timeoutSeconds`.
- Use a smaller model.
- Send shorter voice notes.
- Confirm hardware is not overloaded.

### Poor transcription quality

- Speak clearly and close to the phone microphone.
- Reduce background noise.
- Try a larger Whisper model if speed allows.
- Use English model `base.en` for English-only notes; use multilingual models if needed.

## Security / Public Sharing Checklist

Before sharing this setup publicly, remove:

- Telegram bot tokens.
- Telegram chat IDs and user IDs.
- Bot usernames if you do not want them public.
- Local usernames and absolute personal paths.
- Hostnames, tunnel URLs, or gateway URLs.
- API keys, OAuth tokens, client secrets.
- Customer/client names.
- Logs containing tokens or personal metadata.

## Notes on Telegram vs Discord Voice

Discord exposes voice-channel join/speak/listen APIs that bots commonly use. Telegram bots are much better suited for receiving text, files, photos, and voice notes. Live Telegram voice-chat participation is not clean through the normal Bot API.

For most assistant workflows, Telegram voice-note capture is enough and much simpler:

```text
Voice note → local Whisper transcription → OpenClaw agent turn → Telegram reply
```


## Confirmed Test

A Telegram voice note was sent to the bot and a machine-generated transcript was produced successfully.

Example spoken phrase:

```text
Nox, this is a Telegram voice note test. I don't know that voice capture is working.
```

Result:

- Telegram voice/audio message was received.
- Audio transcript was generated and attached to the conversation context.
- The assistant was able to read and respond to the transcript.

Conclusion: Telegram voice-note capture is working for voice note → STT transcript → agent response.


# OpenClaw Telegram Voice Reply / TTS Add-On

_Last updated: 2026-05-06_

This add-on extends the Telegram voice-note STT setup with outbound assistant voice replies:

```text
Assistant text → TTS MP3 → FFmpeg OGG/Opus conversion → Telegram Bot API sendVoice
```

## Current Status

- Voice-note input/STT: working.
- Outbound TTS voice note: working via helper script.
- TTS provider used in this setup: `edge-tts` / Microsoft Neural voices.
- Telegram send method: Bot API `sendVoice`.
- Recommended usage: send voice replies only when requested, not for every assistant response.

## Install Python Dependencies

```bash
python3 -m pip install --user edge-tts requests
```

FFmpeg is also required:

```bash
brew install ffmpeg
```

## Helper Script Pattern

Create a local helper script, for example:

```text
~/.openclaw/tools/send_telegram_voice.py
```

The helper should:

1. Read the Telegram bot token from OpenClaw config.
2. Generate a temporary MP3 using `edge-tts`.
3. Convert the MP3 to Telegram-friendly OGG/Opus using FFmpeg.
4. Send the voice note with Telegram Bot API `sendVoice`.
5. Print only sanitized results — never print the bot token.

## Example Helper Script Structure

```python
#!/usr/bin/env python3
import argparse, asyncio, json, os, subprocess, tempfile
from pathlib import Path
import edge_tts
import requests


def load_token():
    config_path = Path(os.path.expanduser('~/.openclaw/openclaw.json'))
    data = json.loads(config_path.read_text())
    token = data.get('channels', {}).get('telegram', {}).get('botToken')
    if not token:
        raise RuntimeError('Telegram botToken not found')
    return token


async def synthesize(text, mp3_path, voice='en-US-MichelleNeural'):
    communicate = edge_tts.Communicate(text=text, voice=voice, rate='+0%', pitch='+0Hz')
    await communicate.save(mp3_path)


def convert_to_ogg(mp3_path, ogg_path):
    subprocess.run([
        'ffmpeg', '-y', '-i', mp3_path,
        '-ac', '1', '-ar', '48000',
        '-c:a', 'libopus', '-b:a', '32k',
        '-application', 'voip',
        ogg_path,
    ], check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)


def send_voice(token, chat_id, ogg_path, caption=''):
    url = f'https://api.telegram.org/bot{token}/sendVoice'
    with open(ogg_path, 'rb') as f:
        files = {'voice': ('assistant-voice.ogg', f, 'audio/ogg')}
        data = {'chat_id': chat_id}
        if caption:
            data['caption'] = caption[:1024]
        r = requests.post(url, data=data, files=files, timeout=60)
    r.raise_for_status()
    return r.json()
```

Sanitize the full script before publishing. Do not include real tokens, chat IDs, or local personal paths in public docs.

## Example Usage

```bash
python3 ~/.openclaw/tools/send_telegram_voice.py   --chat-id '<telegram_chat_id>'   --text 'Hello, this is a voice reply test.'   --caption 'Voice test'
```

## Confirmed Outbound Test

A voice note was generated and sent successfully through Telegram Bot API `sendVoice`.

Sanitized result pattern:

```json
{
  "ok": true,
  "message_id": 123
}
```

## Voice Options

Useful Microsoft neural voices:

- `en-US-MichelleNeural`
- `en-US-GuyNeural`
- `en-US-JennyNeural`
- `en-US-AriaNeural`
- `en-US-DavisNeural`

Recommended default for Nox-style assistant replies:

```text
en-US-MichelleNeural
```

## Recommended Voice Reply Policy

Do not make every response a voice note. Use voice replies only when the user explicitly asks, for example:

- “Reply by voice.”
- “Send that as a voice note.”
- “Voice summary please.”

This avoids notification fatigue and keeps the assistant useful instead of noisy.

## Troubleshooting Voice Replies

### TTS generation fails

- Confirm `edge-tts` is installed.
- Confirm internet access is available for Microsoft neural TTS.
- Try a shorter test phrase.

### OGG conversion fails

- Confirm FFmpeg is installed.
- Confirm FFmpeg supports `libopus`.

```bash
ffmpeg -encoders | grep opus
```

### Telegram sendVoice fails

- Confirm bot token is present in config.
- Confirm chat ID is correct.
- Confirm the bot is allowed to message the target chat/user.
- Confirm the generated OGG file is not empty.
- Do not print the token while debugging.
