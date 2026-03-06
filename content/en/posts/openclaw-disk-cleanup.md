---
title: "OpenClaw Disk Cleanup: Reclaiming Storage Space"
date: 2026-02-22T20:45:00+08:00
draft: false
tags: ["openclaw", "maintenance", "disk-cleanup", "troubleshooting"]
categories: ["Maintenance"]
description: "How to identify and clean up disk space used by OpenClaw. Covers logs, caches, and configuration cleanup."
---

## The Problem

After running OpenClaw for a while, you might notice disk space creeping up. Here's how to identify what's using space and safely clean it up.

---

## Finding What's Using Space

### Check OpenClaw Directory Size

```bash
du -sh ~/.openclaw/
```

### Breakdown by Subdirectory

```bash
cd ~/.openclaw
du -h --max-depth=1 | sort -hr
```

Typical output:
```
2.1G    ./node_modules
450M    ./completions
120M    ./logs
85M     ./subagents
12M     ./cron
8.2M    ./config
```

---

## Safe Cleanup Targets

### 1. Old Completions

Completions (AI-generated responses) accumulate over time:

```bash
# Check size
ls -lah ~/.openclaw/completions/

# Remove completions older than 30 days
find ~/.openclaw/completions/ -type f -mtime +30 -delete
```

### 2. Log Files

Logs can grow indefinitely:

```bash
# Check current logs
ls -lah ~/.openclaw/logs/

# Truncate large logs
> ~/.openclaw/logs/commands.log

# Or archive and clear
tar czf ~/openclaw-logs-$(date +%Y%m%d).tar.gz ~/.openclaw/logs/
rm -rf ~/.openclaw/logs/*
```

### 3. Subagent History

Subagent sessions store message history:

```bash
# Check subagent storage
du -sh ~/.openclaw/subagents/

# Review and remove old sessions
ls -lt ~/.openclaw/subagents/
rm -rf ~/.openclaw/subagents/old-session-id
```

### 4. Cache Files

Various caches can be cleared:

```bash
# Clear tool result cache
rm -rf ~/.openclaw/.cache/

# Clear any application caches
rm -rf ~/.cache/openclaw/
```

---

## What NOT to Delete

**Never delete:**
- `~/.openclaw/openclaw.json` – Your main configuration
- `~/.openclaw/credentials/` – Stored credentials
- `~/.openclaw/agents/` – Agent configurations
- `~/.openclaw/cron/jobs.json` – Scheduled tasks

---

## Automated Cleanup Script

Create `~/.openclaw/cleanup.sh`:

```bash
#!/bin/bash
# OpenClaw maintenance cleanup

echo "Starting OpenClaw cleanup..."

# Clean completions older than 30 days
echo "Cleaning old completions..."
find ~/.openclaw/completions/ -type f -mtime +30 -delete 2>/dev/null

# Rotate logs if over 100MB
LOG_SIZE=$(du -m ~/.openclaw/logs/ 2>/dev/null | cut -f1)
if [ "$LOG_SIZE" -gt 100 ]; then
    echo "Rotating logs (current: ${LOG_SIZE}MB)..."
    tar czf ~/openclaw-logs-$(date +%Y%m%d).tar.gz ~/.openclaw/logs/ 2>/dev/null
    > ~/.openclaw/logs/commands.log
fi

# Clean temp files
rm -rf ~/.openclaw/.tmp/* 2>/dev/null

echo "Cleanup complete!"
du -sh ~/.openclaw/
```

Make executable and run:
```bash
chmod +x ~/.openclaw/cleanup.sh
~/.openclaw/cleanup.sh
```

---

## Setting Up Log Rotation

### Using logrotate

Create `/etc/logrotate.d/openclaw`:

```
/home/warwick/.openclaw/logs/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 644 warwick warwick
}
```

### Using systemd timer

Create `~/.config/systemd/user/openclaw-cleanup.service`:
```ini
[Unit]
Description=OpenClaw Cleanup

[Service]
Type=oneshot
ExecStart=/home/warwick/.openclaw/cleanup.sh
```

Create `~/.config/systemd/user/openclaw-cleanup.timer`:
```ini
[Unit]
Description=Run OpenClaw cleanup weekly

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:
```bash
systemctl --user daemon-reload
systemctl --user enable openclaw-cleanup.timer
systemctl --user start openclaw-cleanup.timer
```

---

## Expected Storage Usage

| Component | Typical Size | Growth Rate |
|-----------|-------------|-------------|
| Core files | ~50MB | Stable |
| node_modules | ~2GB | Per version |
| Completions | 100MB-2GB | Depends on usage |
| Logs | 10-100MB | Linear with activity |
| Subagents | 50-500MB | Depends on history |

---

## Monitoring Disk Usage

Add to your shell profile:

```bash
# Show OpenClaw size on login
if [ -d ~/.openclaw ]; then
    SIZE=$(du -sh ~/.openclaw 2>/dev/null | cut -f1)
    echo "OpenClaw storage: $SIZE"
fi
```

---

Regular maintenance keeps your OpenClaw installation lean and responsive.
