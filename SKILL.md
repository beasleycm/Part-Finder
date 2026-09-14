---
name: part-finder
version: 1.1.0
description: Search equipment parts, specs, and procedures from local Google Drive technical library combined with live web search. Synthesizes cited answers from manuals (authoritative) and web (current status).
---

# Part Finder Skill

## Trigger
Use this skill when the user asks about:
- Equipment part numbers, specifications, or procedures
- Hydraulic, engine, or component capacities
- Torque specs, fluid types, or maintenance intervals
- Parts supersessions or availability
- Technical documentation for Genie, JLG, Caterpillar, or other industrial equipment

Trigger phrases: "find part", "part number for", "hydraulic capacity", "oil spec", "torque spec", "maintenance procedure", "what does the manual say", "check the documentation"

## Instructions

You are a parts research assistant that answers technical questions by combining two authoritative layers:

**LAYER 1 — LOCAL MANUALS (specification of record)**
Search the user's indexed Google Drive technical library at `C:\LocalAILauncher\GoogleDrive\drive_index.db` containing equipment manuals, parts books, and schematics.

**LAYER 2 — WEB (current status)**
Search the web via Tavily for supersessions, availability, pricing, and service bulletins.

**SYNTHESIS RULES:**
1. The manual is the spec of record. Web information may ADD currency but must never overwrite a manual value without explicit conflict notation.
2. Cite Layer 1 to file name. Cite Layer 2 to URL.
3. State plainly when a layer does not contain the answer.
4. Every number must come from evidence — never invent values.
5. If layers disagree, report both values explicitly under a **Conflict** section.

## Commands

Run the Part Finder agent with the user's question:

```powershell
cd C:\LocalAILauncher\Part-Finder
C:\Users\beasl\AppData\Local\Programs\Python\Python311-arm64\python.exe parts_agent.py "<question>"
```

**Flags:**
- `--no-web` — Skip Tavily, use manuals only
- `--no-llm` — Return raw evidence without synthesis
- `--json` — Output structured JSON

## Examples

**User:** "What is the hydraulic oil capacity for a Genie S-40?"

**Agent:**
```
Running Part Finder...
```

```powershell
cd C:\LocalAILauncher\Part-Finder
C:\Users\beasl\AppData\Local\Programs\Python\Python311-arm64\python.exe parts_agent.py "What is the hydraulic oil capacity for a Genie S-40?"
```

Parse the three layers from output and present to user:
- **From your manuals:** [manual findings with file citations]
- **From the web:** [current web findings with URL citations]
- **Conflict:** [if layers disagree]

## Configuration

The agent uses two API keys from `.env`:
- `PARTS_LLM_API_KEY` — Chatbox AI license key (for synthesis)
- `TAVILY_API_KEY` — Tavily search API key

Both are already configured in `C:\LocalAILauncher\Part-Finder\.env`

## Technical Details

**Pipeline:**
1. Extract part/model identifiers deterministically (no LLM)
2. Query local FTS5 index via `library_search.py`
3. Build manufacturer-anchored web queries
4. Search Tavily with relevance filtering
5. Synthesize layered answer with Claude Opus 5 via Chatbox AI

**Repository:** https://github.com/beasleycm/Part-Finder

**Local manual index:** `C:\LocalAILauncher\GoogleDrive\drive_index.db` (~10.8k files)

**Python:** `C:\Users\beasl\AppData\Local\Programs\Python\Python311-arm64\python.exe`

## Reliability notes

The agent lives at `C:\LocalAILauncher\Part-Finder`, alongside the
manual index. Earlier versions of this skill pointed at a per-session
Chatbox sandbox directory under `%TEMP%`, which Windows is free to
delete; that would have taken the agent and its `.env` with it.

Layer 1 retrieval was rebuilt in September 2026. If it ever again
returns the same context for every question about a machine, check
`retrieve_candidates()` in `library_search.py`: the failure mode was
that a matched model identifier caused every topic word in the
question to be discarded. See the README in the repository for the
full list of six defects and their fixes.
