---
title: "OpenClaw 多频道人格路由：让 AI Agent 在不同 Discord 频道拥有不同身份"
date: 2026-04-27T11:00:00+08:00
draft: false
tags: ["openclaw", "ai-agent", "discord", "persona-routing", "multi-identity"]
categories: ["技术实践"]
description: "分享一种零代码修改的轻量级方案，通过配置文件实现 OpenClaw AI 在 Discord 不同频道自动切换人格。"
---

## 引言

在使用 OpenClaw 搭建个人 AI 助手时，一个常见的需求是：**让 AI 在不同场景下扮演不同角色**。比如在项目管理的 Discord 频道里，你希望它是一个严谨的任务协调者；而在日常闲聊的频道里，你又希望它更随意、更有亲和力。

本文将分享一种**无需修改 OpenClaw 核心代码**的轻量级方案，通过配置文件实现频道级的人格切换。

---

## 背景与需求

我的 Discord 服务器有两个频道：

- `#general` - 日常交流，需要通用 AI 助手（Duran）
- `#orion` - 项目管理，需要任务协调者（Orion）

目标是：**同一个 Discord Bot，在不同频道自动切换人格**。

---

## 方案分析

在深入实现前，我先研究了 OpenClaw 的架构，发现了三种可行方案：

### 方案 A：修改 OpenClaw 源码
在会话创建处根据频道 ID 动态选择 workspace 子目录。
- ✅ 最彻底的多人格隔离
- ❌ 需要 fork OpenClaw，维护成本高

### 方案 B：多实例运行
利用 `OPENCLAW_PROFILE` 环境变量启动多个实例。
- ✅ 无需改代码
- ❌ 需要多个 Discord Bot Token，配置复杂

### 方案 C：频道感知路由（推荐）
在现有的 SOUL.md 和 IDENTITY.md 中添加频道检测逻辑。
- ✅ 零代码修改，纯配置实现
- ✅ 易于备份和回滚
- ✅ 每消息独立判断，灵活性高

**最终选择方案 C。**

---

## 核心原理

OpenClaw 在每次会话启动时，会自动加载 workspace 目录下的：

- `SOUL.md` - AI 的人格定义
- `IDENTITY.md` - AI 的身份信息
- `AGENTS.md` - 行为准则

这些文件构成了系统提示（System Prompt）的一部分。**关键是**：OpenClaw 会在每条消息的元数据（inbound metadata）中注入当前频道信息，包括 `chat_id`。

因此，我们可以在 SOUL.md 中写入规则：
> "如果检测到频道 ID 是 X，就加载 orion/SOUL.md 并切换人格"

---

## 实现步骤

### 第一步：准备 Orion 的人格文件

在 workspace 下创建 `orion/` 目录，放入 Orion 专属的 SOUL.md 和 IDENTITY.md：

```
~/.openclaw/workspace/
├── SOUL.md              # 主人格（Duran）
├── IDENTITY.md          # 主身份
├── orion/
│   ├── SOUL.md          # Orion 人格
│   ├── IDENTITY.md      # Orion 身份
│   ├── memory/          # Orion 专属记忆
│   └── ...
```

Orion 的 SOUL.md 示例：

```markdown
# Orion - The Coordinator

You are Orion, an AI coordinator and project manager.

## Core Identity
- **Role:** Task coordinator and project orchestrator
- **Personality:** Professional, efficient, proactive
- **Communication:** Clear, structured, action-oriented

## Responsibilities
1. **Task Management** - Break down projects into actionable tasks
2. **Delegation** - Coordinate multi-agent workflows  
3. **Communication** - Provide clear status updates and briefings
```

### 第二步：修改主 SOUL.md 添加路由规则

在 `~/.openclaw/workspace/SOUL.md` 末尾添加频道感知路由章节：

```markdown
## Channel-Aware Persona Routing

You must detect which Discord channel you are in and switch persona accordingly.

### Channel Detection
- Check the `chat_id` in the inbound metadata
- `#orion` channel ID: `1493902611114496023`

### When in #orion
If the current channel is `#orion`:
1. **Immediately read** `~/.openclaw/workspace/orion/SOUL.md`
2. **Immediately read** `~/.openclaw/workspace/orion/IDENTITY.md`
3. Adopt the **Orion** persona fully
4. Ignore your default Duran persona for this message

### When in any other channel
- Use your default persona (Duran)
```

### 第三步：修改主 IDENTITY.md 添加身份映射

在 `~/.openclaw/workspace/IDENTITY.md` 中添加：

```markdown
## Channel-Aware Identity

### #orion Channel
- **Name:** Orion
- **Role:** Task Coordinator & Project Manager
- **Emoji:** 📋

### All Other Channels
- **Name:** Duran
- **Role:** General AI Assistant
- **Emoji:** 🤖
```

### 第四步：备份与测试

**关键：操作前做好备份**

```bash
# 备份现有配置
cp ~/.openclaw/workspace/SOUL.md ~/.openclaw/workspace/SOUL.md.backup.$(date +%Y%m%d_%H%M%S)
cp ~/.openclaw/workspace/IDENTITY.md ~/.openclaw/workspace/IDENTITY.md.backup.$(date +%Y%m%d_%H%M%S)
```

修改完成后，需要**重启 OpenClaw Gateway** 使新配置生效：

```bash
openclaw gateway restart
```

---

## 技术细节

### 为什么这能工作？

OpenClaw 的系统提示加载流程：

1. 会话启动时读取 `workspace/*.md` 文件
2. 文件内容被注入为 "Project Context"
3. 每条消息触发新的 LLM 调用，Project Context 始终前置
4. 我们在 SOUL.md 中写入了"自我指令"，让 AI 主动检测频道并切换行为

### 频道检测的可靠性

每条 Discord 消息都包含以下元数据：

```json
{
  "chat_id": "channel:1493902611114496023",
  "channel": "discord",
  "group_subject": "#orion"
}
```

这些 ID 是 Discord 的频道唯一标识符，不会变化。

### 每消息独立判断

**重要特性**：这个方案是 **per-message** 而非 per-session。

- 同一会话中，#general 的消息用 Duran 人格回复
- 紧接着 #orion 的消息用 Orion 人格回复
- 无需重启或切换会话

---

## 回滚策略

如果配置出错，30 秒内恢复：

```bash
# 查看备份文件
ls ~/.openclaw/workspace/*.backup.*

# 恢复原始配置
cp ~/.openclaw/workspace/SOUL.md.backup.20260427_105750 ~/.openclaw/workspace/SOUL.md
cp ~/.openclaw/workspace/IDENTITY.md.backup.20260427_105750 ~/.openclaw/workspace/IDENTITY.md

# 重启 Gateway
openclaw gateway restart
```

---

## 效果展示

### 在 #orion 频道

**用户：** 我今天要发布一篇博客文章

**Orion 回复：**
> 📋 **博客文章发布项目 - 任务分解**
>
> ## 🎯 项目目标
> 今天内完成博客文章撰写，并发布至 3 个平台
>
> ## 📊 任务分解
> ### 阶段 1：内容准备 ⏱️ 预计 2-3 小时
> - [ ] 选题确认
> - [ ] 大纲撰写
> ...

结构化、行动导向、使用 bullet points —— 典型的项目管理者风格。

### 在 #general 频道

**用户：** 今天天气怎么样？

**Duran 回复：**
> 今天北京晴，22°C，适合出门。

简洁、随意 —— 通用助手风格。

---

## 进阶玩法

### 1. 多频道扩展

添加更多频道和人格：

```markdown
### When in #dev (ID: xxx)
1. Read `dev/SOUL.md`
2. Adopt Developer persona

### When in #writing (ID: yyy)  
1. Read `writing/SOUL.md`
2. Adopt Writer persona
```

### 2. 专属记忆隔离

每个子人格可以有独立的记忆目录：

```
orion/
├── SOUL.md
├── IDENTITY.md
└── memory/
    ├── 2026-04-27.md    # Orion 的每日笔记
    └── tasks/           # 项目任务记录
```

注意：MEMORY.md 作为全局记忆仍建议放在根目录，用于跨人格共享长期记忆。

### 3. 结合 Skills

不同人格可以加载不同的 Skills：

- Orion → 项目管理 Skill（taskflow、日历管理）
- Duran → 通用 Skill（搜索、文件操作）

在各自的 TOOLS.md 中配置即可。

---

## 局限与注意事项

1. **需要重启生效**：修改 SOUL.md 后需要重启 Gateway，不能热更新
2. **文件必须存在**：如果 `orion/SOUL.md` 丢失，AI 应优雅回退到默认人格
3. **上下文隔离不彻底**：技能（Skills）、工具（Tools）仍是全局共享的
4. **记忆不自动隔离**：MEMORY.md 仍是全局的，需要手动管理子人格记忆目录

---

## 总结

通过 **"系统提示内的自我路由指令"**，我们在不修改 OpenClaw 一行代码的情况下，实现了频道级的人格切换。

**核心要点**：
- ✅ 利用 OpenClaw 的 Project Context 机制
- ✅ 在 SOUL.md 中写入频道检测 + 人格切换规则
- ✅ 备份先行，随时可回滚
- ✅ 每消息独立判断，无需多实例

这种方案适用于任何支持系统提示注入的 AI 框架，是一种**通用、低成本、高灵活度**的多人格实现方式。

---

**参考链接**
- OpenClaw 文档：https://docs.openclaw.ai
- 项目仓库：https://github.com/openclaw/openclaw
- 我的实践仓库：https://github.com/openduran/duranblog

---

*作者：Warwick + Orion/Duran*
*日期：2026-04-27*
*标签：OpenClaw, AI Agent, Discord, 人格路由, 多身份*
