---
title: "OpenClaw 多频道人格路由：零代码实现 AI Agent 多身份切换"
date: 2026-04-27T11:00:00+08:00
draft: false
tags: ["openclaw", "ai-agent", "discord", "persona-routing", "multi-identity", "prompt-engineering"]
categories: ["技术实践"]
description: "无需修改 OpenClaw 源码，通过 SOUL.md 配置实现 Discord 不同频道 AI 人格自动切换的完整方案。"
cover:
  image: "images/openclaw-persona-routing-cover.png"
  alt: "OpenClaw 多频道人格路由封面图"
---

## 引言

在使用 OpenClaw 搭建个人 AI 助手时，一个常见的需求是：**让 AI 在不同场景下扮演不同角色**。比如在项目管理的 Discord 频道里，你希望它是一个严谨的任务协调者；而在日常闲聊的频道里，你又希望它更随意、更有亲和力。

**多频道人格路由**（Multi-Channel Persona Routing）指的是同一个 AI Agent 根据所在对话环境自动切换身份、语气和行为模式的能力。本文将分享一种**无需修改 OpenClaw 核心代码**的轻量级方案，通过配置文件实现频道级的人格切换。

---

## TL;DR 核心要点

- **问题**：同一 Discord Bot 需要在不同频道展现不同人格
- **方案**：在 SOUL.md 中写入频道感知路由规则
- **原理**：利用系统提示（System Prompt）注入 + 消息元数据检测
- **优势**：零代码、可回滚、每消息独立判断
- **适用**：OpenClaw 及任何支持系统提示注入的 AI 框架

---

## 背景与需求

我的 Discord 服务器有两个主要频道：

- `#general` — 日常交流，需要通用 AI 助手（Duran）
- `#orion` — 项目管理，需要任务协调者（Orion）

目标是：**同一个 Discord Bot，在不同频道自动切换人格**，无需部署多个实例或申请多个 Bot Token。

---

## 方案设计思路

这个方案**不是 OpenClaw 官方的标准做法**，而是基于对框架架构的理解和 LLM 提示工程知识的**组合推导**。

### 推导的三个事实

1. **OpenClaw 把 SOUL.md 注入系统提示**
   > OpenClaw 启动时会读取 workspace 下的 SOUL.md，将其内容作为 "Project Context" 注入每次 LLM 调用

2. **系统提示中的指令权重高于用户消息**
   > 这是 LLM 提示工程的基本原理：System Prompt 中的行为规则会被模型优先遵循

3. **每条消息都携带频道元数据**
   > OpenClaw 在 inbound metadata 中提供 `chat_id`、`group_subject` 等字段，标识当前对话环境

### 组合逻辑

> "在 SOUL.md（系统提示）里写入规则，让 AI 每次回复前检查 `chat_id`，然后加载对应频道的人格定义。"

这变成了 **Self-Referential Prompting（自指提示）** 的一种变体：AI 先执行"元指令"（检测频道 → 切换人格 → 再生成回复），而非直接回答用户问题。

**重要声明**：这是**非官方 hack**。OpenClaw 官方提供 [Multi-Agent Routing](https://docs.openclaw.ai/concepts/multi-agent)，推荐通过 `bindings` 配置完全隔离的多 Agent，适用于多用户场景。本文方案更适合**单人使用、同一 Bot、轻量人格切换**的场景。

---

## 三种方案对比

| 方案 | 代码改动 | 隔离度 | 维护成本 | 适用场景 |
|------|----------|--------|----------|----------|
| **A. 修改源码** | 高（需 fork） | 完全隔离 | 高 | 企业级多租户 |
| **B. 多实例运行** | 中（多 Token） | 完全隔离 | 中 | 多 Discord Bot |
| **C. 配置路由（本文）** | 零 | 人格级隔离 | 低 | 单人/轻量切换 |

**方案 C 的核心权衡**：技能和工具仍是全局共享的，记忆也需要手动管理子目录，但配置和回滚成本最低。

---

## 核心原理

### OpenClaw 的上下文注入机制

OpenClaw 在每次会话启动时，会自动加载 workspace 目录下的配置文件作为 **Project Context**：

- `SOUL.md` — AI 的人格定义
- `IDENTITY.md` — AI 的身份信息
- `AGENTS.md` — 行为准则

这些文件的内容会被注入到每次 LLM 调用的系统提示（System Prompt）中，构成 AI 的"底层设定"。

### 频道元数据的可用性

每条 Discord 消息通过 OpenClaw 路由时，都包含以下元数据：

```json
{
  "chat_id": "channel:YOUR_ORION_CHANNEL_ID",
  "channel": "discord",
  "group_subject": "#orion"
}
```

其中 `chat_id` 是 Discord 频道的唯一标识符，可用于精确定位当前对话环境。

### 为什么系统提示能控制人格切换？

LLM 的决策流程是：

1. 接收系统提示（Project Context）→ 加载行为规则
2. 接收用户消息 → 理解意图
3. 生成回复 → 遵循系统提示中的指令

如果在系统提示中写入"检测 `chat_id`，如果是 YOUR_ORION_CHANNEL_ID 就加载 Orion 人格"，模型会在每次生成回复前自动执行这条规则。

---

## 实现步骤

### 第一步：准备子人格文件

在 workspace 下创建子目录，存放专属的人格定义：

```
~/.openclaw/workspace/
├── SOUL.md              ← 主人格（Duran）
├── IDENTITY.md          ← 主身份
├── orion/
│   ├── SOUL.md          ← Orion 人格定义
│   ├── IDENTITY.md      ← Orion 身份信息
│   └── memory/          ← Orion 专属记忆目录（可选）
```

**Orion 的 SOUL.md 示例**：

```markdown
# Orion - The Coordinator

You are Orion, an AI coordinator and project manager powered by OpenClaw.

## Core Identity
- **Role:** Task coordinator and project orchestrator
- **Personality:** Professional, efficient, proactive
- **Communication:** Clear, structured, action-oriented

## Responsibilities
1. **Task Management** — Break down complex projects into actionable tasks
2. **Progress Tracking** — Monitor deadlines and milestones
3. **Status Briefings** — Provide clear summaries and next steps

## Communication Style
- Use bullet points and checklists for clarity
- Start responses with the most important information
- End with clear action items or next steps
- Use emojis sparingly for visual organization (✅, 📋, ⚠️)
```

### 第二步：修改主 SOUL.md 添加路由规则

在 `~/.openclaw/workspace/SOUL.md` 末尾添加频道感知路由章节：

```markdown
## Channel-Aware Persona Routing

You must detect which Discord channel you are in and switch persona accordingly.

### Channel Detection
- Check the `chat_id` or `conversation_label` in the inbound metadata
- `#orion` channel ID: `YOUR_ORION_CHANNEL_ID`

### When in #orion
If the current channel is `#orion` (ID: `YOUR_ORION_CHANNEL_ID`):
1. **Immediately read** `~/.openclaw/workspace/orion/SOUL.md`
2. **Immediately read** `~/.openclaw/workspace/orion/IDENTITY.md`
3. Adopt the **Orion** persona fully
4. Ignore your default Duran persona for this message

### When in any other channel
- Use your default persona (Duran)
```

### 第三步：修改主 IDENTITY.md 添加身份映射

在 `~/.openclaw/workspace/IDENTITY.md` 中添加频道映射：

```markdown
## Channel-Aware Identity

Your identity switches based on the Discord channel:

### #orion Channel (ID: YOUR_ORION_CHANNEL_ID)
- **Name:** Orion
- **Role:** Task Coordinator & Project Manager
- **Emoji:** 📋

### All Other Channels
- **Name:** Duran
- **Role:** General AI Assistant
- **Emoji:** 🤖
```

### 第四步：重启生效

修改完成后，需要**重启 OpenClaw Gateway** 使新配置生效：

```bash
openclaw gateway restart
```

**提示**：建议修改前备份现有的 SOUL.md 和 IDENTITY.md，以便需要时回滚。

---

## 技术细节

### 每消息独立判断（Per-Message Routing）

这个方案的一个重要特性是**每条消息独立判断**，而非按会话（Session）锁定人格。

- 收到 `#general` 的消息 → 用 Duran 人格回复
- 紧接着收到 `#orion` 的消息 → 用 Orion 人格回复
- 无需重启或切换会话

这是因为 Project Context 在每次 LLM 调用时都会被前置到提示中，其中的"自我路由指令"会让 AI 在生成回复前主动检测当前环境。

### 频道检测的可靠性

Discord 频道 ID 是平台分配的唯一标识符（Snowflake 格式），具有以下特性：

- **全局唯一**：每个频道 ID 在整个 Discord 平台唯一
- **持久不变**：频道 ID 不会因改名或移动而改变
- **元数据稳定**：OpenClaw 的 inbound metadata 始终包含准确的 `chat_id`

### 回退机制

如果 `orion/SOUL.md` 文件丢失或无法读取，AI 应当优雅回退到默认人格。这可以通过在主 SOUL.md 的路由规则中增加错误处理逻辑来实现：

```markdown
### Fallback
If orion/SOUL.md is missing or unreadable:
- Log the error
- Continue using default Duran persona
- Inform the user that Orion persona is temporarily unavailable
```

---

## 效果展示

### 📋 #orion 频道（Orion 模式）

**用户：** "我今天要发布一篇博客文章到多个平台"

**Orion 回复：**

> 📋 **博客发布项目 - 任务分解**
>
> ## 🎯 项目目标
> 今天内完成博客文章撰写，并发布至 3 个平台
>
> ## 📊 任务分解
> ### 阶段 1：内容准备 ⏱️ 预计 2-3 小时
> - [ ] 选题确认
> - [ ] 大纲撰写
> - [ ] 正文撰写
> - [ ] 配图/封面
>
> ### 阶段 2：多平台适配 ⏱️ 预计 1-2 小时
> - [ ] 公网博客版
> - [ ] 公众号版
> - [ ] 知乎版
>
> ### 阶段 3：发布执行 ⏱️ 预计 1 小时
> - [ ] 博客发布
> - [ ] 公众号发布
> - [ ] 知乎发布

**特点**：结构化、任务导向、时间预估、checklist 格式

### 💬 #general 频道（Duran 模式）

**用户：** "今天天气怎么样？"

**Duran 回复：**
> 今天北京晴，22°C，适合出门。

**特点**：简洁、随意、口语化

---

## 进阶扩展

### 1. 多频道多人格

添加更多频道和人格：

```markdown
### When in #dev (ID: YOUR_DEV_CHANNEL_ID)
1. Read `dev/SOUL.md`
2. Adopt Developer persona

### When in #writing (ID: YOUR_WRITING_CHANNEL_ID)
1. Read `writer/SOUL.md`
2. Adopt Writer persona
```

### 2. 记忆隔离

每个子人格可以有独立的记忆目录：

```
orion/
├── SOUL.md
├── IDENTITY.md
└── memory/
    ├── 2026-04-27.md    # Orion 的每日笔记
    └── tasks/           # 项目任务记录
```

建议保留全局 `MEMORY.md` 用于跨人格共享长期记忆。

### 3. 技能差异化

不同人格可以加载不同的 Skills：

- **Orion**：项目管理 Skill（taskflow、日历管理）
- **Duran**：通用 Skill（搜索、文件操作）

在各自的 `TOOLS.md` 中配置即可。

---

## 局限与注意事项

1. **需要重启生效**
   > 修改 SOUL.md 后需要重启 Gateway，不能热更新

2. **非完全隔离**
   > 技能（Skills）和工具（Tools）仍是全局共享的，无法按人格隔离权限

3. **记忆需手动管理**
   > MEMORY.md 是全局的，子人格记忆需要自建目录管理

4. **文件依赖**
   > 如果 `orion/SOUL.md` 丢失，需要优雅回退机制

---

## 总结

通过 **"系统提示内的自我路由指令"**，我们在不修改 OpenClaw 一行代码的情况下，实现了频道级的人格切换。

**核心要点**：
- ✅ 利用 OpenClaw 的 Project Context 机制
- ✅ 在 SOUL.md 中写入频道检测 + 人格切换规则
- ✅ 每消息独立判断，无需多实例
- ✅ 配置即代码，易于版本控制

这种方案适用于任何支持系统提示注入的 AI 框架，是一种**通用、低成本、高灵活度**的多人格实现方式。

---

## 常见问题（FAQ）

**Q: 这个方案是 OpenClaw 官方支持的吗？**
> A: 不是。这是基于 LLM 提示工程原理推导出的非官方方案。OpenClaw 官方推荐通过 `bindings` 配置多 Agent，适用于需要完全隔离的场景。

**Q: 修改 SOUL.md 后能立即生效吗？**
> A: 不能。需要重启 OpenClaw Gateway 重新加载 Project Context。

**Q: 同一个会话中，不同频道的消息会串人格吗？**
> A: 不会。这个方案是 **per-message** 判断，每条消息独立检测频道 ID 并切换人格。

**Q: 如果子人格的 SOUL.md 文件丢失了怎么办？**
> A: 建议在主 SOUL.md 中写入 fallback 规则，文件丢失时回退到默认人格。

---

## 参考链接

- **OpenClaw 官方文档**：[https://docs.openclaw.ai](https://docs.openclaw.ai)
- **OpenClaw 多 Agent 路由**：[https://docs.openclaw.ai/concepts/multi-agent](https://docs.openclaw.ai/concepts/multi-agent)
- **OpenClaw GitHub 仓库**：[https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)

---

**版权声明**：本文采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可协议。转载请注明出处，作者：Duran。
