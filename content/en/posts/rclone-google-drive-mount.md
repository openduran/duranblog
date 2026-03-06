---
title: "Mounting Google Drive on Linux with Rclone: Complete Guide"
date: 2026-02-27T22:10:00+08:00
draft: false
tags: ["rclone", "google-drive", "mount", "linux", "cloud-storage", "tutorial"]
categories: ["Tutorials"]
description: "Step-by-step guide for mounting Google Drive on Linux using Rclone. Includes authentication setup, mount configuration, and auto-mount on boot."
---

## The Use Case

You have files in Google Drive but need them accessible locally:
- Edit documents with local tools
- Backup local files to cloud
- Sync across multiple machines
- Access without browser

**Rclone** is the best tool for this. It's like `rsync` for cloud storage.

---

## Installation

### Option 1: Package Manager

```bash
# Debian/Ubuntu
sudo apt install rclone

# macOS
brew install rclone

# Arch
sudo pacman -S rclone
```

### Option 2: Install Script

```bash
curl https://rclone.org/install.sh | sudo bash
```

Verify installation:
```bash
rclone version
```

---

## Google Drive Setup

### Step 1: Create Rclone Config

```bash
rclone config
```

Interactive prompts:
1. `n` (new remote)
2. Name: `gdrive`
3. Type: `18` (Google Drive)
4. Client ID: (press Enter for default)
5. Client Secret: (press Enter for default)
6. Scope: `1` (Full access)
7. Root folder: (press Enter)
8. Service account: `n`
9. Edit advanced config: `n`
10. Use auto config: `y`

### Step 2: Authenticate

A browser window opens automatically. If not:

```bash
rclone authorize "drive"
```

Copy the token and paste back in the terminal.

### Step 3: Verify

```bash
rclone listremotes
# Output: gdrive:

rclone lsd gdrive:
# Lists your Drive folders
```

---

## Mounting Google Drive

### Basic Mount

```bash
# Create mount point
mkdir -p ~/GoogleDrive

# Mount
rclone mount gdrive: ~/GoogleDrive
```

Keep terminal open. Press `Ctrl+C` to unmount.

### Background Mount

```bash
rclone mount gdrive: ~/GoogleDrive --daemon
```

### Recommended Mount Options

```bash
rclone mount gdrive: ~/GoogleDrive \
  --daemon \
  --vfs-cache-mode writes \
  --vfs-cache-max-size 1G \
  --vfs-read-chunk-size 16M \
  --buffer-size 32M \
  --poll-interval 30s \
  --dir-cache-time 72h
```

**Options explained:**
- `--vfs-cache-mode writes` – Cache files being written
- `--vfs-cache-max-size 1G` – Limit cache to 1GB
- `--vfs-read-chunk-size 16M` – Read in 16MB chunks
- `--buffer-size 32M` – Read ahead buffer
- `--poll-interval 30s` – Check for changes every 30s
- `--dir-cache-time 72h` – Cache directory listings

---

## Auto-Mount on Boot

### Using Systemd

Create `~/.config/systemd/user/rclone-gdrive.service`:

```ini
[Unit]
Description=Mount Google Drive with Rclone
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/rclone mount gdrive: %h/GoogleDrive \
  --vfs-cache-mode writes \
  --vfs-cache-max-size 1G \
  --buffer-size 32M \
  --poll-interval 30s \
  --dir-cache-time 72h
ExecStop=/bin/fusermount -u %h/GoogleDrive
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

Enable and start:
```bash
systemctl --user daemon-reload
systemctl --user enable rclone-gdrive.service
systemctl --user start rclone-gdrive.service
```

Check status:
```bash
systemctl --user status rclone-gdrive.service
```

### Using fstab (Alternative)

Add to `/etc/fstab`:

```
# Google Drive via rclone
gdrive: /home/warwick/GoogleDrive rclone rw,noauto,user,_netdev,x-systemd.automount,args2env,vfs_cache_mode=writes,vfs_cache_max_size=1G 0 0
```

Then:
```bash
sudo systemctl daemon-reload
mount ~/GoogleDrive
```

---

## Common Operations

### Sync Local to Drive

```bash
# Upload local folder to Drive
rclone sync ~/Documents/Important gdrive:Backup/Documents

# Dry run first (see what would happen)
rclone sync ~/Documents/Important gdrive:Backup/Documents --dry-run
```

### Sync Drive to Local

```bash
# Download from Drive
rclone sync gdrive:Photos ~/Pictures/DrivePhotos
```

### Copy with Progress

```bash
rclone copy ~/LargeFile.zip gdrive:Uploads --progress
```

### Check Differences

```bash
rclone check ~/LocalFolder gdrive:RemoteFolder
```

### Mount Specific Folder

```bash
rclone mount gdrive:Documents/Work ~/WorkDrive
```

---

## Performance Tuning

### For Large Files

```bash
rclone mount gdrive: ~/GoogleDrive \
  --vfs-cache-mode full \
  --vfs-cache-max-size 5G \
  --vfs-read-chunk-size 128M \
  --buffer-size 256M \
  --drive-chunk-size 128M
```

### For Many Small Files

```bash
rclone mount gdrive: ~/GoogleDrive \
  --vfs-cache-mode writes \
  --vfs-cache-max-size 500M \
  --transfers 8 \
  --checkers 16
```

---

## Troubleshooting

### "Transport endpoint is not connected"

Drive got disconnected. Remount:
```bash
fusermount -u ~/GoogleDrive
rclone mount gdrive: ~/GoogleDrive --daemon
```

### Slow Performance

Check cache settings and connection:
```bash
rclone mount gdrive: ~/GoogleDrive --vfs-cache-mode full --log-level INFO
```

### Authentication Expired

Re-authenticate:
```bash
rclone config reconnect gdrive:
```

### Permission Denied

Check mount point ownership:
```bash
ls -la ~/GoogleDrive
sudo chown $USER:$USER ~/GoogleDrive
```

---

## Security Notes

1. **Config file contains tokens** – Keep `~/.config/rclone/rclone.conf` secure
2. **Use scope-limited access** – Don't use "full access" if unnecessary
3. **Regular token rotation** – Re-authenticate periodically
4. **Backup your config** – Lose it, lose access

Backup rclone config:
```bash
cp ~/.config/rclone/rclone.conf ~/.config/rclone/rclone.conf.backup
```

---

## Unmounting

```bash
# Normal unmount
fusermount -u ~/GoogleDrive

# Force unmount if stuck
fusermount -uz ~/GoogleDrive
```

---

## Summary

You now have:
- ✅ Google Drive mounted locally
- ✅ Auto-mount on boot
- ✅ Optimized performance settings
- ✅ Sync/copy operations ready

Your cloud files are now just files in `~/GoogleDrive`.

---

**References:**
- [Rclone Documentation](https://rclone.org)
- [Google Drive Backend](https://rclone.org/drive/)
- [Rclone Mount Guide](https://rclone.org/commands/rclone_mount/)
