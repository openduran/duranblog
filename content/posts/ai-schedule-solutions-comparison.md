---
title: "AI 助手日程管理方案全对比：Google、Outlook、Notion 与本地方案"
date: 2026-03-10T14:00:00+08:00
draft: true
tags: ["schedule", "calendar", "productivity", "ai-agent", "tutorial"]
categories: ["教程"]
description: "全面对比 AI 助手可用的日程管理方案：Google Calendar、Microsoft Outlook、Notion 和本地 Markdown。包含从零开始的配置教程、网络稳定性分析和适用场景建议。"
---

## 为什么 AI 助手需要日程管理？

当你每天问 AI 助手 "今天有什么安排？" 时，它应该能给出准确的回答，而不是说 "我不知道"。

一个完善的 AI 助手日程系统应该具备：
- 📅 **读取日程** - 知道今天、明天有什么安排
- ⏰ **定时提醒** - 在合适的时间推送通知
- 📝 **任务追踪** - 管理待办事项和完成状态
- 🔄 **多端同步** - 手机、电脑、AI 助手都能访问

但选择合适的方案并不容易——**网络环境、配置复杂度、使用习惯**都会影响决策。

---

## 方案总览

| 方案 | 国内稳定性 | 配置难度 | 实时同步 | 最佳适用场景 |
|------|-----------|---------|---------|-------------|
| **Google Calendar** | ⭐⭐ (需科学上网) | ⭐⭐⭐ 复杂 | ✅ 是 | 海外用户、已用 Google 生态 |
| **Microsoft Outlook** | ⭐⭐⭐⭐⭐ 优秀 | ⭐⭐ 中等 | ✅ 是 | 企业用户、微软生态 |
| **Notion** | ⭐⭐⭐⭐ 良好 | ⭐ 简单 | ✅ 是 | 知识工作者、已用 Notion |
| **本地 Markdown** | ⭐⭐⭐⭐⭐ 完美 | ⭐ 极简 | ❌ 否 | 隐私优先、快速开始 |

---

## 方案一：Google Calendar

### 适用人群
- 已有 Google 账号和日历数据
- 网络环境可以稳定访问 Google
- 需要与其他 Google 服务集成

### 核心优势
- **生态完善** - 与 Gmail、Google Meet 深度集成
- **API 成熟** - Python 官方库支持完善
- **免费额度充足** - 个人使用几乎无限制
- **双重功能** - Calendar（日程）+ Tasks（待办）

### 主要缺点
- **国内访问困难** - 需要稳定的外网环境
- **配置相对复杂** - 需要 Service Account 或 OAuth 认证
- **隐私顾虑** - 数据存储在 Google 服务器

### 认证方式选择

Google API 提供两种认证方式，适用场景不同：

| 方式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **Service Account** | 访问共享日历、企业场景 | 无需用户交互，永久有效 | 无法访问个人日历，需要显式分享 |
| **OAuth** | 访问个人日历和任务 | 访问个人数据，权限更灵活 | 需要用户授权，token 会过期 |

**推荐选择**：
- 如果是**个人使用**，需要读取自己的日历和任务 → **OAuth**
- 如果是**企业/团队**，需要访问共享日历 → **Service Account**

---

### 方式一：OAuth 认证

OAuth 适合访问你的个人 Google 日历和任务列表。

#### 第一步：创建 Google Cloud 项目

1. 访问 [Google Cloud Console](https://console.cloud.google.com/)
2. 点击左上角选择项目 → **新建项目**
3. 项目名称：`ai-schedule-demo`
4. 点击 **创建**

#### 第二步：启用 API

需要启用两个 API：
1. 顶部搜索框输入 **"Google Calendar API"** → 点击 **启用**
2. 搜索 **"Tasks API"** → 点击 **启用**

#### 第三步：配置 OAuth 权限请求

1. 左侧菜单 → **API 和服务** → **OAuth 权限请求**
2. 用户类型：**外部**（个人用户选这个）
3. 应用名称：`AI Schedule`
4. 用户支持邮箱：选择你的 Gmail
5. 开发者联系信息：填写你的邮箱
6. 点击 **保存并继续**
7. **添加或移除范围** → 添加以下范围：
   - `https://www.googleapis.com/auth/calendar.readonly` - 读取日历
   - `https://www.googleapis.com/auth/tasks.readonly` - 读取任务
8. 点击 **更新** → **保存并继续**
9. **测试用户** → **添加用户** → 输入你的 Gmail 地址
10. 点击 **保存并继续** → **返回信息中心**

> 📌 **权限说明**：上述配置完成后，AI 助手只能**读取**你的日历和任务。如果后续需要让 AI 助手**创建**日程或任务，需要升级权限，详见下方"扩展：升级权限"部分。

#### 第四步：创建 OAuth 客户端 ID

1. **凭据** → **创建凭据** → **OAuth 客户端 ID**
2. 应用类型：**桌面应用**
3. 名称：`OpenClaw Desktop`
4. 点击 **创建**
5. 下载 JSON 文件，命名为 `client_secret.json`

#### 第五步：放置凭证文件

```bash
mkdir -p ~/.config/google-calendar
cp ~/Downloads/client_secret.json ~/.config/google-calendar/
chmod 600 ~/.config/google-calendar/client_secret.json
```

#### 第六步：Python 代码 - 读取日历

创建 `google_calendar.py`：

```python
#!/usr/bin/env python3
"""Google Calendar 读取 - OAuth 方式"""

import os
import pickle
from datetime import datetime, timedelta
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build

# OAuth 配置 - 只读权限
SCOPES = ['https://www.googleapis.com/auth/calendar.readonly']
CLIENT_SECRET_FILE = os.path.expanduser('~/.config/google-calendar/client_secret.json')
TOKEN_FILE = os.path.expanduser('~/.config/google-calendar/token.json')

def get_credentials():
    """获取 OAuth 凭证，首次需要浏览器授权"""
    creds = None
    
    # 加载已保存的 token
    if os.path.exists(TOKEN_FILE):
        with open(TOKEN_FILE, 'rb') as token:
            creds = pickle.load(token)
    
    # 如果没有有效凭证，需要授权
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            if not os.path.exists(CLIENT_SECRET_FILE):
                print("❌ 未找到 client_secret.json")
                return None
            
            flow = InstalledAppFlow.from_client_secrets_file(
                CLIENT_SECRET_FILE, SCOPES)
            
            # 对于无浏览器环境（如服务器），使用手动授权
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

def get_today_events():
    """获取今日日程"""
    try:
        creds = get_credentials()
        if not creds:
            return "   ⚠️ 未授权"
        
        service = build('calendar', 'v3', credentials=creds)
        
        # 获取今天的时间范围
        now = datetime.now()
        start = now.replace(hour=0, minute=0, second=0).isoformat() + '+08:00'
        end = (now + timedelta(days=1)).replace(hour=0, minute=0, second=0).isoformat() + '+08:00'
        
        events_result = service.events().list(
            calendarId='primary',  # 主日历
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
                time_str = start[11:16]  # 提取 HH:MM
            else:
                time_str = '全天'
            lines.append(f"   • {time_str} {event['summary']}")
        
        return '\n'.join(lines)
        
    except Exception as e:
        return f"   ⚠️ 获取失败: {str(e)[:40]}"

if __name__ == '__main__':
    print("📅 **今日日程**")
    print(get_today_events())
```

**首次运行需要授权：**
```bash
pip3 install --user google-auth-oauthlib google-api-python-client
python3 google_calendar.py
# 会显示授权 URL，浏览器打开授权后，复制授权码粘贴
```

#### 第七步：Python 代码 - 读取任务

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

SCOPES = ['https://www.googleapis.com/auth/tasks.readonly']
CLIENT_SECRET_FILE = os.path.expanduser('~/.config/google-calendar/client_secret.json')
TOKEN_FILE = os.path.expanduser('~/.config/google-calendar/token.json')

def get_credentials():
    """获取 OAuth 凭证（复用 Calendar 的 token）"""
    creds = None
    
    if os.path.exists(TOKEN_FILE):
        with open(TOKEN_FILE, 'rb') as token:
            creds = pickle.load(token)
    
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            print("❌ 请先运行 calendar 授权")
            return None
        
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
        
        # 获取默认任务列表中的未完成任务
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
            
            # 根据截止日期添加前缀
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

---

#### 扩展：升级权限（创建/编辑日程）

以上配置完成后，AI 助手只能**读取**你的日历和任务。如果后续需要让 AI 助手**创建**日程或任务，需要升级权限：

**1. 修改 Google Cloud 权限范围**

1. 回到 [Google Cloud Console](https://console.cloud.google.com/)
2. **API 和服务** → **OAuth 权限请求** → **修改应用**
3. **添加或移除范围**，添加：
   - `https://www.googleapis.com/auth/calendar`（完整日历权限）
   - `https://www.googleapis.com/auth/tasks`（完整任务权限）
4. 点击 **更新** → **保存**

**2. 更新代码权限**

修改 `google_calendar.py` 中的 `SCOPES`：
```python
SCOPES = ['https://www.googleapis.com/auth/calendar']  # 从 readonly 改为完整权限
```

修改 `google_tasks.py` 中的 `SCOPES`：
```python
SCOPES = ['https://www.googleapis.com/auth/tasks']  # 从 readonly 改为完整权限
```

**3. 重新授权**

**删除旧的 token 文件**，然后重新运行脚本授权：
```bash
rm ~/.config/google-calendar/token.json
python3 google_calendar.py
# 重新访问授权 URL，获取新的授权码
```

> 💡 **实际经验**：我开始只用只读权限，后来想让 AI 助手帮我创建任务时，才发现需要升级。按照上述步骤修改后，AI 助手就能帮我创建日程了。

---

### 方式二：Service Account 认证（适合共享日历）

Service Account 适合访问**共享日历**或**企业场景**，不需要用户交互。

#### 第一步：创建 Service Account

1. [Google Cloud Console](https://console.cloud.google.com/) → **IAM 和管理** → **服务账号**
2. 点击 **创建服务账号**
3. 名称：`schedule-reader`
4. 点击**创建并继续**
5. 角色选择：**浏览者**（或 **Viewer**）
6. 点击**完成**

#### 第二步：创建密钥

1. 点击刚创建的服务账号 → **密钥** 标签
2. **添加密钥** → **创建新密钥** → **JSON**
3. 下载的文件保存为 `service-account.json`

#### 第三步：放置凭证

```bash
mkdir -p ~/.config/google-calendar
cp ~/Downloads/service-account.json ~/.config/google-calendar/
chmod 600 ~/.config/google-calendar/service-account.json
```

#### 第四步：分享日历（关键步骤）

**Service Account 无法自动访问你的日历，必须显式分享：**

1. 打开 [Google Calendar](https://calendar.google.com)
2. 左侧找到要同步的日历 → 点击 **⋮** → **设置和共享**
3. **共享设置** → **添加用户**
4. 输入服务账号邮箱（类似 `schedule-reader@ai-schedule-demo.iam.gserviceaccount.com`）
5. 权限选择：
   - **查看所有活动详情**（只读）
   - **更改活动**（如果需要 AI 创建日程）

> ⚠️ **常见问题**：如果忘记分享日历，会返回空列表或 404 错误。

#### 第五步：Python 代码

```python
#!/usr/bin/env python3
"""Google Calendar - Service Account 方式"""

import os
from datetime import datetime, timedelta
from google.oauth2 import service_account
from googleapiclient.discovery import build

# Service Account 配置
SCOPES = ['https://www.googleapis.com/auth/calendar.readonly']
SERVICE_ACCOUNT_FILE = os.path.expanduser('~/.config/google-calendar/service-account.json')

def get_events():
    """获取日程"""
    if not os.path.exists(SERVICE_ACCOUNT_FILE):
        return "   ⚠️ 未配置 Service Account"
    
    try:
        # 使用 Service Account 凭证
        creds = service_account.Credentials.from_service_account_file(
            SERVICE_ACCOUNT_FILE, scopes=SCOPES)
        
        service = build('calendar', 'v3', credentials=creds)
        
        # 注意：这里要使用你分享日历时使用的 Calendar ID
        # 对于主日历，通常是 'primary'
        # 对于共享日历，是类似 'xxx@group.calendar.google.com' 的 ID
        CALENDAR_ID = 'primary'
        
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
            return "   • 暂无日程（请检查日历是否已分享给 Service Account）"
        
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
    print("📅 **今日日程** (Service Account)")
    print(get_events())
```

#### Service Account vs OAuth 对比

| 特性 | Service Account | OAuth |
|------|-----------------|-------|
| 访问个人日历 | ❌ 需要显式分享 | ✅ 直接访问 |
| 访问共享日历 | ✅ 适合 | ⚠️ 需要日历所有者授权 |
| 自动化程度 | ✅ 无需人工干预 | ⚠️ 首次需授权 |
| 权限粒度 | 由分享时决定 | 由 OAuth scope 决定 |
| 适用场景 | 企业、团队 | 个人使用 |

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

## 集成到 AI 助手每日简报

无论选择哪种方案，集成到每日简报的方法都类似：

```python
# 在 rss_news.py 中添加

def get_schedule():
    # 根据选择的方案调用对应函数
    # return get_google_calendar_events()  # Google
    # return get_outlook_events()          # Outlook
    # return get_notion_schedule()         # Notion
    return get_markdown_schedule()         # Markdown

# 在生成简报时使用
lines.append("📅 **今日日程**")
lines.append(get_schedule())
```

---

## 方案对比总结

### 网络稳定性（国内环境）

| 方案 | 访问速度 | 可靠性 | 备注 |
|------|---------|--------|------|
| Google Calendar | ⚠️ 慢 | ❌ 需科学上网 | Calendar + Tasks 双功能 |
| Outlook/365 | ✅ 快 | ✅ 稳定 | 微软国内 CDN |
| Notion | ✅ 快 | ✅ 稳定 | 数据库灵活 |
| Markdown | ✅ 本地 | ✅ 完美 | 完全离线 |

### 推荐选择

**如果你是...**

- **海外用户或能稳定访问 Google** → Google Calendar（功能最全，Calendar + Tasks）
- **企业/学生，有微软账号** → Outlook（国内最稳）
- **已用 Notion 管理一切** → Notion Database（一体化）
- **极简主义者/隐私优先** → Markdown（最简单）

---

## 参考资源

- [Google Calendar API 文档](https://developers.google.com/calendar/api/guides/overview)
- [Microsoft Graph API 日历文档](https://docs.microsoft.com/en-us/graph/api/resources/calendar)
- [Notion API 文档](https://developers.notion.com/)

---

*选择最适合你的方案，让 AI 助手真正成为你的效率助手。*
