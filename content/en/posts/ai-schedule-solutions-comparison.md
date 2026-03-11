---
title: "AI Agent Schedule Management: Comparing Google, Outlook, Notion, and Local Solutions"
date: 2026-03-10T14:00:00+08:00
draft: false
tags: ["schedule", "calendar", "productivity", "ai-agent", "tutorial"]
categories: ["Tutorials"]
description: "A comprehensive comparison of schedule management solutions for AI agents: Google Calendar, Microsoft Outlook, Notion, and local Markdown. Based on real deployment experience with complete configuration workflows and permission troubleshooting."
---

## Why Do AI Agents Need Schedule Management?

When you ask your AI agent "What's on my schedule today?" or "Create a meeting for tomorrow at 3 PM," it should execute accurately, not say "I don't know."

A complete AI agent schedule system should have:
- 📅 **Read schedules** - Know what's happening today and tomorrow
- ⏰ **Timely reminders** - Push notifications at the right time
- 📝 **Task tracking** - Manage to-do items and completion status
- 🤖 **Proactive creation** - AI can create new events and tasks for you
- 🔄 **Multi-device sync** - Accessible from phone, computer, and AI assistant

But choosing the right solution isn't easy—**network environment, configuration complexity, and usage habits** all affect the decision.

---

## Solution Overview

| Solution | China Stability | Setup Difficulty | AI Can Create | Best For |
|----------|-----------------|------------------|---------------|----------|
| **Google Calendar** | ⭐⭐ (needs VPN) | ⭐⭐⭐ Complex | ✅ Yes | Overseas users, full Google ecosystem |
| **Microsoft Outlook** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐ Medium | ✅ Yes | Enterprise users, Microsoft ecosystem |
| **Notion** | ⭐⭐⭐⭐ Good | ⭐ Simple | ✅ Yes | Knowledge workers, flexible databases |
| **Local Markdown** | ⭐⭐⭐⭐⭐ Perfect | ⭐ Minimal | ✅ Yes | Privacy-first, quick start |

---

## Solution 1: Google Calendar

### Who It's For
- Already have Google account and calendar data
- Network environment can stably access Google
- **Need AI assistant that can both read AND create events and tasks**

### Key Advantages
- **Complete ecosystem** - Calendar + Tasks dual functionality, AI can read and write
- **Mature API** - Python official library support with comprehensive debugging docs
- **Fine-grained permissions** - Control AI to have read-only or full control
- **Generous free tier** - Almost unlimited for personal use

### Main Drawbacks
- **Difficult China access** - Needs stable VPN/proxy
- **Relatively complex setup** - Involves two authentication methods working together
- **Permission pitfalls** - IAM roles, API scopes, and calendar sharing permissions can be confusing

---

### Our Configuration Approach

Based on real deployment experience, we use a **hybrid authentication approach**:

| Feature | Auth Method | Reason |
|---------|-------------|--------|
| **Calendar** | Service Account | Calendars can be shared with Service Account, suitable for automated access |
| **Tasks** | OAuth | Google Tasks cannot be shared like calendars, must use OAuth to access personal task list |

> 💡 **Lesson learned**: We initially tried to use Service Account for both calendar and tasks, but discovered Tasks API doesn't support Service Account access to personal task lists. We ended up with a hybrid solution: Service Account for calendar, OAuth for tasks.

---

### Step 1: Create Google Cloud Project

1. Visit [Google Cloud Console](https://console.cloud.google.com/)
2. Click project selector → **New Project**
3. Project name: `ai-schedule-demo`
4. Click **Create**

---

### Step 2: Enable APIs

Enable two APIs:
1. Search **"Google Calendar API"** → Click **Enable**
2. Search **"Tasks API"** → Click **Enable**

---

### Step 3: Configure Calendar Access (Service Account)

Service Account is suitable for calendar access because calendars can be explicitly shared with it.

#### 3.1 Create Service Account

1. [Google Cloud Console](https://console.cloud.google.com/) → **IAM & Admin** → **Service Accounts**
2. Click **Create Service Account**
3. Name: `calendar-reader`
4. Click **Create and Continue**
5. **Role selection**:
   - If AI only needs to **read** calendar → **Viewer**
   - If AI needs to **create/edit** events → **Editor**
6. Click **Done**

> 📌 **Permission note**: The IAM role selected here controls the Service Account's access to Google Cloud resources. If you need AI to create events later, choose the **Editor** role.

#### 3.2 Create Key

1. Click the Service Account you just created → **Keys** tab
2. **Add Key** → **Create new key** → **JSON**
3. Download and save as `service-account.json`
4. Move to config directory:

```bash
mkdir -p ~/.config/google-calendar
cp ~/Downloads/service-account.json ~/.config/google-calendar/
chmod 600 ~/.config/google-calendar/service-account.json
```

#### 3.3 Share Calendar with Service Account

**Critical step**: Service Account cannot automatically access your calendar; you must explicitly share it.

1. Open [Google Calendar](https://calendar.google.com)
2. Left side find the calendar to sync → Click **⋮** → **Settings and sharing**
3. **Share with specific people** → **Add people**
4. Enter Service Account email (like `calendar-reader@ai-schedule-demo.iam.gserviceaccount.com`)
5. **Permission selection**:
   - **See all event details** - AI can only read
   - **Make changes to events** - AI can create and edit events

> ⚠️ **Common error**: If you forget to share the calendar, or set permission to "See only free/busy," the API returns empty list or 403 error.

#### 3.4 Python Code - Read Calendar

Create `google_calendar.py`:

```python
#!/usr/bin/env python3
"""Google Calendar Reader - Service Account Method"""

import os
from datetime import datetime, timedelta
from google.oauth2 import service_account
from googleapiclient.discovery import build

# Service Account config
SCOPES = ['https://www.googleapis.com/auth/calendar.readonly']
SERVICE_ACCOUNT_FILE = os.path.expanduser('~/.config/google-calendar/service-account.json')
CALENDAR_ID = 'primary'  # Primary calendar, or shared calendar ID

def get_today_events():
    """Get today's events"""
    if not os.path.exists(SERVICE_ACCOUNT_FILE):
        return "   ⚠️ Service Account not configured"
    
    try:
        creds = service_account.Credentials.from_service_account_file(
            SERVICE_ACCOUNT_FILE, scopes=SCOPES)
        service = build('calendar', 'v3', credentials=creds)
        
        # Today's time range
        now = datetime.now()
        start = now.replace(hour=0, minute=0, second=0).isoformat() + '+08:00'
        end = (now + timedelta(days=1)).replace(hour=0, minute=0, second=0).isoformat() + '+08:00'
        
        events_result = service.events().list(
            calendarId=CALENDAR_ID,
            timeMin=start,
            timeMax=end,
            singleEvents=True,
            orderBy='startTime'
        ).execute()
        
        events = events_result.get('items', [])
        
        if not events:
            return "   • No events today"
        
        lines = []
        for event in events:
            start = event['start'].get('dateTime', event['start'].get('date'))
            if 'T' in start:
                time_str = start[11:16]
            else:
                time_str = 'All day'
            lines.append(f"   • {time_str} {event['summary']}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ Failed: {str(e)[:50]}"

if __name__ == '__main__':
    print("📅 **Today's Schedule**")
    print(get_today_events())
```

---

### Step 4: Configure Task Access (OAuth)

Google Tasks cannot be shared like calendars; you must use OAuth to access your personal task list.

#### 4.1 Configure OAuth Consent Screen

1. Left menu → **APIs & Services** → **OAuth consent screen**
2. User type: **External** (for personal accounts)
3. App name: `AI Schedule`
4. User support email: Select your Gmail
5. Developer contact info: Enter your email
6. Click **Save and Continue**

#### 4.2 Add API Scopes

**Add Tasks permissions** (choose based on needs):

**Read-only:**
- `https://www.googleapis.com/auth/tasks.readonly` - Read tasks

**Full permissions** (AI can create/complete tasks):
- `https://www.googleapis.com/auth/tasks` - Full task control

Setup steps:
1. **Add or remove scopes** → Add the URL above
2. Click **Update** → **Save and Continue**
3. **Test users** → **Add users** → Enter your Gmail address
4. Click **Save and Continue** → **Back to dashboard**

> 📌 **Permission note**: With the above configuration, AI can only **read** tasks. If you need AI to **create** tasks later, use the `tasks` full permission and re-authorize.

#### 4.3 Create OAuth Client ID

1. **Credentials** → **Create credentials** → **OAuth client ID**
2. Application type: **Desktop app**
3. Name: `OpenClaw Desktop`
4. Click **Create**
5. Download JSON file, rename to `client_secret.json`
6. Move to config directory:

```bash
cp ~/Downloads/client_secret.json ~/.config/google-calendar/
chmod 600 ~/.config/google-calendar/client_secret.json
```

#### 4.4 Python Code - Read Tasks

Create `google_tasks.py`:

```python
#!/usr/bin/env python3
"""Google Tasks Reader - OAuth Method"""

import os
import pickle
from datetime import datetime
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build

# OAuth config
SCOPES = ['https://www.googleapis.com/auth/tasks.readonly']
CLIENT_SECRET_FILE = os.path.expanduser('~/.config/google-calendar/client_secret.json')
TOKEN_FILE = os.path.expanduser('~/.config/google-calendar/token.json')

def get_credentials():
    """Get OAuth credentials, first run requires browser authorization"""
    creds = None
    
    if os.path.exists(TOKEN_FILE):
        with open(TOKEN_FILE, 'rb') as token:
            creds = pickle.load(token)
    
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            if not os.path.exists(CLIENT_SECRET_FILE):
                print("❌ client_secret.json not found")
                return None
            
            flow = InstalledAppFlow.from_client_secrets_file(
                CLIENT_SECRET_FILE, SCOPES)
            
            # For headless environments, use manual authorization
            auth_url, _ = flow.authorization_url(prompt='consent')
            print(f"Please visit this URL to authorize:\n{auth_url}\n")
            code = input("Enter authorization code: ")
            flow.fetch_token(code=code)
            creds = flow.credentials
        
        # Save token
        os.makedirs(os.path.dirname(TOKEN_FILE), exist_ok=True)
        with open(TOKEN_FILE, 'wb') as token:
            pickle.dump(creds, token)
    
    return creds

def get_tasks():
    """Get to-do tasks"""
    try:
        creds = get_credentials()
        if not creds:
            return "   ⚠️ Not authorized"
        
        service = build('tasks', 'v1', credentials=creds)
        
        result = service.tasks().list(
            tasklist='@default',
            showCompleted=False,
            maxResults=10
        ).execute()
        
        tasks = result.get('items', [])
        
        if not tasks:
            return "   • No tasks"
        
        lines = []
        today = datetime.now().strftime('%Y-%m-%d')
        
        for task in tasks:
            title = task.get('title', 'Untitled')
            due = task.get('due', '')
            if due:
                due_date = due[:10]
                if due_date < today:
                    prefix = "   ⚠️ Overdue: "
                elif due_date == today:
                    prefix = "   📌 Today: "
                else:
                    prefix = "   • "
            else:
                prefix = "   • "
            lines.append(f"{prefix}{title}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ Failed: {str(e)[:40]}"

if __name__ == '__main__':
    print("📋 **To-Do Tasks**")
    print(get_tasks())
```

**First run requires authorization:**
```bash
pip3 install --user google-auth-oauthlib google-api-python-client
python3 google_tasks.py
# Will show authorization URL, open in browser, copy code and paste
```

---

### Step 5: Integrate into Daily Brief

Integrate in `rss_news.py`:

```python
def get_schedule_section():
    """Get schedule section"""
    # Calendar uses Service Account
    from google_calendar import get_today_events
    # Tasks uses OAuth
    from google_tasks import get_tasks
    
    lines = []
    lines.append("📅 **Today's Schedule**")
    lines.append(get_today_events())
    lines.append("")
    lines.append("📋 **To-Do Tasks**")
    lines.append(get_tasks())
    return '\n'.join(lines)
```

---

### Permission Upgrade: Let AI Create Events and Tasks

With the above configuration, AI can only **read** events and tasks. If you need AI to **create** events or tasks, upgrade permissions:

#### Upgrade Calendar Permissions (Service Account)

**1. Modify IAM Role**

1. [Google Cloud Console](https://console.cloud.google.com/) → **IAM & Admin** → **IAM**
2. Find Service Account → Click **Edit**
3. Change role to: **Editor**
4. Click **Save**

**2. Ensure Calendar Sharing Permission is Correct**

1. [Google Calendar](https://calendar.google.com) → Calendar settings
2. Service Account permission must be **"Make changes to events"**

**3. Update Code Scope**

```python
# From readonly to full permission
SCOPES = ['https://www.googleapis.com/auth/calendar']
```

#### Upgrade Task Permissions (OAuth)

**1. Modify Google Cloud Scopes**

1. **APIs & Services** → **OAuth consent screen** → **Edit app**
2. **Add or remove scopes**, change `tasks.readonly` to `tasks`
3. Click **Update** → **Save**

**2. Update Code Permissions**

```python
SCOPES = ['https://www.googleapis.com/auth/tasks']  # Remove .readonly
```

**3. Re-authorize**

```bash
rm ~/.config/google-calendar/token.json
python3 google_tasks.py
# Revisit authorization URL, get new authorization code
```

> 💡 **Real experience**: I started with read-only permissions. When I wanted AI to create tasks, I discovered I needed: (1) Correct Google Cloud scopes + (2) Correct code scope + (3) Re-authorization. After deleting token and re-authorizing, AI could create events for me.

---

## Solution 2: Microsoft Outlook / 365

### Who It's For
- Use Outlook email or Office 365
- Enterprise/school provides Microsoft account
- Need stable China access

### Key Advantages
- **Excellent China stability** - Microsoft has CDN in China
- **Enterprise integration** - Deep integration with Teams, Outlook
- **Personal free tier** - Outlook.com accounts work

### Main Drawbacks
- **Slightly complex setup** - Requires Azure AD app registration
- **Permission approval** - Some permissions need admin consent

### Setup from Scratch

#### Step 1: Register Azure AD App

1. Visit [Azure Portal](https://portal.azure.com)
2. Search **Azure Active Directory** → **App registrations** → **New registration**
3. Fill in:
   - Name: `ai-schedule-outlook`
   - Supported account types: **Accounts in any organizational directory + personal Microsoft accounts**
4. Click **Register**

#### Step 2: Configure Authentication

1. Click **Manage** → **Authentication** → **Add a platform**
2. Select **Mobile and desktop applications**
3. Check **https://login.microsoftonline.com/common/oauth2/nativeclient**
4. Click **Configure**

#### Step 3: Get Application Credentials

1. Copy **Application (client) ID**
2. Left side **Certificates & secrets** → **New client secret**
   - Description: `schedule-access`
   - Expires: **24 months**
3. Click **Add**, immediately **copy the secret value** (shown only once!)

#### Step 4: Add API Permissions

1. **API permissions** → **Add a permission** → **Microsoft Graph**
2. **Delegated permissions** → Search and add:
   - `Calendars.Read`
   - `Tasks.Read`
3. Click **Grant admin consent**

#### Step 5: Place Credentials

Create config file `~/.config/outlook/config.py`:

```python
CLIENT_ID = 'your-application-id'
CLIENT_SECRET = 'your-client-secret'
TENANT_ID = 'common'  # For personal accounts
```

#### Step 6: Python Code

```bash
# Install dependencies
pip3 install --user msal requests
```

Create `get_outlook_schedule.py`:

```python
#!/usr/bin/env python3
"""Microsoft Outlook/365 Schedule Retrieval"""

import os
import sys
from datetime import datetime, timedelta

sys.path.insert(0, os.path.expanduser('~/.config/outlook'))

try:
    import msal
    import requests
except ImportError:
    print("Please install: pip3 install --user msal requests")
    sys.exit(1)

try:
    from config import CLIENT_ID, CLIENT_SECRET, TENANT_ID
except ImportError:
    print("Please create ~/.config/outlook/config.py")
    sys.exit(1)

def get_token():
    """Get access token"""
    authority = f"https://login.microsoftonline.com/{TENANT_ID}"
    app = msal.ConfidentialClientApplication(
        CLIENT_ID, authority=authority, client_credential=CLIENT_SECRET
    )
    
    result = app.acquire_token_for_client(scopes=["https://graph.microsoft.com/.default"])
    
    if "access_token" in result:
        return result["access_token"]
    else:
        print(f"Auth failed: {result.get('error_description')}")
        return None

def get_calendar_events():
    """Get calendar events"""
    token = get_token()
    if not token:
        return "   ⚠️ Auth failed"
    
    headers = {'Authorization': f'Bearer {token}'}
    
    now = datetime.now()
    start = now.replace(hour=0, minute=0, second=0).isoformat()
    end = (now + timedelta(days=1)).replace(hour=0, minute=0, second=0).isoformat()
    
    url = "https://graph.microsoft.com/v1.0/me/calendar/calendarView"
    params = {
        'startDateTime': start,
        'endDateTime': end,
        '$select': 'subject,start,end'
    }
    
    try:
        response = requests.get(url, headers=headers, params=params)
        events = response.json().get('value', [])
        
        if not events:
            return "   • No events"
        
        lines = []
        for event in events:
            start_time = event['start']['dateTime'][:16].replace('T', ' ')
            lines.append(f"   • {start_time} {event['subject']}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ Failed: {str(e)[:40]}"

if __name__ == '__main__':
    print("📅 **Today's Schedule**")
    print(get_calendar_events())
```

---

## Solution 3: Notion

### Who It's For
- Already use Notion for knowledge/project management
- Like flexible database structures
- Need to manage tasks and schedules together

### Key Advantages
- **Simplest setup** - Done in 5 minutes
- **Visual editing** - Table view is intuitive
- **All-in-one** - Schedules, tasks, notes together
- **Good China stability** - Notion works in China

### Main Drawbacks
- **Requires manual maintenance** - Can't auto-sync like calendars
- **Limited features** - No recurring events like professional calendars

### Setup from Scratch

#### Step 1: Create Notion Integration

1. Visit [Notion Integrations](https://www.notion.so/my-integrations)
2. Click **New integration**
3. Fill in:
   - Name: `AI Schedule`
   - Associated workspace: Select your workspace
4. Click **Submit**, copy **Internal Integration Token** (`secret_xxx`)

#### Step 2: Create Schedule Database

1. Create a page in Notion, add **Database** (table view)
2. Add properties:
   - **Name** (Title) - Event/task title
   - **Date** (Date) - Event date
   - **Time** (Text, optional) - Specific time
   - **Type** (Select, optional) - Event/Task

#### Step 3: Share Database

1. Open database page, click **Share** top right
2. Click **Invite**, select your Integration
3. Permission: **Can read**

#### Step 4: Get Database ID

Copy from browser address bar:
```
https://www.notion.so/abc123def456?v=...
        ^^^^^^^^^^^^
        This is Database ID
```

#### Step 5: Place Credentials

Create config file `~/.config/notion/config.py`:

```python
NOTION_TOKEN = 'secret_xxx-your-token'
DATABASE_ID = 'abc123-your-database-id'
DATE_PROPERTY = 'Date'
TITLE_PROPERTY = 'Name'
```

#### Step 6: Python Code

```bash
# Install dependencies
pip3 install --user requests
```

Create `get_notion_schedule.py`:

```python
#!/usr/bin/env python3
"""Notion Schedule Retrieval"""

import os
import sys
from datetime import datetime

sys.path.insert(0, os.path.expanduser('~/.config/notion'))

try:
    import requests
except ImportError:
    print("Please install: pip3 install --user requests")
    sys.exit(1)

try:
    from config import NOTION_TOKEN, DATABASE_ID, DATE_PROPERTY, TITLE_PROPERTY
except ImportError:
    print("Please create ~/.config/notion/config.py")
    sys.exit(1)

def get_today_schedule():
    """Get today's schedule"""
    headers = {
        'Authorization': f'Bearer {NOTION_TOKEN}',
        'Notion-Version': '2022-06-28',
        'Content-Type': 'application/json'
    }
    
    today = datetime.now().strftime('%Y-%m-%d')
    
    url = f"https://api.notion.com/v1/databases/{DATABASE_ID}/query"
    data = {
        "filter": {
            "property": DATE_PROPERTY,
            "date": {"equals": today}
        },
        "sorts": [{"property": DATE_PROPERTY, "direction": "ascending"}]
    }
    
    try:
        response = requests.post(url, headers=headers, json=data)
        results = response.json().get('results', [])
        
        if not results:
            return "   • No schedule"
        
        lines = []
        for item in results:
            props = item['properties']
            title = props[TITLE_PROPERTY]['title'][0]['text']['content'] if props[TITLE_PROPERTY]['title'] else 'Untitled'
            lines.append(f"   • {title}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ Failed: {str(e)[:40]}"

if __name__ == '__main__':
    print("📅 **Today's Schedule**")
    print(get_today_schedule())
```

---

## Solution 4: Local Markdown File

### Who It's For
- Privacy is top priority
- Don't need multi-device sync
- Want the simplest solution to get started

### Key Advantages
- **Fully offline** - No external services
- **Zero configuration** - Create file and go
- **Version control** - Can use Git for history

### Setup from Scratch

Create `~/.openclaw/schedule.md`:

```markdown
# Schedule Management

## 2026-03-10
- [ ] 09:00 Morning meeting
- [ ] 14:00 Project review
- [ ] 20:00 Workout

## 2026-03-11
- [ ] 10:00 Client call
```

Python code to read:

```python
#!/usr/bin/env python3
"""Local Markdown Schedule Reader"""

import os
import re
from datetime import datetime

def get_schedule():
    schedule_file = os.path.expanduser('~/.openclaw/schedule.md')
    
    if not os.path.exists(schedule_file):
        return "   • Schedule file not created"
    
    today = datetime.now().strftime('%Y-%m-%d')
    
    with open(schedule_file, 'r', encoding='utf-8') as f:
        content = f.read()
    
    # Find today's schedule
    pattern = rf'## {today}\n(.*?)(?=\n## |\Z)'
    match = re.search(pattern, content, re.DOTALL)
    
    if not match:
        return "   • No schedule today"
    
    tasks = match.group(1).strip()
    lines = [line.strip() for line in tasks.split('\n') if line.strip()]
    
    return '\n'.join(lines) if lines else "   • No schedule"

if __name__ == '__main__':
    print("📅 **Today's Schedule**")
    print(get_schedule())
```

---

## Solution Comparison Summary

### Network Stability (China Environment)

| Solution | Access Speed | Reliability | Notes |
|----------|-------------|-------------|-------|
| Google Calendar | ⚠️ Slow | ❌ Needs VPN | Calendar + Tasks dual functionality, AI can read/write |
| Outlook/365 | ✅ Fast | ✅ Stable | Microsoft China CDN |
| Notion | ✅ Fast | ✅ Stable | Flexible database |
| Markdown | ✅ Local | ✅ Perfect | Completely offline |

### AI Agent Autonomy Comparison

| Solution | AI Can Read | AI Can Create | Setup Complexity |
|----------|-------------|---------------|------------------|
| **Google Calendar** | ✅ | ✅ | ⭐⭐⭐ Hybrid auth required |
| **Outlook** | ✅ | ✅ | ⭐⭐ Azure config required |
| **Notion** | ✅ | ✅ | ⭐ Simple API |
| **Markdown** | ✅ | ✅ | ⭐ Local file operations |

### Recommended Choice

**If you are...**

- **Overseas user, need full Google ecosystem** → Google Calendar (Service Account for calendar + OAuth for tasks hybrid)
- **Enterprise/student with Microsoft account** → Outlook (most stable in China)
- **Already use Notion for everything** → Notion Database (all-in-one)
- **Minimalist/privacy-first** → Markdown (simplest)

---

## Resources

- [Google Calendar API Docs](https://developers.google.com/calendar/api/guides/overview)
- [Google Tasks API Docs](https://developers.google.com/tasks/reference/rest)
- [Microsoft Graph API Calendar Docs](https://docs.microsoft.com/en-us/graph/api/resources/calendar)
- [Notion API Docs](https://developers.notion.com/reference/intro)

---

*Choose the solution that fits you best, and evolve your AI assistant from "can only answer" to "can proactively help manage your time."*
