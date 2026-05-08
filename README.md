# LLM Wiki

A repo for experimenting with **LLM Wiki** — a concept by Andrej Karpathy where an AI agent builds and maintains a persistent, interlinked knowledge base (like a personal Wikipedia) from your documents.

Instead of a one-shot RAG system, LLM Wiki incrementally reads, writes, and maintains a folder of Markdown files. Each file is an entity page connected to others via `[[wiki-links]]`, and knowledge compounds over time.

---

## Implementations Overview

| Implementation | Type | Best For | Local Model Support |
|---|---|---|---|
| [nashsu/llm_wiki](#1-nashsullm_wiki--desktop-app) | Desktop GUI | Visual knowledge graph, document ingestion | Yes (Ollama) |
| [nvk/llm-wiki](#2-nvkllm-wiki--agent-plugin) | CLI Plugin | Parallel research, Claude Code / Codex users | Yes (via AGENTS.md) |
| [Pratiyush/llm-wiki](#3-pratiyushllm-wiki--transcript-to-wiki) | CLI Tool | Converting LLM chat transcripts into a wiki | No (static HTML) |
| [lucasastorian/llmwiki](#4-lucasastorianllmwiki--mcp-powered-wiki) | MCP + Web UI | Auto-maintaining personal wiki from file collections | No (Claude only) |

---

## Which Model Should You Use?

Since you have **Ollama** and **vLLM** available, here is the recommendation:

### TL;DR Recommendation

| Use Case | Tool | Model |
|---|---|---|
| Best quality, long documents | vLLM | `Qwen2.5-14B-Instruct` (128K ctx) |
| Best reasoning, complex topics | vLLM | `Llama-3.1-70B-Instruct` (128K ctx) |
| Very large document batches | vLLM | `Qwen2.5-1M` (1M ctx) |
| Quick local prototyping | Ollama | `qwen2.5:14b` or `llama3.1:8b` |

### Ollama vs vLLM for LLM Wiki

**Use vLLM** when you care about performance:
- ~3x better throughput and ~6x lower latency for concurrent requests
- PagedAttention handles long-context documents efficiently
- Better for batch document ingestion pipelines

**Use Ollama** when you want simplicity:
- One-command model pull and run
- Great for single-request workloads and prototyping
- Easier setup, no GPU config required

### Recommended Models

**For Ollama:**

```bash
# Best balance of quality + speed (recommended starting point)
ollama pull qwen2.5:14b        # 128K context, excellent for structured writing

# Best reasoning quality
ollama pull llama3.1:70b       # 128K context, needs ~48GB VRAM

# Lightweight, fast
ollama pull llama3.1:8b        # 128K context, runs on 8GB VRAM

# Long documents
ollama pull qwen2.5:32b        # 128K context, needs ~24GB VRAM
```

**For vLLM:**

```bash
# Recommended — fast + long context
vllm serve Qwen/Qwen2.5-14B-Instruct --max-model-len 131072

# Best quality
vllm serve meta-llama/Llama-3.1-70B-Instruct --max-model-len 131072

# Extreme long context (research docs, books)
vllm serve Qwen/Qwen2.5-1M --enable-chunked-prefill --max-model-len 1000000
```

Once vLLM is running it exposes an **OpenAI-compatible API at `http://localhost:8000`** — most LLM Wiki tools can use this directly.

---

## 1. nashsu/llm_wiki — Desktop App

**Best for:** Visual knowledge graph, multi-format document ingestion, GUI users.

A cross-platform desktop app (Tauri + React) with a three-column layout: Knowledge Tree, Chat, and Preview. Includes a knowledge graph visualizer, source tracing, and a Chrome extension for web clipping.

### Prerequisites

- Node.js 20+
- Rust 1.70+ (build from source only)
- API key for your LLM provider, **or** a running Ollama/vLLM instance

### Install — Pre-built Binary (Easiest)

Download from the [Releases page](https://github.com/nashsu/llm_wiki/releases):

| OS | File |
|---|---|
| macOS | `.dmg` |
| Windows | `.msi` |
| Linux | `.deb` / `.AppImage` |

### Install — Build from Source

```bash
git clone https://github.com/nashsu/llm_wiki.git
cd llm_wiki
npm install

# Development mode
npm run tauri dev

# Production build
npm run tauri build
```

### Configuration

1. Open the app → **Settings**
2. Set **LLM Provider**:
   - Cloud: Anthropic / OpenAI / Google (requires API key)
   - Local: Select **Ollama** or any **Custom OpenAI-compatible endpoint**
3. For Ollama: set endpoint to `http://localhost:11434`
4. For vLLM: set endpoint to `http://localhost:8000` (OpenAI-compatible)
5. Select your model from the dropdown (e.g. `qwen2.5:14b`)
6. Set **Context Window** size with the slider (match your model's max)

### Running Your First Wiki

1. Launch → **New Project** → choose a template
2. Go to **Sources** → import documents (PDF, DOCX, Markdown, images, video, web URLs)
3. Watch the **Activity Panel** — the agent analyzes then generates wiki pages
4. Open **Knowledge Graph** to explore connections
5. Use **Chat** to query your wiki
6. Run **Lint** to maintain quality and fix broken links

### Chrome Extension Setup

```
chrome://extensions → Enable Developer mode → Load unpacked → select extension/ folder
```

---

## 2. nvk/llm-wiki — Agent Plugin

**Best for:** Parallel multi-agent research, users already using Claude Code or Codex CLI.

A zero-dependency agent plugin that runs 5–10 parallel research agents from different angles, compiles cross-referenced wiki articles with fact traceability, and generates reports, slides, and study guides.

### Install

Choose the mode that matches your agent:

**Claude Code (recommended, 22K token context):**
```bash
claude plugin install wiki@llm-wiki
```

**OpenAI Codex:**
```bash
codex plugin marketplace add nvk/llm-wiki
```

**Any LLM agent (universal):**
```bash
# Clone the repo and copy AGENTS.md into your project
git clone https://github.com/nvk/llm-wiki.git
cp llm-wiki/AGENTS.md ./AGENTS.md
```

**Local models via Pi or Ollama:**

Copy the `AGENTS.md` and pass it as a system prompt or instruction file to your local agent runner. The plugin uses only file read/write and web fetch — no external APIs needed beyond the LLM itself.

### Usage

```bash
# Initialize wiki (creates ~/wiki/ directory)
/wiki init

# Research a topic — launches parallel agents
/wiki:research "topic" --sources 10

# Ingest documents from inbox
/wiki ingest --all

# Query your wiki
@wiki query "your question"

# Audit for gaps and inconsistencies
@wiki audit

# Deep research with iterative gap-closing
/wiki:research "topic" --min-time 30
```

### Configuration

No separate API key needed if using Claude Code or Codex — it reuses your existing subscription. For local models, set your agent runner to use your Ollama/vLLM endpoint and pass the `AGENTS.md` as the system prompt.

### Provider Context Window Comparison

| Mode | Provider | Context |
|---|---|---|
| Claude Code | Claude | 22K token system prompt |
| Codex | OpenAI | 3K tokens |
| OpenCode | Multi-provider | 3K tokens |
| AGENTS.md | Any (local/cloud) | Depends on model |

---

## 3. Pratiyush/llm-wiki — Transcript to Wiki

**Best for:** Converting your existing LLM chat session transcripts into a browsable, searchable static wiki.

Takes `.jsonl` transcript files from Claude Code, Cursor, Codex CLI, Gemini CLI, Copilot, or Obsidian and builds a static HTML knowledge base from them. Fully offline after build.

### Prerequisites

- Python 3.8+
- Git

### Install

```bash
git clone https://github.com/Pratiyush/llm-wiki.git
cd llm-wiki

# Run setup script
./setup.sh          # Linux/macOS
setup.bat           # Windows

# Install Python package (optional extras)
pip install -e .
```

### Usage

```bash
# Step 1: Convert transcripts into wiki entries
llmwiki sync

# Step 2: Build the static HTML site
llmwiki build

# Step 3: Serve locally and browse
llmwiki serve
# Open http://localhost:8080
```

### Supported Transcript Sources

| Tool | Format |
|---|---|
| Claude Code | `.jsonl` |
| Codex CLI | `.jsonl` |
| Cursor | `.jsonl` |
| Gemini CLI | `.jsonl` |
| Obsidian | Vault overlay |
| Copilot Chat/CLI | `.jsonl` |

### Key Features

- **Lifecycle states:** draft → verified → stale → archived
- **AI-consumable exports:** `llms.txt`, JSON-LD graph, RSS feed
- **MCP server** with 12 tools for querying the wiki from agents
- **Auto-redacts** usernames and API keys from transcripts
- **Dark mode** with keyboard shortcuts

### MCP Server (query wiki from agents)

```bash
llmwiki mcp start
# Exposes 12 tools for your agent to search and read the wiki
```

---

## 4. lucasastorian/llmwiki — MCP-Powered Wiki

**Best for:** Auto-maintaining a personal wiki from a folder of documents, with Claude doing the heavy lifting via MCP.

One command points it at a directory, Claude indexes everything, generates wiki pages, and keeps links and summaries up to date automatically. Local SQLite index — no cloud infrastructure needed.

### Prerequisites

- Python 3.11+
- Node.js 20+
- Anthropic API key

### Install

```bash
git clone https://github.com/lucasastorian/llmwiki.git
cd llmwiki

# Install Python API dependencies
cd api && pip install -r requirements.txt && cd ..

# Install Node web UI dependencies
cd web && npm install && cd ..

# Initialize and index your documents
./llmwiki init
```

### Usage

```bash
# One command — open a folder as a wiki
./llmwiki open ~/research

# Or run components separately
./llmwiki serve         # Start the web UI
./llmwiki reindex       # Re-scan and update wiki pages
```

### Configuration

```bash
# Set your Anthropic API key
export ANTHROPIC_API_KEY=your_key_here

# Optional: Mistral API for enhanced PDF/OCR
export MISTRAL_API_KEY=your_key_here
```

Open `http://localhost:3000` to browse your wiki.

### Key Features

- Filesystem remains the **source of truth** — wiki is generated on top of your files, not a replacement
- **Auto-updates** links, summaries, and citations when files change
- Local **SQLite index** — no cloud search service needed
- Optional **Mistral** integration for better PDF and scanned document OCR

---

## Running Locally with Ollama (Quick Reference)

```bash
# 1. Pull your chosen model
ollama pull qwen2.5:14b

# 2. Ollama API is available at http://localhost:11434
#    OpenAI-compatible endpoint: http://localhost:11434/v1

# 3. In nashsu/llm_wiki Settings:
#    Provider → Ollama
#    Endpoint → http://localhost:11434
#    Model → qwen2.5:14b

# 4. In nvk/llm-wiki AGENTS.md mode:
#    Pass AGENTS.md as system prompt to any Ollama-compatible runner
```

## Running Locally with vLLM (Quick Reference)

```bash
# 1. Install vLLM
pip install vllm

# 2. Serve a model (OpenAI-compatible API)
vllm serve Qwen/Qwen2.5-14B-Instruct \
  --max-model-len 131072 \
  --host 0.0.0.0 \
  --port 8000

# 3. In nashsu/llm_wiki Settings:
#    Provider → Custom (OpenAI-compatible)
#    Endpoint → http://localhost:8000/v1
#    API Key → none (or any string)
#    Model → Qwen/Qwen2.5-14B-Instruct

# 4. Test the endpoint directly
curl http://localhost:8000/v1/models
```

---

## Resources

- [nashsu/llm_wiki](https://github.com/nashsu/llm_wiki) — Desktop app
- [nvk/llm-wiki](https://github.com/nvk/llm-wiki) — Agent plugin
- [Pratiyush/llm-wiki](https://github.com/Pratiyush/llm-wiki) — Transcript wiki
- [lucasastorian/llmwiki](https://github.com/lucasastorian/llmwiki) — MCP wiki
- [Anthropic Console](https://console.anthropic.com) — API key
- [Ollama Library](https://ollama.com/library) — Available models
- [vLLM Docs](https://docs.vllm.ai) — vLLM setup and config
