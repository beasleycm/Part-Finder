# Part Finder

Equipment parts research agent using your local Google Drive technical library index combined with live web search.

## What it does

Answers technical questions about equipment parts, specs, and procedures by:

1. **Searching your local manual index** (`C:\LocalAILauncher\GoogleDrive\drive_index.db`) — the authoritative spec of record
2. **Searching the web via Tavily** — for current availability, supersessions, pricing
3. **Synthesizing both layers** using Chatbox AI (Claude Opus 5) with proper citations

The manual is always the spec of record. Web information may add currency but never overwrites a manual value without an explicit note.

## Prerequisites

- **Python 3.11 ARM64** at `C:\Users\beasl\AppData\Local\Programs\Python\Python311-arm64\python.exe`
- **Chatbox AI license key** (you already have this)
- **Tavily API key** (sign up at https://tavily.com if you don't have one)

## Setup

Set your API keys as environment variables:

```powershell
$env:PARTS_LLM_API_KEY = "your-chatbox-license-key"
$env:TAVILY_API_KEY = "your-tavily-api-key"
```

## Usage

Basic question:

```powershell
C:\Users\beasl\AppData\Local\Programs\Python\Python311-arm64\python.exe parts_agent.py "what is the hydraulic oil capacity for a Genie S-40"
```

### Flags

| Flag | What it does |
|------|--------------|
| `--no-web` | Skip Tavily, use manuals only |
| `--no-llm` | Return raw evidence without synthesis |
| `--json` | Output structured JSON instead of prose |

### Quick smoke test (no API keys needed)

Test just the local manual search:

```powershell
C:\Users\beasl\AppData\Local\Programs\Python\Python311-arm64\python.exe parts_agent.py "Genie S-40 hydraulic oil" --no-llm --no-web
```

This hits only your local index — no API keys or running services required. If it returns manual excerpts, the core pipeline works.

## Example output

```
===========================================================================
QUESTION: what is the hydraulic oil capacity for a Genie S-40
===========================================================================
identifiers : ['S-40']
web queries : ['genie boom lift S-40 what hydraulic oil capacity']

--------------------------------------------------------------------------
LAYER 1 -- MANUALS
--------------------------------------------------------------------------
  context: 2847 chars, candidates: 12
   - Genie S-40 Service Manual.pdf
   - Genie S-40 Parts Manual.pdf

  [manual excerpts here]

--------------------------------------------------------------------------
LAYER 2 -- WEB (Tavily)
--------------------------------------------------------------------------
  [1] score=0.94  Genie S-40 Specifications
      https://www.genielift.com/...
      Hydraulic system capacity: 8.5 gallons (32 liters)...

--------------------------------------------------------------------------
SYNTHESIS
--------------------------------------------------------------------------
**From your manuals** -- The Genie S-40 Service Manual specifies 8.5 gallons 
(32 liters) hydraulic oil capacity. (Source: Genie S-40 Service Manual.pdf)

**From the web** -- Current parts suppliers confirm 8.5 gallon capacity and 
recommend ISO 46 AW hydraulic oil. (Source: genielift.com)

  usage: {'prompt_tokens': 892, 'completion_tokens': 127, 'total_tokens': 1019}
  ungrounded numbers: none
```

## Configuration

Override defaults via environment variables:

```powershell
$env:PARTS_LLM_ENDPOINT = "https://ai.chatboxai.app/v1/chat/completions"  # default
$env:PARTS_LLM_MODEL = "claude-opus-5"  # default
$env:PARTS_LLM_API_KEY = "your-key"
$env:TAVILY_API_KEY = "your-key"
```

## Technical notes

- The script deterministically extracts part/model identifiers (no LLM call)
- Web queries are automatically anchored to manufacturer + machine type to avoid false positives
- Irrelevant web results (different products sharing a model number) are filtered out
- All synthesis is done by your local LLM — Tavily provides raw search results only, not AI answers
