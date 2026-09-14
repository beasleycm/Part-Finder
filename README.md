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

## Retrieval fixes (September 2026)

Six defects were causing Layer 1 to return manual pages that did not
contain the answer. The web and synthesis layers were never at fault.

Symptom: every question about a given machine returned an identical
6,026-character context. Asking for a JLG 600AJ starter part number
returned electrical-diagram pages, and the agent correctly reported
that the manuals held no answer -- while the parts book did contain
`303 7012679 1 Starter`.

### In `parts_agent.py` (this repository)

- `IDENT_PATTERNS` only matched letters-then-digits (`S-40`, `TL150`).
  Digit-leading models such as `600AJ`, `800AJ`, and `1930ES` never
  matched, so `extract_identifiers` returned nothing and web queries
  lost their model anchor. Added `\b\d{3,4}[A-Z]{1,3}\b`.
- That pattern also matches ratings like `120V` and `300PSI`, so a
  `UNIT_SUFFIXES` blocklist rejects tokens whose trailing letters are
  a unit.

### In `library_search.py` (lives at `C:\LocalAILauncher\GoogleDrive`, not in this repo)

- `retrieve_candidates()` short-circuited whenever a model identifier
  was present, using only the bare identifier query (`"600AJ"`) and
  discarding every topic word. That query matches 4,063 chunks; after
  the 500-candidate cap the pages holding the answer were gone before
  ranking began. Replaced with `build_identifier_scoped_queries()`,
  which returns precision-first tiers that all still require the
  identifier but let topic words choose the chunks.
- Added `calculate_part_number_proximity_bonus()`, which rewards a
  component name sitting just after a 6-9 digit part number. This is
  what separates a real parts-table row from a wiring-diagram callout;
  without it both scored identically and bm25 noise decided the winner.
- Added `strip_serial_references()` so `S/N 138090` is not mistaken
  for a part number by the bonus above.
- `rank_candidates()` now deduplicates identical chunk text. Several
  manuals are indexed twice (ANSI and CE), and duplicates were
  consuming half the context budget.
- `fallback_match_position()` centred excerpts on the first question
  token, which for a part-number question is `part` -- present in every
  page header. The excerpt was centred on a header and the trim window
  cut the actual table row out of the evidence. It now prefers a
  component name adjacent to a part number.
- `extract_required_identifiers()` required four characters after
  canonicalisation, so `S-40` -> `S40` was discarded and the whole
  Genie S- and Z- range produced no identifier at all. Added
  `SHORT_MODEL_CODE_PATTERN` for one-to-two letters plus two-to-three
  digits.
- `main()` now reconfigures stdout/stderr to UTF-8. Manual text
  containing the ohm sign or degree marks raised `UnicodeEncodeError`
  on the Windows ANSI code page, and the CLI returned
  `{"success": false, "error": "'charmap' codec can't encode ..."}`
  even though retrieval had succeeded.

### Verification

Nine regression cases pass. The JLG 600AJ starter question now returns
`7012679`, `7027408`, and `7026833` from the parts books, and the Genie
S-40 capacity question returns `45 gallons / 170 liters`. A deliberately
nonexistent model (`999ZZ`) still returns no local evidence, so the
precision guarantee is intact.
