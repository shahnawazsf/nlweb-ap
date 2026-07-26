# 00 — Overview: What NLWeb Is and How This System Fits Together

## The problem

Most websites/apps expose content through pages and traditional search boxes (keyword match → list of links). Users increasingly want to just **ask a question** and get an answer, the way they would from ChatGPT — but grounded in *your* content, not the open internet.

**NLWeb** is a protocol (originated by Microsoft, spec at [microsoft/NLWeb](https://github.com/microsoft/NLWeb)) that standardizes how a site/backend exposes itself for natural-language querying, so any client — a web UI, a mobile app, or an AI agent — can talk to it the same way regardless of what's actually indexing the content underneath.

**NLWebNet** is a .NET/ASP.NET Core implementation of that protocol. This tutorial rebuilds its architecture, piece by piece, under a new project we'll call `MyNLWeb`.

## The three query modes

Every NLWeb-protocol server answers a natural-language query in one of three modes:

| Mode | What happens | Analogy |
|------|-------------|---------|
| **List** | Search backend, return ranked raw results | Classic search results page |
| **Summarize** | Search backend, then have an LLM summarize the results | "Here's the gist of what I found" |
| **Generate** | Search backend for context, then have an LLM generate a full answer grounded in it | Full RAG chat answer |

This is a deliberate design: **List** costs nothing beyond your search backend (no LLM call), **Summarize** adds a cheap LLM pass over already-ranked results, **Generate** is full RAG. A client picks the mode that fits its latency/cost/quality needs per-request.

## The two endpoints

1. **`/ask`** — the human/application-facing endpoint. Natural language in, structured JSON (or a stream) out.
2. **`/mcp`** — the same capability, exposed via the **Model Context Protocol**, so AI agents (Claude Desktop, other MCP clients) can discover and call your search/summarize/generate capability as a *tool*, without you writing any agent-specific integration code.

This is the key insight: **build the retrieval + RAG pipeline once, expose it twice** (once for direct HTTP clients, once for AI agents via MCP).

## Request/response shape (protocol contract)

A request to `/ask`:

```json
{
  "query": "what are the main features?",
  "mode": "generate",
  "site": "docs",
  "prev": "previous query 1, previous query 2",
  "streaming": false,
  "query_id": "optional-custom-id"
}
```

A response:

```json
{
  "query_id": "abc123",
  "results": [
    { "url": "...", "name": "...", "site": "...", "score": 8.4, "description": "...", "schema_object": { } }
  ],
  "summary": "optional, present in summarize/generate modes"
}
```

`schema_object` deliberately mirrors [schema.org](https://schema.org) JSON-LD conventions — a pragmatic choice to piggyback on an existing structured-data vocabulary instead of inventing a new one.

## System architecture

```
                         ┌───────────────────────────────┐
  HTTP client ──POST /ask──▶│                                │
                         │        MyNLWeb API layer      │
  AI agent (MCP) ──/mcp──▶│  (Minimal APIs / Controllers)  │
                         └───────────────┬───────────────┘
                                         │
                                 ┌───────▼────────┐
                                 │  NLWebService   │  ← orchestrator
                                 │ (the pipeline)  │
                                 └───┬─────────┬───┘
                     ┌───────────────┘         └───────────────┐
             ┌───────▼────────┐                        ┌───────▼────────┐
             │ IDataBackend    │                        │ IChatClient     │
             │ (search index,  │                        │ (LLM: Azure     │
             │  DB, vector db) │                        │  OpenAI, etc.)  │
             └─────────────────┘                        └─────────────────┘
```

Every box above becomes its own chapter:

- **API layer** → Chapter 01 (Minimal APIs) + Chapter 07 (MCP)
- **Request/response contracts** → Chapter 02
- **`IDataBackend`** → Chapter 03
- **The orchestration pipeline (List/Summarize/Generate)** → Chapter 04
- **`IChatClient` / LLM wiring** → Chapter 05
- **Streaming** → Chapter 06
- **Multi-tool routing on top of the basic pipeline** → Chapter 08
- **Everything needed to run this in production** → Chapters 09–14

## Reference repo structure (what you're rebuilding)

```
MyNLWeb/
├── src/MyNLWeb/                # Core library (the reusable NuGet-able piece)
│   ├── Models/                 # NLWebRequest, NLWebResponse, NLWebResult, QueryMode, options
│   ├── Services/                # IDataBackend, IQueryProcessor, IResultGenerator, NLWebService...
│   ├── Endpoints/                # Minimal API endpoint mappings (/ask)
│   ├── MCP/                      # IMcpService, McpService (/mcp)
│   ├── Middleware/              # Rate limiting, metrics, correlation IDs
│   ├── Health/                   # Health checks
│   └── Extensions/              # DI registration (AddMyNLWeb / MapMyNLWeb)
├── samples/Demo/                 # A runnable ASP.NET Core + Blazor app that hosts the library
├── tests/MyNLWeb.Tests/          # Unit + integration tests
└── deployment/                    # Docker / Kubernetes / Azure
```

This mirrors NLWebNet's actual layout — `src/NLWebNet/{Models,Services,Endpoints,MCP,Middleware,Health,Extensions}`, `samples/Demo`, `tests/NLWebNet.Tests`, `deployment/`. Building `MyNLWeb` in the same shape means anything you don't build yourself, you can read directly from the reference implementation.

## Prerequisites for the tutorial

- .NET 10 SDK (NLWebNet targets .NET 10; .NET 8/9 works fine too if that's what you have installed — nothing here is version-specific beyond Minimal API support, which has existed since .NET 6)
- A code editor (VS Code, Visual Studio, Rider)
- Optional for later chapters: an Azure OpenAI or OpenAI API key, Docker Desktop

Next: [`01-project-setup.md`](01-project-setup.md) — scaffold the solution.
