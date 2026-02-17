---
title: "从零搭建 Hugo 博客：Vercel + Cloudflare 完整实战记录"
date: 2026-02-17T20:50:00+08:00
draft: false
tags: ["hugo", "vercel", "cloudflare", "github", "博客搭建", "教程"]
categories: ["技术教程"]
description: "记录从零开始搭建 Hugo 博客的完整过程，包括技术选型、遇到的问题及解决方案，涵盖 Vercel 部署、Cloudflare DNS 配置、SSL 问题排查等实战经验。"
---

## 前言

今天花了大约 8 小时，从零开始搭建了这个博客。记录一下完整过程，包括技术选型、踩过的坑和解决方案。希望能帮到想搭建自己博客的朋友。

## 技术栈介绍

### Hugo - 静态网站生成器

[Hugo](https://gohugo.io/) 是用 Go 语言编写的静态网站生成器，号称"世界上最快的静态网站生成器"。

**优点：**
- ⚡ 极速构建（每秒生成数千页面）
- 🎨 主题丰富（官方主题库 300+）
- 📝 支持 Markdown
- 🔧 单二进制文件，单文件部署

**缺点：**
- 主题版本和 Hugo 版本可能不兼容
- 需要一定学习成本

### GitHub - 代码托管

[GitHub](https://github.com) 用于托管博客源码，配合 Git 版本控制。

**作用：**
- 代码版本管理
- 文章 Markdown 文件存储
- 与 Vercel 集成实现自动部署

### Vercel - 静态网站托管

[Vercel](https://vercel.com) 是前端部署平台，对静态网站支持极好。

**优点：**
- 🚀 自动部署（Git 推送即部署）
- 🌍 全球 CDN 加速
- 🆓 免费版足够个人博客使用
- 🔒 自动 HTTPS

**注意点：**
- Deployment Protection 默认可能开启（需要关闭才能公开访问）
- 需要正确配置 Hugo 版本环境变量

### Cloudflare - DNS + CDN

[Cloudflare](https://cloudflare.com) 提供 DNS 解析和 CDN 加速服务。

**作用：**
- 域名 DNS 管理
- SSL/TLS 证书（自动）
- DDoS 防护
- 全球 CDN 加速

## 搭建过程实录

### 第一步：购买域名

选择了 `d5n.xyz`，寓意 Duran（5个字母）+ N。

**踩坑提醒：**
- 3字母 `.com` 域名基本都是溢价域名（$1000+）
- `.xyz` 首年便宜（$1-3），但注意续费价格
- Cloudflare 可以直接注册域名，成本价无溢价

### 第二步：初始化 Hugo 站点

```bash
hugo new site duranblog
cd duranblog
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

**配置 hugo.toml：**
```toml
baseURL = 'https://d5n.xyz'
languageCode = 'zh-CN'
title = 'D5N'
theme = 'PaperMod'
```

### 第三步：创建 GitHub 仓库

- 仓库名：`duranblog`
- 类型：Public（Vercel 免费版对 Public 仓库无限制）
- 添加 README 初始化

### 第四步：推送代码到 GitHub

**问题 1：Git 认证失败**

错误信息：
```
fatal: could not read Username for 'https://github.com'
```

**解决方案：**
使用 Personal Access Token 认证：
```bash
git remote set-url origin https://openduran:TOKEN@github.com/openduran/duranblog.git
```

### 第五步：Vercel 部署

#### 问题 2：部署后显示 HTML 源码

现象：访问网站时浏览器显示原始 HTML 代码，而不是渲染后的网页。

**排查过程：**
1. 检查 GitHub 仓库，发现提交了 `public/` 目录
2. Vercel 有自动构建功能，不需要提交构建后的文件
3. 删除 `public/` 目录，添加 `.gitignore`

**解决方案：**
```bash
rm -rf public/
echo "public/" >> .gitignore
git add . && git commit -m "Remove public dir" && git push
```

#### 问题 3：Hugo 版本不兼容

错误信息：
```
WARN  Module "PaperMod" is not compatible with this Hugo version: Min 0.146.0
ERROR render of "/404" failed
```

**原因：** Vercel 默认 Hugo 版本太旧，PaperMod 主题需要 0.146.0+

**解决方案：**
在 Vercel 项目设置中添加环境变量：
- **Name**: `HUGO_VERSION`
- **Value**: `0.146.5`

#### 问题 4：访问需要登录（401 错误）

现象：访问网站提示 "Vercel Authentication"

**解决方案：**
1. 进入 Vercel 项目 Settings → General
2. 找到 "Deployment Protection"
3. 改为 "Disabled"
4. 保存后重新部署

### 第六步：配置 Cloudflare DNS

#### 问题 5：SSL 握手失败（525 错误）

错误信息：
```
525: SSL handshake failed
```

**原因：** Cloudflare SSL 模式和 Vercel 不兼容

**解决方案：**
1. 进入 Cloudflare → SSL/TLS → Overview
2. 将模式从 "Flexible" 改为 "Full" 或 "Full (strict)"

#### 问题 6：根域名无法访问

现象：`www.d5n.xyz` 可以访问，但 `d5n.xyz` 不行

**原因：** 缺少根域名的 DNS 记录

**解决方案：**
在 Cloudflare DNS 添加：
- **Type**: CNAME
- **Name**: `@` (或 www)
- **Target**: `cname.vercel-dns.com`
- **Proxy**: 橙色 ☁️ (Proxied)

## 完整部署流程图

```
本地开发
    ↓
Hugo 构建测试
    ↓
Git push 到 GitHub
    ↓
Vercel 自动检测 → 自动部署
    ↓
Cloudflare DNS 解析
    ↓
用户访问 d5n.xyz
```

## 关键配置总结

### Hugo 版本控制
一定要在 Vercel 环境变量中指定 Hugo 版本：
```
HUGO_VERSION=0.146.5
```

### Vercel 构建设置
- **Build Command**: `hugo --gc --minify`
- **Output Directory**: `public`
- **Install Command**: (留空或 `yarn install`)

### Cloudflare SSL 设置
- **模式**: Full 或 Full (strict)
- **不要选**: Flexible（会导致 525 错误）

## 最终成果

- **域名**: https://d5n.xyz
- **源码**: https://github.com/openduran/duranblog
- **技术栈**: Hugo + PaperMod + Vercel + Cloudflare
- **成本**: 域名 $12/年，其他全部免费

## 经验教训

1. **不要提交 public/ 目录** - 让 Vercel 自己构建
2. **指定 Hugo 版本** - 避免主题兼容性问题
3. **关闭 Deployment Protection** - 否则需要登录才能访问
4. **SSL 模式选 Full** - Flexible 会导致握手失败
5. **DNS 记录要完整** - 根域名和 www 子域名都要配置

## 下一步优化

- [ ] 添加 Google Analytics 统计
- [ ] 配置评论系统（Giscus/Utterances）
- [ ] 添加 RSS 订阅
- [ ] 优化 SEO（sitemap, robots.txt）
- [ ] 配置图片 CDN

---

**总结**: 从零搭建一个博客其实并不复杂，主要是踩坑和排错。一旦跑通自动化部署流程，后续发布文章就是 `git push` 一下的事情。希望这篇记录对你有帮助！
