---
title: "Setting Up OpenClaw with Discord: Complete Guide"
date: 2026-02-12T20:18:00+08:00
draft: false
tags: ["openclaw", "discord", "setup", "bot", "tutorial"]
categories: ["Tutorials"]
description: "Complete walkthrough for setting up OpenClaw with Discord. Covers bot creation, token configuration, permissions, and common troubleshooting."
---

## What We're Building

An AI assistant that lives in your Discord server—capable of answering questions, running tasks, and integrating with your workflows.

**What you'll need:**
- A Discord account
- A server where you're admin
- About 15 minutes

---

## Step 1: Create a Discord Bot

### 1.1 Access the Developer Portal

1. Go to [Discord Developer Portal](https://discord.com/developers/applications)
2. Click "New Application"
3. Name it (e.g., "MyAIAssistant")
4. Accept the terms

### 1.2 Enable Bot Functionality

1. In your app, go to "Bot" section (left sidebar)
2. Click "Add Bot"
3. Confirm with "Yes, do it!"

### 1.3 Get Your Token

**Critical:** The bot token is like a password. Never share it or commit it to git.

1. Under Bot section, click "Reset Token"
2. Copy the new token (starts with something like `MTQ2N...`)
3. Store it securely (password manager or env variable)

---

## Step 2: Configure Bot Permissions

### 2.1 Privileged Gateway Intents

Enable these under Bot → Privileged Gateway Intents:
- ✅ **MESSAGE CONTENT INTENT** (required for reading messages)
- ✅ **SERVER MEMBERS INTENT** (for member-related features)
- ✅ **PRESENCE INTENT** (optional, for presence data)

Without MESSAGE CONTENT INTENT, your bot can't see what people are saying.

### 2.2 OAuth2 Scopes

1. Go to OAuth2 → URL Generator
2. Select scopes:
   - `bot`
   - `applications.commands`
3. Select bot permissions:
   - Send Messages
   - Read Message History
   - Embed Links
   - Attach Files
   - Add Reactions
   - Use Slash Commands

### 2.3 Invite Bot to Server

1. Copy the generated URL
2. Open in browser
3. Select your server
4. Authorize

---

## Step 3: Configure OpenClaw

### 3.1 Set Environment Variable

```bash
export DISCORD_BOT_TOKEN="your-token-here"
```

Or add to `~/.openclaw/.env`:
```
DISCORD_BOT_TOKEN=your-token-here
```

### 3.2 Update hugo.toml

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "${env:DISCORD_BOT_TOKEN}",
      "groupPolicy": "allowlist"
    }
  }
}
```

### 3.3 Configure Channel Permissions

Restrict which channels the bot can access:

```json
"channels": {
  "discord": {
    "enabled": true,
    "token": "${env:DISCORD_BOT_TOKEN}",
    "groupPolicy": "allowlist",
    "guilds": {
      "YOUR_GUILD_ID": {
        "channels": {
          "CHANNEL_ID_1": { "allow": true },
          "CHANNEL_ID_2": { "allow": true }
        }
      }
    }
  }
}
```

**Finding IDs:**
- Enable Developer Mode in Discord (Settings → Advanced)
- Right-click server → "Copy Server ID"
- Right-click channel → "Copy Channel ID"

---

## Step 4: Test the Setup

### 4.1 Start OpenClaw

```bash
openclaw gateway restart
```

### 4.2 Check Logs

```bash
openclaw gateway status
# Or check systemd logs
journalctl --user -u openclaw-gateway -f
```

### 4.3 Test in Discord

1. Go to an allowed channel
2. Mention the bot: `@MyAIAssistant hello`
3. Check for response

---

## Common Issues

### "401 Unauthorized"

**Cause:** Invalid or expired token

**Fix:**
1. Reset token in Discord Developer Portal
2. Update environment variable
3. Restart gateway

### "403 Forbidden"

**Cause:** Bot lacks permissions

**Fix:**
1. Check OAuth2 URL generated correct permissions
2. Re-invite bot with updated scope
3. Verify MESSAGE CONTENT INTENT is enabled

### "Cannot send messages"

**Cause:** Channel permissions override bot permissions

**Fix:**
1. Check channel-specific permissions
2. Ensure bot role is above restricted roles
3. Verify bot is in the channel

### Bot doesn't respond

**Checklist:**
- [ ] Gateway running? (`openclaw gateway status`)
- [ ] Token correct? (check for extra spaces)
- [ ] Channel in allowlist? (if using groupPolicy)
- [ ] Bot has message read permission?
- [ ] Mentioning correctly? (@BotName)

---

## Security Best Practices

1. **Never commit tokens** – Use environment variables
2. **Use allowlists** – Restrict to specific channels
3. **Rotate tokens periodically** – Every 90 days
4. **Monitor bot activity** – Check logs regularly
5. **Limit permissions** – Only what's necessary

---

## What's Next

Now that Discord is connected, you can:
- Set up scheduled tasks (cron jobs)
- Configure multiple channels for different purposes
- Add webhook integrations
- Set up DM responses

See the [OpenClaw Discord docs](https://docs.openclaw.ai/channels/discord) for advanced features.

---

**Full working configuration in the example above. Adjust channel IDs and token for your setup.**
