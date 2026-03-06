---
title: "OpenClaw + Tailscale Remote Access Guide: Two Secure Ways to Expose Your Gateway"
date: 2026-03-06T09:00:00+08:00
draft: false
tags: ["openclaw", "tailscale", "tailscale-funnel", "remote-access", "security"]
---

## Introduction

OpenClaw Gateway runs locally by default (`127.0.0.1:18789`), which means:
- ✅ Secure: No external access
- ❌ Limited: Can only be used locally

If you want to:
- **Run OpenClaw on your home server and access it remotely from your phone**
- **Share an OpenClaw instance with your team**
- **Use your home AI assistant while away**

Then **Tailscale** integration is your best choice.

---

## What is Tailscale?

[Tailscale](https://tailscale.com/) is a zero-config VPN tool based on WireGuard. It lets you easily build a private network (Tailnet) and securely connect any devices.

### Key Benefits

| Feature | Description |
|---------|-------------|
| **Zero Config** | No firewall rules or port forwarding needed |
| **End-to-End Encryption** | WireGuard protocol, secure and reliable |
| **Cross-Platform** | Linux, macOS, Windows, iOS, Android |
| **Free Tier** | Free for personal use, up to 20 devices |

### Two Tailscale Modes

OpenClaw supports two Tailscale modes:

1. **`tailscale serve`** - Tailnet-only access (private)
2. **`tailscale funnel`** - Public internet access (requires password)

---

## What Can OpenClaw + Tailscale Do?

### Scenario 1: Tailscale Serve (Recommended for Personal Use)

**Use Cases:**
- Run OpenClaw on home NAS/server
- Access remotely from phone/laptop via Tailscale
- Only your devices can access

**Network Topology:**
```
[Phone] ←──Tailnet──→ [Tailscale] ←──localhost──→ [OpenClaw Gateway]
[Laptop] ←──Encrypted Tunnel──→ 192.168.x.x:18789
```

### Scenario 2: Tailscale Funnel (Public Access)

**Use Cases:**
- Team collaboration, sharing one OpenClaw instance
- Temporary access from devices without Tailscale
- Access via public URL (e.g., `https://your-machine.tailnet-xx.ts.net`)

**⚠️ Security Warning:**
- Funnel exposes your service to the public internet
- **Password authentication is mandatory**, otherwise anyone can access your Gateway
- Recommended: `gateway.auth.mode: "password"`

---

## Configuration Steps

### Prerequisites

1. **Install Tailscale**
   ```bash
   # Debian/Ubuntu
   curl -fsSL https://tailscale.com/install.sh | sh
   
   # macOS
   brew install tailscale
   ```

2. **Login to Tailscale**
   ```bash
   sudo tailscale up
   # Follow browser prompts to authorize
   ```

3. **Verify Tailscale IP**
   ```bash
   tailscale ip -4
   # Output: 100.x.y.z
   ```

### Configure OpenClaw

Edit `~/.openclaw/openclaw.json`:

#### Option A: Tailscale Serve (Private)

```json
{
  "gateway": {
    "port": 18789,
    "mode": "tailscale",
    "auth": {
      "mode": "token",
      "token": "your-secure-token"
    },
    "tailscale": {
      "mode": "serve",
      "resetOnExit": false
    }
  }
}
```

**Access:** Only devices with Tailscale on the same account

#### Option B: Tailscale Funnel (Public)

```json
{
  "gateway": {
    "port": 18789,
    "mode": "tailscale",
    "auth": {
      "mode": "password",
      "password": "your-strong-password"
    },
    "tailscale": {
      "mode": "funnel",
      "resetOnExit": true
    }
  }
}
```

**⚠️ Password is mandatory for Funnel mode!**

### Restart Gateway

```bash
openclaw gateway restart
```

---

## Security Best Practices

1. **Prefer Serve Mode** - Unless you need public access
2. **Use Strong Passwords for Funnel**
   ```bash
   openssl rand -base64 32
   ```
3. **Enable resetOnExit** for Funnel
4. **Rotate tokens/passwords regularly**

---

## FAQ

**Q: What's the difference between local and Tailscale modes?**

| Feature | Local | Tailscale Serve | Tailscale Funnel |
|---------|-------|-----------------|------------------|
| Access | Local only | Tailnet devices | Public internet |
| Encryption | None | WireGuard | WireGuard + TLS |
| Needs Tailscale | No | Yes | Yes |
| Password | Optional | Optional | **Required** |

**Q: Can I use both local and Tailscale?**

No. Gateway can only bind to one mode. Use Tailscale Serve + install Tailscale on local devices.

---

## Summary

| Need | Recommended |
|------|-------------|
| Local only | `bind: loopback` (default) |
| Multi-device private | `tailscale: serve` |
| Team/public | `tailscale: funnel` + password |

Tailscale makes OpenClaw remote access simple and secure—no firewall configuration, no port forwarding, deployed in minutes.

**References:**
- [Tailscale Docs](https://tailscale.com/kb/)
- [OpenClaw Gateway Config](https://docs.openclaw.ai/gateway/configuration)
