---
title: "From Scripts to Official: My Google Services Management Evolution - An OpenClaw User's CLI Migration Journey"
date: 2026-03-23T15:45:00+08:00
draft: false
tags: ["OpenClaw", "AI Agent", "Google Workspace", "gws", "Productivity", "Automation"]
categories: ["Technology"]
---

## Introduction: An AI Agent Power User's Tool Evolution

As a heavy user of OpenClaw AI assistant, my daily workflow has long been inseparable from automation:

- **Every morning at 8:17**, AI automatically pushes today's schedule and todo tasks
- **Stock analysis** automatically fetches data and generates technical reports
- **Blog publishing** with bilingual Chinese/English auto-deployment
- **Memory management** automatically backs up to GitHub

Behind these automations lies deep integration with Google services: **Google Calendar** for scheduling, **Google Tasks** for tracking todos, and **Google Drive** for file storage.

I previously wrote two articles sharing my approach:
- ["AI Assistant Schedule Management in Practice: OpenClaw + Google Calendar/Tasks Automation"](/en/posts/ai-schedule-solutions-comparison/) - Using Python scripts to connect Google services
- ["Rclone Mount Google Drive: File Management for AI Assistants"](/en/posts/rclone-google-drive-mount/) - Using rclone to manage Drive files

**But recently I encountered several pain points** that prompted me to rethink the entire approach...

---

## Part 1: Problems with the Previous Approach

### 1.1 Issues with Python Scripts

In "AI Assistant Schedule Management in Practice," I used Python scripts with OAuth to access Google Calendar and Tasks:

```python
# Previous approach
from google.oauth2.credentials import Credentials
creds = Credentials.from_authorized_user_file('token.json')
service = build('tasks', 'v1', credentials=creds)
```

**But tokens kept expiring**:
- `invalid_grant: Token has been expired or revoked`
- Had to re-authorize every few days
- Resulted in daily briefs showing "failed to fetch" for task lists

**High maintenance costs**:
- Manual token refresh required
- Scripts scattered, single-purpose
- Different services needed different scripts

### 1.2 Issues with Rclone

In "Rclone Mount Google Drive," I used rclone to manage files:

```bash
rclone mount gdrive: ~/GoogleDrive
```

**But difficult for AI Agent invocation**:
- Rclone mounts as local file system
- OpenClaw needs complex command chaining to operate Drive files
- Upload/download requires local file intermediates

**Scattered configuration**:
- One config for rclone
- Another config for Python scripts
- Management chaos

### 1.3 My New Requirements

As an **AI Agent user** rather than a developer, I wanted:

✅ **Unified management** - One tool for all Google services  
✅ **Automatic token management** - No manual refresh  
✅ **AI-friendly** - OpenClaw can call directly  
✅ **Chinese support** - Email subjects without garbled text  

**Until I discovered Google's official gws (Google Workspace CLI)**

---

## Part 2: What is Google Workspace CLI?

**gws** is Google's official command-line tool. Simply put:

> Just like `kubectl` manages Kubernetes or `aws` manages AWS, `gws` lets you manage all Google services with one-line commands.

### 2.1 Supported Services

| Service | What I Can Do | Replaces Previous |
|---------|--------------|-------------------|
| **Google Tasks** | Create/complete tasks | Python scripts |
| **Google Calendar** | View/create schedules | Python scripts |
| **Gmail** | Send/receive emails | Didn't have before |
| **Google Drive** | Upload/download/manage files | Rclone |
| **Google Sheets** | Read/write spreadsheets | Didn't have before |
| **Google Docs** | Edit documents | Didn't have before |

### 2.2 Value for AI Agent Users

**Previous workflow**:
```
OpenClaw → Python scripts → Google API → Calendar/Tasks
       ↓
     Rclone → Google Drive
```

**Current workflow**:
```
OpenClaw → gws → All Google services
```

**Unified, clean, officially supported**

---

## Part 3: Migration in Practice

### 3.1 Installing gws

```bash
npm install -g @googleworkspace/cli
```

### 3.2 Authentication (One-time Setup)

**Previous pain point**: Python script OAuth tokens expired every few days.

**gws solution**:
1. Create Google Cloud project (one-time)
2. Enable required APIs (Drive, Gmail, Calendar, Tasks)
3. OAuth authorization, get refresh_token
4. **refresh_token valid for 7 days**, auto-renews

After setup, OpenClaw can call directly:

```bash
export GOOGLE_WORKSPACE_CLI_TOKEN="ya29.xxx"

# View tasks
gws tasks tasks list

# Send email
gws gmail users.messages send ...
```

### 3.3 Replacing Previous Python Scripts

**Previous task fetching script** (often failed):

```python
# Old code, token frequently expired
from google_tasks_oauth import get_tasks_service
service = get_tasks_service()  # Often errors
```

**Now with gws**:

```bash
# One command, stable and reliable
gws tasks tasks list --format table
```

**Comparison**:

| Dimension | Previous Python Scripts | Current gws |
|-----------|------------------------|-------------|
| Token management | Manual refresh, frequent expiration | refresh_token auto-renews |
| Feature scope | Single (only Tasks) | Comprehensive (all Google services) |
| Stability | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Ease of use | ⭐⭐⭐⭐ | ⭐⭐ |

### 3.4 Replacing Rclone

**Previously used rclone for Drive**:

```bash
# Mount to local
rclone mount gdrive: ~/GoogleDrive

# Then operate local files
```

**Now with gws**:

```bash
# Direct Drive operations, no mount needed
gws drive files list
gws drive files create --upload ./file.txt
```

**Comparison**:

| Dimension | Previous Rclone | Current gws |
|-----------|----------------|-------------|
| File access | Mounted as local filesystem | Direct API operations |
| AI invocation | Complex (needs local paths) | Simple (direct commands) |
| Batch operations | ✅ Efficient | ⚠️ One-by-one |
| Use case | Large file transfers | Daily file management |

**Conclusion**: Keep rclone for **large file batch transfers**, use gws for **daily file management**

---

## Part 4: OpenClaw Integration Examples

### 4.1 Daily Brief Integration

Previous task fetching often failed (token expiration), now using gws:

```python
# In rss_news.py, modified
def get_google_tasks():
    """Use gws to fetch tasks (replaces previous OAuth script)"""
    import subprocess
    
    result = subprocess.run(
        ['gws', 'tasks', 'tasks', 'list', 
         '--params', '{"tasklist": "@default"}',
         '--format', 'json'],
        capture_output=True,
        text=True,
        env={'GOOGLE_WORKSPACE_CLI_TOKEN': 'ya29.xxx'}
    )
    
    # Parse JSON return task list
    import json
    data = json.loads(result.stdout)
    return data.get('items', [])
```

**Result**: Token valid for 7 days with auto-refresh support, no more frequent failures.

### 4.2 Sending Emails (New Capability)

**Previous situation**:
- My automation workflow lacked email notification capability
- Had to manually open Gmail web interface to send emails

**Now with gws**:

```bash
# Send email (note Chinese encoding)
~/.openclaw/workspace/send-email.sh \
  bauhaushuang@hotmail.com \
  'Test Email' \
  'This is the email content'
```

**Gotcha**: Chinese subjects sent directly will be garbled, requires MIME encoding.

**Solution**: Wrapper script automatically handles UTF-8 Base64 encoding:

```bash
# Correct MIME encoding
Subject: =?UTF-8?B?5rWL6K+V6YKu5Lu2?=  # Base64 encoded "测试邮件"
```

**Result**: Now OpenClaw can directly invoke email sending, e.g., automatic email notification after daily report completion.

### 4.3 File Management

Previously used rclone requiring mount, now direct operations:

```bash
# Upload to Drive
gws drive files create \
  --upload ./document.md \
  --params '{"parents": ["FOLDER_ID"]}'

# Download file
gws drive files get \
  --params '{"fileId": "FILE_ID"}' \
  --output ./downloaded.md
```

**OpenClaw can directly invoke these commands**.

---

## Part 5: Complete Old vs New Comparison

### 5.1 Architecture Comparison

| Component | Previous Approach | Current Approach |
|-----------|------------------|------------------|
| **Google Tasks** | Python OAuth scripts | gws |
| **Google Calendar** | Python OAuth scripts | gws |
| **Gmail** | ❌ Didn't have | gws |
| **Google Drive** | Rclone | gws + rclone (kept) |
| **Google Sheets** | ❌ Didn't have | gws |
| **Token management** | Scattered, prone to expiration | Unified, auto-renews |
| **Configuration maintenance** | Multiple configs | Single config |

### 5.2 Usage Experience Comparison

| Scenario | Before | Now | Evaluation |
|----------|--------|-----|------------|
| **Daily brief** | Token frequently expired | Token stable 7 days | ✅ Significant improvement |
| **Sending emails** | ❌ Didn't have this feature | Supports Chinese | ✅ New capability |
| **File upload** | Rclone mount | Direct commands | ✅ More convenient |
| **Large file transfers** | Rclone efficient | gws one-by-one | ⚠️ Keep rclone |
| **Configuration complexity** | Medium | Higher (initial setup) | ⚠️ Learning curve |

### 5.3 Maintenance Cost Comparison

| Item | Before | Now |
|------|--------|-----|
| Scripts to maintain | 3-4 | 1 (gws wrapper) |
| Token refresh frequency | Every 2-3 days | Every 7 days |
| Official support | ❌ Community solution | ✅ Google official |
| API update sync | Manual updates | Automatic sync |

---

## Part 6: My Recommendations

### 6.1 When to Migrate to gws

✅ **You're a heavy AI Agent user like me**
- Need OpenClaw/Claude to directly call Google services
- Want unified management interface

✅ **Need unified management**
- Don't want to maintain multiple scripts
- Want Drive + Gmail + Calendar + Tasks in one place

✅ **Pursuing stability**
- Tired of frequent token expirations
- Want official long-term support

### 6.2 When to Keep Previous Approach

⚠️ **Only need single functionality**
- Just need to read Calendar, Python script is simpler

⚠️ **Large file batch transfers**
- Rclone is more efficient for batch transfers, keep as supplement

⚠️ **Don't want to tinker with configuration**
- gws initial setup is complex, not worth it for short-term use

### 6.3 My Final Architecture

```
OpenClaw AI Assistant
    ├── Schedule/Task Management → gws (replaces Python scripts)
    ├── Email Sending → gws (new capability)
    ├── Daily File Operations → gws (replaces most rclone scenarios)
    └── Large File Batch Transfers → Rclone (kept)
```

**Not complete replacement, but complementary**

---

## Part 7: Summary

From Python OAuth scripts + rclone to Google Workspace CLI, my tool stack has evolved:

**Problems Solved**:
- ✅ Frequent token expiration → refresh_token 7-day validity
- ✅ Scattered functionality → Unified management
- ✅ Missing email capability → Full Gmail support
- ✅ Garbled Chinese text → Correct MIME encoding

**Costs Paid**:
- ⚠️ Higher initial configuration complexity
- ⚠️ Need to learn new command formats
- ⚠️ Large file operations less efficient than rclone

**Final Evaluation**:

> As an AI Agent user rather than a developer, gws makes my automation workflow more **unified, stable, and scalable**. Although the configuration threshold is higher, it's one-time setup and worth the time investment.

If you're also using OpenClaw or other AI Agent frameworks and deeply depend on Google services, **highly recommend trying gws**.

---

## References

- [My previous article: AI Assistant Schedule Management in Practice](/en/posts/ai-schedule-solutions-comparison/)
- [My previous article: Rclone Mount Google Drive](/en/posts/rclone-google-drive-mount/)
- Google Workspace CLI GitHub: https://github.com/googleworkspace/cli

---

*The author is a user of OpenClaw AI assistant, not a Google developer. This article shares real migration experience from a user perspective.*
