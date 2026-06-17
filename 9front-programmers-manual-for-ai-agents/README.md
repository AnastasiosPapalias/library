# 9front Programmer's Manual for AI Agents — Community Edition

**Author:** Anastasios Papalias  
**License:** Free — Community Edition  
**Published:** 2026  

---

## What This Is

A structured AI training dataset for 9front and Plan 9 C programming. Built so that AI agents (Claude, GPT, local models) can write correct native 9front C code without hallucinating POSIX, Linux, or BSD patterns.

This is not a tutorial for humans. It is a precision reference engineered for machine consumption: typed, relational, anti-pattern-aware, and RAG-ready.

---

## Files in This Release

| File | Description |
| --- | --- |
| `plan9_programmers_manual_community_edition.md` | Full manual — all 6 parts in Markdown |
| `plan9_programmers_manual_community_edition.docx` | Full manual in Word format |
| `symbol_map.jsonl` | 107 typed symbols — syscalls, library functions, types, macros, constants |
| `plan9_dataset.jsonl` | 47 RAG-ready chunks with topic, embedding tags, and importance field |

---

## Contents of the Manual

| Part | Description |
| --- | --- |
| Part 1 — Structured API Reference | 20 concept blocks: open, read, rfork, exec, bind, dial, thread, Biobuf, notify, stat, and more |
| Part 2 — Code Patterns | 10 complete compilable 9front C programs |
| Part 3 — Anti-POSIX Training | 16 WRONG→CORRECT→WHY blocks |
| Part 4 — Symbol Map | 107 typed symbols with importance ratings |
| Part 5 — Integration Guide | RAG pipelines, SFT fine-tuning, code completion instructions |
| Part 6 — Supplementary Reference | AI Reasoning Protocol, dirread, ioproc, factotum/auth, tokenize |

---

## How to Use

### As a system prompt (simplest)
Paste the contents of `plan9_programmers_manual_community_edition.md` into your system prompt before asking an AI agent to write 9front C code.

### As a RAG pipeline
```python
import json

chunks = []
with open('plan9_dataset.jsonl') as f:
    for line in f:
        chunks.append(json.loads(line))

# Each chunk has: id, topic, title, content, source, embedding_tags, importance
# Embed 'content', store metadata, query by topic or importance
```

### As a symbol resolver
```python
import json

symbols = {}
with open('symbol_map.jsonl') as f:
    for line in f:
        s = json.loads(line)
        symbols[s['symbol']] = s

# Look up any syscall or library function
print(symbols['rfork']['signature'])
print(symbols['dial']['importance'])
```

---

## The 4-Step AI Reasoning Protocol

Before writing any 9front C code, an AI agent must:

1. **Identify** — is this native 9front, APE/POSIX, or hosted?
2. **Reject** — check against the forbidden POSIX symbol list
3. **Include** — `#include <u.h>` first, `#include <libc.h>` second, always
4. **Verify** — `void main(...)`, `exits(nil)`, never `return 0`

---

## Man Page Sources

All API documentation sourced from [man.aiju.de](https://man.aiju.de) — the authoritative 9front man pages.

---

## Related

- [Plan9 repo](https://github.com/AnastasiosPapalias/Plan9) — 9front programs and tools by the same author
