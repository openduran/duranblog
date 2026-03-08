---
title: "Leveraging Cloudflare Markdown for Agents: Optimize AI Content Fetching"
date: 2026-03-08T22:00:00+08:00
draft: false
tags: ["cloudflare", "markdown-for-agents", "ai", "web-scraping", "token-optimization"]
categories: ["Tutorials"]
description: "Learn how to leverage Cloudflare Markdown for Agents to optimize AI content fetching, reducing token consumption by 80%. Includes complete tool source code and implementation methods."
---

## The Problem: Pain Points of AI Web Scraping

When you ask an AI Agent to fetch web content, you typically encounter these issues:

- **Too much HTML noise** - Navigation bars, ads, sidebars, scripts, styles...
- **Massive token consumption** - 2,000 words of content might require 15,000+ tokens of HTML
- **Difficult parsing** - AI needs to extract useful info from complex HTML
- **High costs** - With token-based pricing, this directly means money

**Cloudflare Markdown for Agents** was created to solve this problem.

---

## What is Cloudflare Markdown for Agents?

Launched by Cloudflare in February 2026, this feature automatically converts HTML to Markdown when AI Agents scrape websites that have it enabled.

### How Significant is the Effect?

According to Cloudflare's official data:
- A blog post in HTML format: ~**16,180 tokens**
- Converted to Markdown: only ~**3,150 tokens**
- **~80% reduction in token consumption**

### How It Works

When an AI Agent sends an HTTP request with this header:

```
Accept: text/markdown
```

If the website has Cloudflare Markdown for Agents enabled, Cloudflare converts the HTML to Markdown at the edge and returns it to the AI Agent.

**The returned content:**
- ✅ Automatically removes HTML tags, CSS, JavaScript
- ✅ Preserves semantic structure (headings, lists, links, etc.)
- ✅ Easier for AI to parse, less noise
- ✅ Significantly reduces token consumption

---

## Practical: How to Make AI Agents Fetch Markdown Format

Regardless of whether the target website has Cloudflare Markdown for Agents enabled, you can optimize your scraping using the following methods.

### Method 1: Request Markdown Format (If Supported)

The simplest approach is to declare in the HTTP request header that you accept Markdown format:

```python
import requests

headers = {
    'Accept': 'text/markdown, text/html;q=0.8'
}

response = requests.get('https://example.com/article/', headers=headers)

# Check the returned content type
if 'markdown' in response.headers.get('Content-Type', ''):
    print("✅ Got Markdown format")
    content = response.text
else:
    print("ℹ️ Got HTML, needs conversion")
    content = html_to_markdown(response.text)
```

**Check if website supports it:**
- If the returned `Content-Type` contains `text/markdown`, it's supported
- Currently, not many websites support this, but the number is growing

### Method 2: Try Markdown Version URLs

Some websites actively provide Markdown versions, typically with these URL patterns:

```
https://example.com/posts/article-title/index.md
https://example.com/posts/article-title.md
https://example.com/api/content/article-title?format=md
```

**Scraping strategy:**
1. First try URLs with `.md` or `/index.md` suffix
2. If not found, fall back to regular HTML scraping
3. Convert HTML to Markdown

### Method 3: Use the Smart Fetch Tool

I've written a complete tool that automates the above workflow:

**`smart_fetch.py` core features:**
- Prioritizes Markdown format requests
- Automatically detects return type
- If HTML is returned, automatically converts to Markdown
- Extracts main content, removes navigation and ads

**Complete source code:**

```python
#!/usr/bin/env python3
"""
Smart Fetch - Intelligent Web Scraping Tool
Supports Cloudflare Markdown for Agents
Auto-detects and handles Markdown/HTML responses
"""

import sys
import urllib.request
import urllib.error
from html.parser import HTMLParser
import re


class HTMLToMarkdown(HTMLParser):
    """HTML to Markdown converter"""
    
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
    """Smart web content fetching"""
    
    headers = {
        'User-Agent': 'Mozilla/5.0 (compatible; AI-Agent/1.0; +https://www.d5n.xyz)',
        'Accept': 'text/markdown, text/plain;q=0.9, text/html;q=0.8',
        'Accept-Language': 'en-US,en;q=0.9',
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
                print(f"✅ Got Markdown format", file=sys.stderr)
                return content[:max_chars]
            
            if 'text/plain' in content_type:
                return content[:max_chars]
            
            print(f"🔄 Got HTML, converting to Markdown", file=sys.stderr)
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

**Usage examples:**

```bash
# Fetch web page, auto-handle Markdown/HTML
python3 smart_fetch.py "https://example.com/article/"

# Limit returned characters
python3 smart_fetch.py "https://example.com/article/" 3000
```

---

## Advanced: Search + Fetch Integration

In practice, you usually need to search first, then fetch detailed content. I've combined SearXNG search and Smart Fetch into a complete tool chain.

**`search_and_fetch.py` complete source code:**

```python
#!/usr/bin/env python3
"""
SearXNG + Smart Fetch combo tool
Search first, then intelligently fetch detailed content
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
    """Search using SearXNG"""
    try:
        url = f"{SEARXNG_URL}/search?q={urllib.parse.quote(query)}&format=json"
        req = urllib.request.Request(url, headers={
            'User-Agent': 'Mozilla/5.0 (compatible; AI-Agent/1.0)'
        })
        
        with urllib.request.urlopen(req, timeout=30) as response:
            data = json.loads(response.read().decode('utf-8'))
            return data.get('results', [])[:num_results]
    except Exception as e:
        print(f"❌ Search failed: {e}", file=sys.stderr)
        return []

def smart_fetch(url, max_chars=3000):
    """Call smart_fetch.py to get content"""
    try:
        result = subprocess.run(
            ['python3', os.path.join(SCRIPT_DIR, 'smart_fetch.py'), url, str(max_chars)],
            capture_output=True,
            text=True,
            timeout=30
        )
        return result.stdout
    except Exception as e:
        return f"❌ Fetch failed: {e}"

def main():
    if len(sys.argv) < 2:
        print("""Usage: python3 search_and_fetch.py "query" [num_results] [brief|full]

Options:
  num_results - Number of search results (default: 5)
  fetch_depth - brief (summary) | full (complete) (default: brief)

Examples:
  python3 search_and_fetch.py "OpenClaw tutorial"
  python3 search_and_fetch.py "AI news" 3 full
""")
        sys.exit(1)
    
    query = sys.argv[1]
    num_results = int(sys.argv[2]) if len(sys.argv) > 2 else 5
    fetch_depth = sys.argv[3] if len(sys.argv) > 3 else 'brief'
    
    print(f"🔍 Searching: {query}\n")
    
    # 1. Search
    results = searxng_search(query, num_results)
    
    if not results:
        print("No results found")
        sys.exit(1)
    
    # 2. Fetch details
    for i, result in enumerate(results, 1):
        title = result.get('title', 'No title')
        url = result.get('url', '')
        content = result.get('content', '')
        
        print(f"\n{'='*60}")
        print(f"{i}. {title}")
        print(f"   URL: {url}")
        print(f"{'='*60}\n")
        
        if content:
            print(f"📄 Summary: {content[:200]}...")
        
        if fetch_depth == 'full' and url:
            print(f"\n🔄 Fetching full content...")
            detail = smart_fetch(url, 3000)
            print(f"\n📄 Full content:\n{detail[:1500]}...")
        
        print()

if __name__ == "__main__":
    main()
```

**Usage:**

```bash
# Search and get summaries
./search-and-fetch.sh "OpenClaw tutorial" 5 brief

# Search and fetch full articles
./search-and-fetch.sh "AI safety research" 3 full
```

For setting up SearXNG search, check out my previous post:
- [Search Solutions for AI Agents: SearXNG vs. Tavily vs. Custom](https://www.d5n.xyz/en/posts/openclaw-search-solutions-comparison/)

---

## Real-World Impact

### Test Scenario: Scraping a Technical Blog Post

| Method | Content-Type | Token Count | Effect |
|--------|-------------|-------------|--------|
| Regular HTML | text/html | ~5,000 | Contains navigation, styles, noise |
| Markdown format | text/markdown | ~1,000 | Only main content |
| **Savings** | - | **~80%** | ✅ Significant optimization |

### Benefits for AI Agents

1. **Lower costs** - 60-80% reduction in token consumption
2. **Faster processing** - Less content to parse
3. **Better accuracy** - Reduced HTML noise interference
4. **Longer context** - Same context window can hold more content

---

## Appendix: Making Your Website Support Markdown Format

If you want your own website to support Markdown for Agents, here are implementation methods.

### Example: Hugo

Configure in `hugo.toml`:

```toml
[outputs]
  page = ["HTML", "Markdown"]

[outputFormats.Markdown]
  mediatype = "text/markdown"
  baseName = "index"
  isPlainText = true
```

Create `layouts/_default/single.md` template:

```markdown
---
title: "{{ .Title }}"
date: {{ .Date }}
---

{{ .RawContent }}
```

After building, each post generates both `index.html` and `index.md`.

### For Other Platforms

- **WordPress**: Use plugins to generate Markdown versions
- **Next.js/Gatsby**: Generate `.md` files at build time
- **Docusaurus/VitePress**: Markdown source files, provide direct access
- **Custom systems**: Write both HTML and Markdown when publishing

---

## Summary

### Key Points

1. **Request headers are key** - Use `Accept: text/markdown` to request Markdown format
2. **Try Markdown URLs** - Some websites provide `/index.md` direct access
3. **Auto-conversion fallback** - Use Smart Fetch tool for automatic HTML→Markdown conversion
4. **Integrated tools for efficiency** - Search+fetch integration, complete workflow

### Applicable Scenarios

- ✅ AI assistant real-time Q&A (needs to fetch external sources)
- ✅ Content aggregation and analysis (batch processing articles)
- ✅ Automated monitoring (regular update checks)
- ✅ Research assistance (quick access to clean content)

---

## Resources

- [Cloudflare Markdown for Agents docs](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)
- [Hugo Configure Outputs](https://gohugo.io/configuration/outputs/)
- [Search Solutions Comparison](https://www.d5n.xyz/en/posts/openclaw-search-solutions-comparison/)

---

*Complete source code examples available on GitHub. Feedback welcome!*
