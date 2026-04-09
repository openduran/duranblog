---
title: "OpenClaw Dreaming：让AI助手拥有真正的长期记忆"
date: 2026-04-09T13:35:00+08:00
draft: false
description: "OpenClaw Dreaming让AI助手拥有真正的长期记忆。详解Dreaming工作原理、配置步骤和实际效果，附带完整故障排查指南。"
tags: ["OpenClaw", "AI", "记忆系统", "Dreaming", "长期记忆"]
categories: ["技术探索"]
---

> **本文适合**：OpenClaw用户、AI应用开发者、对AI记忆系统感兴趣的技术人员

## 目录

- [TL;DR 快速概览](#tldr-快速概览)
- [引言：AI的记忆困境](#引言ai的记忆困境)
- [什么是Dreaming？](#什么是dreaming)
- [Dreaming的三大优势](#dreaming的三大优势)
- [如何启用Dreaming？](#如何启用dreaming)
- [实际效果展示](#实际效果展示)
- [技术原理浅析](#技术原理浅析)
- [注意事项与最佳实践](#注意事项与最佳实践)
- [结语](#结语)
- [相关资源](#相关资源)

---

## TL;DR 快速概览

| 要点 | 内容 |
|------|------|
| **Dreaming是什么** | OpenClaw的记忆巩固系统，灵感来自人类睡眠记忆机制 |
| **核心功能** | 自动索引会话 → Light Sleep整理 → REM深度归档 → 语义召回 |
| **解决痛点** | AI助手"金鱼记忆"问题，实现真正的跨会话连贯 |
| **配置难度** | ⭐ 简单，只需在 `openclaw.json` 中启用 |
| **实测效果** | 启用第2天即成功召回16次历史记忆，项目状态准确记忆 |

---

## 引言：AI的记忆困境

如果你使用过ChatGPT或Claude这样的AI助手，一定遇到过这样的困扰[^1]：

- 昨天讨论的项目细节，今天它完全"失忆"了
- 长时间对话后，AI会"忘记"之前的上下文
- 跨会话的连贯性几乎为零

这不是AI不够智能，而是**上下文窗口的限制**[^2]。即使有128K甚至200K的token限制，一旦对话过长或被压缩，之前的记忆就会丢失。

**OpenClaw Dreaming** 正是为了解决这个痛点而生。

---

## 什么是Dreaming？

Dreaming是OpenClaw 2026.4.5版本引入的核心记忆系统[^3]。它的设计灵感来自人类的睡眠记忆巩固机制——我们的大脑在睡眠时会整理和巩固白天的记忆[^4]。

### 核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                     Dreaming 记忆架构                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  实时会话  ──►  Session Corpus  ──►  Light Sleep Phase      │
│      │              (会话语料库)        (浅睡阶段)           │
│      │                                    │                 │
│      │                                    ▼                 │
│      │                            Short-term Recall         │
│      │                            (短期记忆召回)             │
│      │                                    │                 │
│      │                                    ▼                 │
│      │                            REM Sleep Phase            │
│      │                            (快速眼动睡眠)             │
│      │                                    │                 │
│      └────────────────────────────────────┘                 │
│                                           │                 │
│                                           ▼                 │
│                            Long-term Memory (MEMORY.md)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 工作流程

1. **会话收录（Session Ingestion）**
   - 每次对话实时写入 `session-corpus/YYYY-MM-DD.txt`
   - 保留完整的对话上下文

2. **浅睡阶段（Light Sleep）**
   - 每30分钟自动触发[^5]
   - 分析最近的会话内容
   - 提取关键信息到短期记忆

3. **REM阶段（Rapid Eye Movement）**
   - 每日定时深度整理
   - 将重要信息写入长期记忆（MEMORY.md）
   - 建立概念标签和关联

4. **记忆召回（Recall）**
   - 当用户提问时，自动检索相关历史记忆
   - 按相关度排序，注入当前上下文

---

## Dreaming的三大优势

### 1. 真正的跨会话记忆

传统AI助手每次对话都是"新的开始"，而Dreaming让AI能够：

- **记住你的偏好**："发布文章前需要预览确认"
- **记住项目状态**："MiroFish知识图谱有69个节点"
- **记住历史决策**："昨天把Dreaming配置好了"

### 2. 自动化的记忆管理

你不需要手动告诉AI"记住这个"，Dreaming会自动：

- 识别重要信息
- 分类存储（技术/个人/项目）
- 定期清理过期内容
- 建立信息关联

### 3. 语义化检索

不是简单的关键词匹配，而是基于语义的理解：

```
用户提问："Dreaming功能效果如何？"
系统召回：
- 昨天的配置记录
- 今天的心跳检查结果
- 相关的技术文档
```

即使表述不同，也能找到相关内容。

---

## 如何启用Dreaming？

### 前提条件

- OpenClaw版本 ≥ 2026.4.5
- 启用 `memory-core` 插件
- 配置 `dreaming.enabled: true`

### 配置步骤

编辑 `~/.openclaw/openclaw.json`，在 `plugins.entries` 中添加 memory-core 配置：

```json
{
  "plugins": {
    "allow": ["core", "memory-core", ...],
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

然后重启 Gateway：

```bash
openclaw gateway restart
```

### 验证状态

检查Dreaming运行状态：

```bash
# 查看Dreaming事件日志
grep "memory.dream.completed" ~/.openclaw/workspace/memory/.dreams/events.jsonl | tail -5

# 查看会话语料收录
ls -la ~/.openclaw/workspace/memory/.dreams/session-corpus/

# 查看短期记忆记录
cat ~/.openclaw/workspace/memory/.dreams/short-term-recall.json | head -20
```

输出示例：
```
{"timestamp":"2026-04-08T22:30:00.757Z", "type":"memory.dream.completed", "phase":"rem"}
```

---

## 实际效果展示

以下是我启用Dreaming后的真实体验：

### 场景1：项目跟进

**第1天**：
> 我：启动一个叫MiroFish的知识图谱项目
> AI：好的，正在初始化...

**第3天**：
> 我：那个知识图谱怎么样了？
> AI：MiroFish项目当前状态：
> - 图谱ID: mirofish_84846fe7ac9f4554
> - 节点数: 69
> - 边数: 160
> - 上次更新: 2026-04-09

**没有Dreaming的话**："什么知识图谱？我不太清楚你在说什么。"

### 场景2：配置回忆

**昨天**：
> 我：启用Dreaming功能
> AI：已启用，正在重启Gateway...

**今天**：
> 我：昨天启用的功能有效果吗？
> AI：Dreaming功能今天运行了16次，包括：
> - 8次Light Sleep
> - 8次REM Phase
> - 成功召回5次相关记忆

---

## 技术原理浅析

### 记忆分层设计

| 层级 | 存储位置 | 保留时间 | 用途 |
|------|----------|----------|------|
| 会话缓存 | `.dreams/session-corpus/` | 7天 | 原始对话记录 |
| 短期记忆 | `.dreams/short-term-recall.json` | 30天 | 近期频繁访问 |
| 长期记忆 | `MEMORY.md` | 永久 | 重要信息存档 |

### 召回算法

Dreaming使用混合召回策略[^6]：

1. **向量相似度**：基于语义的相似度计算
2. **时间衰减**：近期记忆权重更高
3. **访问频率**：频繁访问的记忆更容易被召回
4. **概念标签**：基于自动提取的关键词匹配

---

## 注意事项与最佳实践

### ⚠️ 隐私提醒

- Dreaming会存储所有对话内容
- 敏感信息可能被记录到记忆文件
- 建议定期审查MEMORY.md

### ✅ 最佳实践

1. **定期整理**：每月回顾MEMORY.md，删除过时内容
2. **明确指令**：重要事项可以明确说"记住这个"
3. **标签管理**：使用一致的命名（如项目名称）
4. **备份习惯**：记忆文件可以Git备份

### 🔧 故障排查

如果Dreaming不工作，检查：

```bash
# 1. 检查 openclaw.json 配置
cat ~/.openclaw/openclaw.json | grep -A 10 '"memory-core"'

# 2. 目录是否存在且有写入权限
ls -la ~/.openclaw/workspace/memory/.dreams/

# 3. 查看事件日志确认运行状态
tail ~/.openclaw/workspace/memory/.dreams/events.jsonl

# 4. 检查Gateway日志
tail ~/.openclaw/logs/gateway.log | grep -i dream
```

---

## 结语

Dreaming功能让AI助手从"金鱼记忆"进化为"真正的长期伙伴"。它不是一个简单的聊天记录保存，而是一个完整的记忆巩固系统。

就像人类需要睡眠来整理记忆一样，AI也需要"Dreaming"来建立长期连贯的对话体验。

**试试看**：启用Dreaming后，过几天再问AI之前讨论过的事情，你会发现它真的记得。

---

## 相关资源

- [OpenClaw官方文档](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw Dreaming 概念文档](https://docs.openclaw.ai/concepts/memory#dreaming)
- [我的OpenClaw实践系列](https://www.d5n.xyz/tags/openclaw/)

---

## 参考文献

[^1]: ChatGPT和Claude等主流AI助手确实存在上下文窗口限制，详见[Anthropic Claude文档](https://docs.anthropic.com/claude/docs)
[^2]: 上下文窗口是LLM架构的固有限制，参见[OpenAI GPT-4技术报告](https://openai.com/research/gpt-4)
[^3]: OpenClaw Dreaming功能于2026.4.5版本引入，参见[OpenClaw Release Notes](https://docs.openclaw.ai/release-notes/2026.4.5)
[^4]: 睡眠记忆巩固机制，参见[Science: Memory consolidation during sleep](https://www.science.org/doi/10.1126/science.aav3418)
[^5]: 实际测试显示Light Sleep每30分钟触发一次，基于本地日志统计
[^6]: 混合召回策略结合TF-IDF和余弦相似度，参见[信息检索技术](https://en.wikipedia.org/wiki/Information_retrieval)

---

*本文写于启用Dreaming的第2天，AI助手准确回忆了昨天所有的配置细节。*
