# Part 5 — Integration Guide


**Using the Dataset with RAG, SFT & Code Completion**


> Complete instructions for integrating this dataset into three use modes: RAG pipeline (vector database embedding and retrieval), SFT fine-tuning (generating instruction pairs from the structured files), and symbol-aware code completion. Includes the full topic taxonomy and live man page URL list for keeping the dataset current.


# 9front Dataset — Integration Guide


## How to use all five files to train or power an AI agent


## What Each File Is For


| File | Format | Use case |
| --- | --- | --- |
| plan9_dataset.md | Markdown | Human-readable reference; system prompt injection; fine-tuning source |
| plan9_dataset.jsonl | JSONL | RAG vector database; chunk retrieval; semantic search |
| symbol_map.jsonl | JSONL | Type-safe API lookup; symbol resolution; code completion hints |
| code_patterns.md | Markdown | Few-shot examples; code generation prompts |
| anti_patterns.md | Markdown | Negative examples; POSIX rejection training |


## Option A: RAG Pipeline (Recommended for Production)


### Step 1 — Embed the chunks


```c
import json
```


```c
chunks = []
with open('plan9_dataset.jsonl') as f:
for line in f:
if line.strip():
chunks.append(json.loads(line))
```


```c
# Each chunk has: id, topic, content, source, embedding_tags, importance
# Embed 'content' field with your embedding model
# Store id, topic, embedding_tags, importance as metadata
```


```c
for chunk in chunks:
embedding = embed(chunk['content'])   # your embedding function
vector_db.upsert(
id=chunk['id'],
vector=embedding,
metadata={
'topic':          chunk['topic'],
'source':         chunk['source'],
'importance':     chunk['importance'],
'embedding_tags': chunk['embedding_tags'],
}
)
```


### Step 2 — Query strategy


```c
def query_9front(user_question, top_k=5):
# Boost critical chunks
results = vector_db.query(
vector=embed(user_question),
top_k=top_k * 2,
filter={'importance': {'$in': ['critical', 'high']}}
)
# Always include the reasoning protocol chunk
reasoning = vector_db.fetch('ai_reasoning_...')  # always prepend this
return [reasoning] + results[:top_k]
```


### Step 3 — System prompt construction


```c
SYSTEM = """You are an expert 9front/Plan 9 C programmer.
You write ONLY native Plan 9 C code — never POSIX, never Linux, never BSD.
```


```c
MANDATORY RULES:
1. Always #include <u.h> first, then #include <libc.h>
2. main() returns void; exit with exits(nil) for success, exits("msg") for failure
3. Error handling: check return values; use %r in format strings for errstr
4. Process creation: rfork(flags), not fork()/pthread
5. Signals: atnotify(), not signal()
6. I/O: open/read/write or Biobuf, never FILE* or stdio
7. Networking: dial()/announce()/listen()/accept(), never socket()/connect()
```


```c
Retrieved context:
{retrieved_chunks}
"""
```


## Option B: Fine-Tuning (SFT)


### Generating instruction pairs from the dataset


```c
import json, re
```


```c
# Extract WRONG/CORRECT pairs from anti_patterns.md
anti = open('anti_patterns.md').read()
blocks = re.split(r'(?=## POSIX MISCONCEPTION)', anti)
```


```c
pairs = []
for block in blocks:
if '## POSIX MISCONCEPTION' not in block:
continue
title = re.search(r'POSIX MISCONCEPTION: (.+)', block).group(1).strip()
wrong_m   = re.search(r'### WRONG\n```c(.+?)```', block, re.DOTALL)
correct_m = re.search(r'### CORRECT\n```c(.+?)```', block, re.DOTALL)
why_m     = re.search(r'### WHY\n(.+?)(?=\n###|\n---|\Z)', block, re.DOTALL)
```


```c
if wrong_m and correct_m and why_m:
pairs.append({
"instruction": f"Write Plan 9 C code to: {title}",
"input": f"Wrong POSIX approach:\n```c{wrong_m.group(1)}```",
"output": f"Correct Plan 9 approach:\n```c{correct_m.group(1)}```\n\nExplanation:\n{why_m.group(1).strip()}",
"source": "anti_patterns.md"
})
```


```c
# Extract examples from code_patterns.md
code = open('code_patterns.md').read()
patterns = re.split(r'(?=## Pattern \d+)', code)
for p in patterns[1:]:
title_m = re.match(r'## Pattern \d+: (.+)', p)
if not title_m:
continue
title = title_m.group(1).strip()
code_m = re.search(r'```c(.+?)```', p, re.DOTALL)
if code_m:
pairs.append({
"instruction": f"Write a complete 9front C program that demonstrates: {title}",
"input": "",
"output": f"```c{code_m.group(1)}```",
"source": "code_patterns.md"
})
```


```c
# Extract from structured blocks in plan9_dataset.md
md = open('plan9_dataset.md').read()
sections = re.split(r'\n(?=## )', md)
for sec in sections[1:]:
lines = sec.split('\n')
title = lines[0].lstrip('#').strip()
sig_m    = re.search(r'### SIGNATURE\n```c(.+?)```', sec, re.DOTALL)
desc_m   = re.search(r'### DESCRIPTION\n(.+?)(?=\n###)', sec, re.DOTALL)
when_m   = re.search(r'### WHEN TO USE\n(.+?)(?=\n###)', sec, re.DOTALL)
example_m= re.search(r'### EXAMPLE\n```c(.+?)```', sec, re.DOTALL)
```


```c
if sig_m and desc_m:
pairs.append({
"instruction": f"Explain the Plan 9 {title} API and show how to use it correctly",
"input": "",
"output": (
f"## {title}\n\n"
f"**Signature:**\n```c{sig_m.group(1)}```\n\n"
f"**Description:**\n{desc_m.group(1).strip()}\n\n"
+ (f"**When to use:**\n{when_m.group(1).strip()}\n\n" if when_m else "")
+ (f"**Example:**\n```c{example_m.group(1)}```" if example_m else "")
),
"source": "plan9_dataset.md"
})
```


```c
print(f"Generated {len(pairs)} instruction pairs")
with open('plan9_sft_pairs.jsonl', 'w') as f:
for p in pairs:
f.write(json.dumps(p, ensure_ascii=False) + '\n')
```


## Option C: Symbol-Aware Code Completion


```c
import json
```


```c
symbols = {}
with open('symbol_map.jsonl') as f:
for line in f:
if line.strip():
s = json.loads(line)
symbols[s['symbol']] = s
```


```c
def get_signature(name):
s = symbols.get(name)
if s:
return s.get('signature', f"/* see man.aiju.de/2/{name} */")
return None
```


```c
def get_related(name):
s = symbols.get(name)
if s:
return s.get('related', [])
return []
```


```c
def get_header(name):
s = symbols.get(name)
if s:
return s.get('header', '<libc.h>')
return '<libc.h>'
```


```c
# Query: what do I need to make a TCP connection?
sym = symbols.get('dial')
print(f"dial: {sym['signature']}")
print(f"related: {sym['related']}")
print(f"header: {sym['header']}")
print(f"importance: {sym['importance']}")
print()
```


```c
# Check if something sets errstr (for error handling code gen)
for name in ['open', 'rfork', 'exits', 'rendezvous']:
s = symbols.get(name)
print(f"{name}: sets_errstr={s.get('sets_errstr', '?')}")
```


## Chunk Importance Tiers


Use importance field to prioritise retrieval:


| Importance | Meaning | Count |
| --- | --- | --- |
| critical | Always retrieve for any 9front question | 42 chunks |
| high | Retrieve for relevant questions | 34 chunks |
| medium | Retrieve for specific queries | 1 chunk |


For a system with limited context window, always include critical chunks first.


## Topic Taxonomy


Query by topic field for targeted retrieval:


| Topic | What it covers |
| --- | --- |
| process | rfork, exec, exits, wait, notes |
| file_io | open, create, read, write, seek, dup, pipe |
| namespace | bind, mount, unmount, rfork namespace flags |
| error_handling | errstr, werrstr, sysfatal, ERRMAX |
| networking | dial, announce, listen, accept |
| buffered_io | Biobuf, Bopen, Brdline, Bgetrune |
| concurrency | libthread, Channel, threadcreate |
| synchronisation | QLock, RWLock, rendezvous, Rendez |
| file_metadata | Dir, Qid, dirstat, dirwstat |
| unicode | Rune, chartorune, utflen, UTFmax |
| formatting | print, fprint, seprint, smprint, %r |
| anti_posix | WRONG→CORRECT misconception blocks |
| code_pattern | Complete working example programs |
| ai_reasoning | Step-by-step reasoning protocol |
| 9p | 9P protocol, file server programming |
| kernel | Dev interface, Chan, kernel driver structure |
| security | factotum, auth, TLS, secstore |


## Man Page URLs (live source)


All man page content was fetched from https://man.aiju.de/. To refresh:


```c
man.aiju.de/2/open       man.aiju.de/2/read      man.aiju.de/2/fork
man.aiju.de/2/stat       man.aiju.de/2/bind       man.aiju.de/2/errstr
man.aiju.de/2/exits      man.aiju.de/2/wait       man.aiju.de/2/notify
man.aiju.de/2/print      man.aiju.de/2/dial       man.aiju.de/2/lock
man.aiju.de/2/rendezvous man.aiju.de/2/bio        man.aiju.de/2/thread
man.aiju.de/2/exec       man.aiju.de/2/dup        man.aiju.de/2/pipe
man.aiju.de/2/seek       man.aiju.de/2/postnote   man.aiju.de/2/getfields
man.aiju.de/2/dirread    man.aiju.de/2/remove     man.aiju.de/2/alarm
man.aiju.de/2/fauth      man.aiju.de/3/proc       man.aiju.de/3/ip
man.aiju.de/4/namespace  man.aiju.de/5/intro
```


## Coverage Summary


| Layer (spec) | Status | Detail |
| --- | --- | --- |
| L1: Core Knowledge | ✅ Complete | 24 man pages verbatim from man.aiju.de |
| L2: Structured Markdown | ✅ Complete | 20 concept blocks, all fields present |
| L3: Symbol Map | ✅ Complete | 107 symbols, all required fields |
| L4: Code Patterns | ✅ Complete | 10 patterns × all 7 required types |
| L5: Anti-POSIX | ✅ Complete | 16 balanced WRONG/CORRECT/WHY blocks |
| L6: AI Reasoning | ✅ Complete | Explicit 4-step protocol in dedicated chunk |
| L7: RAG Chunks | ✅ Complete | 77 chunks, normalised topics, all fields |


All 47 spec requirements: **47/47 ✅**


