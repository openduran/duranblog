---
title: "Building AI-Friendly Content: How to Cut Token Costs by 80%"
date: 2026-03-08T22:00:00+08:00
draft: false
tags: ["cloudflare", "markdown-for-agents", "ai", "hugo", "token-optimization"]
categories: ["Tutorials"]
description: "Learn how to implement Cloudflare's Markdown for Agents feature on your own site, reducing AI token consumption by 80% and making your content more accessible to agents."
---

## The Problem: HTML is Wasteful for AI

Here's something that became obvious once I started building AI agents: **HTML is terrible for machine consumption**.

Think about what happens when an AI agent tries to read a typical blog post:
- It downloads the HTML (navigation, sidebars, ads, scripts, styles)
- It has to parse through all the markup
- It extracts the actual content
- Finally, it can process what matters

A blog post with 2,000 words of actual content might require 15,000+ tokens in HTML format. That's expensive when you're paying per token.

---

## Cloudflare's Solution

Enter **Markdown for Agents**.

Cloudflare recently launched this feature that automatically converts HTML to Markdown when AI agents request your content. The numbers are impressive:

> **16,180 tokens** (HTML) → **3,150 tokens** (Markdown)
> 
> **~80% reduction**

### How It Works

When an AI crawler sends a request with this header:

```
Accept: text/markdown
```

Cloudflare intercepts it, converts your HTML to clean Markdown at the edge, and serves that instead.

**The result:** AI agents get exactly what they need—structured content without the presentation layer.

---

## The Catch: Pro Plans Only

Here's where it gets frustrating. This feature requires:
- Cloudflare Pro ($20/month) or higher
- Orange-cloud enabled (traffic going through Cloudflare)
- Manual activation in dashboard

For hobby projects or Free plan users, this isn't accessible.

**So I built an alternative.**

---

## DIY: Generate Markdown Versions for Your Content

Since Cloudflare's official feature requires a paid plan, we can achieve similar results by providing both HTML and Markdown formats ourselves. The core idea: **make every article available in both formats**.

### Example: Hugo Implementation

If you're using Hugo static site generator, here's how to do it:

#### Step 1: Configure Output Formats

Add this to your `hugo.toml`:

```toml
[outputs]
  page = ["HTML", "Markdown"]

[outputFormats.Markdown]
  mediatype = "text/markdown"
  baseName = "index"
  isPlainText = true
```

#### Step 2: Create a Markdown Template

Create `layouts/_default/single.md`:

```markdown
---
title: "{{ .Title }}"
date: {{ .Date }}
---

{{ .RawContent }}
```

#### Step 3: Access Patterns

Your content is now available at:

| Format | URL |
|--------|-----|
| HTML | `/posts/my-article/` |
| Markdown | `/posts/my-article/index.md` |

### For Other Platforms

If you don't use Hugo, the approach is the same:
- **WordPress**: Use plugins to generate Markdown versions or provide via URL parameters
- **Next.js/Gatsby**: Generate `.md` files at build time
- **Docusaurus/VitePress**: Already Markdown-based, just provide direct access
- **Custom systems**: Write both HTML and Markdown versions when publishing

---

## Making It Useful: Smart Fetch

Generating Markdown is half the battle. You also need tools that can take advantage of it.

### Complete Source Code

**`smart_fetch.py`**:

```python
#!/usr/bin/env python3
"""
Smart Web Fetch with Markdown for Agents Support
Supports Cloudflare Markdown for Agents
Auto-detects and handles Markdown/HTML responses
"""

import sys
import urllib.request
import urllib.error
from html.parser import HTMLParser
import re

class HTMLToMarkdown(HTMLParser):
    """Simple HTML to Markdown converter"""
    
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

**Usage:**

```bash
# Try to get Markdown version
python3 smart_fetch.py "https://example.com/posts/article/index.md"

# Or let it figure out the best format
python3 smart_fetch.py "https://example.com/posts/article/"
```

---

## The Complete Workflow: Search + Fetch

To make this actually useful, I combined search and fetch into a single tool.

### Complete Source Code

**`search-and-fetch.sh`**:

```bash
#!/bin/bash
# Search and Fetch - SearXNG + Smart Fetch combo tool

cd "$(dirname "$0")"
python3 search_and_fetch.py "$@"
```

**`search_and_fetch.py`** (core logic):

```python
#!/usr/bin/env python3
"""
SearXNG + Smart Fetch combo tool
Search first, then intelligently fetch detailed content
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
        print("Usage: python3 search_and_fetch.py 'query' [num_results] [brief|full]")
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

**The workflow:**

```
User Query → SearXNG Search → Get URLs → Smart Fetch → Clean Markdown
```

This gives you:
1. Privacy-respecting search (self-hosted SearXNG)
2. Intelligent content fetching
3. Markdown optimization when available
4. Clean, token-efficient output

For setting up SearXNG search, check out my previous post:
- [Search Solutions for AI Agents: SearXNG vs. Tavily vs. Custom](https://www.d5n.xyz/en/posts/openclaw-search-solutions-comparison/)

### Example Usage

```bash
# Search and get summaries
./search-and-fetch.sh "OpenClaw tutorial" 5 brief

# Search and fetch full articles
./search-and-fetch.sh "AI safety research" 3 full
```

---

## Real-World Impact

I tested this on my own blog. Here's what happened:

**Before (HTML):**
- Average post: ~5,000 tokens
- Includes: navigation, footer, scripts, styles
- AI has to parse and filter

**After (Markdown):**
- Same content: ~1,000 tokens  
- Includes: just the article
- AI processes immediately

**Cost implication:** If you're using GPT-4 at $0.03/1K tokens, analyzing 100 articles:
- HTML: 500K tokens = $15
- Markdown: 100K tokens = $3

**Savings: $12 per batch** — and that's just for small-scale usage.

---

## Why This Matters

### For Content Creators

Your content is increasingly being consumed by AI, not just humans:
- AI search engines (Perplexity, Bing Copilot)
- Research assistants (Claude, ChatGPT with browsing)
- Content aggregators
- Automated analysis tools

Making your content **AI-friendly** is becoming as important as making it **SEO-friendly**.

### For AI Developers

Token costs are real. If you're building:
- Web scraping pipelines
- Research assistants  
- Content analysis tools
- Automated monitoring

Every token you save is money in your pocket. An 80% reduction isn't just optimization—it's a different cost structure entirely.

---

## Implementation Checklist

Want to implement this on your site?

**Generate Markdown versions:**
- [ ] Configure your CMS/SSG to output Markdown
- [ ] Set up URL patterns (e.g., `/post/index.md`)
- [ ] Test that Markdown URLs work
- [ ] Document your Markdown endpoints

**Use Smart Fetch:**
- [ ] Deploy the `smart_fetch.py` script
- [ ] Test with your Markdown URLs
- [ ] Integrate into your AI workflows

**Optimize for AI:**
- [ ] Test with actual AI tools
- [ ] Measure token savings
- [ ] Adjust content structure if needed

---

## The Bigger Picture

This isn't just about saving tokens. It's about **the future of web content**.

As AI agents become primary consumers of information, we need to think about:

1. **Structured formats** - Machines need structure, not presentation
2. **Semantic clarity** - Clear hierarchies, not visual styling  
3. **Multiple representations** - Same content, different formats
4. **Machine-readable metadata** - Schema, frontmatter, linked data

The sites that adapt to this reality will have an advantage. Those that don't will be expensive to process and harder to discover.

---

## Resources

- [Cloudflare Markdown for Agents docs](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)
- [Hugo Configure Outputs](https://gohugo.io/configuration/outputs/)
- [Search Solutions Comparison](https://www.d5n.xyz/en/posts/openclaw-search-solutions-comparison/)

---

**What's your take?** Are you optimizing your content for AI consumption? What approaches have worked for you?

*This post is also available in Markdown format at `/posts/markdown-for-agents-guide/index.md` — try fetching it!*
