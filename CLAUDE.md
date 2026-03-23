# CLAUDE.md — obsiLM

This file provides guidance for AI assistants working on this codebase.

---

## Project Overview

**obsiLM** is an Obsidian-based NotebookLM — a local-first AI notebook CLI inspired by Google NotebookLM.

**Core concept:** 1 Topic = 1 Notebook = 1 Obsidian Vault

Users collect sources (PDFs, URLs, YouTube videos) on a topic; Claude AI analyzes them for Q&A, summaries, and learning materials. All data is stored as local Markdown files in Obsidian vault format.

**Current status:** Specification phase. `SPEC.md` contains the full design doc. No source code exists yet.

---

## Tech Stack

| Component | Choice |
|---|---|
| Language | Python 3.12+ |
| CLI framework | Typer |
| AI API | Claude API (Anthropic) — default model: `claude-opus-4-6` |
| Obsidian integration | Obsidian official CLI (`obsidian` command, v1.12.0+) |
| Dependency manager | `uv` |

---

## Planned Module Structure

```
obsilm/
├── cli.py              # Typer CLI entry point
├── config.py           # Config management (~/.obsilm/config.toml, state.toml)
├── notebook.py         # Notebook (Vault) CRUD + activate/deactivate
├── source/
│   ├── manager.py      # Source add/delete/list
│   ├── fetcher.py      # URL crawling, PDF extraction, YouTube transcripts
│   └── converter.py    # Convert source types to Markdown
├── ai/
│   ├── client.py       # Claude API client
│   ├── chat.py         # Chat session management
│   ├── memory.py       # memory/ folder I/O
│   └── generator.py    # Content generation (summary, study-guide, flashcards)
└── obsidian.py         # Obsidian CLI wrapper
```

---

## Vault File Structure

Each notebook is an Obsidian vault at `~/obsiLM/notebooks/<notebook-slug>/`:

```
<notebook-slug>/
├── .obsidian/          # Obsidian config (auto-generated)
├── .obsilm             # obsiLM vault identifier flag (YAML)
├── sources/            # Sources converted to Markdown
│   ├── source-001.md
│   └── source-NNN.md
├── generated/          # AI-generated content
│   ├── summary.md
│   ├── study-guide.md
│   └── flashcards.md
├── memory/             # AI memory store (date-based)
│   └── YYYY-MM-DD.md
├── chat/               # Conversation history (date-based)
│   └── YYYY-MM-DD.md
└── notebook.md         # Notebook metadata
```

---

## File Format Conventions

### `.obsilm` (vault identifier flag)
```yaml
version: "1.0"
slug: ai-safety-research
title: "AI Safety Research"
created: 2026-03-18T09:00:00
model: claude-opus-4-6
```

### `notebook.md` frontmatter
```yaml
---
title: "AI Safety Research"
slug: ai-safety-research
created: 2026-03-18T09:00:00
model: claude-opus-4-6
tags: [ai, safety, research]
source_count: 3
---
```

### Source files (`sources/source-NNN.md`) frontmatter
```yaml
---
id: source-001
type: url          # url | pdf | youtube | file
origin: https://example.com/article
title: "Article Title"
added: 2026-03-18T10:00:00
tags: [ai, research]
---
```

### Memory files (`memory/YYYY-MM-DD.md`)
```yaml
---
date: 2026-03-18
type: memory
---
```

---

## Global Config Files

Located at `~/.obsilm/`:

**`config.toml`** — API keys and settings:
```toml
[general]
notebooks_dir = "~/obsiLM/notebooks"
default_model = "claude-opus-4-6"

[api]
anthropic_api_key = ""  # or env var ANTHROPIC_API_KEY
```

**`state.toml`** — Active notebook state:
```toml
[active]
notebook = "ai-safety-research"
path = "/Users/user/obsiLM/notebooks/ai-safety-research"
```

---

## CLI Interface

The CLI uses an **activate-based UX**: activate a notebook once, and all subsequent commands target it automatically. Use `--notebook <slug>` to override explicitly.

### Notebook Management
```bash
obsilm create <title>          # Create + auto-activate notebook
obsilm activate <notebook>     # Activate existing notebook
obsilm deactivate              # Deactivate current notebook
obsilm status                  # Show active notebook info
obsilm list                    # List all obsiLM notebooks (detected by .obsilm file)
obsilm delete [notebook]       # Delete notebook (requires confirmation prompt)
obsilm open [notebook]         # Open vault in Obsidian GUI
obsilm info [notebook]         # Show detailed notebook info
```

### Source Management
```bash
obsilm add --url <url>         # Add URL source
obsilm add --file <path>       # Add file source (.pdf, .txt, .md)
obsilm add --youtube <url>     # Add YouTube transcript as source
obsilm sources                 # List sources in active notebook
obsilm remove-source <id>      # Remove a source (e.g., source-001)
```

### AI Features
```bash
obsilm chat                    # Start interactive chat session
obsilm generate summary        # Generate summary → generated/summary.md
obsilm generate study-guide    # Generate study guide → generated/study-guide.md
obsilm generate flashcards     # Generate flashcards → generated/flashcards.md
```

### Chat Session Commands
Inside `obsilm chat`:
- `/remember <text>` — Save to `memory/YYYY-MM-DD.md`
- `/save` — Save conversation to `chat/YYYY-MM-DD.md`
- `exit` — Exit the session

---

## Key Conventions

1. **No API keys in code** — Always read from `~/.obsilm/config.toml` or `ANTHROPIC_API_KEY` env var.
2. **All vault files are standard Markdown** — Every generated file must be readable in Obsidian without modification.
3. **`.obsilm` flag is the vault identifier** — `obsilm list` and other commands use this file to discover notebooks. Never mistake a plain Obsidian vault for an obsiLM notebook.
4. **Source citations in AI responses** — Responses should include `[source-NNN]` inline citations referencing the source files.
5. **Source file naming** — Sequential zero-padded: `source-001.md`, `source-002.md`, etc.
6. **Date-based file naming** — memory/ and chat/ files use `YYYY-MM-DD.md`.
7. **Notebook slug** — Generated from title: lowercase, spaces replaced with hyphens.
8. **AI model** — Default is `claude-opus-4-6`. Configurable per-notebook in `.obsilm`.

---

## Error Handling

| Situation | Behavior |
|---|---|
| URL unreachable | Print error, abort |
| PDF parse failure | Fallback to pypdf text extraction |
| YouTube no transcript | Print error + manual add instructions |
| File not found | Clear error message |
| API rate limit | Auto-retry with exponential backoff (max 3 attempts) |
| Context window exceeded | Inform user (Phase 2: chunking + RAG) |
| API key not set | Immediate clear error message at startup |
| Interrupted command | Auto-clean partial files |
| `.obsilm` corrupted | Print recovery instructions |
| `notebook.md` missing | Auto-regenerate |

---

## MVP Priorities (Phase 1)

| Feature | Priority |
|---|---|
| Notebook create/list/delete | P0 |
| activate / deactivate / status | P0 |
| URL source add | P0 |
| PDF source add | P0 |
| AI Chat (source-based Q&A) | P0 |
| `generate summary` | P0 |
| memory/ basics (`/remember`) | P1 |
| Obsidian open integration | P1 |
| YouTube source add | P1 |
| `generate study-guide` | P1 |
| `generate flashcards` | P1 |

## Phase 2 (Future)

- Audio overview (podcast-style)
- Mind map generation (Obsidian Canvas)
- Obsidian plugin auto-install
- Context window overflow handling (chunking + RAG)
- AI long-term memory accumulation
- Web UI (optional)

---

## Development Setup

```bash
# Install uv (if not present)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies (once pyproject.toml exists)
uv sync

# Run CLI
uv run obsilm <command>
```

**External dependency:** Obsidian CLI v1.12.0+ (released February 2026) must be installed and on PATH.

---

## Git Workflow

- Main branch: `main` (remote) / `master` (local)
- Feature branches: created per task
- All commits should be descriptive and focused
