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

## DIY: Generating Markdown with Hugo

If you're using Hugo (or any static site generator), you can achieve similar results without paying for Cloudflare Pro.

### The Strategy

Generate **two versions** of every page:
1. `index.html` - For human visitors
2. `index.md` - For AI agents

### Step 1: Configure Hugo

Add this to your `hugo.toml`:

```toml
[outputs]
  page = ["HTML", "Markdown"]

[outputFormats.Markdown]
  mediatype = "text/markdown"
  baseName = "index"
  isPlainText = true
```

### Step 2: Create a Markdown Template

Create `layouts/_default/single.md`:

```markdown
---
title: "{{ .Title }}"
date: {{ .Date }}
---

{{ .RawContent }}
```

That's it. Hugo now generates both formats.

### Step 3: Access Patterns

Your content is now available at:

| Format | URL |
|--------|-----|
| HTML | `/posts/my-article/` |
| Markdown | `/posts/my-article/index.md` |

---

## Making It Useful: Smart Fetch

Generating Markdown is half the battle. You also need tools that can take advantage of it.

I built `smart_fetch.py` with this logic:

```python
# 1. Request Markdown first
headers = {
    'Accept': 'text/markdown, text/html;q=0.8'
}

# 2. If server returns Markdown → use it
# 3. If server returns HTML → convert to Markdown
# 4. Return clean, structured content
```

**Key features:**
- Tries Markdown endpoints first (`/index.md`)
- Falls back to HTML parsing if needed
- Extracts main content, removes navigation/noise
- Returns consistent Markdown format

### Usage

```bash
# Try to get Markdown version
python3 smart_fetch.py "https://example.com/posts/article/index.md"

# Or let it figure out the best format
python3 smart_fetch.py "https://example.com/posts/article/"
```

---

## The Complete Workflow: Web Search Skill

To make this actually useful, I combined search and fetch into a single tool.

**The workflow:**

```
User Query → SearXNG Search → Get URLs → Smart Fetch → Clean Markdown
```

This gives you:
1. Privacy-respecting search (self-hosted SearXNG)
2. Intelligent content fetching
3. Markdown optimization when available
4. Clean, token-efficient output

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

**For Hugo users:**
- [ ] Add Markdown output format to `hugo.toml`
- [ ] Create `single.md` template
- [ ] Test build generates `.md` files
- [ ] Verify `/index.md` URLs work

**For other platforms:**
- [ ] Find your SSG's multi-format output option
- [ ] Create plain text/Markdown template
- [ ] Set up URL routing

**For all sites:**
- [ ] Consider adding `Content-Type: text/markdown` headers
- [ ] Document your Markdown endpoints
- [ ] Test with actual AI tools

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
- [Hugo Output Formats](https://gohugo.io/templates/output-formats/)
- [Example code on GitHub](https://github.com/openduran/duranblog)

---

**What's your take?** Are you optimizing your content for AI consumption? What approaches have worked for you?

*This post is also available in Markdown format at `/posts/markdown-for-agents-guide/index.md` — try fetching it!*
