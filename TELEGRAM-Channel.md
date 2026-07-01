# OpenJarvis Telegram Channel — Setup & Usage

Chat with your local `qwen-122b` from the **Telegram app on any device (incl. iPhone)**.
Messages you send to a private bot are answered by OpenJarvis running on this Mac.

- **How it works:** OpenJarvis uses Telegram **long polling** — it *pulls* messages from
  Telegram's servers. So **no public URL, no open ports, no tunnel** are needed. The Mac
  just needs internet + the gateway daemon running.
- **Where the model runs:** locally (`qwen-122b` via LiteLLM). Only the message text
  transits Telegram's servers; inference stays on your machine.
- **Config file:** `~/.openjarvis/config.toml` (the `[channel]` / `[channel.telegram]`
  sections were already added — see step 2).

---

## What's already done vs. what you must do

| Done for you | You must do (one-time) |
|--------------|------------------------|
| `[channel] enabled = true`, `default_channel = "telegram"` added to config | Create a bot with **@BotFather** and get its **token** |
| `[channel.telegram]` section scaffolded with a placeholder token | Paste the token into config (step 2) |
| `python-telegram-bot` installed; config verified to parse | (Recommended) lock the bot to your chat ID (step 3) |

`jarvis channel status --channel-type telegram` currently reports **disconnected**
because the token is still the placeholder `REPLACE_WITH_BOTFATHER_TOKEN`.

---

## Step 1 — Create your bot (in the Telegram app)

1. Open Telegram, search for **@BotFather** (the official one, blue check), start a chat.
2. Send `/newbot`.
3. Give it a **name** (display name, anything) and a **username** (must end in `bot`,
   e.g. `ohama_jarvis_bot`).
4. BotFather replies with a **token** like:
   `123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
   Keep it secret — it's the full credential to control the bot.

Optional BotFather niceties: `/setdescription`, `/setuserpic`, and `/setprivacy` →
**Disable** if you later want it in group chats.

---

## Step 2 — Add the token

Edit `~/.openjarvis/config.toml`, in the `[channel.telegram]` section, replace the
placeholder:

```toml
[channel]
enabled = true
default_channel = "telegram"

[channel.telegram]
bot_token = "123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"   # <- your BotFather token
# allowed_chat_ids = "..."   # see step 3
```

**Alternative (keeps the secret out of the file):** leave `bot_token = ""` and export
an env var in the shell that runs the gateway:
```bash
export TELEGRAM_BOT_TOKEN="123456789:AAE..."
```
OpenJarvis reads `TELEGRAM_BOT_TOKEN` when `bot_token` is empty.

---

## Step 3 — Lock it to you (recommended)

Without an allow-list, **anyone who discovers the bot's username can use it** (and burn
your local GPU / read whatever tools you enabled). Restrict it to your own chat ID:

1. Temporarily start the gateway (step 4) with just the token set.
2. Send any message to your bot from Telegram.
3. Run `jarvis channel status --channel-type telegram` (or check `gateway logs`) to see
   the incoming chat ID. Alternatively message **@userinfobot** in Telegram — it replies
   with your numeric ID.
4. Put it in config and restart the gateway:
   ```toml
   [channel.telegram]
   allowed_chat_ids = "123456789"          # comma-separate multiple: "111,222"
   ```

Messages from any other chat ID are then ignored.

---

## Step 4 — Start the gateway

The **gateway** is the daemon that runs your enabled channels.

```bash
cd /Users/ohama/projs/openjarvis-test/OpenJarvis

.venv/bin/jarvis gateway start      # start the multi-channel gateway (background daemon)
.venv/bin/jarvis gateway status     # confirm it's running
.venv/bin/jarvis gateway logs       # tail logs (useful for the step-3 chat ID)
.venv/bin/jarvis gateway stop       # stop it
```

Verify the channel itself connected:
```bash
.venv/bin/jarvis channel status --channel-type telegram   # want: Status: connected
```

> The Mac must stay awake for the bot to respond. For always-on use, disable App Nap /
> sleep, or keep it plugged in with "prevent sleeping" on.

> ⚠️ **This dev build:** `jarvis gateway start` generates a plist that runs
> `openjarvis.daemon.gateway`, but that module is still an incomplete stub and exits
> immediately. To actually run the channel, use **`jarvis serve`** as shown in Step 5 —
> when `[channel] enabled = true`, `serve` connects the Telegram channel alongside the
> API server and stays up (blocking) under `uvicorn`.

---

## Step 5 — Run it as a launchd service (always-on, survives reboot)

To auto-start the bot at login and auto-restart it if the process dies, register a
launchd user agent (LaunchAgent). Use **`jarvis serve`** as the service command (not the
stub `daemon.gateway`), since it actually brings up the channel.

Create `~/Library/LaunchAgents/com.openjarvis.gateway.plist` (adjust the paths and
`HOME` for your environment):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openjarvis.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/ohama/projs/openjarvis-test/OpenJarvis/.venv/bin/jarvis</string>
        <string>serve</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/ohama/projs/openjarvis-test/OpenJarvis</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>/Users/ohama</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/ohama/.openjarvis/gateway.out.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/ohama/.openjarvis/gateway.err.log</string>
</dict>
</plist>
```

Load and manage it:

```bash
launchctl load   ~/Library/LaunchAgents/com.openjarvis.gateway.plist   # start (+ enable at login)
launchctl list | grep openjarvis                                       # a PID in col 1 = running
launchctl unload ~/Library/LaunchAgents/com.openjarvis.gateway.plist   # stop and unregister
```

- `RunAtLoad`: starts **at login** (not at boot before you log in). `KeepAlive`: restarts on crash.
- **Only one poller.** If you run `jarvis serve` again while the service is up, Telegram
  returns `409 Conflict`.

### Verify the Telegram connection

```bash
# 1) Process alive + port listening
launchctl list | grep openjarvis
lsof -iTCP:8090 -sTCP:LISTEN -n -P

# 2) Confirm the channel connected in the log (rich console output goes to the err log)
grep -E "Channel:|Uvicorn running|startup complete" ~/.openjarvis/gateway.err.log
#   → "Channel: telegram", "Model:  qwen-122b", "Uvicorn running on http://127.0.0.1:8090"

# 3) Confirm the bot itself is alive (using your BotFather token)
curl -s "https://api.telegram.org/bot<BOT_TOKEN>/getMe"
#   → {"ok":true,"result":{"username":"<your_bot>", ...}}
```

Finally, message the bot from Telegram on your iPhone — if local `qwen-122b` replies,
the always-on service is complete.

---

## Using it

- Open your bot in Telegram (from the BotFather link or search its username) and just
  **send a message** — e.g. *"summarize the news on fusion energy in 3 bullets"*.
- It replies using your **default agent + model** from config (`orchestrator` + `qwen-122b`).
  Long replies are auto-split at Telegram's 4096-char limit.
- Works from **any device signed into your Telegram** — iPhone, iPad, desktop — since
  the bot lives in your Telegram account, not on one device.

### Send an outbound message from the CLI (notifications)
You can also push messages *to* Telegram from scripts:
```bash
.venv/bin/jarvis channel send telegram "Build finished ✅"
```

---

## Managing & troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `Status: disconnected` | Token still placeholder / wrong | Put the real BotFather token in config (step 2) |
| Bot never replies | Gateway not running | `jarvis gateway start`; check `gateway logs` |
| Bot ignores you specifically | Your chat ID not in `allowed_chat_ids` | Add your ID, or clear the allow-list to test |
| Replies stop when you close the laptop | Mac slept | Keep it awake / plugged in |
| Someone else could use it | No allow-list | Set `allowed_chat_ids` (step 3) |
| Token leaked | — | In BotFather: `/revoke` to issue a new token, then update config |

Useful:
```bash
.venv/bin/jarvis config show                         # confirm channel config loaded
.venv/bin/jarvis channel status --channel-type telegram
.venv/bin/jarvis gateway logs
```

---

## Security notes

- **Set `allowed_chat_ids`** unless you truly want the bot open — it consumes your local
  compute and can invoke whatever tools the default agent has (`shell_exec`, `file_write`, …).
- Consider giving the Telegram-facing setup a **less powerful default agent/tools** if you
  won't lock the allow-list.
- The **token is a secret**. Prefer the `TELEGRAM_BOT_TOKEN` env var over committing it to
  files; `/revoke` in BotFather if it ever leaks.
- Message text does pass through **Telegram's servers** (not end-to-end encrypted for bots).
  The model and your data stay local, but don't send secrets you wouldn't put in a normal
  Telegram chat.

---

## Related docs
- `USING-OpenJarvis.md` — full CLI/usage guide (Telegram is one of several channels)
- `LLM-STACK-REFERENCE.md` — the local model endpoint the bot answers with
- `SETUP-OpenJarvis.md` — how this OpenJarvis install was built
