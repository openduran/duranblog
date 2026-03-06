---
title: "OpenClaw Discord Bot 配置指南：解决 401 错误和离线问题"
date: 2026-03-05T20:30:00+08:00
draft: true
tags: ["openclaw", "discord", "bot", " troubleshooting"]
---

## 前言

本文基于 OpenClaw 2026.3.2 实际测试，记录从配置 Discord Bot 到解决常见问题的完整过程。

## 一、检查当前 Discord 配置状态

```bash
openclaw status --deep
```

**正常状态：**
```
│ Discord  │ ON      │ OK     │ token config (${env:DISCORD_BOT_TOKEN}) │
```

**常见问题 1：401 Unauthorized**
```
│ Discord  │ WARN    │ failed (401) - getMe failed (401)            │
```

**原因：** Token 无效或过期

**解决：**
1. 访问 https://discord.com/developers/applications
2. 选择你的 Application → Bot → Reset Token
3. 复制新 Token
4. 更新环境变量：

```bash
# 编辑 systemd 环境变量文件
vim ~/.openclaw/secrets/gateway.env

# 添加或更新
DISCORD_BOT_TOKEN=你的新Token
```

5. 重启服务：
```bash
systemctl --user restart openclaw-gateway
```

---

**常见问题 2：连接成功但 Bot 离线**

`openclaw status` 显示 OK，但 Discord 里看不到 Bot 在线。

**原因：** Privileged Gateway Intents 未启用

**解决：**

1. 访问 https://discord.com/developers/applications
2. 选择你的 Application → **Bot** 标签页
3. 找到 **Privileged Gateway Intents**，全部开启：
   - ✅ **PRESENCE INTENT**
   - ✅ **SERVER MEMBERS INTENT**  
   - ✅ **MESSAGE CONTENT INTENT**

4. 保存后等待几秒，Bot 应该会显示在线

## 二、OpenClaw 配置结构

### 2.1 配置文件位置

```
~/.openclaw/openclaw.json          # 主配置
~/.openclaw/secrets/gateway.env    # 环境变量（Discord Token 等）
```

### 2.2 Discord 配置示例

`openclaw.json` 中的 Discord 配置：

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "${env:DISCORD_BOT_TOKEN}",
      "groupPolicy": "allowlist",
      "guilds": {
        "你的服务器ID": {
          "channels": {
            "频道ID": { "allow": true }
          }
        }
      }
    }
  }
}
```

**注意：** OpenClaw 只支持 `${env:VAR_NAME}` 格式引用环境变量。

### 2.3 systemd 服务配置

查看当前服务配置：

```bash
systemctl --user cat openclaw-gateway
```

关键部分：
```ini
[Service]
EnvironmentFile=/home/warwick/.openclaw/secrets/gateway.env
ExecStart=/usr/bin/node /path/to/openclaw/dist/index.js gateway
```

`EnvironmentFile` 指定了环境变量文件路径，这是 Token 被加载的方式。

## 三、环境变量文件格式

`~/.openclaw/secrets/gateway.env`：

```bash
# 注释以 # 开头
DISCORD_BOT_TOKEN=MTQ2Njc4MDY2NzgwNjIyMDM2NA.xxx.xxx
KIMI_API_KEY=sk-kimi-xxx
```

**要求：**
- 纯文本格式，一行一个变量
- `KEY=VALUE` 格式，不需要引号
- 文件权限建议设为 600：`chmod 600 ~/.openclaw/secrets/gateway.env`

## 四、完整配置流程

### 步骤 1：获取 Discord Bot Token

1. https://discord.com/developers/applications → New Application
2. 左侧 **Bot** → Add Bot
3. Reset Token → 复制（只显示一次！）
4. 开启所有 **Privileged Gateway Intents**

### 步骤 2：邀请 Bot 到服务器

1. OAuth2 → URL Generator
2. Scopes: `bot`
3. Bot Permissions: 至少勾选 `Send Messages`, `Read Message History`
4. 复制生成的 URL，在浏览器中打开，选择服务器添加

### 步骤 3：配置 OpenClaw

```bash
# 1. 创建环境变量文件
mkdir -p ~/.openclaw/secrets
cat > ~/.openclaw/secrets/gateway.env << 'EOF'
DISCORD_BOT_TOKEN=你的Token
EOF

chmod 600 ~/.openclaw/secrets/gateway.env

# 2. 确保 openclaw.json 使用 env 引用
cat ~/.openclaw/openclaw.json | grep '"token"'
# 应该显示: "token": "${env:DISCORD_BOT_TOKEN}"

# 3. 重启服务
systemctl --user restart openclaw-gateway

# 4. 验证
openclaw status --deep
```

### 步骤 4：测试

```bash
# 发送测试消息
openclaw message send --channel discord --to "channel:频道ID" --message "Hello from OpenClaw!"
```

## 五、故障排查

### 5.1 检查 Token 是否正确加载

```bash
# 查看 Gateway 日志
journalctl --user -u openclaw-gateway -n 50

# 查找 401 错误
journalctl --user -u openclaw-gateway | grep "401"
```

### 5.2 检查环境变量

```bash
# 查看进程环境变量
cat /proc/$(pgrep -f "openclaw-gateway")/environ | tr '\0' '\n' | grep DISCORD
```

如果为空，说明 EnvironmentFile 未正确加载。

### 5.3 手动验证 Token

```bash
# 直接测试 Token 是否有效
curl -H "Authorization: Bot 你的Token" \
  https://discord.com/api/v10/users/@me
```

成功应返回 bot 的用户信息，失败返回 401。

## 六、总结

| 问题 | 现象 | 解决 |
|-----|------|------|
| Token 无效 | 401 Unauthorized | 重置 Token 并更新 gateway.env |
| Intents 未开 | 连接成功但离线 | 开启 Privileged Gateway Intents |
| 未加入服务器 | 无法发送消息 | 通过 OAuth2 URL 邀请 Bot |
| 权限不足 | 消息发送失败 | 检查 Bot Permissions |

**关键要点：**
- OpenClaw 使用 `${env:VAR}` 引用环境变量
- Token 通过 systemd `EnvironmentFile` 加载
- Discord Bot 需要正确的 Intents 才能正常工作

---

**本文环境：**
- OpenClaw: 2026.3.2
- OS: Debian 13
- Date: 2026-03-05

**参考：**
- OpenClaw 官方文档: https://docs.openclaw.ai
- Discord Developer Portal: https://discord.com/developers/applications
