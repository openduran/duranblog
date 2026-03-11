---
title: "AI 助手日程管理方案全对比：Google、Outlook、Notion 与本地方案"
date: 2026-03-10T14:00:00+08:00
draft: false
tags: ["schedule", "calendar", "productivity", "ai-agent", "tutorial"]
categories: ["教程"]
description: "全面对比 AI 助手可用的日程管理方案：Google Calendar、Microsoft Outlook、Notion 和本地 Markdown。基于实际部署经验，包含完整的配置流程和权限问题解决方案。"
---

## 为什么 AI 助手需要日程管理？

当你问 AI 助手 "今天有什么安排？" 或 "帮我创建一个明天下午 3 点的会议" 时，它应该能准确执行，而不是说 "我不知道"。

一个完善的 AI 助手日程系统应该具备：
- 📅 **读取日程** - 知道今天、明天有什么安排
- ⏰ **定时提醒** - 在合适的时间推送通知
- 📝 **任务追踪** - 管理待办事项和完成状态
- 🤖 **主动创建** - AI 能帮你新建日程和任务
- 🔄 **多端同步** - 手机、电脑、AI 助手都能访问

但选择合适的方案并不容易——**网络环境、配置复杂度、使用习惯**都会影响决策。

---

## 方案总览

| 方案 | 国内稳定性 | 配置难度 | AI 可创建 | 最佳适用场景 |
|------|-----------|---------|-----------|-------------|
| **Google Calendar** | ⭐⭐ (需科学上网) | ⭐⭐⭐ 复杂 | ✅ 是 | 海外用户、完整 Google 生态 |
| **Microsoft Outlook** | ⭐⭐⭐⭐⭐ 优秀 | ⭐⭐ 中等 | ✅ 是 | 企业用户、微软生态 |
| **Notion** | ⭐⭐⭐⭐ 良好 | ⭐ 简单 | ✅ 是 | 知识工作者、灵活数据库 |
| **本地 Markdown** | ⭐⭐⭐⭐⭐ 完美 | ⭐ 极简 | ✅ 是 | 隐私优先、快速开始 |

---

## 方案一：Google Calendar

### 适用人群
- 已有 Google 账号和日历数据
- 网络环境可以稳定访问 Google
- **需要 AI 助手既能读取、又能创建日程和任务**

### 核心优势
- **完整生态** - Calendar + Tasks 双功能，AI 可读写
- **API 成熟** - Python 官方库支持，调试文档完善
- **权限精细** - 可控制 AI 只有读取权，或给予完整控制权
- **免费额度充足** - 个人使用几乎无限制

### 主要缺点
- **国内访问困难** - 需要稳定的外网环境
- **配置相对复杂** - 涉及两种认证方式配合使用
- **权限容易踩坑** - IAM 角色、API Scope、日历分享三层权限容易混淆

---

### 我们的配置方案

基于实际部署经验，我们采用**混合认证方案**：

| 功能 | 认证方式 | 原因 |
|------|---------|------|
| **Calendar（日历）** | Service Account | 日历可以共享给 Service Account，适合自动化访问 |
| **Tasks（任务）** | OAuth | Google Tasks 无法像日历那样共享，必须用 OAuth 访问个人任务列表 |

> 💡 **踩坑经验**：我们最初尝试用 Service Account 同时访问日历和任务，结果发现 Tasks API 不支持 Service Account 访问个人任务列表。最终采用混合方案，日历用 Service Account，任务用 OAuth。

---

### 第一步：创建 Google Cloud 项目

1. 访问 [Google Cloud Console](https://console.cloud.google.com/)
2. 点击左上角选择项目 → **新建项目**
3. 项目名称：`ai-schedule-demo`
4. 点击 **创建**

---

### 第二步：启用 API

需要启用两个 API：
1. 顶部搜索框输入 **"Google Calendar API"** → 点击 **启用**
2. 搜索 **"Tasks API"** → 点击 **启用**

---

### 第三步：配置日历访问（Service Account）

Service Account 适合日历访问，因为日历可以显式共享给它。

#### 3.1 创建 Service Account

1. [Google Cloud Console](https://console.cloud.google.com/) → **IAM 和管理** → **服务账号**
2. 点击 **创建服务账号**
3. 名称：`calendar-reader`
4. 点击**创建并继续**
5. **角色选择**：
   - 如果 AI **只需要读取**日历 → **浏览者**（Viewer）
   - 如果 AI **需要创建/编辑**日程 → **编辑者**（Editor）
6. 点击**完成**

> 📌 **权限说明**：这里选择的 IAM 角色控制 Service Account 对 Google Cloud 资源的访问权限。如果后续需要让 AI 创建日程，需要选择 **Editor** 角色。

#### 3.2 创建密钥

1. 点击刚创建的 Service Account → **密钥** 标签
2. **添加密钥** → **创建新密钥** → **JSON**
3. 下载的文件保存为 `service-account.json`
4. 移动到配置目录：

```bash
mkdir -p ~/.config/google-calendar
cp ~/Downloads/service-account.json ~/.config/google-calendar/
chmod 600 ~/.config/google-calendar/service-account.json
```

#### 3.3 分享日历给 Service Account

**关键步骤**：Service Account 无法自动访问你的日历，必须显式分享。

1. 打开 [Google Calendar](https://calendar.google.com)
2. 左侧找到要同步的日历 → 点击 **⋮** → **设置和共享**
3. **共享设置** → **添加用户**
4. 输入 Service Account 邮箱（类似 `calendar-reader@ai-schedule-demo.iam.gserviceaccount.com`）
5. **权限选择**：
   - **查看所有活动详情** - AI 只能读取
   - **更改活动** - AI 可以创建和编辑日程

> ⚠️ **常见错误**：如果忘记分享日历，或权限设为"仅查看忙碌状态"，API 会返回空列表或 403 错误。

#### 3.4 Python 代码 - 读取日历

创建 `google_calendar.py`：

```python
#!/usr/bin/env python3
"""Google Calendar 读取 - Service Account 方式"""

import os
from datetime import datetime, timedelta
from google.oauth2 import service_account
from googleapiclient.discovery import build

# Service Account 配置
SCOPES = ['https://www.googleapis.com/auth/calendar.readonly']
SERVICE_ACCOUNT_FILE = os.path.expanduser('~/.config/google-calendar/service-account.json')
CALENDAR_ID = 'primary'  # 主日历，或共享日历的 ID

def get_today_events():
    """获取今日日程"""
    if not os.path.exists(SERVICE_ACCOUNT_FILE):
        return "   ⚠️ 未配置 Service Account"
    
    try:
        creds = service_account.Credentials.from_service_account_file(
            SERVICE_ACCOUNT_FILE, scopes=SCOPES)
        service = build('calendar', 'v3', credentials=creds)
        
        # 今天的时间范围
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
            return "   • 暂无日程"
        
        lines = []
        for event in events:
            start = event['start'].get('dateTime', event['start'].get('date'))
            if 'T' in start:
                time_str = start[11:16]
            else:
                time_str = '全天'
            lines.append(f"   • {time_str} {event['summary']}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ 获取失败: {str(e)[:50]}"

if __name__ == '__main__':
    print("📅 **今日日程**")
    print(get_today_events())
```

---

### 第四步：配置任务访问（OAuth）

Google Tasks 无法像日历那样共享，必须使用 OAuth 访问你的个人任务列表。

#### 4.1 配置 OAuth 权限请求

1. 左侧菜单 → **API 和服务** → **OAuth 权限请求**
2. 用户类型：**外部**（个人用户选这个）
3. 应用名称：`AI Schedule`
4. 用户支持邮箱：选择你的 Gmail
5. 开发者联系信息：填写你的邮箱
6. 点击 **保存并继续**

#### 4.2 添加 API 权限范围

**添加 Tasks 权限**（根据需求选择）：

**只读权限：**
- `https://www.googleapis.com/auth/tasks.readonly` - 读取任务

**完整权限**（AI 可以创建/完成任务）：
- `https://www.googleapis.com/auth/tasks` - 完全控制任务

配置步骤：
1. **添加或移除范围** → 添加上述 URL
2. 点击 **更新** → **保存并继续**
3. **测试用户** → **添加用户** → 输入你的 Gmail 地址
4. 点击 **保存并继续** → **返回信息中心**

> 📌 **权限说明**：上述配置完成后，AI 助手只能**读取**任务。如果后续需要让 AI 助手**创建**任务，需要使用 `tasks` 完整权限，并重新授权。

#### 4.3 创建 OAuth 客户端 ID

1. **凭据** → **创建凭据** → **OAuth 客户端 ID**
2. 应用类型：**桌面应用**
3. 名称：`OpenClaw Desktop`
4. 点击 **创建**
5. 下载 JSON 文件，命名为 `client_secret.json`
6. 移动到配置目录：

```bash
cp ~/Downloads/client_secret.json ~/.config/google-calendar/
chmod 600 ~/.config/google-calendar/client_secret.json
```

#### 4.4 Python 代码 - 读取任务

创建 `google_tasks.py`：

```python
#!/usr/bin/env python3
"""Google Tasks 读取 - OAuth 方式"""

import os
import pickle
from datetime import datetime
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build

# OAuth 配置
SCOPES = ['https://www.googleapis.com/auth/tasks.readonly']
CLIENT_SECRET_FILE = os.path.expanduser('~/.config/google-calendar/client_secret.json')
TOKEN_FILE = os.path.expanduser('~/.config/google-calendar/token.json')

def get_credentials():
    """获取 OAuth 凭证，首次需要浏览器授权"""
    creds = None
    
    if os.path.exists(TOKEN_FILE):
        with open(TOKEN_FILE, 'rb') as token:
            creds = pickle.load(token)
    
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            if not os.path.exists(CLIENT_SECRET_FILE):
                print("❌ 未找到 client_secret.json")
                return None
            
            flow = InstalledAppFlow.from_client_secrets_file(
                CLIENT_SECRET_FILE, SCOPES)
            
            # 对于无浏览器环境，使用手动授权
            auth_url, _ = flow.authorization_url(prompt='consent')
            print(f"请访问这个 URL 授权：\n{auth_url}\n")
            code = input("输入授权码：")
            flow.fetch_token(code=code)
            creds = flow.credentials
        
        # 保存 token
        os.makedirs(os.path.dirname(TOKEN_FILE), exist_ok=True)
        with open(TOKEN_FILE, 'wb') as token:
            pickle.dump(creds, token)
    
    return creds

def get_tasks():
    """获取待办任务"""
    try:
        creds = get_credentials()
        if not creds:
            return "   ⚠️ 未授权"
        
        service = build('tasks', 'v1', credentials=creds)
        
        result = service.tasks().list(
            tasklist='@default',
            showCompleted=False,
            maxResults=10
        ).execute()
        
        tasks = result.get('items', [])
        
        if not tasks:
            return "   • 暂无任务"
        
        lines = []
        today = datetime.now().strftime('%Y-%m-%d')
        
        for task in tasks:
            title = task.get('title', '无标题')
            due = task.get('due', '')
            if due:
                due_date = due[:10]
                if due_date < today:
                    prefix = "   ⚠️ 过期: "
                elif due_date == today:
                    prefix = "   📌 今天: "
                else:
                    prefix = "   • "
            else:
                prefix = "   • "
            lines.append(f"{prefix}{title}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ 获取失败: {str(e)[:40]}"

if __name__ == '__main__':
    print("📋 **待办任务**")
    print(get_tasks())
```

**首次运行需要授权：**
```bash
pip3 install --user google-auth-oauthlib google-api-python-client
python3 google_tasks.py
# 会显示授权 URL，浏览器打开授权后，复制授权码粘贴
```

---

### 第五步：整合到每日简报

在 `rss_news.py` 中整合：

```python
def get_schedule_section():
    """获取日程板块"""
    # 日历使用 Service Account
    from google_calendar import get_today_events
    # 任务使用 OAuth
    from google_tasks import get_tasks
    
    lines = []
    lines.append("📅 **今日日程**")
    lines.append(get_today_events())
    lines.append("")
    lines.append("📋 **待办任务**")
    lines.append(get_tasks())
    return '\n'.join(lines)
```

---

### 权限升级：让 AI 创建日程和任务

以上配置完成后，AI 只能**读取**日程和任务。如果需要 AI **创建**日程或任务，需要升级权限：

#### 升级日历权限（Service Account）

**1. 修改 IAM 角色**

1. [Google Cloud Console](https://console.cloud.google.com/) → **IAM 和管理** → **IAM**
2. 找到 Service Account → 点击 **修改**
3. 角色改为：**编辑者**（Editor）
4. 点击 **保存**

**2. 确保日历分享权限正确**

1. [Google Calendar](https://calendar.google.com) → 日历设置
2. Service Account 的权限必须是 **"更改活动"**

**3. 更新代码 Scope**

```python
# 从 readonly 改为完整权限
SCOPES = ['https://www.googleapis.com/auth/calendar']
```

#### 升级任务权限（OAuth）

**1. 修改 Google Cloud 权限范围**

1. **API 和服务** → **OAuth 权限请求** → **修改应用**
2. **添加或移除范围**，将 `tasks.readonly` 改为 `tasks`
3. 点击 **更新** → **保存**

**2. 更新代码权限**

```python
SCOPES = ['https://www.googleapis.com/auth/tasks']  # 去掉 .readonly
```

**3. 重新授权**

```bash
rm ~/.config/google-calendar/token.json
python3 google_tasks.py
# 重新访问授权 URL，获取新的授权码
```

> 💡 **实际经验**：我开始只用只读权限，后来想让 AI 助手帮我创建任务时，才发现需要同时满足：(1) Google Cloud 权限范围正确 + (2) 代码 scope 正确 + (3) 重新授权。删除 token 重新授权后，AI 就能帮我创建日程了。

---

## 方案二：Microsoft Outlook / 365

### 适用人群
- 使用 Outlook 邮箱或 Office 365
- 企业/学校提供微软账号
- 需要国内稳定访问

### 核心优势
- **国内访问稳定** - 微软在国内有 CDN
- **企业集成** - 与 Teams、Outlook 深度整合
- **个人免费** - Outlook.com 账号即可使用

### 主要缺点
- **配置稍复杂** - 需要 Azure AD 注册应用
- **权限申请** - 需要管理员同意某些权限

### 从零开始配置

#### 第一步：注册 Azure AD 应用

1. 访问 [Azure Portal](https://portal.azure.com)
2. 搜索 **Azure Active Directory** → **应用注册** → **新注册**
3. 填写：
   - 名称：`ai-schedule-outlook`
   - 支持的账户类型：**任何组织目录中的账户 + 个人 Microsoft 账户**
4. 点击**注册**

#### 第二步：配置认证

1. 点击 **管理** → **身份验证** → **添加平台**
2. 选择 **移动和桌面应用程序**
3. 勾选 **https://login.microsoftonline.com/common/oauth2/nativeclient**
4. 点击 **配置**

#### 第三步：获取应用凭证

1. 复制 **应用程序(客户端) ID**
2. 左侧 **证书和密码** → **新客户端密码**
   - 描述：`schedule-access`
   - 有效期：**24 个月**
3. 点击**添加**，立即**复制密码值**（只显示一次！）

#### 第四步：添加 API 权限

1. **API 权限** → **添加权限** → **Microsoft Graph**
2. **委托的权限** → 搜索并添加：
   - `Calendars.Read`
   - `Tasks.Read`
3. 点击 **代表管理员同意**

#### 第五步：放置凭证

创建配置文件 `~/.config/outlook/config.py`：

```python
CLIENT_ID = '你的应用程序ID'
CLIENT_SECRET = '你的客户端密码'
TENANT_ID = 'common'  # 个人账号用 'common'
```

#### 第六步：Python 代码

```bash
# 安装依赖
pip3 install --user msal requests
```

创建 `get_outlook_schedule.py`：

```python
#!/usr/bin/env python3
"""Microsoft Outlook/365 日程获取"""

import os
import sys
from datetime import datetime, timedelta

sys.path.insert(0, os.path.expanduser('~/.config/outlook'))

try:
    import msal
    import requests
except ImportError:
    print("请安装依赖: pip3 install --user msal requests")
    sys.exit(1)

try:
    from config import CLIENT_ID, CLIENT_SECRET, TENANT_ID
except ImportError:
    print("请创建 ~/.config/outlook/config.py 配置文件")
    sys.exit(1)

def get_token():
    """获取访问令牌"""
    authority = f"https://login.microsoftonline.com/{TENANT_ID}"
    app = msal.ConfidentialClientApplication(
        CLIENT_ID, authority=authority, client_credential=CLIENT_SECRET
    )
    
    result = app.acquire_token_for_client(scopes=["https://graph.microsoft.com/.default"])
    
    if "access_token" in result:
        return result["access_token"]
    else:
        print(f"认证失败: {result.get('error_description')}")
        return None

def get_calendar_events():
    """获取日历事件"""
    token = get_token()
    if not token:
        return "   ⚠️ 认证失败"
    
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
            return "   • 暂无日程"
        
        lines = []
        for event in events:
            start_time = event['start']['dateTime'][:16].replace('T', ' ')
            lines.append(f"   • {start_time} {event['subject']}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ 获取失败: {str(e)[:40]}"

if __name__ == '__main__':
    print("📅 **今日日程**")
    print(get_calendar_events())
```

---

## 方案三：Notion

### 适用人群
- 已在使用 Notion 管理知识/项目
- 喜欢灵活的数据库结构
- 需要同时管理任务和日程

### 核心优势
- **配置最简单** - 5分钟搞定
- **可视化编辑** - 表格视图直观
- **一体化** - 日程、任务、笔记在一起
- **国内访问稳定** - Notion 在国内可用

### 主要缺点
- **需要手动维护** - 不能像日历自动同步
- **功能有限** - 不如专业日历的重复事件等功能

### 从零开始配置

#### 第一步：创建 Notion Integration

1. 访问 [Notion Integrations](https://www.notion.so/my-integrations)
2. 点击 **New integration**
3. 填写：
   - Name: `AI Schedule`
   - Associated workspace: 选择你的工作区
4. 点击 **Submit**，复制 **Internal Integration Token**（`secret_xxx`）

#### 第二步：创建日程数据库

1. 在 Notion 创建一个页面，添加 **Database**（表格视图）
2. 添加属性：
   - **名称**（Title）- 日程标题
   - **日期**（Date）- 日程日期
   - **时间**（Text，可选）- 具体时间
   - **类型**（Select，可选）- 日程/任务

#### 第三步：分享数据库

1. 打开数据库页面，点击右上角 **Share**
2. 点击 **Invite**，选择刚创建的 Integration
3. 权限选择 **Can read**

#### 第四步：获取 Database ID

从浏览器地址栏复制：
```
https://www.notion.so/abc123def456?v=...
        ^^^^^^^^^^^^
        这是 Database ID
```

#### 第五步：放置凭证

创建配置文件 `~/.config/notion/config.py`：

```python
NOTION_TOKEN = 'secret_xxx你的token'
DATABASE_ID = 'abc123你的数据库ID'
DATE_PROPERTY = '日期'
TITLE_PROPERTY = '名称'
```

#### 第六步：Python 代码

```bash
# 安装依赖
pip3 install --user requests
```

创建 `get_notion_schedule.py`：

```python
#!/usr/bin/env python3
"""Notion 日程获取"""

import os
import sys
from datetime import datetime

sys.path.insert(0, os.path.expanduser('~/.config/notion'))

try:
    import requests
except ImportError:
    print("请安装依赖: pip3 install --user requests")
    sys.exit(1)

try:
    from config import NOTION_TOKEN, DATABASE_ID, DATE_PROPERTY, TITLE_PROPERTY
except ImportError:
    print("请创建 ~/.config/notion/config.py 配置文件")
    sys.exit(1)

def get_today_schedule():
    """获取今日日程"""
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
            return "   • 暂无日程"
        
        lines = []
        for item in results:
            props = item['properties']
            title = props[TITLE_PROPERTY]['title'][0]['text']['content'] if props[TITLE_PROPERTY]['title'] else '无标题'
            lines.append(f"   • {title}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ 获取失败: {str(e)[:40]}"

if __name__ == '__main__':
    print("📅 **今日日程**")
    print(get_today_schedule())
```

---

## 方案四：本地 Markdown 文件

### 适用人群
- 隐私要求极高
- 不需要多端同步
- 想要最简单的方案快速开始

### 核心优势
- **完全离线** - 不依赖任何外部服务
- **零配置** - 创建文件就能用
- **版本控制** - 可用 Git 管理历史

### 从零开始配置

创建 `~/.openclaw/schedule.md`：

```markdown
# 日程管理

## 2026-03-10
- [ ] 09:00 晨会
- [ ] 14:00 项目评审
- [ ] 20:00 健身

## 2026-03-11
- [ ] 10:00 客户电话
```

Python 读取代码：

```python
#!/usr/bin/env python3
"""本地 Markdown 日程读取"""

import os
import re
from datetime import datetime

def get_schedule():
    schedule_file = os.path.expanduser('~/.openclaw/schedule.md')
    
    if not os.path.exists(schedule_file):
        return "   • 未创建日程文件"
    
    today = datetime.now().strftime('%Y-%m-%d')
    
    with open(schedule_file, 'r', encoding='utf-8') as f:
        content = f.read()
    
    # 查找今天的日程
    pattern = rf'## {today}\n(.*?)(?=\n## |\Z)'
    match = re.search(pattern, content, re.DOTALL)
    
    if not match:
        return "   • 暂无今日日程"
    
    tasks = match.group(1).strip()
    lines = [line.strip() for line in tasks.split('\n') if line.strip()]
    
    return '\n'.join(lines) if lines else "   • 暂无日程"

if __name__ == '__main__':
    print("📅 **今日日程**")
    print(get_schedule())
```

---

## 方案对比总结

### 网络稳定性（国内环境）

| 方案 | 访问速度 | 可靠性 | 备注 |
|------|---------|--------|------|
| Google Calendar | ⚠️ 慢 | ❌ 需科学上网 | Calendar + Tasks 双功能，AI 可读写 |
| Outlook/365 | ✅ 快 | ✅ 稳定 | 微软国内 CDN |
| Notion | ✅ 快 | ✅ 稳定 | 数据库灵活 |
| Markdown | ✅ 本地 | ✅ 完美 | 完全离线 |

### AI Agent 自主性对比

| 方案 | AI 可读取 | AI 可创建 | 配置复杂度 |
|------|-----------|-----------|-----------|
| **Google Calendar** | ✅ | ✅ | ⭐⭐⭐ 需混合认证 |
| **Outlook** | ✅ | ✅ | ⭐⭐ 需 Azure 配置 |
| **Notion** | ✅ | ✅ | ⭐ 简单 API |
| **Markdown** | ✅ | ✅ | ⭐ 本地文件操作 |

### 推荐选择

**如果你是...**

- **海外用户，需要完整 Google 生态** → Google Calendar（日历 Service Account + 任务 OAuth 混合方案）
- **企业/学生，有微软账号** → Outlook（国内最稳）
- **已用 Notion 管理一切** → Notion Database（一体化）
- **极简主义者/隐私优先** → Markdown（最简单）

---

## 参考资源

- [Google Calendar API 文档](https://developers.google.com/calendar/api/guides/overview)
- [Google Tasks API 文档](https://developers.google.com/tasks/reference/rest)
- [Microsoft Graph API 日历文档](https://docs.microsoft.com/en-us/graph/api/resources/calendar)
- [Notion API 文档](https://developers.notion.com/reference/intro)

---

*选择最适合你的方案，让 AI 助手从 "只能回答" 进化为 "能主动帮你管理时间" 的真正助手。*
