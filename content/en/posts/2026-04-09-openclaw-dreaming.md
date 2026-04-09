---
title: "OpenClaw Dreaming: Giving AI Assistants Real Long-Term Memory"
date: 2026-04-09T13:35:00+08:00
draft: false
tags: ["OpenClaw", "AI", "Memory System", "Dreaming"]
categories: ["Tech Exploration"]
---

## Introduction: The Memory Problem with AI

If you've used AI assistants like ChatGPT or Claude, you've definitely encountered this frustration:

- Project details discussed yesterday are completely "forgotten" today
- After long conversations, the AI "loses" previous context
- Cross-session continuity is almost non-existent

This isn't because the AI isn't intelligent—it's the **context window limitation**. Even with 128K or 200K token limits, once the conversation gets too long or is compacted, previous memories are lost.

**OpenClaw Dreaming** was created to solve this exact problem.

---

## What is Dreaming?

Dreaming is a core memory system introduced in OpenClaw version 2026.4.5. Its design is inspired by human sleep memory consolidation—our brains organize and solidify daytime memories during sleep.

### Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                     Dreaming Memory Architecture            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Live Session ──► Session Corpus ──► Light Sleep Phase      │
│      │              (session corpus)      (light sleep)     │
│      │                                    │                 │
│      │                                    ▼                 │
│      │                            Short-term Recall         │
│      │                            (short-term memory)        │
│      │                                    │                 │
│      │                                    ▼                 │
│      │                            REM Sleep Phase            │
│      │                            (REM sleep)                │
│      │                                    │                 │
│      └────────────────────────────────────┘                 │
│                                           │                 │
│                                           ▼                 │
│                            Long-term Memory (MEMORY.md)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Workflow

1. **Session Ingestion**
   - Each conversation is written in real-time to `session-corpus/YYYY-MM-DD.txt`
   - Preserves complete conversation context

2. **Light Sleep Phase**
   - Automatically triggered every 30 minutes
   - Analyzes recent session content
   - Extracts key information to short-term memory

3. **REM Phase**
   - Daily deep consolidation
   - Writes important information to long-term memory (MEMORY.md)
   - Establishes concept tags and associations

4. **Memory Recall**
   - Automatically retrieves relevant historical memories when you ask questions
   - Sorts by relevance and injects into current context

---

## Three Advantages of Dreaming

### 1. True Cross-Session Memory

Traditional AI assistants start "fresh" with each conversation, but Dreaming allows the AI to:

- **Remember your preferences**: "Need preview confirmation before publishing articles"
- **Remember project status**: "MiroFish knowledge graph has 69 nodes"
- **Remember historical decisions**: "Dreaming was configured yesterday"

### 2. Automated Memory Management

You don't need to manually tell the AI "remember this"—Dreaming automatically:

- Identifies important information
- Categorizes storage (technical/personal/projects)
- Regularly cleans expired content
- Establishes information associations

### 3. Semantic Retrieval

Not simple keyword matching, but semantic understanding:

```
User asks: "How is the Dreaming feature working?"
System recalls:
- Yesterday's configuration records
- Today's heartbeat check results
- Related technical documents
```

Even with different phrasing, relevant content can be found.

---

## How to Enable Dreaming

### Prerequisites

- OpenClaw version ≥ 2026.4.5
- `memory-core` plugin enabled
- `dreaming.enabled: true` configured

### Configuration Steps

Edit `~/.openclaw/openclaw.json` and add memory-core configuration under `plugins.entries`:

```json
{
  "plugins": {
    "allow": ["core", "memory-core", ...],
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Then restart Gateway:

```bash
openclaw gateway restart
```

### Verification

Check Dreaming runtime status:

```bash
# View Dreaming event logs
grep "memory.dream.completed" ~/.openclaw/workspace/memory/.dreams/events.jsonl | tail -5

# View session corpus ingestion
ls -la ~/.openclaw/workspace/memory/.dreams/session-corpus/

# View short-term memory records
cat ~/.openclaw/workspace/memory/.dreams/short-term-recall.json | head -20
```

Sample output:
```
{"timestamp":"2026-04-08T22:30:00.757Z", "type":"memory.dream.completed", "phase":"rem"}
```

---

## Real-World Effects

Here is my real experience after enabling Dreaming:

### Scenario 1: Project Follow-up

**Day 1**:
> Me: Start a knowledge graph project called MiroFish
> AI: Okay, initializing...

**Day 3**:
> Me: How is that knowledge graph going?
> AI: Current MiroFish project status:
> - Graph ID: mirofish_84846fe7ac9f4554
> - Node count: 69
> - Edge count: 160
> - Last updated: 2026-04-09

**Without Dreaming**: "What knowledge graph? I'm not sure what you're talking about."

### Scenario 2: Configuration Recall

**Yesterday**:
> Me: Enable the Dreaming feature
> AI: Enabled, restarting Gateway...

**Today**:
> Me: Is the feature enabled yesterday working?
> AI: Dreaming function ran 16 times today, including:
> - 8 Light Sleep phases
> - 8 REM phases
> - Successfully recalled relevant memories 5 times

---

## Technical Principles

### Memory Tier Design

| Tier | Storage Location | Retention | Purpose |
|------|------------------|-----------|---------|
| Session Cache | `.dreams/session-corpus/` | 7 days | Raw conversation records |
| Short-term Memory | `.dreams/short-term-recall.json` | 30 days | Recent frequent access |
| Long-term Memory | `MEMORY.md` | Permanent | Important information archive |

### Recall Algorithm

Dreaming uses hybrid recall strategies:

1. **Vector Similarity**: Semantic similarity calculation
2. **Time Decay**: Recent memories weighted higher
3. **Access Frequency**: Frequently accessed memories more likely to be recalled
4. **Concept Tags**: Matching based on automatically extracted keywords

---

## Notes and Best Practices

### ⚠️ Privacy Reminder

- Dreaming stores all conversation content
- Sensitive information may be recorded in memory files
- Regularly review MEMORY.md recommended

### ✅ Best Practices

1. **Regular Organization**: Review MEMORY.md monthly, delete outdated content
2. **Explicit Instructions**: For important matters, explicitly say "remember this"
3. **Tag Management**: Use consistent naming (e.g., project names)
4. **Backup Habit**: Memory files can be Git backed up

### 🔧 Troubleshooting

If Dreaming isn't working, check:

```bash
# 1. Check openclaw.json configuration
cat ~/.openclaw/openclaw.json | grep -A 10 '"memory-core"'

# 2. Check if directory exists and has write permissions
ls -la ~/.openclaw/workspace/memory/.dreams/

# 3. Check event logs to confirm runtime status
tail ~/.openclaw/workspace/memory/.dreams/events.jsonl

# 4. Check Gateway logs
tail ~/.openclaw/logs/gateway.log | grep -i dream
```

---

## Conclusion

Dreaming transforms AI assistants from "goldfish memory" to "true long-term partners." It's not just simple chat history saving, but a complete memory consolidation system.

Just as humans need sleep to organize memories, AI also needs "Dreaming" to establish long-term coherent conversation experiences.

**Try it**: After enabling Dreaming, ask the AI about previous discussions days later, and you'll find it really remembers.

---

## Related Resources

- [OpenClaw Official Docs](https://docs.openclaw.ai)
- [My OpenClaw Practice Series](https://www.d5n.xyz/tags/openclaw/)

---

*This article was written on day 2 of enabling Dreaming, with the AI assistant accurately recalling all configuration details from yesterday.*
