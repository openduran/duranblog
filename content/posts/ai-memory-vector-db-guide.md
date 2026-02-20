---
title: "为AI助手构建记忆系统：轻量级向量数据库实战指南"
date: 2026-02-20T22:45:00+08:00
draft: false
tags: ["openclaw", "向量数据库", "AI记忆", "python", "教程"]
categories: ["技术教程"]
description: "从零开始为AI助手构建记忆系统，对比主流向量数据库方案，实现语义搜索和自动关联功能。包含完整代码和部署步骤，可直接上手操作。"
---

## 问题背景

我的AI助手（OpenClaw）每次重启后都会"失忆"。虽然通过文件系统保存了历史记录，但存在几个问题：

1. **关键词匹配局限**：搜索"博客RSS配置"，如果原文写的是"订阅功能优化"，就找不到
2. **缺乏关联性**：不知道"RSS配置"和"SEO优化"其实是同一批工作
3. **检索效率低**：每次都要读取全部文件，token消耗大

解决方案：**引入向量数据库**，实现语义搜索和自动关联。

---

## 向量数据库方案对比

在动手之前，我调研了主流方案：

| 方案 | 类型 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| **Chroma** | 本地嵌入式 | Python原生、零配置、易集成 | 性能一般、功能简单 | 原型开发、小规模数据 |
| **Qdrant** | 本地/云服务 | Rust编写、高性能、支持过滤 | 需独立部署、稍复杂 | 中等规模、生产环境 |
| **Milvus** | 本地/云服务 | 功能最全、分布式支持 | 资源占用大、配置复杂 | 大规模、企业级应用 |
| **Pinecone** | 全托管云 | 免维护、自动扩展 | 需API Key、有费用、数据外泄风险 | 快速启动、无需运维 |
| **pgvector** | PostgreSQL插件 | 与SQL结合、事务支持 | 需PostgreSQL基础 | 已有PG基础设施 |

### 我的选择

考虑到：
- 个人项目，数据量小（<1000条记忆）
- 不希望引入额外依赖（pip安装可能失败）
- 需要完全本地可控（数据隐私）

最终选择：**纯Python实现轻量级方案**（基于TF-IDF + 余弦相似度）

优点：
- ✅ 零依赖，只使用Python标准库
- ✅ 完全本地，数据不上云
- ✅ 足够简单，代码可读懂和修改
- ✅ 对于文本记忆，精度足够

---

## 系统设计

### 三层记忆架构

```
┌─────────────────────────────────────────┐
│  Layer 3: 自动关联 (Memory Linker)      │
│  - 实体提取、共现分析、关系图谱          │
├─────────────────────────────────────────┤
│  Layer 2: 向量搜索 (Memory Search)      │
│  - TF-IDF、余弦相似度、语义检索          │
├─────────────────────────────────────────┤
│  Layer 1: 文件存储 (Markdown)           │
│  - 每日日志、长期记忆、原始记录          │
└─────────────────────────────────────────┘
```

### 数据流向

```
用户提问 
   ↓
[向量搜索] 找到相关记忆片段
   ↓
[自动关联] 发现相关实体和上下文
   ↓
整合信息 → 生成回答
```

---

## 实战部署

### 第一步：创建项目结构

```bash
mkdir -p ~/openclaw/workspace/memory
cd ~/openclaw/workspace/memory
```

### 第二步：向量搜索核心代码

创建 `memory_search.py`：

```python
#!/usr/bin/env python3
"""
轻量级记忆向量搜索系统
基于TF-IDF + 余弦相似度，无需额外依赖
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
        """简单分词：中文按字，英文按词"""
        text = re.sub(r'[^\u4e00-\u9fa5a-zA-Z0-9]', ' ', text)
        tokens = []
        for char in text:
            if '\u4e00' <= char <= '\u9fa5':
                tokens.append(char)  # 中文字
            elif char.isalnum():
                tokens.append(char.lower())  # 英文数字
        return tokens
    
    def _compute_tf(self, tokens):
        """计算词频"""
        counter = Counter(tokens)
        total = len(tokens)
        return {term: count/total for term, count in counter.items()}
    
    def add_document(self, doc_id, content, metadata=None):
        """添加文档到索引"""
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
        """构建索引"""
        N = len(self.documents)
        # 计算IDF
        for term, df in self.doc_freq.items():
            self.idf[term] = math.log(N / (df + 1)) + 1
            
        # 计算TF-IDF向量
        for doc in self.documents:
            doc["vector"] = {}
            for term, tf in doc["tf"].items():
                doc["vector"][term] = tf * self.idf.get(term, 0)
                
    def _cosine_similarity(self, vec1, vec2):
        """计算余弦相似度"""
        terms = set(vec1.keys()) | set(vec2.keys())
        dot_product = sum(vec1.get(t, 0) * vec2.get(t, 0) for t in terms)
        
        norm1 = math.sqrt(sum(v**2 for v in vec1.values()))
        norm2 = math.sqrt(sum(v**2 for v in vec2.values()))
        
        if norm1 == 0 or norm2 == 0:
            return 0
        return dot_product / (norm1 * norm2)
    
    def search(self, query, top_k=5):
        """语义搜索"""
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
        """索引所有记忆文件"""
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
        print(f"✅ 索引完成：{len(self.documents)} 个文档片段")
        
    def save_index(self):
        """保存索引"""
        data = {
            "documents": [{k: v for k, v in doc.items() if k != "vector"} for doc in self.documents],
            "idf": self.idf,
            "doc_freq": dict(self.doc_freq)
        }
        with open(self.index_file, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
            
    def load_index(self):
        """加载索引"""
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
        print("🔄 首次运行，正在构建索引...")
        searcher.index_memory_files()
        searcher.save_index()
    else:
        print(f"✅ 已加载索引：{len(searcher.documents)} 个文档")
    
    if len(sys.argv) > 1:
        query = " ".join(sys.argv[1:])
        print(f"\n🔍 搜索: {query}\n")
        results = searcher.search(query, top_k=5)
        for i, r in enumerate(results, 1):
            print(f"{i}. [{r['score']}] {r['id']}")
            print(f"   {r['content'][:150]}...\n")
    else:
        print("\n💡 使用方法: python3 memory_search.py '查询内容'")


if __name__ == "__main__":
    main()
```

### 第三步：自动关联系统

创建 `memory_linker.py`：

```python
#!/usr/bin/env python3
"""
记忆自动关联系统
基于实体提取 + 共现分析
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
        """提取实体"""
        entities = set()
        
        tech_patterns = [
            r'\b[A-Z][a-zA-Z0-9]*[A-Z][a-zA-Z0-9]*\b',
            r'`([^`]+)`',
            r'\b([A-Z]{2,})\b',
        ]
        
        for pattern in tech_patterns:
            matches = re.findall(pattern, text)
            entities.update(matches)
        
        cn_terms = re.findall(r'[\u4e00-\u9fa5]{2,6}(?:系统|框架|工具|配置|优化)', text)
        entities.update(cn_terms)
        
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
        
        print(f"✅ 分析了 {len(self.documents)} 个文档片段")
        print(f"✅ 提取了 {len(self.entities)} 个实体")
        
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
            print(f"❌ 未找到实体: {entity}")
            return
        
        docs = self.entities[entity]
        print(f"\n🔗 实体 '{entity}' 关联图谱")
        print(f"   出现在 {len(docs)} 个文档中:\n")
        
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
            print("🔄 构建记忆关联图谱...\n")
            core_links = linker.build_links()
            linker.save_links()
            
            print("\n📊 核心关联实体:")
            for i, link in enumerate(core_links[:10], 1):
                print(f"{i}. {link['entity']} - 出现在 {link['doc_count']} 个文档中")
                
        elif cmd == "related" and len(sys.argv) > 2:
            doc_id = sys.argv[2]
            if not linker.load_links():
                print("❌ 未找到关联数据，请先运行 build")
                return
            
            print(f"\n🔍 与 '{doc_id}' 相关的记忆:\n")
            related = linker.find_related(doc_id, top_k=5)
            for i, r in enumerate(related, 1):
                print(f"{i}. [{r['score']}] {r['id']}")
                print(f"   共享: {', '.join(r['shared_entities'])}")
                print(f"   {r['preview']}\n")
                
        elif cmd == "entity" and len(sys.argv) > 2:
            entity = sys.argv[2]
            if not linker.load_links():
                print("❌ 未找到关联数据，请先运行 build")
                return
            linker.show_entity_graph(entity)


if __name__ == "__main__":
    main()
```

### 第四步：创建快捷命令

创建 `search.sh`：

```bash
#!/bin/bash
cd "$(dirname "$0")"
python3 memory_search.py "$@"
```

创建 `link.sh`：

```bash
#!/bin/bash
cd "$(dirname "$0")"
python3 memory_linker.py "$@"
```

创建 `reindex.sh`：

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
print('✅ 索引重建完成！')
"
```

赋予执行权限：

```bash
chmod +x search.sh link.sh reindex.sh
```

---

## 使用方法

### 1. 语义搜索

```bash
./search.sh "博客RSS配置"

🔍 搜索结果:
1. [0.4534] 2026-02-19.md#4
   博客优化文章 - 撰写并发布了 Hugo + PaperMod 博客进阶配置...

2. [0.2983] 2026-02-20.md#6
   博客RSS配置优化 - 添加了RSS订阅链接...
```

### 2. 构建关联图谱

```bash
./link.sh build

📊 核心关联实体:
1. API - 出现在 12 个文档中
2. GSC - 出现在 5 个文档中
3. OpenClaw - 出现在 5 个文档中
4. RSS - 出现在 3 个文档中
```

### 3. 查找相关记忆

```bash
./link.sh related "2026-02-20.md#5"

🔍 相关记忆:
1. [0.25] 2026-02-19.md#17
   共享: /Twitter, 多平台
   后续计划 - 微信公众号、今日头条、小红书...
```

### 4. 查看实体图谱

```bash
./link.sh entity "OpenClaw"

🔗 实体 'OpenClaw' 关联图谱:
   出现在 5 个文档中:
   
   • 2026-02-19.md#16
     知乎文章发布成功...
   
   • 2026-02-19.md#9
     OpenClaw 更新...
```

---

## 性能评估

在我的环境中（54个记忆文档，约500KB文本）：

| 操作 | 耗时 | 内存占用 |
|------|------|---------|
| 构建索引 | ~2秒 | ~50MB |
| 搜索 | ~50ms | 可忽略 |
| 加载索引 | ~100ms | ~30MB |

对于个人使用完全足够。

---

## 扩展建议

### 1. 升级到专业向量数据库

当数据量超过1000条时，建议迁移到Chroma或Qdrant：

```python
# Chroma示例
import chromadb
client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection("memory")

collection.add(
    documents=["记忆内容"],
    ids=["doc_id"],
    metadatas=[{"date": "2026-02-20"}]
)

results = collection.query(
    query_texts=["搜索内容"],
    n_results=5
)
```

### 2. 增加Embedding模型

使用 sentence-transformers 获得更好的语义理解：

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')

embeddings = model.encode(["搜索内容"])
```

### 3. 集成到AI助手启动流程

```python
# 在AI助手启动时加载记忆
searcher = MemorySearch()
searcher.load_index()

# 用户提问时先搜索相关记忆
relevant = searcher.search(user_query, top_k=3)
context = "\n".join([r["content"] for r in relevant])

# 将上下文加入prompt
prompt = f"基于以下记忆:\n{context}\n\n用户问题: {user_query}"
```

---

## 总结

通过纯Python实现，我们在**零依赖**的情况下构建了完整的向量记忆系统：

✅ **语义搜索**：告别关键词匹配，理解查询意图  
✅ **自动关联**：发现记忆间的隐藏联系  
✅ **轻量级**：单文件可运行，无外部依赖  
✅ **可扩展**：代码清晰，易于升级  

这套方案特别适合：
- 个人AI助手项目
- 对数据隐私有要求（完全本地）
- 不想维护复杂基础设施
- 快速原型验证

以上代码完整可复制，直接保存即可使用。如需改进，欢迎参考和自行修改！

---

**参考链接：**
- [TF-IDF Wikipedia](https://en.wikipedia.org/wiki/Tf-idf)
- [余弦相似度](https://en.wikipedia.org/wiki/Cosine_similarity)
- [ChromaDB](https://www.trychroma.com/)
- [Qdrant](https://qdrant.tech/)
