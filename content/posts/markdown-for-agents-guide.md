---
title: "实现 Cloudflare Markdown for Agents 效果：为 AI 优化你的内容输出"
date: 2026-03-08T22:00:00+08:00
draft: false
tags: ["cloudflare", "markdown-for-agents", "ai", "hugo", "optimization"]
categories: ["技术教程"]
description: "学习如何实现 Cloudflare Markdown for Agents 效果，让 AI 更高效地抓取和理解你的内容，节省 80% token 消耗。包含 Hugo 配置和实用工具。"
---

## 背景：AI 时代的内容消费变革

Cloudflare 最近推出了 **Markdown for Agents** 功能，这是一项针对 AI 时代的创新优化。简单来说，当 AI Agent（如 ChatGPT、Claude 等）抓取网页内容时，网站可以返回 Markdown 格式而非 HTML，从而大幅减少 token 消耗。

**效果有多显著？**

根据 Cloudflare 官方数据：
- 一篇博客文章在 HTML 格式下约 **16,180 tokens**
- 转换为 Markdown 后仅 **3,150 tokens**
- **节省约 80% 的 token 消耗**

这对于依赖大模型处理网页内容的应用来说，意味着显著的成本降低和效率提升。

---

## Cloudflare Markdown for Agents 是什么？

### 工作原理

当 AI Agent 发送 HTTP 请求时，如果请求头包含：

```
Accept: text/markdown
```

Cloudflare 会在边缘节点实时将 HTML 转换为 Markdown，然后返回给 AI Agent。

**核心优势：**
- ✅ 自动去除 HTML 标签、CSS、JavaScript
- ✅ 保留内容的语义结构（标题、列表、链接等）
- ✅ AI 更容易解析，减少噪声干扰
- ✅ 大幅减少 token 消耗

### 限制条件

**必须满足：**
1. 网站使用 Cloudflare CDN（橙云开启）
2. Cloudflare 计划为 Pro 及以上（$20/月起）
3. 需要手动在后台开启功能

**对于 Free 计划用户：** 无法使用官方功能，但可以通过其他方式实现类似效果。

---

## 替代方案：为每篇文章生成 Markdown 版本

既然 Cloudflare 的官方功能需要付费，我们可以用自己的方式实现类似效果。核心思路是：**让每篇文章同时提供 HTML 和 Markdown 两种格式**。

### 以 Hugo 为例

如果你使用 Hugo 静态网站生成器，可以通过以下方式实现：

#### 步骤 1：配置输出格式

在 `hugo.toml` 中添加 Markdown 输出配置：

```toml
[outputs]
  home = ["HTML", "RSS", "JSON"]
  section = ["HTML", "RSS"]
  page = ["HTML", "Markdown"]  # 关键：为每篇文章生成 Markdown

[outputFormats]
  [outputFormats.RSS]
    mediatype = "application/rss"
    baseName = "index"
  
  [outputFormats.Markdown]
    mediatype = "text/markdown"
    baseName = "index"
    isPlainText = true
```

#### 步骤 2：创建 Markdown 模板

创建 `layouts/_default/single.md`：

```markdown
---
title: "{{ .Title }}"
date: {{ .Date.Format "2006-01-02T15:04:05-07:00" }}
author: "{{ .Params.author | default .Site.Params.author }}"
categories: [{{ range .Params.categories }}"{{ . }}"{{ end }}]
tags: [{{ range .Params.tags }}"{{ . }}"{{ end }}]
---

{{ .RawContent }}
```

#### 步骤 3：构建并验证

运行 Hugo 构建：

```bash
hugo --gc
```

构建完成后，每篇文章会同时生成：
- `index.html` - HTML 版本（人类阅读）
- `index.md` - Markdown 版本（AI 阅读）

**访问方式：**
- HTML: `https://example.com/posts/article-title/`
- Markdown: `https://example.com/posts/article-title/index.md`

### 其他平台的思路

如果你不使用 Hugo，核心思路是一样的：
- **WordPress**：使用插件生成 Markdown 版本，或通过 URL 参数提供
- **Next.js/Gatsby**：在构建时生成 `.md` 文件
- **Docusaurus/VitePress**：本身就有 Markdown 源文件，可直接提供访问
- **自建系统**：在发布文章时同时写入 HTML 和 Markdown 两个文件

---

## 配套工具：Smart Fetch

为了让 AI Agent 更好地利用这个功能，我创建了一个智能抓取工具。

### 完整源码

**`smart_fetch.py`**：

```python
#!/usr/bin/env python3
"""
Smart Web Fetch with Markdown for Agents Support
支持 Cloudflare Markdown for Agents 功能
自动检测并处理 Markdown/HTML 响应
"""

import sys
import urllib.request
import urllib.error
from html.parser import HTMLParser
import re

class HTMLToMarkdown(HTMLParser):
    """简单的 HTML 转 Markdown 转换器"""
    
    def __init__(self):
        super().__init__()
        self.result = []
        self.in_script = False
        self.in_style = False
        self.skip_tags = {'script', 'style', 'nav', 'header', 'footer', 'aside'}
        
    def handle_starttag(self, tag, attrs):
        if tag in ('script', 'style'):
            self.in_script = tag == 'script'
            self.in_style = tag == 'style'
        elif tag in self.skip_tags:
            pass
        elif tag == 'h1':
            self.result.append('\n# ')
        elif tag == 'h2':
            self.result.append('\n## ')
        elif tag == 'h3':
            self.result.append('\n### ')
        elif tag == 'h4':
            self.result.append('\n#### ')
        elif tag == 'p':
            self.result.append('\n')
        elif tag == 'br':
            self.result.append('\n')
        elif tag == 'a':
            attrs_dict = dict(attrs)
            if 'href' in attrs_dict:
                self.result.append(f'[{attrs_dict.get("title", "") or attrs_dict.get("href", "")}](')
        elif tag == 'img':
            attrs_dict = dict(attrs)
            alt = attrs_dict.get('alt', '')
            src = attrs_dict.get('src', '')
            if src:
                self.result.append(f'![{alt}]({src})')
        elif tag in ('ul', 'ol'):
            self.result.append('\n')
        elif tag == 'li':
            self.result.append('- ')
        elif tag in ('strong', 'b'):
            self.result.append('**')
        elif tag in ('em', 'i'):
            self.result.append('*')
        elif tag == 'code':
            self.result.append('`')
        elif tag == 'pre':
            self.result.append('\n```\n')
        
    def handle_endtag(self, tag):
        if tag == 'script':
            self.in_script = False
        elif tag == 'style':
            self.in_style = False
        elif tag in self.skip_tags:
            pass
        elif tag in ('h1', 'h2', 'h3', 'h4', 'p', 'li'):
            self.result.append('\n')
        elif tag == 'a':
            self.result.append(')')
        elif tag in ('strong', 'b'):
            self.result.append('**')
        elif tag in ('em', 'i'):
            self.result.append('*')
        elif tag == 'code':
            self.result.append('`')
        elif tag == 'pre':
            self.result.append('\n```\n')
            
    def handle_data(self, data):
        if self.in_script or self.in_style:
            return
        text = data.strip()
        if text:
            self.result.append(text)
    
    def get_markdown(self):
        return ''.join(self.result)


def smart_fetch(url, max_chars=5000):
    """智能抓取网页内容"""
    
    headers = {
        'User-Agent': 'Mozilla/5.0 (compatible; AI-Agent/1.0; +https://www.d5n.xyz)',
        'Accept': 'text/markdown, text/plain;q=0.9, text/html;q=0.8, application/xhtml+xml;q=0.8',
        'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
        'Accept-Encoding': 'identity',
        'Connection': 'keep-alive',
    }
    
    try:
        req = urllib.request.Request(url, headers=headers, method='GET')
        
        with urllib.request.urlopen(req, timeout=30) as response:
            content_type = response.headers.get('Content-Type', '').lower()
            raw_data = response.read()
            
            try:
                content = raw_data.decode('utf-8')
            except UnicodeDecodeError:
                try:
                    content = raw_data.decode('gbk')
                except:
                    content = raw_data.decode('utf-8', errors='ignore')
            
            if 'markdown' in content_type:
                print(f"✅ 获取到 Markdown 格式", file=sys.stderr)
                return content[:max_chars]
            
            if 'text/plain' in content_type:
                return content[:max_chars]
            
            print(f"🔄 返回 HTML，转换为 Markdown", file=sys.stderr)
            converter = HTMLToMarkdown()
            
            body_match = re.search(r'<body[^>]*>(.*?)</body>', content, re.DOTALL | re.IGNORECASE)
            if body_match:
                body_content = body_match.group(1)
            else:
                body_content = content
            
            converter.feed(body_content)
            markdown = converter.get_markdown()
            markdown = re.sub(r'\n{3,}', '\n\n', markdown)
            
            return markdown[:max_chars]
            
    except Exception as e:
        return f"❌ Error: {str(e)}"


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python3 smart_fetch.py <URL> [max_chars]")
        sys.exit(1)
    
    url = sys.argv[1]
    max_chars = int(sys.argv[2]) if len(sys.argv) > 2 else 5000
    print(smart_fetch(url, max_chars))
```

**使用方式：**

```bash
# 抓取 Markdown 版本（如果可用）
python3 smart_fetch.py "https://example.com/posts/article/index.md"

# 普通抓取（自动处理）
python3 smart_fetch.py "https://example.com/posts/article/"
```

---

## 进阶：搜索+抓取一体化工具

为了更方便地使用搜索+抓取功能，我将 SearXNG 搜索和 Smart Fetch 组合成了一个完整的工具链。

### 完整源码

**`search-and-fetch.sh`**：

```bash
#!/bin/bash
# Search and Fetch - SearXNG + Smart Fetch 组合工具

cd "$(dirname "$0")"
python3 search_and_fetch.py "$@"
```

**`search_and_fetch.py`**（核心代码）：

```python
#!/usr/bin/env python3
"""
SearXNG + Smart Fetch 组合工具
先搜索，再智能抓取详细内容
"""

import sys
import urllib.request
import urllib.parse
import json
import subprocess
import os

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
SEARXNG_URL = "http://localhost:8888"

def searxng_search(query, num_results=5):
    """使用 SearXNG 搜索"""
    try:
        url = f"{SEARXNG_URL}/search?q={urllib.parse.quote(query)}&format=json"
        req = urllib.request.Request(url, headers={
            'User-Agent': 'Mozilla/5.0 (compatible; AI-Agent/1.0)'
        })
        
        with urllib.request.urlopen(req, timeout=30) as response:
            data = json.loads(response.read().decode('utf-8'))
            return data.get('results', [])[:num_results]
    except Exception as e:
        print(f"❌ 搜索失败: {e}", file=sys.stderr)
        return []

def smart_fetch(url, max_chars=3000):
    """调用 smart_fetch.py 抓取内容"""
    try:
        result = subprocess.run(
            ['python3', os.path.join(SCRIPT_DIR, 'smart_fetch.py'), url, str(max_chars)],
            capture_output=True,
            text=True,
            timeout=30
        )
        return result.stdout
    except Exception as e:
        return f"❌ 抓取失败: {e}"

def main():
    if len(sys.argv) < 2:
        print("Usage: python3 search_and_fetch.py '搜索关键词' [结果数量] [brief|full]")
        sys.exit(1)
    
    query = sys.argv[1]
    num_results = int(sys.argv[2]) if len(sys.argv) > 2 else 5
    fetch_depth = sys.argv[3] if len(sys.argv) > 3 else 'brief'
    
    print(f"🔍 搜索: {query}\n")
    
    # 1. 搜索
    results = searxng_search(query, num_results)
    
    if not results:
        print("未找到结果")
        sys.exit(1)
    
    # 2. 抓取详情
    for i, result in enumerate(results, 1):
        title = result.get('title', '无标题')
        url = result.get('url', '')
        content = result.get('content', '')
        
        print(f"\n{'='*60}")
        print(f"{i}. {title}")
        print(f"   URL: {url}")
        print(f"{'='*60}\n")
        
        if content:
            print(f"📄 摘要: {content[:200]}...")
        
        if fetch_depth == 'full' and url:
            print(f"\n🔄 正在抓取详细内容...")
            detail = smart_fetch(url, 3000)
            print(f"\n📄 详细内容:\n{detail[:1500]}...")
        
        print()

if __name__ == "__main__":
    main()
```

**使用方式：**

```bash
# 仅搜索（返回摘要）
./search-and-fetch.sh "OpenClaw 教程" 5 brief

# 搜索 + 抓取全文
./search-and-fetch.sh "AI 新闻" 3 full
```

关于 SearXNG 搜索的搭建，可以参考我之前写的文章：
- [自建搜索引擎方案对比：SearXNG、Tavily 与自定义实现](https://www.d5n.xyz/posts/openclaw-search-solutions-comparison/)

---

## 实际效果对比

### 测试场景：抓取一篇技术博客

| 方式 | Content-Type | Token 数量 | 效果 |
|------|-------------|-----------|------|
| 普通 HTML | text/html | ~5,000 | 包含导航、样式等噪声 |
| Markdown 版本 | text/markdown | ~1,000 | 仅保留正文内容 |
| **节省** | - | **~80%** | ✅ 显著优化 |

### 对 AI Agent 的好处

1. **成本降低** - Token 消耗减少 60-80%
2. **处理更快** - 需要解析的内容更少
3. **准确性提升** - 减少 HTML 噪声干扰
4. **上下文更长** - 同样的上下文窗口可以容纳更多内容

---

## 适用场景

### 推荐启用的情况

- ✅ 技术博客、文档站点
- ✅ 内容聚合平台
- ✅ AI 助手需要频繁抓取的网站
- ✅ 需要优化 SEO 的站点（AI 搜索越来越重要）

### 实际应用案例

**场景 1：AI 助手回答实时问题**
```
用户：最新的人工智能新闻是什么？
AI：让我搜索一下... → 抓取 Markdown 版本 → 生成回答
```

**场景 2：内容分析汇总**
```
研究人员：分析这 10 篇论文的共同点
AI：抓取每篇的 Markdown 版本 → 分析结构 → 生成报告
```

**场景 3：自动化监控**
```
定时任务：检查技术博客更新 → 抓取 Markdown → 发送摘要
```

---

## 总结与展望

### 今天完成的内容

1. ✅ 分析了 Cloudflare Markdown for Agents 的工作原理
2. ✅ 实现了为每篇文章生成 Markdown 版本的替代方案
3. ✅ 创建了 Smart Fetch 智能抓取工具（附完整源码）
4. ✅ 打包了搜索+抓取组合工具（附完整源码）

### 给网站管理者的建议

**如果你有 Cloudflare Pro：**
- 直接开启官方功能，最简单
- 监控效果，优化内容结构

**如果你使用 Free 计划：**
- 为自己的网站生成 Markdown 版本
- 引导 AI Agent 使用 `/index.md` 路径
- 长期来看，这有助于 AI 搜索优化

### 未来趋势

随着 AI Agent 的普及，**AI 友好的内容格式**将变得越来越重要：

- 结构化数据（Schema.org）
- 机器可读的元数据
- 清晰的语义标记
- Markdown 等轻量级格式

**提前布局，让你的内容在 AI 时代更具竞争力。**

---

## 参考资源

- [Cloudflare Markdown for Agents 官方文档](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)
- [自建搜索引擎方案对比](https://www.d5n.xyz/posts/openclaw-search-solutions-comparison/)

---

*本文示例代码可在 GitHub 获取，欢迎尝试和反馈。*
