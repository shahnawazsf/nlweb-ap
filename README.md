# Build Your Own NLWeb-Protocol AI App — Tutorial & Reference Docs

This folder is a self-contained curriculum for building a **natural-language query API for web/data content** — the same class of system as [NLWebNet](https://github.com/nlweb-ai/nlweb-net) (a .NET implementation of Microsoft's [NLWeb protocol](https://github.com/microsoft/NLWeb)).

It's written for someone starting a **new** project from an empty folder who wants to understand — and rebuild, piece by piece — every domain that a project like this touches: minimal APIs, RAG, LLM integration, streaming, the Model Context Protocol (MCP), configuration, observability, UI, and deployment.

Each chapter is self-contained, explains the concept **from scratch** (not assuming you've read NLWebNet's source), and ends with working code for a running example project we call **`MyNLWeb`**. Code samples throughout are grounded in the real NLWebNet source (referenced by file path) so you can cross-check against a production-quality implementation whenever you want more depth than the tutorial covers.

## How to use this

- If you just want to **understand the architecture**, read `docs/00-overview.md` and stop there.
- If you want to **build the whole thing**, go in order — each chapter's code is the prerequisite for the next.
- If you only care about **one domain** (e.g. "how do I add MCP support to my existing API?"), jump straight to that chapter — each one lists its prerequisites at the top.

## Chapter Index

| # | Chapter | Domain |
|---|---------|--------|
| 00 | [`docs/00-overview.md`](docs/00-overview.md) | What NLWeb is, protocol shape, system architecture |
| 01 | [`docs/01-project-setup.md`](docs/01-project-setup.md) | .NET solution layout, Minimal APIs from scratch |
| 02 | [`docs/02-data-models.md`](docs/02-data-models.md) | Request/response contracts, options pattern |
| 03 | [`docs/03-data-backend.md`](docs/03-data-backend.md) | Pluggable retrieval layer (the "R" in RAG) |
| 04 | [`docs/04-rag-modes.md`](docs/04-rag-modes.md) | List / Summarize / Generate pipeline, RAG from first principles |
| 05 | [`docs/05-ai-integration.md`](docs/05-ai-integration.md) | LLM abstraction via `Microsoft.Extensions.AI`, provider wiring |
| 06 | [`docs/06-streaming.md`](docs/06-streaming.md) | Server-Sent Events, streaming LLM output |
| 07 | [`docs/07-mcp-protocol.md`](docs/07-mcp-protocol.md) | Model Context Protocol — exposing your API as agent tools |
| 08 | [`docs/08-tool-selection.md`](docs/08-tool-selection.md) | Multi-tool routing framework (search/compare/details/etc.) |
| 09 | [`docs/09-configuration.md`](docs/09-configuration.md) | JSON/YAML/XML config, secrets, multi-backend config |
| 10 | [`docs/10-middleware-observability.md`](docs/10-middleware-observability.md) | Rate limiting, metrics, health checks, OpenTelemetry |
| 11 | [`docs/11-testing.md`](docs/11-testing.md) | Unit + integration testing strategy |
| 12 | [`docs/12-ui-layer.md`](docs/12-ui-layer.md) | Blazor front end for the API |
| 13 | [`docs/13-aspire-orchestration.md`](docs/13-aspire-orchestration.md) | Multi-service local orchestration with .NET Aspire |
| 14 | [`docs/14-deployment.md`](docs/14-deployment.md) | Docker, Kubernetes, Azure deployment |
| 15 | [`docs/15-build-order-checklist.md`](docs/15-build-order-checklist.md) | End-to-end build checklist to go from empty folder to running app |

## Source of truth

Everything here is inspired by, and cross-referenced against, the real repository:

- **Reference repo:** https://github.com/nlweb-ai/nlweb-net
- **Protocol spec:** https://github.com/microsoft/NLWeb
- **MCP spec:** https://modelcontextprotocol.io/
- **.NET AI abstractions:** https://learn.microsoft.com/en-us/dotnet/ai/

That repo is explicitly an **alpha / proof-of-concept** — not production ready. Treat it (and this tutorial) as a learning reference, and apply your own hardening (auth, real backends, caching) before shipping.

## License

This tutorial is licensed under [CC BY 4.0](LICENSE).
