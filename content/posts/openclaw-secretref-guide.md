---
title: "OpenClaw API 密钥管理完全指南：从明文到 SecretRef"
date: 2026-03-03T22:00:00+08:00
draft: true
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

### 1.2 明文存储的安全隐患

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

预期输出示例：
```
🔍 审计结果：
  - Discord Token: 明文存储于 openclaw.json
  - Kimi API Key: 明文存储于 auth-profiles.json  
  - GitHub PAT: 明文存储于 git remote URL
  - Qwen OAuth: 遗留凭证（OAuth 不支持静态迁移）

建议：使用 'openclaw secrets configure' 开始迁移
```

### 3.2 交互式迁移配置

**第三步：启动配置向导**

```bash
openclaw secrets configure --output /tmp/openclaw-secrets-plan.json
```

**配置流程：**

1. **选择存储后端**
   ```
   选择密钥存储后端：
   1) keychain    - macOS 系统钥匙串
   2) pass        - Linux Password Store
   3) file        - 加密文件（通用）
   4) env         - 环境变量
   5) vault       - HashiCorp Vault
   
   选择 [1-5]: 3  # 这里选择 file（跨平台兼容）
   ```

2. **配置加密文件存储**
   ```
   加密文件存储路径 [~/.openclaw/secrets.enc]:
   加密密码（留空生成随机密码）: ********
   确认密码: ********
   ```

3. **选择要迁移的密钥**
   ```
   发现以下明文密钥：
   
   [✓] Discord Token (channels.discord.token)
   [✓] Kimi API Key (auth-profiles.kimi-coding.key)
   [✓] GitHub PAT (git.remote.origin)
   [ ] Qwen OAuth (OAuth 不支持静态迁移)
   
   选择要迁移的密钥（空格选择，回车确认）: 
   ```

4. **生成迁移计划**
   
   配置向导会生成 `/tmp/openclaw-secrets-plan.json`：
   ```json
   {
     "backend": {
       "type": "file",
       "path": "/home/warwick/.openclaw/secrets.enc"
     },
     "secrets": [
       {
         "name": "discord-token",
         "value": "MTQ2Njc4MDY2NzgwNjIyMDM2NA.GvboSs.xxx",
         "target": "openclaw.json:channels.discord.token"
       },
       {
         "name": "kimi-api-key",
         "value": "sk-kimi-xxx",
         "target": "auth-profiles.json:kimi-coding.default.key"
       },
       {
         "name": "github-pat",
         "value": "ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
         "target": "git.remote.origin"
       }
     ]
   }
   ```

### 3.3 应用迁移计划

**第四步：预览迁移（dry-run）**

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
```

输出示例：
```
🔍 迁移预览（dry-run）：

文件: ~/.openclaw/openclaw.json
  变更: channels.discord.token
  旧值: "MTQ6...9KHm" (明文)
  新值: "${secret:discord-token}" (引用)

文件: ~/.openclaw/agents/main/agent/auth-profiles.json  
  变更: kimi-coding.default.key
  旧值: "sk-kimi-xxx" (明文)
  新值: "${secret:kimi-api-key}" (引用)

Git Remote:
  变更: origin
  旧值: "https://openduran:ghp_xxx@github.com/..."
  新值: "https://openduran:${secret:github-pat}@github.com/..."

⚠️ 注意：此操作不会创建回滚备份
建议手动备份重要配置
```

**第五步：正式应用**

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --yes
```

**预期输出：**
```
✅ 密钥已安全存储
✅ 配置文件已更新
✅ 共迁移 3 个密钥

下一步：运行 'openclaw secrets reload' 激活新配置
```

### 3.4 激活新配置

**第六步：运行时重载（无需重启）**

```bash
openclaw secrets reload
```

输出示例：
```
🔄 重载密钥快照...
✅ 已加载 3 个密钥引用
✅ Discord Token: 已解析
✅ Kimi API Key: 已解析  
✅ GitHub PAT: 已解析

配置已生效，无需重启 Gateway
```

### 3.5 验证迁移结果

**第七步：验证配置**

```bash
# 检查配置文件（应显示引用而非明文）
cat ~/.openclaw/openclaw.json | grep -A2 '"discord"'
```

预期输出：
```json
"discord": {
  "token": "${secret:discord-token}",
  "enabled": true
}
```

```bash
# 检查密钥存储
cat ~/.openclaw/secrets.enc | head -c 100
echo "..."
```

预期输出：
```
U2FsdGVkX1+7J8v2...  (加密内容，无法直接读取)
```

**第八步：功能测试**

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

迁移后，需要更新 git remote URL 使用引用格式：

```bash
cd ~/.openclaw/workspace/duranblog

# 查看当前 remote
git remote -v
# 输出：origin https://openduran:ghp_xxx@github.com/openduran/duranblog.git

# 更新为引用格式
git remote set-url origin 'https://openduran:${secret:github-pat}@github.com/openduran/duranblog.git'

# 验证
git remote -v
# 输出：origin https://openduran:${secret:github-pat}@github.com/openduran/duranblog.git
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

```bash
# 1. 更新存储的密钥值
openclaw secrets set discord-token "新token值"

# 2. 重载配置
openclaw secrets reload

# 3. 验证
openclaw gateway status
```

### 5.3 清理计划文件

迁移完成后，安全删除计划文件：

```bash
# 安全删除（覆盖后删除）
shred -u /tmp/openclaw-secrets-plan.json

# 或者移动到加密存储
mv /tmp/openclaw-secrets-plan.json ~/.openclaw/backups/
chmod 600 ~/.openclaw/backups/openclaw-secrets-plan.json
```

## 六、常见问题

### Q1: 迁移后启动报错 "无法解析 SecretRef"

**原因：** 存储后端配置错误或密钥未正确存储

**解决：**
```bash
# 检查存储后端状态
openclaw secrets backend status

# 重新存储密钥
openclaw secrets set discord-token "实际token值"

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
# 列出所有已配置的密钥引用
openclaw secrets list

# 输出示例：
# discord-token    - 最后更新: 2026-03-03 22:00
# kimi-api-key     - 最后更新: 2026-03-03 22:00
# github-pat       - 最后更新: 2026-03-03 22:00
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
| 存储方式 | 明文存储 | 加密存储 + 引用 |
| 安全风险 | 高（配置文件泄露=密钥泄露） | 低（配置文件只含引用） |
| 审计合规 | 不满足 | 满足 |
| 密钥轮换 | 需修改多处 | 只需更新一处 |
| 团队协作 | 密钥共享困难 | 可安全共享配置 |

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
