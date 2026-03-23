---
title: "Google Workspace CLI 实战评测：官方工具 vs 自建脚本，企业级自动化方案选型指南"
date: 2026-03-23T12:15:00+08:00
draft: false
tags: ["Google Workspace", "CLI", "自动化", "gws", "DevOps"]
categories: ["技术"]
---

## 引言

Google 最近推出了官方的 **Google Workspace CLI**（`gws`），宣称可以一站式管理 Drive、Gmail、Calendar、Tasks 等所有服务。作为长期使用自建 Python 脚本管理 Google 服务的开发者，我第一时间进行了深度测试。

本文将对比 **官方 CLI 工具** 与 **自建 Python 脚本** 的优劣，并提供完整的配置教程，帮助你做出技术选型决策。

---

## 一、Google Workspace CLI 简介

### 1.1 什么是 gws？

Google Workspace CLI（简称 `gws`）是 Google 官方推出的命令行工具，基于 **Google Discovery Service** 动态构建，支持：

- **Google Drive** - 文件管理、同步
- **Gmail** - 收发邮件、搜索
- **Google Calendar** - 日程管理
- **Google Tasks** - 任务管理
- **Google Sheets/Docs/Slides** - 文档协作
- **Google Chat/Meet** - 团队协作
- **100+ AI Agent Skills** - 自动化技能

### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| **动态构建** | 自动同步最新 API，无需等待工具更新 |
| **多格式输出** | JSON / Table / CSV / YAML |
| **Git 工作流** | `pull`/`push` 本地编辑 Drive 文件 |
| **AI Skills** | 内置 100+ 个可复用的 Agent 技能 |
| **跨平台** | 支持 Linux/macOS/Windows |

---

## 二、功能对比：官方 CLI vs 自建脚本

### 2.1 功能覆盖对比

| 功能 | 自建 Python 脚本 | Google Workspace CLI | 评价 |
|------|-----------------|---------------------|------|
| **Drive 文件管理** | ⚠️ 需自行实现 | ✅ 完整支持 | gws 支持 Git 式 pull/push |
| **Gmail 收发** | ⚠️ 基础功能 | ✅ 完整支持 | gws 支持复杂搜索 |
| **Calendar 查看** | ✅ 简单实现 | ✅ 完整支持 | 两者相当 |
| **Tasks 管理** | ✅ 已实现 | ✅ 完整支持 | 两者相当 |
| **Sheets/Docs 编辑** | ❌ 无 | ✅ 支持 | gws 优势明显 |
| **批量操作** | ❌ 需自己实现 | ✅ 内置分页 | gws 更稳定 |
| **AI Skills** | ❌ 无 | ✅ 100+ 技能 | gws 独有 |

### 2.2 使用体验对比

#### 自建脚本示例（Python）

```python
# 发送邮件（之前的方式）
from google.oauth2 import service_account
from googleapiclient.discovery import build
import base64

creds = service_account.Credentials.from_service_account_file(
    'service-account.json',
    scopes=['https://www.googleapis.com/auth/gmail.send']
)
service = build('gmail', 'v1', credentials=creds)

message = MIMEText('邮件内容')
message['To'] = 'user@example.com'
message['Subject'] = '测试邮件'
raw = base64.urlsafe_b64encode(message.as_bytes()).decode()

service.users().messages().send(userId='me', body={'raw': raw}).execute()
```

**优点**：代码清晰，完全可控
**缺点**：需自己维护，功能单一

#### 官方 CLI 示例

```bash
# 发送邮件（gws）
gws gmail users.messages send \
  --params '{"userId": "me"}' \
  --json '{"raw": "base64encodedstring"}'
```

**优点**：一行命令，功能全面
**缺点**：参数复杂，易出错

### 2.3 中文支持对比

**痛点**：中文主题乱码

```bash
# ❌ 错误方式（会导致乱码）
Subject: 测试邮件

# ✅ 正确方式（RFC 2047 MIME 编码）
Subject: =?UTF-8?B?5rWL6K+V6YKu5Lu2?= 
```

自建脚本需要手动处理编码，gws 同样需要。建议封装脚本解决。

---

## 三、详细配置教程

### 3.1 前置条件

- Google 账号（Gmail 或 Google Workspace）
- 可访问 Google Cloud Console
- Node.js 环境（用于安装 gws）

### 3.2 安装 gws

```bash
# 使用 npm 安装
npm install -g @googleworkspace/cli

# 验证安装
gws --version
# 输出: gws 0.18.1
```

### 3.3 创建 Google Cloud 项目

1. 访问 https://console.cloud.google.com/
2. 创建新项目（如 `my-cli-project`）
3. 记录项目 ID

### 3.4 启用 API

必须启用的 API：

```
✅ Gmail API
✅ Google Drive API
✅ Google Calendar API
✅ Google Tasks API
✅ Google Sheets API
✅ Google Docs API
```

启用地址：
- `https://console.cloud.google.com/apis/library/gmail.googleapis.com`
- `https://console.cloud.google.com/apis/library/drive.googleapis.com`
- （其他 API 类似）

### 3.5 创建 OAuth 客户端

1. 进入 **API 和服务 → 凭据**
2. 点击 **创建凭据 → OAuth 客户端 ID**
3. 应用类型选择 **桌面应用**
4. 名称填 `gws-cli`
5. 点击 **创建**
6. 下载 `client_secret.json`

### 3.6 首次授权

```bash
# 设置环境变量
export GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE="/path/to/client_secret.json"

# 运行授权
gws auth login
```

会输出授权 URL，在浏览器中打开并授权，然后复制回调 URL 中的 `code` 参数。

### 3.7 获取 Access Token

```bash
# 使用 code 换取 token（示例）
curl -X POST https://oauth2.googleapis.com/token \
  -d "code=YOUR_CODE" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "redirect_uri=http://localhost" \
  -d "grant_type=authorization_code"
```

### 3.8 配置持久化

将 token 添加到 `~/.bashrc`：

```bash
echo 'export GOOGLE_WORKSPACE_CLI_TOKEN="ya29.xxx"' >> ~/.bashrc
source ~/.bashrc
```

---

## 四、实战脚本封装

### 4.1 发送中文邮件脚本

由于 gws 对中文支持不完善，建议封装：

```bash
#!/bin/bash
# send-email.sh - 支持中文的邮件发送

export GOOGLE_WORKSPACE_CLI_TOKEN="your-token"

TO="$1"
SUBJECT="$2"
BODY="$3"

python3 << PYEOF
import base64
import json
import urllib.request

subject_encoded = "=?UTF-8?B?" + base64.b64encode("$SUBJECT".encode()).decode() + "?="

email_content = f"""From: me
To: $TO
Subject: {subject_encoded}
Content-Type: text/plain; charset=utf-8

$BODY
"""

encoded = base64.b64encode(email_content.encode()).decode()
data = json.dumps({"raw": encoded}).encode()

req = urllib.request.Request(
    "https://gmail.googleapis.com/gmail/v1/users/me/messages/send",
    data=data,
    headers={
        "Authorization": f"Bearer $GOOGLE_WORKSPACE_CLI_TOKEN",
        "Content-Type": "application/json"
    }
)

with urllib.request.urlopen(req) as response:
    print("邮件发送成功!")
PYEOF
```

### 4.2 文件传输脚本

```bash
#!/bin/bash
# gdrive-transfer.sh

export GOOGLE_WORKSPACE_CLI_TOKEN="your-token"

case "$1" in
    upload)
        gws drive files create \
          --upload "$2" \
          --params "{\"parents\": [\"$3\"]}"
        ;;
    download)
        gws drive files get \
          --params "{\"fileId\": \"$2\"}" \
          --output "$3"
        ;;
    list)
        gws drive files list \
          --params "{\"q\": \"'$2' in parents\"}"
        ;;
esac
```

---

## 五、使用场景建议

### 5.1 适合使用 gws 的场景

✅ **企业自动化流程**
- 需要整合 Drive + Gmail + Calendar 的复杂工作流
- 定期备份 Drive 文件到本地
- 自动化邮件发送和任务管理

✅ **AI Agent 集成**
- 使用 gws 的 100+ Skills 构建智能助手
- 与 OpenClaw 等 Agent 框架结合

✅ **团队协作**
- 多人共享的自动化脚本
- 标准化的 DevOps 流程

### 5.2 不适合使用 gws 的场景

❌ **个人临时使用**
- 配置成本太高（Cloud 项目 + OAuth）
- 不如直接使用网页版高效

❌ **简单单一需求**
- 只需要读取 Calendar
- 自建脚本更简单直接

❌ **短期项目**
- Token 维护成本高
- 学习曲线陡峭

---

## 六、Token 管理最佳实践

### 6.1 Token 有效期

| Token 类型 | 有效期 | 说明 |
|-----------|--------|------|
| access_token | 1 小时 | 短期访问令牌 |
| refresh_token | 7 天 | 用于刷新 access_token |

### 6.2 自动刷新方案

```bash
# 添加到 crontab，每天自动刷新
0 8 * * * /path/to/refresh-token.sh >> /var/log/gws-token.log 2>&1
```

### 6.3 安全存储

```bash
# 使用 pass 或 keyring 存储 token
# 不要明文存储在脚本中

# 推荐方式
export GOOGLE_WORKSPACE_CLI_TOKEN=$(pass show google/workspace-token)
```

---

## 七、总结与建议

### 7.1 核心结论

| 维度 | 评分 | 说明 |
|------|------|------|
| **功能完整性** | ⭐⭐⭐⭐⭐ | 官方工具覆盖最全 |
| **易用性** | ⭐⭐ | 配置复杂，学习曲线陡峭 |
| **稳定性** | ⭐⭐⭐⭐ | 官方维护，API 同步更新 |
| **中文支持** | ⭐⭐ | 需要额外处理编码 |
| **性价比** | ⭐⭐⭐ | 适合企业，个人需谨慎 |

### 7.2 我的建议

**对于个人开发者**：
- 如果只需要 1-2 个功能（如 Tasks + Calendar），**自建脚本更简单**
- 如果需要全面管理 Google 生态，**gws 是终极方案**

**对于企业团队**：
- **强烈推荐 gws**，标准化 + 官方支持
- 配合封装脚本解决中文和易用性问题

**最佳实践**：
```
gws（官方功能）+ 自建封装脚本（易用性）= 完美组合
```

---

## 参考链接

- Google Workspace CLI GitHub: https://github.com/googleworkspace/cli
- Google API Console: https://console.cloud.google.com/
- Gmail API 文档: https://developers.google.com/gmail/api

---

*本文基于 Google Workspace CLI v0.18.1 版本测试，部分功能可能随版本更新而变化*
