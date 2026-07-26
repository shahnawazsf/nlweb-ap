# 15 — Build Order Checklist: Empty Folder to Deployed App

Use this as a literal checklist. Each item links back to the chapter that covers it in depth.

## Phase 1 — Skeleton (get *something* running)

- [ ] `dotnet new sln` + `src/`, `samples/Demo/`, `tests/` projects — [Ch.01](01-project-setup.md)
- [ ] Placeholder `/ask` endpoint returning an echo — [Ch.01](01-project-setup.md)
- [ ] `dotnet run` works, `curl` round-trips — [Ch.01](01-project-setup.md)

## Phase 2 — Protocol contracts

- [ ] `QueryMode`, `NLWebRequest`, `NLWebResult`, `NLWebResponse`, `NLWebOptions` — [Ch.02](02-data-models.md)

## Phase 3 — Retrieval (the part that has to be real before anything else matters)

- [ ] `IDataBackend` interface — [Ch.03](03-data-backend.md)
- [ ] `MockDataBackend` with seed data you can search — [Ch.03](03-data-backend.md)
- [ ] Verify: List mode returns ranked results with no LLM involved

## Phase 4 — The pipeline

- [ ] `IQueryProcessor` (pass-through v1) — [Ch.04](04-rag-modes.md)
- [ ] `IResultGenerator` (stubbed v1) — [Ch.04](04-rag-modes.md)
- [ ] `NLWebService` orchestrator wiring all three modes — [Ch.04](04-rag-modes.md)
- [ ] Verify: all three modes (`list`/`summarize`/`generate`) return *something*, even if generation is a placeholder string

## Phase 5 — Make it actually AI-powered

- [ ] Pick a provider (OpenAI / Azure OpenAI / GitHub Models), install its `Microsoft.Extensions.AI.*` package — [Ch.05](05-ai-integration.md)
- [ ] User secrets set up, key never in `appsettings.json` — [Ch.05](05-ai-integration.md)
- [ ] Real `ResultGenerator` implementation calling `IChatClient` — [Ch.05](05-ai-integration.md)
- [ ] `NullChatClient` fallback so the app runs without a key — [Ch.05](05-ai-integration.md)
- [ ] Verify: Summarize/Generate modes produce real, grounded answers referencing your seed data

## Phase 6 — Streaming

- [ ] `IAsyncEnumerable` streaming path through the pipeline — [Ch.06](06-streaming.md)
- [ ] SSE `/ask/stream` endpoint — [Ch.06](06-streaming.md)
- [ ] Verify: tokens arrive incrementally in a browser `EventSource` test page

## Phase 7 — Expose to AI agents

- [ ] `IMcpService` + `McpService` wrapping the same `INLWebService` — [Ch.07](07-mcp-protocol.md)
- [ ] `/mcp` endpoint: `list_tools`, `call_tool`, `list_prompts`, `get_prompt` — [Ch.07](07-mcp-protocol.md)
- [ ] Verify: `list_tools` returns valid schemas; `call_tool` round-trips through the real pipeline

## Phase 8 — Ship the basics to production (can happen in parallel with Phase 9–10)

- [ ] Correlation IDs in every log line — [Ch.10](10-middleware-observability.md)
- [ ] Rate limiting on `/ask` and `/mcp` — [Ch.10](10-middleware-observability.md)
- [ ] Health checks for backend + AI provider, wired to `/health` — [Ch.10](10-middleware-observability.md)
- [ ] Decide on auth (API key / OAuth / gateway) — don't ship with none by accident — [Ch.14](14-deployment.md)
- [ ] Unit tests for routing + error handling; integration test with `NullChatClient` — [Ch.11](11-testing.md)
- [ ] Dockerfile + `.dockerignore`; confirm no secrets baked into the image — [Ch.14](14-deployment.md)
- [ ] Deploy target chosen (Container Apps / App Service / Kubernetes) — [Ch.14](14-deployment.md)

## Phase 9 — Only if you actually need them (don't build speculatively)

- [ ] Multi-tool routing (`IToolSelector`/`IToolHandler`) — only once generic RAG demonstrably underperforms for specific query intents — [Ch.08](08-tool-selection.md)
- [ ] Multi-format config (YAML/XML) — only if ops actually wants YAML, or tool routing (Ch.08) needs declarative definitions — [Ch.09](09-configuration.md)
- [ ] Metrics/OpenTelemetry tracing — once you have more than one backend/provider to debug across — [Ch.10](10-middleware-observability.md)
- [ ] Blazor (or any) UI — once you want humans, not just API clients, exercising the system — [Ch.12](12-ui-layer.md)
- [ ] .NET Aspire orchestration — once you have more than one local process that needs to talk to another (e.g. a real vector DB) — [Ch.13](13-aspire-orchestration.md)
- [ ] Real backend beyond mock data (Azure AI Search / vector DB / EF) — as soon as you're ready to index real content — [Ch.03.4](03-data-backend.md), [Ch.13.5](13-aspire-orchestration.md)

## A note on scope discipline

The reference NLWebNet repo is explicitly labeled alpha/proof-of-concept despite having all of the above built out — that's a signal, not a warning to ignore. Every phase above adds real operational surface area. Build Phases 1–7 first and get a genuinely useful, demoable system; only pull in Phase 9 items when a real requirement — not "the reference repo has it" — is pulling you toward them.
