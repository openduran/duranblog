---
title: "OpenClaw + Tailscale 远程访问指南：安全暴露 Gateway 的两种方式"
date: 2026-03-06T09:00:00+08:00
draft: false
tags: ["openclaw", "tailscale", "tailscale-funnel", "remote-access", "security"]
---

## 前言

OpenClaw Gateway 默认只在本地运行（`127.0.0.1:18789`），这意味着：
- ✅ 安全：外部无法直接访问
- ❌ 局限：只能在本地使用，无法远程控制

如果你希望：
- **在家里的服务器运行 OpenClaw，用手机远程访问**
- **团队协作时共享一个 OpenClaw 实例**
- **出门在外时仍能使用家里的 AI 助手**

那么 **Tailscale** 集成是你的最佳选择。

---

## 一、Tailscale 是什么？

[Tailscale](https://tailscale.com/) 是一个基于 WireGuard 的零配置 VPN 工具，它让你可以轻松构建私有网络（Tailnet），将任意设备安全地连接在一起。

### 核心优势

| 特性 | 说明 |
|------|------|
| **零配置** | 无需配置防火墙规则、端口转发 |
| **端到端加密** | WireGuard 协议，安全可靠 |
| **跨平台** | Linux、macOS、Windows、iOS、Android 全支持 |
| **免费额度** | 个人用户免费，最多 20 台设备 |

### Tailscale 的两种模式

OpenClaw 支持 Tailscale 的两种工作模式：

1. **`tailscale serve`** - 仅限 Tailnet 内部访问（私有）
2. **`tailscale funnel`** - 公共互联网可访问（公开，需密码保护）

---

## 二、OpenClaw + Tailscale 能做什么？

### 场景 1：Tailscale Serve（推荐个人使用）

**适用场景：**
- 在家里的 NAS/服务器运行 OpenClaw
- 手机、笔记本通过 Tailscale 远程连接
- 只有你自己的设备能访问

**网络拓扑：**
```
[手机] ←──Tailnet──→ [Tailscale] ←──localhost──→ [OpenClaw Gateway]
[笔记本] ←──加密隧道──→ 192.168.x.x:18789
```

### 场景 2：Tailscale Funnel（需要公共访问）

**适用场景：**
- 团队协作，多人共享一个 OpenClaw 实例
- 需要在没有安装 Tailscale 的设备上临时访问
- 通过公共 URL 访问（如 `https://your-machine.tailnet-xx.ts.net`）

**⚠️ 安全警告：**
- Funnel 会将服务暴露到公共互联网
- **必须启用密码认证**，否则任何人都能访问你的 Gateway
- 建议配合 `gateway.auth.mode: "password"` 使用

---

## 三、配置步骤

### 前置条件

1. **安装 Tailscale**
   ```bash
   # Debian/Ubuntu
   curl -fsSL https://tailscale.com/install.sh | sh
   
   # macOS
   brew install tailscale
   
   # 其他系统见官方文档
   ```

2. **登录 Tailscale**
   ```bash
   sudo tailscale up
   # 按提示在浏览器中完成授权
   ```

3. **确认设备已获得 Tailscale IP**
   ```bash
   tailscale ip -4
   # 输出类似：100.x.y.z
   ```

### 配置 OpenClaw

编辑 `~/.openclaw/openclaw.json`：

#### 方案 A：Tailscale Serve（私有访问）

```json
{
  "gateway": {
    "port": 18789,
    "mode": "tailscale",
    "auth": {
      "mode": "token",
      "token": "your-secure-token-here"
    },
    "tailscale": {
      "mode": "serve",
      "resetOnExit": false
    }
  }
}
```

**访问方式：**
- 只有安装了 Tailscale 并登录同一账户的设备可以访问
- 使用 `http://your-hostname.tailnet-xx.ts.net:18789`

#### 方案 B：Tailscale Funnel（公共访问）

```json
{
  "gateway": {
    "port": 18789,
    "mode": "tailscale",
    "auth": {
      "mode": "password",
      "password": "your-strong-password-here"
    },
    "tailscale": {
      "mode": "funnel",
      "resetOnExit": true
    }
  }
}
```

**⚠️ 必须设置密码！** Funnel 模式拒绝无密码配置。

**访问方式：**
- 任何设备通过 `https://your-hostname.tailnet-xx.ts.net` 访问
- 需要输入密码认证

### 重启 Gateway

```bash
openclaw gateway restart
# 或
systemctl --user restart openclaw-gateway.service
```

### 验证连接

**从另一台已安装 Tailscale 的设备：**
```bash
# 测试连接
curl http://your-hostname.tailnet-xx.ts.net:18789/status

# 或使用浏览器访问 Dashboard
http://your-hostname.tailnet-xx.ts.net:18789
```

---

## 四、安全最佳实践

### 1. 优先使用 Serve 模式

除非确实需要公共访问，否则始终使用 `serve` 模式。

### 2. Funnel 必须使用强密码

```bash
# 生成强密码
openssl rand -base64 32
```

### 3. 限制设备访问（可选）

在 Tailscale ACL 中限制哪些设备可以访问 OpenClaw 端口：

```json
// tailnet policy file (admin console)
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["tag:openclaw:18789"]
    }
  ]
}
```

### 4. 启用退出时重置

```json
"tailscale": {
  "mode": "funnel",
  "resetOnExit": true
}
```

这样 Gateway 停止时会自动关闭 Funnel，防止意外暴露。

### 5. 定期轮换 Token/密码

```bash
# 设置新密码
openclaw config set gateway.auth.password "$(openssl rand -base64 24)"
openclaw gateway restart
```

---

## 五、常见问题

### Q1: Tailscale 和本地模式有什么区别？

| 对比项 | 本地模式 (`bind: loopback`) | Tailscale Serve | Tailscale Funnel |
|--------|---------------------------|-----------------|------------------|
| 访问范围 | 仅限本机 | Tailnet 内设备 | 公共互联网 |
| 加密 | 无 | WireGuard | WireGuard + TLS |
| 需要 Tailscale | 否 | 是 | 是 |
| 需要密码 | 可选 | 可选 | **必须** |
| 适用场景 | 单机使用 | 多设备私有访问 | 团队协作/公共访问 |

### Q2: 可以同时使用本地和 Tailscale 吗？

不可以。OpenClaw Gateway 只能绑定一种模式：
- `bind: loopback` → 本地访问
- `mode: tailscale` → Tailscale 访问

如果需要同时支持，建议：
1. 使用 Tailscale Serve 模式
2. 本地设备也安装 Tailscale

### Q3: Funnel 模式提示 "refusing to start without password"

这是安全设计。编辑配置：
```json
"auth": {
  "mode": "password",
  "password": "your-password"
}
```

### Q4: 如何查看 Tailscale 分配的域名？

```bash
tailscale status
# 查看 "DNS name" 字段
```

### Q5: 手机如何连接？

1. 安装 Tailscale App
2. 登录同一账户
3. 开启 VPN
4. 使用 `http://hostname:18789` 访问

---

## 六、配置示例汇总

### 个人私有访问（推荐）

```json
{
  "gateway": {
    "port": 18789,
    "mode": "tailscale",
    "auth": {
      "mode": "token",
      "token": "${env:OPENCLAW_GATEWAY_TOKEN}"
    },
    "tailscale": {
      "mode": "serve",
      "resetOnExit": false
    }
  },
  "channels": {
    "discord": {
      "enabled": true,
      "token": "${env:DISCORD_BOT_TOKEN}"
    }
  }
}
```

### 团队协作（Funnel + 密码）

```json
{
  "gateway": {
    "port": 18789,
    "mode": "tailscale",
    "auth": {
      "mode": "password",
      "password": "${env:OPENCLAW_GATEWAY_PASSWORD}"
    },
    "tailscale": {
      "mode": "funnel",
      "resetOnExit": true
    }
  }
}
```

环境变量文件 `~/.openclaw/.env`：
```bash
OPENCLAW_GATEWAY_TOKEN=your-secure-random-token
OPENCLAW_GATEWAY_PASSWORD=your-strong-password
```

---

## 七、总结

| 需求 | 推荐方案 |
|------|----------|
| 仅本地使用 | `bind: loopback`（默认） |
| 多设备私有访问 | `tailscale: serve` |
| 团队协作/公共访问 | `tailscale: funnel` + 密码 |

Tailscale 让 OpenClaw 的远程访问变得简单安全，无需配置防火墙、无需端口转发，几分钟即可完成部署。

**下一步：**
- [Tailscale 官方文档](https://tailscale.com/kb/)
- [OpenClaw Gateway 配置参考](https://docs.openclaw.ai/gateway/configuration)

---

**本文环境：**
- OpenClaw: 2026.3.2
- Tailscale: 1.80.x
- OS: Debian 13
