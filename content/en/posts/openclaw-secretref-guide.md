---
title: "OpenClaw API Key Management: Environment Variables Best Practices"
date: 2026-03-03T22:00:00+08:00
draft: true
tags: ["openclaw", "security", "environment-variables", "best-practices"]
---

## The Problem with Plaintext Keys

When setting up OpenClaw, you're dealing with sensitive credentials:
- Discord Bot Tokens
- AI API Keys (Kimi, OpenAI, etc.)
- Service credentials

**The temptation:** Just paste them into `openclaw.json`

**The risk:** One accidental git commit, and your keys are public.

---

## The Solution: Environment Variables

OpenClaw supports referencing environment variables in configuration. Your config file only contains placeholders, actual values live in environment variables.

### How It Works

```json
{
  "channels": {
    "discord": {
      "token": "${env:DISCORD_BOT_TOKEN}"
    }
  }
}
```

The `${env:VAR_NAME}` syntax tells OpenClaw to read from environment variables at runtime.

---

## Supported Environment Variables

Based on OpenClaw source code, these are officially supported:

| Service | Variable Name | Config Path |
|---------|--------------|-------------|
| Discord | `DISCORD_BOT_TOKEN` | `channels.discord.token` |
| Kimi AI | `KIMI_API_KEY` | Auth profiles |
| Moonshot | `MOONSHOT_API_KEY` | Auth profiles |
| OpenAI | `OPENAI_API_KEY` | Model providers |
| Anthropic | `ANTHROPIC_API_KEY` | Model providers |
| Gateway | `OPENCLAW_GATEWAY_TOKEN` | `gateway.auth.token` |

Full list from source:
```
OPENAI_API_KEY, ANTHROPIC_API_KEY, ANTHROPIC_OAUTH_TOKEN,
GEMINI_API_KEY, ZAI_API_KEY, OPENROUTER_API_KEY,
AI_GATEWAY_API_KEY, MINIMAX_API_KEY, SYNTHETIC_API_KEY,
KILOCODE_API_KEY, ELEVENLABS_API_KEY, TELEGRAM_BOT_TOKEN,
DISCORD_BOT_TOKEN, SLACK_BOT_TOKEN, SLACK_APP_TOKEN,
OPENCLAW_GATEWAY_TOKEN, OPENCLAW_GATEWAY_PASSWORD,
KIMI_API_KEY, MOONSHOT_API_KEY
```

---

## Setup Methods

### Method 1: Shell Environment

```bash
export DISCORD_BOT_TOKEN="your-token-here"
export KIMI_API_KEY="your-key-here"
openclaw gateway restart
```

**Pros:** Quick, good for testing  
**Cons:** Lost on shell exit, not persistent

### Method 2: Environment File (Recommended)

Create `~/.openclaw/.env`:
```bash
DISCORD_BOT_TOKEN=your-token-here
KIMI_API_KEY=your-key-here
```

OpenClaw automatically loads this on startup.

**Pros:** Persistent, organized, no shell pollution  
**Cons:** File permissions matter

Secure the file:
```bash
chmod 600 ~/.openclaw/.env
```

### Method 3: Systemd Service

For systemd-managed gateway, edit the service file:

```ini
[Service]
EnvironmentFile=/home/warwick/.openclaw/.env
```

Then reload:
```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

---

## Complete Configuration Example

### 1. Create Environment File

`~/.openclaw/.env`:
```bash
# Discord
DISCORD_BOT_TOKEN=MTQ2Njc4MDY2NzgwNjIyMDM2NA.GvboSs.xxxxxxxxxxxxxxxxxxxxx

# AI Services
KIMI_API_KEY=sk-kimi-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Gateway Auth
OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)
```

### 2. Update openclaw.json

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "${env:DISCORD_BOT_TOKEN}"
    }
  },
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "${env:OPENCLAW_GATEWAY_TOKEN}"
    }
  }
}
```

### 3. Restart Gateway

```bash
openclaw gateway restart
```

---

## Security Best Practices

### 1. Never Commit .env Files

Add to `.gitignore`:
```
.env
.env.local
*.env
openclaw.json.bak
```

### 2. Use Different Tokens for Different Environments

```bash
# Production
DISCORD_BOT_TOKEN_PROD=xxx

# Development  
DISCORD_BOT_TOKEN_DEV=yyy
```

### 3. Rotate Keys Regularly

Set a calendar reminder every 90 days to regenerate tokens.

### 4. Audit Your Config

```bash
openclaw secrets audit
```

This shows which keys are still in plaintext.

### 5. Backup Strategy

```bash
# Backup config (without secrets)
cp ~/.openclaw/openclaw.json ~/backup/

# Backup .env separately (encrypt it)
gpg -c ~/.openclaw/.env
```

---

## Migration Guide

### From Plaintext to Environment Variables

**Step 1: Extract current keys**
```bash
grep -E '"token"|"key"|"password"' ~/.openclaw/openclaw.json
```

**Step 2: Create .env file**
```bash
cat > ~/.openclaw/.env << 'EOF'
DISCORD_BOT_TOKEN=your-extracted-token
KIMI_API_KEY=your-extracted-key
EOF
chmod 600 ~/.openclaw/.env
```

**Step 3: Update config to use env vars**
Replace `"token": "actual-token"` with `"token": "${env:DISCORD_BOT_TOKEN}"`

**Step 4: Verify**
```bash
openclaw secrets audit
# Should show no plaintext keys
```

**Step 5: Restart**
```bash
openclaw gateway restart
```

---

## Troubleshooting

### "Cannot resolve env variable"

**Check:** Variable is actually set
```bash
echo $DISCORD_BOT_TOKEN
```

**Check:** No spaces around `=` in .env file
```bash
# Wrong
DISCORD_BOT_TOKEN = token-here

# Right  
DISCORD_BOT_TOKEN=token-here
```

### Gateway can't find .env

**Check:** File location
```bash
ls -la ~/.openclaw/.env
```

**Check:** File permissions
```bash
chmod 600 ~/.openclaw/.env
```

### Environment variables not loading

**For systemd:**
```bash
# Check if EnvironmentFile is set
systemctl --user cat openclaw-gateway.service | grep Environment

# Reload and restart
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

---

## Alternative: Password Store

For even better security, use a password manager:

### With `pass`

```bash
# Store token
pass insert openclaw/discord-token

# Retrieve in script
export DISCORD_BOT_TOKEN=$(pass openclaw/discord-token)
openclaw gateway restart
```

### With 1Password CLI

```bash
export DISCORD_BOT_TOKEN=$(op read "op://Private/OpenClaw/discord-token")
```

---

## Summary

| Approach | Security | Convenience | Best For |
|----------|----------|-------------|----------|
| Plaintext config | ❌ Poor | ✅ Easy | Never |
| Environment variables | ✅ Good | ✅ Easy | Most users |
| .env file | ✅ Good | ✅ Easy | Development |
| Password store | ✅ Excellent | ⚠️ Setup | Security-focused |

**Recommendation:** Use `.env` file for most setups, password store for high-security environments.

---

**Remember:** Security is about trade-offs. Environment variables hit the sweet spot between security and convenience for most OpenClaw deployments.
