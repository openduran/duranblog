---
title: "从脚本到官方：我的 Google 服务管理进化史 - 一个 OpenClaw 用户的 CLI 工具迁移实战"
date: 2026-03-23T15:30:00+08:00
draft: false
tags: ["OpenClaw", "AI Agent", "Google Workspace", "gws", "效率工具", "自动化"]
categories: ["技术"]
---

## 引言：一个 AI Agent 重度使用者的工具进化

作为一名 OpenClaw AI 助手的重度使用者，我的日常工作流早已离不开自动化：

- **每天早上 8:17**，AI 自动推送今日日程和待办任务
- **股票分析** 自动抓取数据并生成技术报告
- **博客发布** 中英文双语自动部署
- **记忆管理** 自动备份到 GitHub

这些自动化的背后，离不开对 Google 服务的深度整合：**Google Calendar** 管理日程、**Google Tasks** 追踪待办、**Google Drive** 存储文件。

之前我写过两篇文章分享我的方案：
- [《AI 助手日程管理实战：OpenClaw + Google Calendar/Tasks 自动化配置》](/posts/ai-schedule-solutions-comparison/) - 用 Python 脚本接入 Google 服务
- [《rclone 挂载 Google Drive：AI 助手的文件管理方案》](/posts/rclone-google-drive-mount/) - 用 rclone 管理 Drive 文件

**但最近我遇到了几个痛点**，促使我重新思考整个方案...

---

## 一、之前方案的问题

### 1.1 Python OAuth 脚本的问题

在《AI 助手日程管理实战》中，我用 Python 脚本 + OAuth 接入 Google Calendar 和 Tasks：

```python
# 之前的方案
from google.oauth2.credentials import Credentials
creds = Credentials.from_authorized_user_file('token.json')
service = build('tasks', 'v1', credentials=creds)
```

**但 Token 经常过期**：
- `invalid_grant: Token has been expired or revoked`
- 每隔几天就要重新授权
- 导致每日简报的任务列表显示 "获取失败"

**维护成本高**：
- 需要手动刷新 token
- 脚本分散，功能单一
- 不同服务需要不同脚本

### 1.2 rclone 的问题

在《rclone 挂载 Google Drive》中，我用 rclone 管理文件：

```bash
rclone mount gdrive: ~/GoogleDrive
```

**但 AI Agent 调用困难**：
- rclone 是文件系统层面的挂载
- OpenClaw 想操作 Drive 文件需要复杂的命令拼接
- 上传下载需要本地文件中转

**配置分散**：
- rclone 一套配置
- Python 脚本另一套配置
- 管理混乱

### 1.3 我的新需求

作为一个 **AI Agent 使用者** 而非开发者，我想要：

✅ **一站式管理** - 一个工具管理所有 Google 服务  
✅ **Token 自动管理** - 不用手动刷新  
✅ **AI 友好** - OpenClaw 可以直接调用  
✅ **中文支持** - 邮件主题不乱码  

**直到我发现 Google 官方推出的 gws（Google Workspace CLI）**

---

## 二、Google Workspace CLI 是什么？

**gws** 是 Google 官方推出的命令行工具，简单来说：

> 就像 `kubectl` 管理 Kubernetes、`aws` 管理 AWS 一样，`gws` 让你用一行命令管理所有 Google 服务。

### 2.1 覆盖的服务

| 服务 | 我能做什么 | 对应之前方案 |
|------|-----------|-------------|
| **Google Tasks** | 创建/完成任务 | 替代 Python OAuth 脚本 |
| **Google Calendar** | 查看/创建日程 | 替代 Python OAuth 脚本 |
| **Gmail** | 收发邮件 | 之前没有 |
| **Google Drive** | 上传/下载/管理文件 | 替代 rclone |
| **Google Sheets** | 读写表格 | 之前没有 |
| **Google Docs** | 编辑文档 | 之前没有 |

### 2.2 对我这个 AI Agent 用户的价值

**之前的工作流**：
```
OpenClaw → Python 脚本 → Google API → Calendar/Tasks
       ↓
     rclone → Google Drive
```

**现在的工作流**：
```
OpenClaw → gws → 所有 Google 服务
```

**统一、简洁、官方支持**

---

## 三、实战：从旧方案迁移到新方案

### 3.1 安装 gws

```bash
npm install -g @googleworkspace/cli
```

### 3.2 认证配置（一劳永逸）

**之前的痛点**：Python 脚本的 OAuth token 几天就过期。

**gws 的解决方案**：
1. 创建 Google Cloud 项目（一次性）
2. 启用需要的 API（Drive、Gmail、Calendar、Tasks）
3. OAuth 授权，获取 refresh_token
4. **refresh_token 有效期 7 天**，自动续期

配置完成后，OpenClaw 可以直接调用：

```bash
export GOOGLE_WORKSPACE_CLI_TOKEN="ya29.xxx"

# 查看任务
gws tasks tasks list

# 发送邮件
gws gmail users.messages send ...
```

### 3.3 替换之前的 Python 脚本

**之前获取 Tasks 的脚本**（经常失效）：

```python
# 之前的代码，token 经常过期
from google_tasks_oauth import get_tasks_service
service = get_tasks_service()  # 经常报错
```

**现在用 gws**：

```bash
# 一行命令，稳定可靠
gws tasks tasks list --format table
```

**对比**：

| 维度 | 之前 Python 脚本 | 现在 gws |
|------|-----------------|---------|
| Token 管理 | 手动刷新，经常过期 | refresh_token 自动续期 |
| 功能范围 | 单一（只能 Tasks） | 全面（所有 Google 服务）|
| 稳定性 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 易用性 | ⭐⭐⭐⭐ | ⭐⭐ |

### 3.4 替换 rclone

**之前用 rclone 管理 Drive**：

```bash
# 挂载到本地
rclone mount gdrive: ~/GoogleDrive

# 然后操作本地文件
```

**现在用 gws**：

```bash
# 直接操作 Drive，无需挂载
gws drive files list
gws drive files create --upload ./file.txt
```

**对比**：

| 维度 | 之前 rclone | 现在 gws |
|------|------------|---------|
| 文件访问 | 挂载为本地文件系统 | API 直接操作 |
| AI 调用 | 复杂（需要本地路径）| 简单（直接命令）|
| 批量操作 | ✅ 高效 | ⚠️ 逐个处理 |
| 适用场景 | 大文件传输 | 日常文件管理 |

**结论**：rclone 保留用于**大文件批量传输**，gws 用于**日常文件管理**

---

## 四、实战：OpenClaw 集成示例

### 4.1 每日简报集成

之前的简报任务获取经常失败（Token 过期），现在改为 gws：

```python
# rss_news.py 中修改
def get_google_tasks():
    """使用 gws 获取任务（替代之前的 OAuth 脚本）"""
    import subprocess
    
    result = subprocess.run(
        ['gws', 'tasks', 'tasks', 'list', 
         '--params', '{"tasklist": "@default"}',
         '--format', 'json'],
        capture_output=True,
        text=True,
        env={'GOOGLE_WORKSPACE_CLI_TOKEN': 'ya29.xxx'}
    )
    
    # 解析 JSON 返回任务列表
    import json
    data = json.loads(result.stdout)
    return data.get('items', [])
```

**效果**：Token 有效期 7 天，且支持自动刷新，不再频繁失效。

### 4.2 发送邮件（新增功能）

**之前的情况**：
- 我的自动化流程中缺少邮件通知能力
- 如果需要发送邮件，只能手动打开 Gmail 网页

**现在用 gws**：

```bash
# 发送邮件（需注意中文编码）
~/.openclaw/workspace/send-email.sh \
  bauhaushuang@hotmail.com \
  '测试邮件' \
  '这是邮件内容'
```

**遇到的坑**：中文主题直接发送会乱码，需要 MIME 编码处理。

**解决方案**：封装脚本自动处理 UTF-8 Base64 编码：

```bash
# 正确的 MIME 编码
Subject: =?UTF-8?B?5rWL6K+V6YKu5Lu2?=  # "测试邮件"的Base64编码
```

**效果**：现在 OpenClaw 可以直接调用发送邮件，比如日报完成后自动邮件通知。

### 4.3 文件管理

之前用 rclone 需要挂载，现在直接操作：

```bash
# 上传到 Drive
gws drive files create \
  --upload ./document.md \
  --params '{"parents": ["FOLDER_ID"]}'

# 下载文件
gws drive files get \
  --params '{"fileId": "FILE_ID"}' \
  --output ./downloaded.md
```

**OpenClaw 可以直接调用这些命令**。

---

## 五、新旧方案完整对比

### 5.1 架构对比

| 组件 | 之前方案 | 现在方案 |
|------|---------|---------|
| **Google Tasks** | Python OAuth 脚本 | gws |
| **Google Calendar** | Python OAuth 脚本 | gws |
| **Gmail** | ❌ 没有 | gws |
| **Google Drive** | rclone | gws + rclone（保留）|
| **Google Sheets** | ❌ 没有 | gws |
| **Token 管理** | 分散，易过期 | 统一，自动续期 |
| **配置维护** | 多套配置 | 一套配置 |

### 5.2 使用体验对比

| 场景 | 之前 | 现在 | 评价 |
|------|------|------|------|
| **每日简报** | Token 经常过期 | Token 稳定 7 天 | ✅ 显著提升 |
| **发送邮件** | ❌ 没有此功能 | 支持中文 | ✅ 新增能力 |
| **文件上传** | rclone 挂载 | 直接命令 | ✅ 更便捷 |
| **大文件传输** | rclone 高效 | gws 逐个处理 | ⚠️ rclone 保留 |
| **配置复杂度** | 中等 | 较高（初始配置）| ⚠️ 学习成本 |

### 5.3 维护成本对比

| 项目 | 之前 | 现在 |
|------|------|------|
| 需要维护的脚本数量 | 3-4 个 | 1 个（gws 封装）|
| Token 刷新频率 | 每 2-3 天 | 每 7 天 |
| 官方支持 | ❌ 社区方案 | ✅ Google 官方 |
| API 更新同步 | 手动更新 | 自动同步 |

---

## 六、我的建议

### 6.1 适合迁移到 gws 的场景

✅ **你和我一样是 AI Agent 重度用户**
- 需要让 OpenClaw/Claude 直接调用 Google 服务
- 希望统一的管理接口

✅ **需要一站式管理**
- 不想维护多个脚本
- 希望 Drive + Gmail + Calendar + Tasks 统一管理

✅ **追求稳定性**
- 受够了 Token 频繁过期
- 希望官方长期支持

### 6.2 保留原有方案的场景

⚠️ **只需要单一功能**
- 只需要读取 Calendar，Python 脚本更简单

⚠️ **大文件批量传输**
- rclone 在批量传输上更高效，保留作为补充

⚠️ **不想折腾配置**
- gws 初始配置较复杂，短期使用不值得

### 6.3 我的最终架构

```
OpenClaw AI 助手
    ├── 日程/任务管理 → gws (替代 Python 脚本)
    ├── 邮件发送 → gws (新增功能)
    ├── 日常文件操作 → gws (替代 rclone 大部分场景)
    └── 大文件批量传输 → rclone (保留)
```

**不是完全替代，而是互补**

---

## 七、总结

从 Python OAuth 脚本 + rclone 到 Google Workspace CLI，我的工具栈完成了一次进化：

**解决的问题**：
- ✅ Token 频繁过期 → refresh_token 7 天有效期
- ✅ 功能分散 → 一站式管理
- ✅ 缺少邮件功能 → 完整 Gmail 支持
- ✅ 中文乱码 → 正确的 MIME 编码

**付出的代价**：
- ⚠️ 初始配置复杂度提升
- ⚠️ 需要学习新的命令格式
- ⚠️ 大文件操作不如 rclone 高效

**最终评价**：

> 作为一个 AI Agent 使用者而非开发者，gws 让我的自动化工作流更加**统一、稳定、可扩展**。虽然配置门槛较高，但一劳永逸，值得投入时间。

如果你也在用 OpenClaw 或其他 AI Agent 框架，并且深度依赖 Google 服务，**强烈推荐尝试 gws**。

---

## 参考

- [我之前的文章：AI 助手日程管理实战](/posts/ai-schedule-solutions-comparison/)
- [我之前的文章：rclone 挂载 Google Drive](/posts/rclone-google-drive-mount/)
- Google Workspace CLI GitHub: https://github.com/googleworkspace/cli

---

*本文作者是一个 OpenClaw AI 助手的使用者，而非 Google 开发者。文章从用户视角出发，分享真实的迁移经验。*
