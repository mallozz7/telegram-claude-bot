# Telegram → Claude Code CLI Bot

Remote interface for **Claude Code CLI**: send text or voice messages from
Telegram and the bot executes them as prompts via `claude --print` on your
local PC, returning the response in chat.

Cross-platform: **Windows**, **Linux**, **macOS**. Localized: **en**, **it**, **fr**, **de**, **es**.

```
Smartphone (Telegram)
      │ text / voice / commands
      ▼
Telegram Bot API  ←── long polling, no port forwarding
      │
      ▼
telegram-claude-bot  (local PC)
├── text ──────────────────────────────┐
├── voice → Whisper → text ────────────┤
├── voice commands (ok/punto/...) ─────┤
│                                      ▼
│                            claude --print "<prompt>"
│                                      │
│                                      ▼
│                            response → Telegram (+ TTS optional)
│
├── /webcam  → capture video + audio from webcam
├── /stamp   → screenshot of primary monitor
├── /windows → list open windows (for /stamp <title>)
├── /task    → background execution with push notification
└── /shutdown -restart N → stop and wake after N seconds
```

---

## Prerequisites

| Component | Windows | Linux | macOS |
|---|---|---|---|
| Python >= 3.10 | python.org or winget | `apt install python3` | `brew install python` |
| Claude Code CLI | `npm install -g @anthropic-ai/claude-code` | same | same |
| ffmpeg | [gyan.dev](https://www.gyan.dev/ffmpeg/builds/) or winget | `apt install ffmpeg` | `brew install ffmpeg` |
| Telegram account | Any | Any | Any |

---

## Project structure

```
telegram-claude-bot/              # Source repository
  pyproject.toml                  # Package metadata
  src/telegram_claude_bot/        # Installable package
    __init__.py
    __main__.py                   # Entry point: python -m telegram_claude_bot
    paths.py                      # Runtime paths (~/.telegram-bot/)
    config.py                     # Configuration from .env
    bot.py                        # Telegram handlers, commands
    transcriber.py                # Audio transcription (Whisper)
    tts.py                        # Text-to-Speech (gTTS/OpenAI)
    session_manager.py            # Claude Code session management
    locales/                      # Built-in locale files
      en.json, it.json, fr.json, de.json, es.json
  service/                        # Service management (cross-platform)
    install.py                    # Setup runtime (~/.telegram-bot/)
    install-service.ps1           # Register service (Windows, admin)
    install-service.sh            # Register service (Linux/macOS)
    bot_watchdog.pyw              # Cross-platform watchdog
    run_bot.bat                   # Manual launcher (Windows)
  tests/                          # 136 tests (pytest)
  .env.example                    # Configuration template

~/.telegram-bot/                  # Runtime (created by install.py)
  .env                            # Configuration
  venv/                           # Dedicated virtual environment
  logs/                           # bot.log + watchdog.log
  service/                        # Deployed watchdog + lock file
  locales/                        # User locale overrides (optional)
```

---

## Installation

### 1. Clone the project

```bash
git clone <repo-url> telegram-claude-bot
cd telegram-claude-bot
```

### 2. Create the Telegram bot

1. Message **@BotFather** on Telegram and send `/newbot`
2. Choose a name and username, copy the **token**
3. Get your user ID from **@userinfobot** → `/start`

### 3. Run the installer

```bash
python service/install.py
```

This creates `~/.telegram-bot/` with:
- Dedicated virtual environment with all dependencies
- Copy of `.env.example` to configure
- Watchdog deployed to `service/`

### 4. Configure

<details>
<summary><b>Windows</b></summary>

```powershell
notepad $env:USERPROFILE\.telegram-bot\.env
```
</details>

<details>
<summary><b>Linux / macOS</b></summary>

```bash
nano ~/.telegram-bot/.env
```
</details>

Fill in at least:
```ini
TELEGRAM_BOT_TOKEN=your_token
ALLOWED_USER_IDS=your_user_id:YourName
WORKDIR=/home/youruser/projects
WHISPER_LANGUAGE=en
```

### 5. Register the service

<details>
<summary><b>Windows</b> — Task Scheduler (requires admin)</summary>

```powershell
.\service\install-service.ps1
```

Registers a scheduled task that:
- Starts the watchdog at login
- Restarts on wake from standby (wake trigger)
- Prevents duplicate instances (`MultipleInstances = IgnoreNew`)

Verify:
```powershell
Get-ScheduledTask -TaskName TelegramClaudeBot | Select State
```
</details>

<details>
<summary><b>Linux</b> — systemd user service</summary>

```bash
bash service/install-service.sh
```

Creates a `systemd --user` service that:
- Starts automatically at login
- Restarts on crash (`Restart=on-failure`)
- With `enable-linger`, stays active without a login session

Useful commands:
```bash
systemctl --user status telegram-claude-bot
systemctl --user restart telegram-claude-bot
journalctl --user -u telegram-claude-bot -f
```
</details>

<details>
<summary><b>macOS</b> — launchd LaunchAgent</summary>

```bash
bash service/install-service.sh
```

Creates a LaunchAgent that:
- Starts automatically at login (`RunAtLoad`)
- Restarts on crash (`KeepAlive`)
- 30s throttle between restarts

Useful commands:
```bash
launchctl list | grep telegram
launchctl unload ~/Library/LaunchAgents/com.telegram-claude-bot.plist
launchctl load ~/Library/LaunchAgents/com.telegram-claude-bot.plist
tail -f ~/.telegram-bot/logs/watchdog.log
```
</details>

### 6. Manual start (development)

```bash
python -m telegram_claude_bot
```

---

## Updating

```bash
cd telegram-claude-bot
git pull
python service/install.py --update
```

The `--update` flag re-installs the package and watchdog without overwriting `.env`.

---

## Telegram commands

### Sessions and conversation

| Command | Description |
|---|---|
| `/start` | Welcome message and help |
| `/help` | Show all commands |
| `/reset` | New conversation |
| `/sessions [-N]` | List saved sessions |
| `/resume <n>` | Resume a session (also changes directory) |
| `/task <prompt>` | Run in background with notification |
| `/tasks` | List running/recent tasks |
| `/cancel [id]` | Cancel a task |

### Navigation and files

| Command | Description |
|---|---|
| `/cd <path>` | Change working directory |
| `/ls [opts] [path]` | List files |
| `/head <file> [n]` | First n lines of a file |
| `/tail <file> [n]` | Last n lines of a file |
| `/sendme <path>` | Send file as attachment |
| `/windows [filter]` | List open windows (for use with /stamp) |
| `/stamp [window title]` | Screenshot (primary monitor or specific window) |
| `/webcam [seconds]` | Record webcam video + audio (default 3s, max 30s) |

### Settings and info

| Command | Description |
|---|---|
| `/settings` | Current settings |
| `/set <key> <value>` | Change a setting |
| `/usage` | Token usage |
| `/last` | Last prompt and response |
| `/status` | Session and task info |
| `/shutdown [-restart N]` | Stop bot and watchdog |

> **Note**: When a locale is active (e.g. `WHISPER_LANGUAGE=it`), all commands also have
> localized voice aliases. For example with Italian: `/sessioni`, `/riprendi`, `/cattura`, etc.
> Type `/help` to see the full list with aliases for your language.

### Shutdown with scheduled restart

```
/shutdown -restart 1800
```

Stops the bot, allows the PC to sleep, and schedules a restart after 1800 seconds (30 min).

| Platform | Mechanism | Wake from standby |
|---|---|---|
| Windows | Task Scheduler with `WakeToRun` | Yes (if BIOS supports wake timers) |
| Linux | `systemd-run --user --on-active` | No (runs after resume) |
| macOS | `at` command | No (runs after resume) |

Verify wake timers on Windows:
```powershell
powercfg /waketimers
```

---

## Localization

The bot UI is fully localized. The `WHISPER_LANGUAGE` setting in `.env` controls:

| Feature | Example (it) |
|---|---|
| Voice transcription language | Whisper hint → Italian |
| Voice command aliases | `/sessions` → also responds to `sessioni` |
| Voice trigger words | `"ok"`, `"punto"`, `"comando"` |
| UI messages | Errors, confirmations, labels in Italian |
| Help descriptions | Command descriptions in Italian |
| Safety patterns | Detects dangerous words in Italian |

### Built-in locales

| Code | Language | Trigger words | Example aliases |
|---|---|---|---|
| `en` | English (default) | `ok` | (none — English command names only) |
| `it` | Italian | `ok`, `punto`, `comando` | `sessioni`, `riprendi`, `cattura`, `spegni` |
| `fr` | French | `ok`, `commande` | `seances`, `reprendre`, `capture`, `eteindre` |
| `de` | German | `ok`, `befehl` | `sitzungen`, `fortsetzen`, `bildschirm`, `beenden` |
| `es` | Spanish | `ok`, `comando`, `oye` | `sesiones`, `reanudar`, `captura`, `apagar` |

### Custom locales

Create a file in `~/.telegram-bot/locales/{lang}.json` to:
- **Add a new language**: create the full file following `en.json` as template
- **Override built-in translations**: only include the keys you want to change

User files are merged on top of built-in defaults. Example partial override:
```json
{
  "aliases": {
    "webcam": ["telecamera"]
  },
  "messages": {
    "bot_started": "🟢 {greeting}Ready to go!"
  }
}
```

### Locale file structure

Each locale JSON has these sections (all optional):

```json
{
  "danger_patterns": [{"pattern": "\\bdelete\\b", "label": "file deletion"}],
  "trigger_words": ["ok", "hey"],
  "aliases":      {"sessions": ["my-alias"], "resume": ["continue"]},
  "descriptions": {"sessions": "list saved sessions"},
  "messages":     {"unauthorized": "⛔ Not allowed.", "bot_started": "🟢 {greeting}Ready!"}
}
```

---

## Voice messages

### Voice prompt
Send a voice message with any phrase → it gets transcribed and sent to Claude as a prompt.

### Voice commands
Start a voice message with a **trigger word** followed by a command:
- **"Ok"** + command: `"Ok sessions"`, `"Ok resume 1"`
- With locale triggers: `"Punto status"` (it), `"Commande capture"` (fr)

Works with both English command names and localized aliases.

### Voice response (TTS)
Responses can also be sent as audio:
```ini
TTS_PROVIDER=gtts          # gtts (free) or openai_tts
```

---

## Audio transcription

Configurable fallback chain:

```ini
TRANSCRIBER=groq_whisper,openai_whisper,faster_whisper
```

| Provider | Requirements | Notes |
|---|---|---|
| `groq_whisper` | `GROQ_API_KEY` | Cloud, fastest, free |
| `openai_whisper` | `OPENAI_API_KEY` | Cloud, reliable |
| `faster_whisper` | No key | Local, slow first load |

---

## Webcam

Captures video + audio from the webcam and sends it on Telegram.
Pipeline: native MJPEG capture (2s warm-up + duration) → trim → encode H264+AAC MP4.

```ini
# Device names (leave empty for auto-detect)
# Windows: DirectShow names, Linux: /dev/video0, macOS: AVFoundation
#WEBCAM_VIDEO_DEVICE=Integrated Camera
#WEBCAM_AUDIO_DEVICE=Microphone Array

# Resolution and framerate (must be supported by your webcam)
WEBCAM_RESOLUTION=1280x720
WEBCAM_FRAMERATE=30
```

Use `/windows` to list open windows, then `/stamp <title>` to capture a specific one.

---

## Watchdog

The bot runs under a cross-platform watchdog (`bot_watchdog.pyw`):

| Feature | Windows | Linux | macOS |
|---|---|---|---|
| Single instance | Lock file | Lock file | Lock file |
| Auto-restart | 10s delay | 10s delay | 10s delay |
| Heartbeat check | 30s write / 120s timeout | same | same |
| Crash loop alert | Telegram after 5 crashes in 5 min | same | same |
| Prevent sleep | `SetThreadExecutionState` | (systemd) | `caffeinate` |
| Kill process tree | `taskkill /T /F` | `killpg SIGTERM` | `killpg SIGTERM` |
| Logging | `~/.telegram-bot/logs/watchdog.log` | same | same |

---

## Security

> **ALLOWED_USER_IDS is critical.**
> Claude Code CLI has filesystem access and can execute commands.
> Without a whitelist, anyone with the bot link can control your PC.

- Always set `ALLOWED_USER_IDS` with your Telegram user_id
- The bot includes a **safety check** that detects potentially dangerous prompts
  (deletions, shell commands, credential access) and asks for confirmation
- Safety patterns are localized — dangerous words are detected in the active language
- Commands `/webcam`, `/stamp`, `/cd` require authorization

---

## Tests

```bash
pip install -e ".[dev,local-whisper,tts]"
pytest tests/ -v
```

136 tests covering:
- Bot utilities (safety check, split message, authorization, path resolution)
- Session manager (path encoding, session search, formatting)
- Video pipeline (ffmpeg capture → trim → encode, audio present, faststart, duration)
- Audio transcription (gTTS → faster-whisper round-trip, Italian recognition)
- TTS (synthesis, truncation, fallback chain)
- Voice commands (trigger words, locale aliases, argument parsing)
- Localization (all locales consistent, aliases match commands, user overrides)
