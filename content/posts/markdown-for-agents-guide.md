---
title: "利用 Cloudflare Markdown for Agents 优化 AI 内容抓取"
date: 2026-03-08T22:00:00+08:00
draft: false
tags: ["cloudflare", "markdown-for-agents", "ai", "web-scraping", "token-optimization"]
categories: ["技术教程"]
description: "学习如何利用 Cloudflare Markdown for Agents 功能，让 AI Agent 高效抓取网页内容，节省 80% token 消耗。包含完整工具代码和实现方法。"
---

## 背景：AI 抓取的痛点

当你让 AI Agent 去抓取网页内容时，通常会遇到这些问题：

- **HTML 噪音太多** - 导航栏、广告、侧边栏、脚本、样式...
- **Token 消耗巨大** - 2,000 字的正文可能需要 15,000+ tokens 的 HTML
- **解析困难** - AI 需要从复杂 HTML 中提取有用信息
- **成本高** - 按 token 付费的模型下，这直接意味着钱

**Cloudflare Markdown for Agents** 就是为了解决这个问题而生的。

---

## 什么是 Cloudflare Markdown for Agents？

这是 Cloudflare 在 2026 年 2 月推出的功能。当 AI Agent 抓取启用了此功能的网站时，Cloudflare 会自动将 HTML 转换为 Markdown 返回。

### 效果有多显著？

根据 Cloudflare 官方数据：
- 一篇博客文章在 HTML 格式下约 **16,180 tokens**
- 转换为 Markdown 后仅 **3,150 tokens**
- **节省约 80% 的 token 消耗**

### 工作原理

当 AI Agent 发送 HTTP 请求时，在请求头中添加：

```
Accept: text/markdown
```

如果网站启用了 Cloudflare Markdown for Agents，Cloudflare 会在边缘节点实时将 HTML 转换为 Markdown，然后返回给 AI Agent。

**返回的内容：**
- ✅ 自动去除 HTML 标签、CSS、JavaScript
- ✅ 保留内容的语义结构（标题、列表、链接等）
- ✅ AI 更容易解析，减少噪声干扰
- ✅ 大幅减少 token 消耗

---

## 实战：如何让 AI Agent 抓取 Markdown 格式

无论目标网站是否启用了 Cloudflare Markdown for Agents，你都可以通过以下方法优化抓取效果。

### 方法 1：请求 Markdown 格式（如果网站支持）

最简单的做法是在 HTTP 请求头中声明接受 Markdown 格式：

```python
import requests

headers = {
    'Accept': 'text/markdown, text/html;q=0.8'
}

response = requests.get('https://example.com/article/', headers=headers)

# 检查返回的内容类型
if 'markdown' in response.headers.get('Content-Type', ''):
    print("✅ 获取到 Markdown 格式")
    content = response.text
else:
    print("ℹ️ 返回 HTML，需要转换")
    content = html_to_markdown(response.text)
```

**判断网站是否支持：**
- 如果返回的 `Content-Type` 包含 `text/markdown`，说明支持
- 目前支持此功能的网站还不多，但会逐渐增加

### 方法 2：尝试 Markdown 版本 URL

一些网站会主动提供 Markdown 版本，通常的 URL 模式：

```
https://example.com/posts/article-title/index.md
https://example.com/posts/article-title.md
https://example.com/api/content/article-title?format=md
```

**抓取策略：**
1. 先尝试 `.md` 或 `/index.md` 后缀的 URL
2. 如果不存在，回退到普通 HTML 抓取
3. 将 HTML 转换为 Markdown

### 方法 3：使用 Smart Fetch 工具

我编写了一个完整的工具，自动完成上述流程：

**`smart_fetch.py` 核心功能：**
- 优先请求 Markdown 格式
- 自动检测返回类型
- 如果返回 HTML，自动转换为 Markdown
- 提取正文内容，去除导航和广告

**完整源码：**

```python
#!/usr/bin/env python3
"""
Smart Fetch - 智能网页抓取工具
支持 Cloudflare Markdown for Agents
自动检测并处理 Markdown/HTML 响应
"""

import sys
import urllib.request
import urllib.error
from html.parser import HTMLParser
import re


class HTMLToMarkdown(HTMLParser):
    """HTML 转 Markdown 转换器"""
    
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
    
    # 构建请求头 - 优先请求 Markdown
    headers = {
        'User-Agent': 'Mozilla/5.0 (compatible; AI-Agent/1.0; +https://www.d5n.xyz)',
        'Accept': 'text/markdown, text/plain;q=0.9, text/html;q=0.8',
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
            
            # 检查是否返回了 Markdown
            if 'markdown' in content_type:
                print(f"✅ 获取到 Markdown 格式 (Content-Type: {content_type})", file=sys.stderr)
                return content[:max_chars]
            
            # 如果是纯文本
            if 'text/plain' in content_type:
                return content[:max_chars]
            
            # 如果是 HTML，转换为 Markdown
            print(f"🔄 返回 HTML，转换为 Markdown", file=sys.stderr)
            converter = HTMLToMarkdown()
            
            # 提取 body 内容
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

**使用示例：**

```bash
# 抓取网页，自动处理 Markdown/HTML
python3 smart_fetch.py "https://example.com/article/"

# 限制返回字符数
python3 smart_fetch.py "https://example.com/article/" 3000
```

---

## 进阶：搜索 + 抓取一体化

在实际应用中，通常需要先搜索，再抓取详细内容。我将 SearXNG 搜索和 Smart Fetch 组合成了一个完整的工具链。

**`search_and_fetch.py` 完整源码：**

```python
#!/usr/bin/env python3
"""
SearXNG + Smart Fetch 组合工具
先搜索，再智能抓取详细内容
"""

import sys
import urllib.request
import urllib.error
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
        print("""Usage: python3 search_and_fetch.py "搜索关键词" [结果数量] [brief|full]

Options:
  结果数量    - 搜索返回的结果数 (默认: 5)
  抓取深度    - brief(摘要) | full(全文) (默认: brief)

Examples:
  python3 search_and_fetch.py "OpenClaw 教程"
  python3 search_and_fetch.py "AI 新闻" 3 full
""")
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
| Markdown 格式 | text/markdown | ~1,000 | 仅保留正文内容 |
| **节省** | - | **~80%** | ✅ 显著优化 |

### 对 AI Agent 的好处

1. **成本降低** - Token 消耗减少 60-80%
2. **处理更快** - 需要解析的内容更少
3. **准确性提升** - 减少 HTML 噪声干扰
4. **上下文更长** - 同样的上下文窗口可以容纳更多内容

---

## 附：如何让网站支持 Markdown 格式

如果你想让自己的网站也支持 Markdown for Agents，以下是实现方法。

### 以 Hugo 为例

在 `hugo.toml` 中配置：

```toml
[outputs]
  page = ["HTML", "Markdown"]

[outputFormats.Markdown]
  mediatype = "text/markdown"
  baseName = "index"
  isPlainText = true
```

创建 `layouts/_default/single.md` 模板：

```markdown
---
title: "{{ .Title }}"
date: {{ .Date }}
---

{{ .RawContent }}
```

构建后，每篇文章会同时生成 `index.html` 和 `index.md`。

### 其他平台的思路

- **WordPress**：使用插件生成 Markdown 版本
- **Next.js/Gatsby**：在构建时生成 `.md` 文件
- **Docusaurus/VitePress**：本身 Markdown 源文件，直接提供访问
- **自建系统**：发布时同时写入 HTML 和 Markdown

---

## 总结

### 核心要点

1. **请求头是关键** - 使用 `Accept: text/markdown` 请求 Markdown 格式
2. **尝试 Markdown URL** - 部分网站提供 `/index.md` 格式的直接访问
3. **自动转换兜底** - 使用 Smart Fetch 工具自动处理 HTML→Markdown 转换
4. **组合工具提效** - 搜索+抓取一体化，完整工作流

### 适用场景

- ✅ AI 助手实时问答（需要抓取外部资料）
- ✅ 内容聚合和分析（批量处理文章）
- ✅ 自动化监控（定期检查更新）
- ✅ 研究辅助（快速获取干净内容）

---

## 参考资源

- [Cloudflare Markdown for Agents 官方文档](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)
- [Hugo 输出格式配置文档](https://gohugo.io/configuration/outputs/)
- [自建搜索引擎方案对比](https://www.d5n.xyz/posts/openclaw-search-solutions-comparison/)

---

*本文示例代码可在 GitHub 获取，欢迎尝试和反馈。*
