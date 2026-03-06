---
title: "Building a Hugo Blog from Scratch: Vercel + Cloudflare Complete Guide"
date: 2026-02-17T20:50:00+08:00
draft: false
tags: ["hugo", "vercel", "cloudflare", "github", "blog", "tutorial"]
categories: ["Tutorials"]
description: "A complete walkthrough of building a Hugo blog from scratch, including tech stack choices, common pitfalls, and solutions. Covers Vercel deployment, Cloudflare DNS configuration, and SSL troubleshooting."
---

## Introduction

Today I spent about 8 hours building this blog from scratch. This post documents the complete process, including technology choices, pitfalls encountered, and their solutions. Hope this helps anyone looking to build their own blog.

## Tech Stack Overview

### Hugo - Static Site Generator

[Hugo](https://gohugo.io/) is a static site generator written in Go, marketed as "the world's fastest static site generator."

**Pros:**
- ⚡ Lightning-fast builds (thousands of pages per second)
- 🎨 Rich theme ecosystem (300+ official themes)
- 📝 Native Markdown support
- 🔧 Single binary deployment

**Cons:**
- Theme versions may be incompatible with Hugo versions
- Learning curve involved

### GitHub - Code Hosting

[GitHub](https://github.com) hosts the blog source code with Git version control.

**Functions:**
- Code version management
- Markdown file storage
- Automatic deployment integration with Vercel

### Vercel - Static Site Hosting

[Vercel](https://vercel.com) is a frontend deployment platform with excellent static site support.

**Pros:**
- 🚀 Automatic deployment (deploy on every push)
- 🌍 Global CDN acceleration
- 🆓 Free tier sufficient for personal blogs
- 🔒 Automatic HTTPS

**Notes:**
- Deployment Protection may be enabled by default (needs to be disabled for public access)
- Hugo version environment variable needs to be configured correctly

### Cloudflare - DNS + CDN

[Cloudflare](https://cloudflare.com) provides DNS resolution and CDN acceleration.

**Functions:**
- Domain DNS management
- SSL/TLS certificates (automatic)
- DDoS protection
- Global CDN acceleration

## Step-by-Step Setup

### Step 1: Domain Purchase

I chose `d5n.xyz` - Duran (5 letters) + N.

**Tips:**
- 3-letter `.com` domains are mostly premium ($1000+)
- `.xyz` is cheap for the first year ($1-3), but check renewal prices
- Cloudflare offers domain registration at cost (no markup)

### Step 2: Initialize Hugo Site

```bash
hugo new site duranblog
cd duranblog
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

**Configure hugo.toml:**
```toml
baseURL = 'https://d5n.xyz'
languageCode = 'zh-CN'
title = 'D5N'
theme = 'PaperMod'
```

### Step 3: Create GitHub Repository

- Repository name: `duranblog`
- Type: Public (Vercel free tier has no limits for public repos)
- Initialize with README

### Step 4: Push Code to GitHub

**Issue 1: Git Authentication Failure**

Error:
```
fatal: could not read Username for 'https://github.com'
```

**Solution:**
Use Personal Access Token authentication:
```bash
git remote set-url origin https://openduran:TOKEN@github.com/openduran/duranblog.git
```

### Step 5: Vercel Deployment

#### Issue 2: Raw HTML Source Displayed

Symptom: Browser shows raw HTML code instead of rendered webpage.

**Investigation:**
1. Checked GitHub repo, found `public/` directory was committed
2. Vercel has auto-build; no need to commit built files
3. Remove `public/` and add `.gitignore`

**Solution:**
```bash
rm -rf public/
echo "public/" >> .gitignore
git add . && git commit -m "Remove public dir" && git push
```

#### Issue 3: Hugo Version Incompatibility

Error:
```
WARN  Module "PaperMod" is not compatible with this Hugo version: Min 0.146.0
ERROR render of "/404" failed
```

**Cause:** Vercel's default Hugo version is too old; PaperMod requires 0.146.0+

**Solution:**
Add environment variable in Vercel project settings:
- **Name**: `HUGO_VERSION`
- **Value**: `0.146.5`

#### Issue 4: Login Required (401 Error)

Symptom: Website shows "Vercel Authentication"

**Solution:**
1. Go to Vercel project Settings → General
2. Find "Deployment Protection"
3. Change to "Disabled"
4. Save and redeploy

### Step 6: Configure Cloudflare DNS

#### Issue 5: SSL Handshake Failed (525 Error)

Error:
```
525: SSL handshake failed
```

**Cause:** Cloudflare SSL mode incompatible with Vercel

**Solution:**
1. Go to Cloudflare → SSL/TLS → Overview
2. Change mode from "Flexible" to "Full" or "Full (strict)"

#### Issue 6: Root Domain Not Accessible

Symptom: `www.d5n.xyz` works, but `d5n.xyz` doesn't

**Cause:** Missing DNS record for root domain

**Solution:**
Add in Cloudflare DNS:
- **Type**: CNAME
- **Name**: `@` (or www)
- **Target**: `cname.vercel-dns.com`
- **Proxy**: Orange ☁️ (Proxied)

## Deployment Workflow

```
Local Development
    ↓
Hugo Build Test
    ↓
Git push to GitHub
    ↓
Vercel Auto-detect → Auto-deploy
    ↓
Cloudflare DNS Resolution
    ↓
User visits d5n.xyz
```

## Key Configuration Summary

### Hugo Version Control
Always specify Hugo version in Vercel environment variables:
```
HUGO_VERSION=0.146.5
```

### Vercel Build Settings
- **Build Command**: `hugo --gc --minify`
- **Output Directory**: `public`
- **Install Command**: (leave blank or `yarn install`)

### Cloudflare SSL Settings
- **Mode**: Full or Full (strict)
- **Don't use**: Flexible (causes 525 error)

## Final Result

- **Domain**: https://d5n.xyz
- **Source**: https://github.com/openduran/duranblog
- **Stack**: Hugo + PaperMod + Vercel + Cloudflare
- **Cost**: $12/year for domain, everything else free

## Lessons Learned

1. **Don't commit public/ directory** - Let Vercel build it
2. **Specify Hugo version** - Avoid theme compatibility issues
3. **Disable Deployment Protection** - Otherwise login is required
4. **Use Full SSL mode** - Flexible causes handshake failures
5. **Complete DNS records** - Both root and www subdomains need configuration

## Next Steps

- [ ] Add Google Analytics
- [ ] Configure comments (Giscus/Utterances)
- [ ] Add RSS feed
- [ ] Optimize SEO (sitemap, robots.txt)
- [ ] Configure image CDN

---

**Conclusion**: Building a blog from scratch isn't complicated—it's mostly about troubleshooting. Once the automated deployment pipeline is set up, publishing posts is just a `git push` away. Hope this guide helps!
