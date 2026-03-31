# Claude Code Watchdog v2

Auto-kill hung Claude Code processes before they freeze your machine.

Part of [`bujosa/claude-telegram-server`](https://github.com/bujosa/claude-telegram-server) under `scripts/watchdog/`.

---

## Table of Contents

- [Why This Exists](#why-this-exists)
- [What Changed in v2](#what-changed-in-v2)
- [Watchdog Checks](#watchdog-checks)
- [Configuration](#configuration)
- [Safe Launcher](#safe-launcher-claude-safe-launchsh)
- [Quick Test](#quick-test)
- [Installation](#installation)
  - [macOS (LaunchAgent)](#macos-launchagent)
  - [Linux (systemd)](#linux-systemd)
- [Bot Launch Example](#bot-launch-example)
- [Log Examples](#log-examples)
- [Uninstall](#uninstall)
- [Related GitHub Issues](#related-github-issues)
- [License](#license)

---

## Why This Exists

Claude Code (the CLI) has several known resource-leak bugs that remain open as of March 2026. Left unchecked, any one of them can render a machine completely unresponsive within minutes.

| Bug | Symptom | Impact |
|-----|---------|--------|
| CPU burn | Processes pin at 80-100% CPU after a session closes | Machine becomes sluggish, fans run constantly |
| Memory leak (ArrayBuffers) | RSS grows at ~36 MB/s indefinitely | All RAM consumed in 2-5 minutes, system freezes or OOM-kills critical processes |
| caffeinate leak (macOS) | Thousands of `caffeinate` processes spawned | PID table exhaustion, prevents sleep, wastes resources |
| Orphaned processes | Desktop app and VS Code do not kill child processes on exit | Zombie Claude processes accumulate across sessions |

If you run Claude Code as a long-lived bot (Telegram, Slack, etc.) or even just use it interactively throughout the day, you need something watching for these failures. The watchdog runs on a tight loop, detects the problems early, and kills the offending processes before damage spreads.

---

## What Changed in v2

v1 ran every 5 minutes and only checked CPU. That is far too slow -- the memory leak can consume all RAM in under 2 minutes, freezing the machine before the watchdog even wakes up.

| Feature | v1 | v2 |
|---------|----|----|
| Check interval | 300s (5 min) | **60s** |
| Memory monitoring | None | **Kills processes using >3 GB RSS** |
| Memory growth tracking | None | **Kills processes growing >500 MB between checks** |
| Minimum process age | 120s | **60s** |
| Safe launcher script | None | **`claude-safe-launch.sh` with `nice` + `ulimit`** |
| Auto-restart | None | **Restarts bot up to 5 times in a 5-minute window** |
| caffeinate cleanup | Basic | Unchanged |
| CPU hog detection | >80% CPU, >120s | >80% CPU, **>60s**, no TTY |
| Orphan detection | Basic | Parent=PID 1, >10% CPU, >60s |

---

## Watchdog Checks

The checks run in a specific order. Memory issues are checked first because they are the most dangerous -- a runaway ArrayBuffer leak will freeze your machine before high CPU ever matters.

### 1. Kill orphaned caffeinate (macOS only)

Finds `caffeinate` processes whose parent Claude process no longer exists and kills them. This addresses the caffeinate leak where thousands of these processes accumulate.

### 2. Kill memory hogs (>3 GB RSS)

Any Claude-related process using more than 3 GB of resident memory is killed immediately. This is checked first because the memory leak is the fastest path to a frozen machine.

### 3. Kill memory growth (>500 MB between checks)

The watchdog snapshots each process's RSS on every run. If a process grew by more than 500 MB since the last check (roughly 60 seconds ago), it is killed. This catches the ArrayBuffer leak early, before the process reaches the 3 GB hard ceiling.

Snapshots are stored as small files in the directory specified by `MEM_SNAPSHOT_DIR`.

### 4. Kill CPU hogs (>80% CPU, >60s old, no TTY)

Processes consuming more than 80% CPU, older than 60 seconds, and not attached to a terminal (no TTY) are killed. The TTY check prevents the watchdog from killing interactive Claude sessions where the user is actively working.

### 5. Kill orphaned children (parent=PID 1, >10% CPU, >60s)

When a parent process (VS Code, Claude Desktop, a terminal) exits without cleaning up, child Claude processes are re-parented to PID 1 (init/launchd). These orphans continue consuming resources. The watchdog finds them by checking for Claude processes whose PPID is 1, that use more than 10% CPU, and are older than 60 seconds.

---

## Configuration

All settings are controlled via environment variables. Defaults are sane for most setups.

| Variable | Default | Description |
|----------|---------|-------------|
| `CPU_THRESHOLD` | `80` | CPU usage percentage above which a process is considered a hog |
| `MEM_THRESHOLD_MB` | `3072` | RSS in MB above which a process is killed immediately (3 GB) |
| `MEM_GROWTH_THRESHOLD_MB` | `500` | RSS growth in MB between checks that triggers a kill |
| `MIN_AGE_SECONDS` | `60` | Minimum process age before it becomes eligible for killing |
| `LOG_FILE` | `/tmp/claude-watchdog.log` | Path to the log file |
| `DRY_RUN` | `false` | Set to `true` to log what would be killed without actually killing anything |
| `MEM_SNAPSHOT_DIR` | `/tmp/claude-watchdog-snapshots` | Directory for per-process memory snapshots used by growth tracking |

Example override:

```bash
MEM_THRESHOLD_MB=2048 CPU_THRESHOLD=70 bash claude-watchdog.sh
```

---

## Safe Launcher (claude-safe-launch.sh)

The safe launcher wraps Claude Code with OS-level resource limits so that even if the watchdog is not running, a single process cannot consume the entire machine.

### What it does

**`nice -n 10`** -- Lowers the CPU scheduling priority of the Claude process. The process still gets CPU time, but the kernel will prefer other processes (your desktop, SSH, the watchdog itself) when the system is under load. This keeps the machine responsive even during a CPU burn bug.

**`ulimit -v 4194304`** (4 GB) -- Sets a virtual memory ceiling for the process. If Claude tries to allocate more than 4 GB of RAM, the OS kills it with a signal. This is a hard backstop that prevents the ArrayBuffer memory leak from consuming all system RAM and freezing the machine. The process dies, the system survives.

**Auto-restart** -- If Claude dies (whether killed by the watchdog, killed by `ulimit`, or a plain crash), the launcher waits 10 seconds and restarts it. This repeats up to 5 times within a 5-minute window. If the process keeps dying faster than that, the launcher gives up and exits -- something is seriously wrong and restarting will only make it worse.

### Launcher configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_RAM_KB` | `4194304` | Virtual memory limit in KB (4 GB). Passed to `ulimit -v`. |
| `NICE_LEVEL` | `10` | CPU scheduling priority offset. Higher = lower priority. Range: 0-19. |
| `BOT_MODE` | `false` | Set to `true` to launch Claude in MCP bot mode (Telegram, etc.) |
| `TELEGRAM_PLUGIN` | `""` | Path to the Telegram MCP plugin, if applicable |
| `TMUX_SESSION` | `claude-bot` | Name of the tmux session to create (if using tmux) |
| `RESTART_ON_CRASH` | `true` | Whether to auto-restart Claude if it exits unexpectedly |
| `RESTART_DELAY` | `10` | Seconds to wait before restarting |
| `MAX_RESTARTS` | `5` | Maximum number of restarts allowed within the restart window |
| `RESTART_WINDOW` | `300` | Window in seconds for counting restarts (5 minutes) |

---

## Quick Test

Run the watchdog in dry-run mode to see what it would kill without actually killing anything:

```bash
DRY_RUN=true bash claude-watchdog.sh
```

Expected output (no problems found):

```
[2026-03-31 14:22:01] Watchdog v2 check started
[2026-03-31 14:22:01] Checking for orphaned caffeinate processes...
[2026-03-31 14:22:01] Checking for memory hogs (>3072 MB)...
[2026-03-31 14:22:01] Checking for memory growth (>500 MB since last check)...
[2026-03-31 14:22:01] Checking for CPU hogs (>80%, >60s, no TTY)...
[2026-03-31 14:22:01] Checking for orphaned children (PPID=1, >10% CPU, >60s)...
[2026-03-31 14:22:01] Watchdog v2 check complete. 0 processes killed.
```

Expected output (problem detected, dry run):

```
[2026-03-31 14:22:01] Watchdog v2 check started
[2026-03-31 14:22:01] Checking for memory hogs (>3072 MB)...
[2026-03-31 14:22:01] [DRY RUN] Would kill PID 48291 (claude, RSS=4812 MB, age=184s)
[2026-03-31 14:22:01] Watchdog v2 check complete. 0 processes killed (1 would be killed).
```

---

## Installation

### macOS (LaunchAgent)

A LaunchAgent runs the watchdog every 60 seconds under your user account. No root required.

**1. Copy the watchdog script:**

```bash
sudo cp claude-watchdog.sh /usr/local/bin/claude-watchdog.sh
sudo chmod +x /usr/local/bin/claude-watchdog.sh
```

**2. Copy the LaunchAgent plist:**

```bash
cp com.bujosa.claude-watchdog.plist ~/Library/LaunchAgents/
```

The plist should contain:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.bujosa.claude-watchdog</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/usr/local/bin/claude-watchdog.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>60</integer>
    <key>StandardOutPath</key>
    <string>/tmp/claude-watchdog-launchd.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/claude-watchdog-launchd.log</string>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

**3. Load the agent:**

```bash
launchctl load ~/Library/LaunchAgents/com.bujosa.claude-watchdog.plist
```

**4. Verify it is running:**

```bash
launchctl list | grep claude-watchdog
```

You should see a line with the label `com.bujosa.claude-watchdog` and a PID or status code.

### Linux (systemd)

On Linux, a systemd timer replaces the LaunchAgent. This requires root for system-wide installation.

**1. Copy the watchdog script:**

```bash
sudo cp claude-watchdog.sh /usr/local/bin/claude-watchdog.sh
sudo chmod +x /usr/local/bin/claude-watchdog.sh
```

**2. Create the service unit** at `/etc/systemd/system/claude-watchdog.service`:

```ini
[Unit]
Description=Claude Code Watchdog v2
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/bin/claude-watchdog.sh
Environment=LOG_FILE=/var/log/claude-watchdog.log
```

**3. Create the timer unit** at `/etc/systemd/system/claude-watchdog.timer`:

```ini
[Unit]
Description=Run Claude Code Watchdog every 60 seconds

[Timer]
OnBootSec=60
OnUnitActiveSec=60
AccuracySec=5

[Install]
WantedBy=timers.target
```

**4. Enable and start:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now claude-watchdog.timer
```

**5. Verify:**

```bash
systemctl status claude-watchdog.timer
systemctl list-timers | grep claude
```

---

## Bot Launch Example

To run Claude Code as a Telegram bot with full protection (watchdog + safe launcher):

```bash
# Authenticate first
claude auth login --claudeai

# Launch bot with protections
BOT_MODE=true bash claude-safe-launch.sh
```

For a persistent session using tmux:

```bash
tmux new-session -d -s claude-bot "BOT_MODE=true bash claude-safe-launch.sh"
```

The safe launcher applies `nice` and `ulimit` limits, and the watchdog (running separately via LaunchAgent or systemd) monitors for any processes that escape those limits.

---

## Log Examples

**Normal operation (nothing killed):**

```
[2026-03-31 14:22:01] Watchdog v2 check started
[2026-03-31 14:22:01] Checking for orphaned caffeinate processes...
[2026-03-31 14:22:01] Checking for memory hogs (>3072 MB)...
[2026-03-31 14:22:01] Checking for memory growth (>500 MB since last check)...
[2026-03-31 14:22:01] Checking for CPU hogs (>80%, >60s, no TTY)...
[2026-03-31 14:22:01] Checking for orphaned children (PPID=1, >10% CPU, >60s)...
[2026-03-31 14:22:01] Watchdog v2 check complete. 0 processes killed.
```

**Memory hog killed:**

```
[2026-03-31 14:23:01] Watchdog v2 check started
[2026-03-31 14:23:01] Checking for memory hogs (>3072 MB)...
[2026-03-31 14:23:01] KILLED PID 48291 (claude, RSS=4812 MB, age=184s) -- memory hog
[2026-03-31 14:23:01] Watchdog v2 check complete. 1 process killed.
```

**Memory growth detected:**

```
[2026-03-31 14:24:01] Watchdog v2 check started
[2026-03-31 14:24:01] Checking for memory growth (>500 MB since last check)...
[2026-03-31 14:24:01] KILLED PID 51003 (node, RSS=1847 MB, growth=+612 MB in 60s) -- memory leak
[2026-03-31 14:24:01] Watchdog v2 check complete. 1 process killed.
```

**CPU hog killed:**

```
[2026-03-31 14:25:01] Watchdog v2 check started
[2026-03-31 14:25:01] Checking for CPU hogs (>80%, >60s, no TTY)...
[2026-03-31 14:25:01] KILLED PID 49102 (claude, CPU=98.2%, age=312s, no TTY) -- CPU hog
[2026-03-31 14:25:01] Watchdog v2 check complete. 1 process killed.
```

**Orphaned caffeinate cleanup (macOS):**

```
[2026-03-31 14:26:01] Watchdog v2 check started
[2026-03-31 14:26:01] Checking for orphaned caffeinate processes...
[2026-03-31 14:26:01] KILLED 37 orphaned caffeinate processes
[2026-03-31 14:26:01] Watchdog v2 check complete. 37 processes killed.
```

**Safe launcher auto-restart:**

```
[2026-03-31 14:30:12] Claude process exited with code 137 (killed)
[2026-03-31 14:30:12] Restart 1/5 -- waiting 10s before relaunch
[2026-03-31 14:30:22] Relaunching Claude Code...
```

**Safe launcher giving up after too many restarts:**

```
[2026-03-31 14:32:45] Claude process exited with code 137 (killed)
[2026-03-31 14:32:45] 5 restarts in the last 300s -- exceeds MAX_RESTARTS (5)
[2026-03-31 14:32:45] Giving up. Something is persistently wrong. Check logs and issues.
```

---

## Uninstall

### macOS

```bash
# Stop and unload the LaunchAgent
launchctl unload ~/Library/LaunchAgents/com.bujosa.claude-watchdog.plist

# Remove files
rm ~/Library/LaunchAgents/com.bujosa.claude-watchdog.plist
sudo rm /usr/local/bin/claude-watchdog.sh

# Clean up logs and snapshots
rm -f /tmp/claude-watchdog.log
rm -f /tmp/claude-watchdog-launchd.log
rm -rf /tmp/claude-watchdog-snapshots
```

### Linux

```bash
# Stop and disable the timer
sudo systemctl disable --now claude-watchdog.timer

# Remove unit files
sudo rm /etc/systemd/system/claude-watchdog.service
sudo rm /etc/systemd/system/claude-watchdog.timer
sudo systemctl daemon-reload

# Remove the script
sudo rm /usr/local/bin/claude-watchdog.sh

# Clean up logs and snapshots
sudo rm -f /var/log/claude-watchdog.log
rm -rf /tmp/claude-watchdog-snapshots
```

---

## Related GitHub Issues

These are the upstream Claude Code bugs that the watchdog mitigates. All were still open as of March 2026.

| Issue | Title | Summary |
|-------|-------|---------|
| [#22275](https://github.com/anthropics/claude-code/issues/22275) | CPU burn after session close | Processes run at 80-100% CPU indefinitely after a session ends |
| [#32729](https://github.com/anthropics/claude-code/issues/32729) | ArrayBuffer memory leak | ArrayBuffers grow at ~36 MB/s, consuming all RAM in minutes |
| [#33280](https://github.com/anthropics/claude-code/issues/33280) | Memory leak (duplicate/related) | Additional reports of unbounded memory growth |
| [#40706](https://github.com/anthropics/claude-code/issues/40706) | Orphaned processes on exit | Desktop and VS Code integrations do not kill child processes when the parent exits |

---

## License

MIT
