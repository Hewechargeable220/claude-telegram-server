# Claude Code Telegram Server

Run Claude Code as a 24/7 Telegram bot on a headless Mac mini / MacBook server. Send messages from your phone — Claude works on your files, runs commands, and replies directly in Telegram.

## What This Is

A complete guide to setting up a dedicated Mac as an always-on Claude Code server accessible via Telegram. No screen, no keyboard — just SSH and your phone.

## Architecture

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────┐
│   Telegram   │────▶│  Mac Server (tmux)   │────▶│  Your Code   │
│   (phone)    │◀────│  Claude Code + Plugin │◀────│  & Terminal  │
└──────────────┘     └──────────────────────┘     └──────────────┘
```

You message your Telegram bot → the plugin pushes it into Claude Code → Claude reads/edits files, runs commands → replies back to Telegram.

## Requirements

- A Mac (mini, MacBook, etc.) running macOS 26+
- Claude account with **Max** or **Pro** subscription
- Stable Wi-Fi or Ethernet connection
- Fake HDMI adapter (for headless MacBooks — prevents GPU sleep)
- Telegram account

## Server Setup

### 1. Network Configuration

Set a static IP so the server is always reachable via SSH:

```bash
# Set static IP (replace with your values)
sudo networksetup -setmanual Wi-Fi 10.0.0.19 255.255.255.0 10.0.0.1

# Set DNS (Pi-hole + Google fallback)
sudo networksetup -setdnsservers Wi-Fi 10.0.0.17 8.8.8.8
```

> **Tip:** For persistence across reboots, configure the static IP via **System Settings > Network > Wi-Fi > Details > TCP/IP** instead of the command line.

### 2. Disable Sleep (24/7 Operation)

```bash
sudo pmset -a sleep 0 displaysleep 0 disksleep 0 \
  standby 0 autopoweroff 0 hibernatemode 0 networkoversleep 0
```

Verify with:

```bash
pmset -g | grep -E 'sleep|standby|hibernate|autopoweroff'
```

All values should be `0`.

### 3. Enable Remote Login (SSH)

```bash
sudo systemsetup -setremotelogin on
```

Or: **System Settings > General > Sharing > Remote Login**

### 4. SSH Key Setup (Passwordless Access)

From your development machine:

```bash
ssh-copy-id user@10.0.0.19
```

Add an alias to `~/.ssh/config`:

```
Host myserver
    HostName 10.0.0.19
    User bujosa
    IdentityFile ~/.ssh/id_ed25519
```

Now just `ssh myserver` — no password needed.

## Installing Dependencies

### Homebrew

```bash
NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval $(/opt/homebrew/bin/brew shellenv)' >> ~/.zshrc
```

### Node.js

```bash
brew install node
```

### Bun

```bash
curl -fsSL https://bun.sh/install | bash
echo 'export BUN_INSTALL="$HOME/.bun"' >> ~/.zshrc
echo 'export PATH="$BUN_INSTALL/bin:$PATH"' >> ~/.zshrc
```

### tmux

```bash
brew install tmux
```

### GitHub CLI

```bash
brew install gh
gh auth login
```

## Claude Code Setup

### Install

```bash
npm install -g @anthropic-ai/claude-code
```

### Authenticate

```bash
claude auth login --claudeai
```

This opens a browser URL. Visit it, sign in with your Claude Max/Pro account, and paste the code back.

## Telegram Bot Setup

### 1. Create a Bot

1. Open [@BotFather](https://t.me/BotFather) in Telegram
2. Send `/newbot`
3. Choose a name and username (must end in `bot`)
4. Copy the token

### 2. Install the Plugin

```bash
# Add the official marketplace
claude plugin marketplace add anthropics/claude-plugins-official

# Install the Telegram plugin
claude plugin install telegram@claude-plugins-official
```

### 3. Configure the Token

```bash
mkdir -p ~/.claude/channels/telegram
echo 'TELEGRAM_BOT_TOKEN=your-token-here' > ~/.claude/channels/telegram/.env
```

### 4. Launch with Telegram Channel

```bash
claude --channels plugin:telegram@claude-plugins-official
```

### 5. Pair Your Account

1. Send any message to your bot in Telegram
2. The bot replies with a **pairing code**
3. In Claude Code, run:

```
/telegram:access pair <code>
```

### 6. Lock Access (Allowlist)

Only your Telegram account can interact with the bot:

```
/telegram:access policy allowlist
```

## Running 24/7 with tmux

```bash
# Start a detached tmux session
tmux new-session -d -s claude-telegram \
  'claude --channels plugin:telegram@claude-plugins-official'

# Check it's running
tmux list-sessions

# Attach to see what's happening
tmux attach -t claude-telegram

# Detach without stopping: Ctrl+B, then D
```

### Auto-start on Boot (LaunchAgent)

Create `~/Library/LaunchAgents/com.claude.telegram.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.telegram</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/zsh</string>
        <string>-c</string>
        <string>export PATH=/opt/homebrew/bin:$HOME/.bun/bin:$PATH && tmux new-session -d -s claude-telegram 'claude --channels plugin:telegram@claude-plugins-official'</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>
</dict>
</plist>
```

Load it:

```bash
launchctl load ~/Library/LaunchAgents/com.claude.telegram.plist
```

## Usage

Once everything is running, just message your Telegram bot:

- *"list all files in the project"*
- *"fix the login bug in auth.ts"*
- *"run the tests and tell me if they pass"*
- *"create a new API endpoint for user profiles"*
- *"commit the changes with a descriptive message"*

Claude Code works with your actual files and terminal on the server — it's like having your IDE in your pocket.

## Headless MacBook Tips

| Issue | Solution |
|-------|----------|
| GPU sleep with lid closed | Use a **fake HDMI adapter** |
| Wi-Fi drops with lid closed | Use **Ethernet** (USB-C adapter) or keep lid open |
| Static IP resets after reboot | Configure via **System Settings GUI**, not CLI |
| SSH stops after reboot | Enable Remote Login in System Settings |

## Security Notes

- **Allowlist**: Only your Telegram ID can message the bot
- **Permissions**: Claude Code asks for approval before running potentially dangerous commands
- **OAuth**: Uses your Claude subscription, not API keys
- The tmux session must be running for the bot to respond

## Stack

- macOS 26+ (Apple Silicon)
- Claude Code v2.1.81+
- Telegram Bot API
- Claude Code Telegram Plugin (official)
- tmux for session persistence
- Homebrew, Node.js, Bun

## License

MIT

---

Built by [openclaw](https://github.com/bujosa) 🐾
