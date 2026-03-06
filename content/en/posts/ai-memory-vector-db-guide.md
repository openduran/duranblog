---
title: "Building an AI Memory System: A Lightweight Vector Database Guide"
date: 2026-02-20T22:45:00+08:00
draft: false
tags: ["openclaw", "vector-database", "ai-memory", "python", "tutorial"]
categories: ["Tutorials"]
description: "Build a semantic search and auto-linking memory system for your AI assistant from scratch. Includes complete code and deployment steps—ready to use immediately."
---

## The Problem

My AI assistant (OpenClaw) had a memory problem. Every restart, it started fresh. While I was saving conversation history to files, this approach had serious limitations:

1. **Keyword matching fails**: Searching for "blog RSS config" wouldn't find content about "subscription optimization"
2. **No connections**: The system couldn't see that "RSS config" and "SEO optimization" were related
3. **Inefficient retrieval**: Reading all files every time burned through tokens

The solution? **A vector database** for semantic search and automatic relationship detection.

---

## Vector Database Options

Before building, I evaluated the landscape:

| Option | Type | Pros | Cons | Best For |
|--------|------|------|------|----------|
| **Chroma** | Local/Embedded | Python-native, zero-config, easy integration | Mediocre performance, simple features | Prototyping, small datasets |
| **Qdrant** | Local/Cloud | Rust-based, high performance, filtering support | Requires separate deployment, more complex | Medium scale, production |
| **Milvus** | Local/Cloud | Feature-complete, distributed support | Resource-heavy, complex setup | Large scale, enterprise |
| **Pinecone** | Managed Cloud | Zero maintenance, auto-scaling | API key required, costs, data privacy concerns | Quick starts, no-ops teams |
| **pgvector** | Postgres Plugin | SQL integration, transaction support | Requires PostgreSQL knowledge | Existing PG infrastructure |

### My Choice

Given my constraints:
- Personal project with <1000 memories
- No extra dependencies (pip can fail)
- Full local control (data privacy matters)

I went with: **Pure Python implementation** (TF-IDF + Cosine Similarity)

**Why:**
- ✅ Zero dependencies—standard library only
- ✅ Fully local—no cloud, no API keys
- ✅ Simple enough to read and modify
- ✅ Accurate enough for text memories

---

## System Architecture

### Three-Layer Memory Stack

```
┌─────────────────────────────────────────┐
│  Layer 3: Auto-Linking                  │
│  Entity extraction, co-occurrence,      │
│  relationship graphs                    │
├─────────────────────────────────────────┤
│  Layer 2: Vector Search                 │
│  TF-IDF, cosine similarity,             │
│  semantic retrieval                     │
├─────────────────────────────────────────┤
│  Layer 1: File Storage (Markdown)       │
│  Daily logs, long-term memory,          │
│  raw records                            │
└─────────────────────────────────────────┘
```

### Data Flow

```
User asks a question
        ↓
[Vector Search] finds relevant memory snippets
        ↓
[Auto-Linking] discovers related entities and context
        ↓
Combine insights → Generate response
```

---

## Implementation

### Project Structure

```bash
mkdir -p ~/openclaw/workspace/memory
cd ~/openclaw/workspace/memory
```

### The Vector Search Engine

Create `memory_search.py`:

```python
#!/usr/bin/env python3
"""
Lightweight Vector Memory Search
TF-IDF + Cosine Similarity, zero dependencies
"""

import os
import json
import math
import re
from collections import defaultdict, Counter
from datetime import datetime

class MemorySearch:
    def __init__(self, memory_dir="/home/warwick/.openclaw/workspace/memory"):
        self.memory_dir = memory_dir
        self.index_file = os.path.join(memory_dir, ".vector_index.json")
        self.documents = []
        self.term_freq = {}
        self.doc_freq = defaultdict(int)
        self.idf = {}
        
    def _tokenize(self, text):
        """Simple tokenizer: Chinese characters + English words"""
        text = re.sub(r'[^\u4e00-\u9fa5a-zA-Z0-9]', ' ', text)
        tokens = []
        for char in text:
            if '\u4e00' <= char <= '\u9fa5':
                tokens.append(char)  # Chinese character
            elif char.isalnum():
                tokens.append(char.lower())  # English/alphanumeric
        return tokens
    
    def _compute_tf(self, tokens):
        """Compute term frequencies"""
        counter = Counter(tokens)
        total = len(tokens)
        return {term: count/total for term, count in counter.items()}
    
    def add_document(self, doc_id, content, metadata=None):
        """Add document to index"""
        tokens = self._tokenize(content)
        tf = self._compute_tf(tokens)
        
        doc = {
            "id": doc_id,
            "content": content,
            "tf": tf,
            "metadata": metadata or {},
            "added_at": datetime.now().isoformat()
        }
        self.documents.append(doc)
        
        for term in set(tokens):
            self.doc_freq[term] += 1
            
    def build_index(self):
        """Build the search index"""
        N = len(self.documents)
        # Compute IDF
        for term, df in self.doc_freq.items():
            self.idf[term] = math.log(N / (df + 1)) + 1
            
        # Compute TF-IDF vectors
        for doc in self.documents:
            doc["vector"] = {}
            for term, tf in doc["tf"].items():
                doc["vector"][term] = tf * self.idf.get(term, 0)
                
    def _cosine_similarity(self, vec1, vec2):
        """Calculate cosine similarity between two vectors"""
        terms = set(vec1.keys()) | set(vec2.keys())
        dot_product = sum(vec1.get(t, 0) * vec2.get(t, 0) for t in terms)
        
        norm1 = math.sqrt(sum(v**2 for v in vec1.values()))
        norm2 = math.sqrt(sum(v**2 for v in vec2.values()))
        
        if norm1 == 0 or norm2 == 0:
            return 0
        return dot_product / (norm1 * norm2)
    
    def search(self, query, top_k=5):
        """Semantic search"""
        query_tokens = self._tokenize(query)
        query_tf = self._compute_tf(query_tokens)
        query_vec = {}
        for term, tf in query_tf.items():
            query_vec[term] = tf * self.idf.get(term, 0)
        
        results = []
        for doc in self.documents:
            score = self._cosine_similarity(query_vec, doc.get("vector", {}))
            if score > 0:
                results.append({
                    "id": doc["id"],
                    "content": doc["content"][:200] + "..." if len(doc["content"]) > 200 else doc["content"],
                    "score": round(score, 4),
                    "metadata": doc["metadata"]
                })
        
        results.sort(key=lambda x: x["score"], reverse=True)
        return results[:top_k]
    
    def index_memory_files(self):
        """Index all memory markdown files"""
        import glob
        
        md_files = glob.glob(os.path.join(self.memory_dir, "*.md"))
        
        for filepath in md_files:
            if os.path.basename(filepath).startswith("."):
                continue
                
            with open(filepath, 'r', encoding='utf-8') as f:
                content = f.read()
                
            sections = re.split(r'\n##+\s+', content)
            for i, section in enumerate(sections):
                if section.strip():
                    doc_id = f"{os.path.basename(filepath)}#{i}"
                    date_match = re.search(r'(\d{4}-\d{2}-\d{2})', filepath)
                    metadata = {"date": date_match.group(1) if date_match else None}
                    self.add_document(doc_id, section, metadata)
        
        self.build_index()
        print(f"✅ Indexed {len(self.documents)} document chunks")
        
    def save_index(self):
        """Save index to disk"""
        data = {
            "documents": [{k: v for k, v in doc.items() if k != "vector"} for doc in self.documents],
            "idf": self.idf,
            "doc_freq": dict(self.doc_freq)
        }
        with open(self.index_file, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
            
    def load_index(self):
        """Load index from disk"""
        if not os.path.exists(self.index_file):
            return False
            
        with open(self.index_file, 'r', encoding='utf-8') as f:
            data = json.load(f)
            
        self.documents = data.get("documents", [])
        self.idf = data.get("idf", {})
        self.doc_freq = defaultdict(int, data.get("doc_freq", {}))
        
        for doc in self.documents:
            doc["vector"] = {}
            for term, tf in doc.get("tf", {}).items():
                doc["vector"][term] = tf * self.idf.get(term, 0)
                
        return True


def main():
    import sys
    
    searcher = MemorySearch()
    
    if not searcher.load_index():
        print("🔄 First run, building index...")
        searcher.index_memory_files()
        searcher.save_index()
    else:
        print(f"✅ Loaded index: {len(searcher.documents)} documents")
    
    if len(sys.argv) > 1:
        query = " ".join(sys.argv[1:])
        print(f"\n🔍 Searching: {query}\n")
        results = searcher.search(query, top_k=5)
        for i, r in enumerate(results, 1):
            print(f"{i}. [{r['score']}] {r['id']}")
            print(f"   {r['content'][:150]}...\n")
    else:
        print("\n💡 Usage: python3 memory_search.py 'your query'")


if __name__ == "__main__":
    main()
```

### Auto-Linking System

Create `memory_linker.py`:

```python
#!/usr/bin/env python3
"""
Memory Auto-Linking System
Based on entity extraction + co-occurrence analysis
"""

import os
import json
import re
from collections import defaultdict
from datetime import datetime

class MemoryLinker:
    def __init__(self, memory_dir="/home/warwick/.openclaw/workspace/memory"):
        self.memory_dir = memory_dir
        self.links_file = os.path.join(memory_dir, ".memory_links.json")
        self.entities = defaultdict(set)
        self.documents = {}
        
    def _extract_entities(self, text):
        """Extract technical entities and terms"""
        entities = set()
        
        # Technical patterns
        tech_patterns = [
            r'\b[A-Z][a-zA-Z0-9]*[A-Z][a-zA-Z0-9]*\b',  # CamelCase
            r'`([^`]+)`',  # Inline code
            r'\b([A-Z]{2,})\b',  # Acronyms
        ]
        
        for pattern in tech_patterns:
            matches = re.findall(pattern, text)
            entities.update(matches)
        
        # Chinese technical terms
        cn_terms = re.findall(r'[\u4e00-\u9fa5]{2,6}(?:系统|框架|工具|配置|优化)', text)
        entities.update(cn_terms)
        
        # URLs and paths
        urls = re.findall(r'https?://[^\s]+|/[^\s\)]+', text)
        entities.update(urls)
        
        return entities
    
    def _extract_tags(self, text):
        return set(re.findall(r'#([\w\u4e00-\u9fa5]+)', text))
    
    def analyze_document(self, doc_id, content):
        entities = self._extract_entities(content)
        tags = self._extract_tags(content)
        
        self.documents[doc_id] = {
            "content": content[:500],
            "entities": list(entities),
            "tags": list(tags),
        }
        
        for entity in entities:
            self.entities[entity].add(doc_id)
        for tag in tags:
            self.entities[f"#{tag}"].add(doc_id)
    
    def find_related(self, doc_id, top_k=5):
        if doc_id not in self.documents:
            return []
        
        doc = self.documents[doc_id]
        doc_entities = set(doc["entities"]) | set(f"#{t}" for t in doc["tags"])
        
        related_scores = defaultdict(int)
        for entity in doc_entities:
            for other_doc in self.entities[entity]:
                if other_doc != doc_id:
                    related_scores[other_doc] += 1
        
        results = []
        for other_id, score in related_scores.items():
            if other_id in self.documents:
                other_doc = self.documents[other_id]
                other_entities = set(other_doc["entities"]) | set(f"#{t}" for t in other_doc["tags"])
                union = len(doc_entities | other_entities)
                similarity = score / union if union > 0 else 0
                shared = doc_entities & other_entities
                
                results.append({
                    "id": other_id,
                    "score": round(similarity, 4),
                    "shared_entities": list(shared)[:5],
                    "preview": other_doc["content"][:100] + "..."
                })
        
        results.sort(key=lambda x: x["score"], reverse=True)
        return results[:top_k]
    
    def build_links(self):
        import glob
        
        md_files = glob.glob(os.path.join(self.memory_dir, "*.md"))
        
        for filepath in md_files:
            if os.path.basename(filepath).startswith("."):
                continue
                
            with open(filepath, 'r', encoding='utf-8') as f:
                content = f.read()
            
            sections = re.split(r'\n##+\s+', content)
            for i, section in enumerate(sections):
                if section.strip() and len(section) > 50:
                    doc_id = f"{os.path.basename(filepath)}#{i}"
                    self.analyze_document(doc_id, section)
        
        print(f"✅ Analyzed {len(self.documents)} document chunks")
        print(f"✅ Extracted {len(self.entities)} entities")
        
        strong_links = []
        for entity, docs in self.entities.items():
            if len(docs) >= 2 and not entity.startswith('#'):
                strong_links.append({
                    "entity": entity,
                    "doc_count": len(docs),
                    "docs": list(docs)[:5]
                })
        
        strong_links.sort(key=lambda x: x["doc_count"], reverse=True)
        return strong_links[:20]
    
    def save_links(self):
        data = {
            "documents": self.documents,
            "entities": {k: list(v) for k, v in self.entities.items()},
            "built_at": datetime.now().isoformat()
        }
        with open(self.links_file, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    
    def load_links(self):
        if not os.path.exists(self.links_file):
            return False
            
        with open(self.links_file, 'r', encoding='utf-8') as f:
            data = json.load(f)
            
        self.documents = data.get("documents", {})
        self.entities = defaultdict(set, {k: set(v) for k, v in data.get("entities", {}).items()})
        return True
    
    def show_entity_graph(self, entity):
        if entity not in self.entities:
            print(f"❌ Entity not found: {entity}")
            return
        
        docs = self.entities[entity]
        print(f"\n🔗 Entity '{entity}' relationship graph")
        print(f"   Appears in {len(docs)} documents:\n")
        
        for doc_id in list(docs)[:10]:
            if doc_id in self.documents:
                preview = self.documents[doc_id]["content"][:80]
                print(f"   • {doc_id}")
                print(f"     {preview}...\n")


def main():
    import sys
    
    linker = MemoryLinker()
    
    if len(sys.argv) > 1:
        cmd = sys.argv[1]
        
        if cmd == "build":
            print("🔄 Building memory link graph...\n")
            core_links = linker.build_links()
            linker.save_links()
            
            print("\n📊 Core linked entities:")
            for i, link in enumerate(core_links[:10], 1):
                print(f"{i}. {link['entity']} - appears in {link['doc_count']} documents")
                
        elif cmd == "related" and len(sys.argv) > 2:
            doc_id = sys.argv[2]
            if not linker.load_links():
                print("❌ No link data found. Run 'build' first.")
                return
            
            print(f"\n🔍 Memories related to '{doc_id}':\n")
            related = linker.find_related(doc_id, top_k=5)
            for i, r in enumerate(related, 1):
                print(f"{i}. [{r['score']}] {r['id']}")
                print(f"   Shared: {', '.join(r['shared_entities'])}")
                print(f"   {r['preview']}\n")
                
        elif cmd == "entity" and len(sys.argv) > 2:
            entity = sys.argv[2]
            if not linker.load_links():
                print("❌ No link data found. Run 'build' first.")
                return
            linker.show_entity_graph(entity)


if __name__ == "__main__":
    main()
```

### Shell Scripts

`search.sh`:
```bash
#!/bin/bash
cd "$(dirname "$0")"
python3 memory_search.py "$@"
```

`link.sh`:
```bash
#!/bin/bash
cd "$(dirname "$0")"
python3 memory_linker.py "$@"
```

`reindex.sh`:
```bash
#!/bin/bash
cd "$(dirname "$0")"

if [ -f ".vector_index.json" ]; then
    mv .vector_index.json ".vector_index.json.backup.$(date +%Y%m%d%H%M%S)"
fi

python3 -c "
import sys
sys.path.insert(0, '.')
from memory_search import MemorySearch
searcher = MemorySearch()
searcher.index_memory_files()
searcher.save_index()
print('✅ Index rebuilt!')
"
```

Make executable:
```bash
chmod +x search.sh link.sh reindex.sh
```

---

## Usage Examples

### Semantic Search

```bash
./search.sh "blog RSS configuration"

🔍 Search results:
1. [0.4534] 2026-02-19.md#4
   Blog optimization article covers RSS feeds...

2. [0.2983] 2026-02-20.md#6
   RSS configuration improvements added...
```

### Build Link Graph

```bash
./link.sh build

📊 Core linked entities:
1. API - appears in 12 documents
2. GSC - appears in 5 documents
3. OpenClaw - appears in 5 documents
4. RSS - appears in 3 documents
```

### Find Related Memories

```bash
./link.sh related "2026-02-20.md#5"

🔍 Related memories:
1. [0.25] 2026-02-19.md#17
   Shared: Twitter, multi-platform
   Future plans - WeChat, Toutiao, Xiaohongshu...
```

### View Entity Graph

```bash
./link.sh entity "OpenClaw"

🔗 Entity 'OpenClaw' relationship graph:
   Appears in 5 documents:
   
   • 2026-02-19.md#16
     Zhihu article published successfully...
   
   • 2026-02-19.md#9
     OpenClaw update notes...
```

---

## Performance

On my setup (54 memory files, ~500KB text):

| Operation | Time | Memory |
|-----------|------|--------|
| Build index | ~2s | ~50MB |
| Search | ~50ms | Negligible |
| Load index | ~100ms | ~30MB |

More than fast enough for personal use.

---

## Future Upgrades

### 1. Migrate to Professional Vector DB

When you hit 1000+ memories, move to Chroma or Qdrant:

```python
import chromadb
client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection("memory")

collection.add(
    documents=["memory content"],
    ids=["doc_id"],
    metadatas=[{"date": "2026-02-20"}]
)

results = collection.query(
    query_texts=["search query"],
    n_results=5
)
```

### 2. Add Embedding Model

For better semantic understanding:

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')
embeddings = model.encode(["search query"])
```

### 3. Integrate into AI Startup

```python
# Load on startup
searcher = MemorySearch()
searcher.load_index()

# Search before generating
relevant = searcher.search(user_query, top_k=3)
context = "\n".join([r["content"] for r in relevant])

# Include in prompt
prompt = f"Based on memory:\n{context}\n\nUser: {user_query}"
```

---

## Summary

Using pure Python, we built a complete vector memory system with **zero dependencies**:

✅ **Semantic search** – No more keyword matching, understands intent  
✅ **Auto-linking** – Discovers hidden connections between memories  
✅ **Lightweight** – Single-file executable, no external deps  
✅ **Extensible** – Clean code, easy to upgrade  

Perfect for:
- Personal AI assistant projects
- Privacy-conscious setups (fully local)
- Quick prototypes without infrastructure
- Learning vector search fundamentals

The code above is complete and copy-paste ready. Save and run immediately!

---

**References:**
- [TF-IDF on Wikipedia](https://en.wikipedia.org/wiki/Tf-idf)
- [Cosine Similarity](https://en.wikipedia.org/wiki/Cosine_similarity)
- [ChromaDB](https://www.trychroma.com/)
- [Qdrant](https://qdrant.tech/)
