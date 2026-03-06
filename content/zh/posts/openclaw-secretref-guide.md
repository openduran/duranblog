---
title: "OpenClaw API 密钥管理完全指南：从明文到 SecretRef"
date: 2026-03-03T22:00:00+08:00
draft: false
tags: ["openclaw", "security", "secretref", "tutorial"]
---

## 前言

在使用 OpenClaw 的过程中，我们不可避免地会接触到各种 API 密钥：Discord Bot Token、Kimi API Key、GitHub PAT 等。这些密钥如果明文存储在配置文件中，存在严重的安全隐患。

本文将系统性地介绍 OpenClaw 2026.3.2 引入的 **SecretRef** 功能，帮助你把明文密钥迁移到安全存储，实现生产级的密钥管理。

## 一、明文存储的风险

### 1.1 当前密钥分布现状

| 密钥类型 | 典型存储位置 | 风险等级 |
|---------|-------------|---------|
| Discord Token | `openclaw.json` | 🔴 高 |
| AI API Keys | `auth-profiles.json` | 🔴 高 |
| GitHub PAT | git remote URL | 🔴 高 |
| OAuth Tokens | 配置文件 | 🟡 中 |

### 1.2 OpenClaw 2026.3.2 自动迁移范围

**升级到 2026.3.2 后，以下配置会被自动迁移：**

| 配置项 | 自动迁移 | 迁移后格式 | 说明 |
|--------|---------|-----------|------|
| `channels.discord.token` | ✅ 是 | `${env:DISCORD_TOKEN}` | 自动转为 env 引用 |
| `channels.telegram.token` | ✅ 是 | `${env:TELEGRAM_TOKEN}` | 自动转为 env 引用 |
| 其他 channel tokens | ✅ 是 | `${env:XXX_TOKEN}` | 自动转为 env 引用 |
| `auth-profiles.json` API Keys | ❌ 否 | 仍是明文 | **需要手动迁移** |
| Git remote URL 中的 PAT | ❌ 否 | 仍是明文 | **需要手动迁移** |

**检查自动迁移结果：**

```bash
# 检查 channel 配置是否已迁移
grep -E '\$\{env:' ~/.openclaw/openclaw.json

# 检查 auth-profiles 是否仍是明文
grep -E '"key":\s*"sk-' ~/.openclaw/agents/main/agent/auth-profiles.json

# 检查 git remote 是否仍是明文
git remote -v | grep 'ghp_'
```

### 1.3 明文存储的安全隐患

```json
// 危险：明文存储在配置文件中
{
  "channels": {
    "discord": {
      "token": "MTQ2Njc4MDY2NzgwNjIyMDM2NA.GvboSs.xxx"  // ❌ 泄露风险
    }
  }
}
```

**潜在风险：**
- 配置文件可能被提交到 Git 仓库
- 多人协作时密钥暴露范围扩大
- 日志中可能意外打印敏感信息
- 无法满足合规审计要求

## 二、SecretRef 架构介绍

### 2.1 核心概念

**SecretRef（密钥引用）** 是一种"引用而非持有"的安全模型：

```
传统方式：配置文件持有明文密钥
    ↓
SecretRef 方式：配置文件只存储引用，密钥存储在安全后端
```

### 2.2 工作原理

```yaml
# 传统方式（明文）
token: "sk-xxx1234567890abcdef"

# SecretRef 方式（安全引用）
token: "${secret:discord-token}"
```

**工作流程：**
1. 配置文件只包含 `${secret:key-name}` 格式的引用
2. OpenClaw 启动时从安全后端获取实际密钥
3. 密钥在内存中使用，不持久化到配置文件
4. 支持运行时重载，无需重启 Gateway

### 2.3 支持的存储后端

| 后端 | 类型 | 适用场景 |
|-----|------|---------|
| `keychain` | 系统钥匙串 | macOS 用户首选 |
| `pass` | Password Store | Linux 用户首选 |
| `file` | 加密文件 | 跨平台兼容 |
| `env` | 环境变量 | 容器化部署 |
| `vault` | HashiCorp Vault | 企业级部署 |

## 三、实战迁移操作

### 3.1 准备工作

**第一步：备份当前配置**

```bash
# 创建配置备份目录
mkdir -p ~/.openclaw/backups

# 备份关键配置文件
cp ~/.openclaw/openclaw.json ~/.openclaw/backups/openclaw.json.bak.$(date +%Y%m%d_%H%M%S)
cp ~/.openclaw/agents/main/agent/auth-profiles.json ~/.openclaw/backups/auth-profiles.json.bak.$(date +%Y%m%d_%H%M%S)

# 备份博客仓库的 git 配置
cd ~/.openclaw/workspace/duranblog
git remote get-url origin > ~/.openclaw/backups/git-remote-backup.txt

echo "✅ 备份完成"
```

**第二步：检查当前密钥状态**

```bash
# 审计检查
openclaw secrets audit
```

**实际输出示例：**
```
Secrets audit: findings. plaintext=2, unresolved=0, shadowed=0, legacy=1.
- [PLAINTEXT_FOUND] /home/warwick/.openclaw/openclaw.json:channels.discord.token channels.discord.token is stored as plaintext.
- [LEGACY_RESIDUE] /home/warwick/.openclaw/agents/main/agent/auth-profiles.json:profiles.qwen-portal:default OAuth credentials are present (out of scope for static SecretRef migration).
- [PLAINTEXT_FOUND] /home/warwick/.openclaw/agents/main/agent/auth-profiles.json:profiles.kimi-coding:default.key Auth profile API key is stored as plaintext.
```

**审计结果解读：**

| 发现项 | 位置 | 状态 | 说明 |
|--------|------|------|------|
| Discord Token | `openclaw.json` | 🔴 明文 | 需要迁移 |
| Kimi API Key | `auth-profiles.json` | 🔴 明文 | 需要迁移 |
| Qwen OAuth | `auth-profiles.json` | 🟡 遗留 | OAuth 不支持静态迁移 |

**注意：** `secrets audit` 不会检查 git remote 中的 GitHub PAT，需要手动确认：

```bash
# 检查 git remote 是否包含明文 PAT
cd ~/.openclaw/workspace/duranblog
git remote -v
```

如果输出包含 `https://用户名:ghp_xxx@github.com/...`，说明 PAT 是明文存储的。

### 3.2 手动迁移剩余密钥

**如果 OpenClaw 2026.3.2 升级后仍有明文密钥（auth-profiles.json 中的 API Keys 和 git remote 中的 PAT），按以下步骤手动迁移：**

#### 迁移 auth-profiles.json 中的 API Keys

**第一步：检查当前明文密钥**

```bash
# 查看 auth-profiles.json 中的明文 API Key
cat ~/.openclaw/agents/main/agent/auth-profiles.json | grep -E '"key":\s*"sk-'
```

**第二步：更新 gateway.env 文件**

将 API Key 添加到环境变量文件：

```bash
# 编辑 env 文件
vim ~/.openclaw/secrets/gateway.env
```

添加 Kimi API Key：
```bash
# 已有的 Discord Token
DISCORD_TOKEN=MTQ2Njc4MDY2NzgwNjIyMDM2NA.GvboSs.xxx

# 添加 Kimi API Key（从 auth-profiles.json 中复制）
KIMI_API_KEY=sk-kimi-xxx
```

**第三步：修改 auth-profiles.json 使用 SecretRef**

编辑 `~/.openclaw/agents/main/agent/auth-profiles.json`：

```json
{
  "profiles": {
    "kimi-coding:default": {
      "type": "api_key",
      "provider": "kimi-coding",
      "key": "${env:KIMI_API_KEY}"
    }
  }
}
```

**第四步：重载配置**

```bash
openclaw secrets reload
```

#### 迁移 Git Remote 中的 GitHub PAT

**第一步：备份当前 remote URL**

```bash
cd ~/.openclaw/workspace/duranblog
git remote get-url origin > ~/.openclaw/backups/git-remote-backup.txt
```

**第二步：添加 GitHub PAT 到 env 文件**

```bash
# 编辑 env 文件
vim ~/.openclaw/secrets/gateway.env
```

添加 GitHub PAT：
```bash
DISCORD_TOKEN=MTQ2Njc4MDY2NzgwNjIyMDM2NA.GvboSs.xxx
KIMI_API_KEY=sk-kimi-xxx
GITHUB_PAT=ghp_xxx
```

**第三步：更新 git remote URL**

```bash
cd ~/.openclaw/workspace/duranblog

# 更新为使用 env 变量
git remote set-url origin 'https://openduran:${env:GITHUB_PAT}@github.com/openduran/duranblog.git'

# 验证
git remote -v
```

**注意**：git 本身不支持 `${env:...}` 语法，需要通过 credential helper 或手动配置 git 来支持。

**替代方案（推荐）**：使用 Git Credential Manager 或手动输入密码：

```bash
# 移除 URL 中的密码
git remote set-url origin 'https://openduran@github.com/openduran/duranblog.git'

# 配置 git 缓存密码
git config --global credential.helper cache
```

**第四步：重启 Gateway**

```bash
openclaw gateway restart
```

### 3.3 使用交互式配置向导（可选）

对于更复杂的场景，可以使用交互式配置向导：

```bash
openclaw secrets configure
```

**注意**：交互式向导适合初次配置或需要添加新的 provider（如 file/exec），对于已有 env provider 的情况，直接编辑配置文件更简单。

---

**第三步：启动配置向导**

```bash
openclaw secrets configure
```

**配置流程（完整步骤）：**

**1. 初始界面**
```
Configure secret providers (only env refs are available until file/exec providers are added)

● Add provider (Define a new env/file/exec provider)

Select: 1  # 选择 Add provider
```

**2. 选择 Provider Source**
```
Provider source

1) env  - Environment variables
2) file - Read from file (JSON or single-value)
3) exec - Execute command and read stdout

Select: 2  # 选择 file
```

**3. 配置 File Provider**
```
File path (absolute): /home/warwick/.openclaw/secrets/credentials.json

File mode

1) json        - Read multiple key-value pairs from JSON file
2) singleValue - Read a single value from file

Select: 1  # 选择 json

Timeout ms (blank for default):    # 按回车

Max bytes (blank for default):     # 按回车
```

**4. 回到 Provider 配置界面**
```
Configure secret providers

● Continue (Continue to credential field mapping)

Select: 1  # 选择 Continue
```

**5. 选择要配置的凭证字段**
```
Select credential field

● discord-token       (openclaw.json)
● kimi-api-key        (auth-profiles.json)
○ Create auth profile mapping
○ Done

Select: 1  # 选择 discord-token
```

**6. 配置 Secret 引用**
```
Secret source

1) file

Select: 1  # 选择 file

Provider alias: default  # 输入或确认 provider 别名

Secret id: discord-token  # 输入密钥在 JSON 文件中的 key
```

此时会验证引用是否能解析到值。

**7. 继续配置其他字段**
```
Configure another credential? (Y/n): Y

# 重复步骤 5-6，选择 kimi-api-key 等其他字段
```

**8. 完成配置**
```
Select credential field

○ discord-token       (configured)
○ kimi-api-key        (configured)
● Done                (Finish and run preflight)

Select: 3  # 选择 Done
```

**9. Preflight 检查和 Apply**
```
Preflight: changed=true, files=2, warnings=0.
Plan: targets=2, providerUpserts=1, providerDeletes=0.

Apply this plan now? (Y/n): Y

This migration is one-way for migrated plaintext values. Continue with apply? (Y/n): Y

Secrets applied. Updated 2 file(s).
```

**注意**：
- 配置向导会自动创建 JSON 密钥文件并写入密钥值
- 如果过程中出错，可以使用 `--plan-out` 参数生成计划文件检查
- 或者在出错时选择不 apply，然后使用手动配置方式

**当前版本限制**：
- 仅支持 env/file/exec 三种 provider，不支持 keychain/pass/vault
- 配置流程分为两个阶段：先配置 provider，再配置密钥映射

### 3.4 验证迁移结果

**验证配置**

```bash
# 检查配置文件（应显示引用而非明文）
cat ~/.openclaw/openclaw.json | grep -A2 '"discord"'
```

预期输出：
```json
"discord": {
  "token": "${env:DISCORD_TOKEN}",
  "enabled": true
}
```

**注意**：根据使用的 provider 不同，引用格式也不同：
- `${env:VAR_NAME}` - 环境变量 provider（OpenClaw 自动迁移使用）
- `${file:/path/to/file:key}` - 文件 provider（手动配置使用）

**功能测试**

```bash
# 测试 Discord 连接
openclaw message send --channel discord --to "channel:ID" --message "测试消息"

# 测试 AI 功能
openclaw chat "你好"

# 测试 Git 推送
cd ~/.openclaw/workspace/duranblog
git push origin main --dry-run
```

## 四、GitHub PAT 特殊处理

### 4.1 更新 Git Remote URL

手动配置时，使用 file provider 格式更新 remote：

```bash
cd ~/.openclaw/workspace/duranblog

# 查看当前 remote
git remote -v

# 更新为引用格式（使用 file provider）
git remote set-url origin 'https://openduran:${file:/home/warwick/.openclaw/secrets/credentials.json:github-pat}@github.com/openduran/duranblog.git'

# 验证
git remote -v
```
```

### 4.2 配置 Git Credential Helper

为了让 git 能够解析 SecretRef，需要配置自定义 credential helper：

```bash
# 配置 git 使用 OpenClaw 作为 credential helper
git config --global credential.helper '!openclaw secrets git-credential'
```

## 五、后续维护

### 5.1 定期审计

```bash
# 每月执行一次审计
openclaw secrets audit
```

### 5.2 密钥轮换

当密钥泄露或需要轮换时：

**如果是 env provider（OpenClaw 自动迁移的格式）：**

```bash
# 1. 直接编辑 env 文件更新密钥值
vim ~/.openclaw/secrets/gateway.env

# 2. 重载配置
openclaw secrets reload

# 3. 验证
openclaw gateway status
```

**如果是手动配置的 file provider：**

```bash
# 1. 编辑密钥文件
vim ~/.openclaw/secrets/credentials.json

# 2. 重载配置
openclaw secrets reload

# 3. 验证
openclaw gateway status
```

### 5.3 安全删除旧配置备份

迁移完成后，旧备份文件仍包含明文密钥，建议安全删除：

```bash
# 安全删除（覆盖后删除）
shred -u ~/.openclaw/backups/openclaw.json.bak.*
shred -u ~/.openclaw/backups/auth-profiles.json.bak.*
```

## 六、常见问题

### Q1: 迁移后启动报错 "无法解析 SecretRef"

**原因：** 密钥文件路径错误或格式不正确

**解决：**
```bash
# 检查密钥文件是否存在
ls -la ~/.openclaw/secrets/credentials.json

# 检查 JSON 格式是否正确
jq . ~/.openclaw/secrets/credentials.json

# 检查文件权限（应为 600）
chmod 600 ~/.openclaw/secrets/credentials.json

# 检查配置文件中的引用路径是否正确
grep "file:" ~/.openclaw/openclaw.json

# 重载配置
openclaw secrets reload
```

### Q2: OAuth 凭证如何处理？

OAuth 凭证（如 Qwen）不支持静态迁移，因为：
- OAuth Token 会定期过期刷新
- 需要动态获取而非静态存储

**建议：** 继续使用 OAuth 原生流程，不要尝试静态迁移。

### Q3: 如何查看当前有哪些 SecretRef？

```bash
# 检查配置文件中的引用
grep -r "\${file:" ~/.openclaw/

# 或者查看所有 SecretRef 引用
grep -rE '\$\{(file|env|exec|secret):' ~/.openclaw/
```

### Q4: 如何回滚到明文配置？

```bash
# 使用备份恢复
cp ~/.openclaw/backups/openclaw.json.bak.xxx ~/.openclaw/openclaw.json
cp ~/.openclaw/backups/auth-profiles.json.bak.xxx ~/.openclaw/agents/main/agent/auth-profiles.json

# 重启 Gateway
openclaw gateway restart
```

## 七、总结

### 迁移前后对比

| 维度 | 迁移前 | 迁移后 |
|-----|-------|-------|
| 存储方式 | 明文存储 | 分离存储（密钥文件 + 配置引用） |
| 安全风险 | 高（配置文件泄露=密钥泄露） | 低（配置文件只含引用） |
| 审计合规 | 不满足 | 满足 |
| 密钥轮换 | 需修改多处 | 只需更新密钥文件 |
| 团队协作 | 密钥共享困难 | 可安全共享配置（不含密钥） |

### 核心优势

✅ **安全**：密钥不再明文存储在配置文件中  
✅ **灵活**：支持多种存储后端，适应不同环境  
✅ **便捷**：运行时重载，无需重启服务  
✅ **合规**：满足企业安全审计要求  

---

**参考文档：**
- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [SecretRef 设计文档](https://github.com/openclaw/openclaw/blob/main/docs/secrets.md)
- [Git Credential Helper](https://git-scm.com/docs/gitcredentials)

**本文环境：**
- OpenClaw: 2026.3.2
- OS: Debian 13
- Date: 2026-03-03
