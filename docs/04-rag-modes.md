# 04 — RAG From First Principles: The List / Summarize / Generate Pipeline

**Prerequisites:** Chapter 03 (`IDataBackend` working).

## 4.1 What RAG actually is (from scratch)

**Retrieval-Augmented Generation** solves one specific problem: an LLM only knows what was in its training data (stale, generic) plus whatever you put in its prompt (the "context window"). RAG is the pattern of:

1. **Retrieve** — search *your* data for the few documents relevant to the user's question
2. **Augment** — stuff those documents into the LLM's prompt as context
3. **Generate** — ask the LLM to answer the question *using only that context*

That's it. There's no special "RAG API" — it's search + prompt engineering + an LLM call, orchestrated in that order. The value is that the LLM's answer is grounded in your actual content instead of hallucinated from training data.

The NLWeb protocol's three modes are RAG taken apart into its constituent steps, each independently addressable:

- **List** = step 1 only (retrieve, return raw)
- **Summarize** = step 1 + a lightweight step 3 (generate a summary *of the results*, not a full answer)
- **Generate** = the full pipeline (retrieve → augment → generate a grounded answer)

## 4.2 The pieces we're assembling

```
NLWebRequest
    │
    ▼
IQueryProcessor      — decontextualize the query (resolve "the sequel?" using Prev/history)
    │
    ▼
IDataBackend.SearchAsync   — retrieval (Chapter 03)
    │
    ▼
IResultGenerator     — mode-specific step: pass through (List) / summarize / generate (RAG)
    │
    ▼
NLWebResponse
```

## 4.3 `IQueryProcessor` — decontextualization

```csharp
namespace MyNLWeb.Services;

public interface IQueryProcessor
{
    Task<string> ProcessQueryAsync(string query, string? prev, string? decontextualizedQuery,
        CancellationToken cancellationToken = default);
}
```

A minimal first implementation: if the client already sent `DecontextualizedQuery`, use it as-is; otherwise, if there's conversation history in `Prev`, concatenate it and (optionally) ask an LLM to rewrite the query as a standalone question. Start simple:

```csharp
public class QueryProcessor : IQueryProcessor
{
    public Task<string> ProcessQueryAsync(string query, string? prev, string? decontextualizedQuery,
        CancellationToken cancellationToken = default)
    {
        if (!string.IsNullOrWhiteSpace(decontextualizedQuery))
            return Task.FromResult(decontextualizedQuery);

        // v1: no history rewriting yet — just pass the query through.
        // v2 (once Chapter 05's IChatClient exists): ask the LLM to rewrite
        // `query` into a standalone question using `prev` as context.
        return Task.FromResult(query);
    }
}
```

Don't reach for an LLM call here on day one — get the List mode working end-to-end with a pass-through processor first, then come back and add LLM-based rewriting once Chapter 05 gives you a chat client to call.

## 4.4 `IResultGenerator` — the mode-specific step

```csharp
using MyNLWeb.Models;

namespace MyNLWeb.Services;

public interface IResultGenerator
{
    Task<string> GenerateSummaryAsync(string query, IEnumerable<NLWebResult> results,
        CancellationToken cancellationToken = default);

    Task<string> GenerateResponseAsync(string query, IEnumerable<NLWebResult> results,
        CancellationToken cancellationToken = default);
}
```

`GenerateSummaryAsync` (Summarize mode) — a cheap prompt: *"Summarize these N search results in relation to the query."* `GenerateResponseAsync` (Generate mode) — the real RAG prompt: *"Answer the query using only the following context; cite sources."* We implement both for real once Chapter 05 gives us an `IChatClient`. For now, stub them:

```csharp
public class ResultGenerator : IResultGenerator
{
    public Task<string> GenerateSummaryAsync(string query, IEnumerable<NLWebResult> results, CancellationToken ct = default)
        => Task.FromResult($"Found {results.Count()} results for '{query}'."); // replaced in Ch.05

    public Task<string> GenerateResponseAsync(string query, IEnumerable<NLWebResult> results, CancellationToken ct = default)
        => Task.FromResult($"(placeholder RAG answer for '{query}')"); // replaced in Ch.05
}
```

## 4.5 `NLWebService` — the orchestrator

This is the piece that ties everything together. Modeled on the real `src/NLWebNet/Services/NLWebService.cs`:

```csharp
using MyNLWeb.Models;

namespace MyNLWeb.Services;

public interface INLWebService
{
    Task<NLWebResponse> ProcessRequestAsync(NLWebRequest request, CancellationToken cancellationToken = default);
}

public class NLWebService : INLWebService
{
    private readonly IQueryProcessor _queryProcessor;
    private readonly IDataBackend _backend;
    private readonly IResultGenerator _resultGenerator;
    private readonly ILogger<NLWebService> _logger;

    public NLWebService(IQueryProcessor queryProcessor, IDataBackend backend,
        IResultGenerator resultGenerator, ILogger<NLWebService> logger)
    {
        _queryProcessor = queryProcessor;
        _backend = backend;
        _resultGenerator = resultGenerator;
        _logger = logger;
    }

    public async Task<NLWebResponse> ProcessRequestAsync(NLWebRequest request, CancellationToken cancellationToken = default)
    {
        var queryId = request.QueryId ?? Guid.NewGuid().ToString();
        _logger.LogInformation("Processing {QueryId} mode={Mode}", queryId, request.Mode);

        try
        {
            var effectiveQuery = await _queryProcessor.ProcessQueryAsync(
                request.Query, request.Prev, request.DecontextualizedQuery, cancellationToken);

            var results = (await _backend.SearchAsync(effectiveQuery, request.Site, cancellationToken: cancellationToken)).ToList();

            var response = new NLWebResponse { QueryId = queryId, Results = results };

            response.Summary = request.Mode switch
            {
                QueryMode.List => null,
                QueryMode.Summarize => await _resultGenerator.GenerateSummaryAsync(effectiveQuery, results, cancellationToken),
                QueryMode.Generate => await _resultGenerator.GenerateResponseAsync(effectiveQuery, results, cancellationToken),
                _ => null
            };

            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed processing {QueryId}", queryId);
            return new NLWebResponse
            {
                QueryId = queryId,
                Results = new(),
                Summary = "An error occurred while processing your query."
            };
        }
    }
}
```

Note the **graceful degradation** pattern here (also used by the reference implementation, including around its optional tool-execution step covered in Chapter 08): catch exceptions at the orchestration boundary and return a usable (if apologetic) response rather than a 500, since a partial/degraded answer is usually better UX than a hard failure for a conversational interface.

## 4.6 Wire it into the endpoint

Update `AskEndpoints.cs` from Chapter 02:

```csharp
group.MapPost("/", async (NLWebRequest request, INLWebService service, CancellationToken ct) =>
{
    if (string.IsNullOrWhiteSpace(request.Query))
        return Results.BadRequest(new { title = "Bad Request", detail = "query is required" });

    var response = await service.ProcessRequestAsync(request, ct);
    return Results.Ok(response);
});
```

Register services:

```csharp
builder.Services.AddSingleton<IDataBackend, MockDataBackend>();
builder.Services.AddScoped<IQueryProcessor, QueryProcessor>();
builder.Services.AddScoped<IResultGenerator, ResultGenerator>();
builder.Services.AddScoped<INLWebService, NLWebService>();
```

## 4.7 Test all three modes

```bash
curl -X POST localhost:5037/ask -H "Content-Type: application/json" -d '{"query":"getting started","mode":"list"}'
curl -X POST localhost:5037/ask -H "Content-Type: application/json" -d '{"query":"getting started","mode":"summarize"}'
curl -X POST localhost:5037/ask -H "Content-Type: application/json" -d '{"query":"getting started","mode":"generate"}'
```

You now have a fully working (if not yet AI-powered) NLWeb-protocol server. Everything from here — real LLM calls, streaming, MCP, tool routing — augments this pipeline; none of it replaces it.

Next: [`05-ai-integration.md`](05-ai-integration.md) — replace the placeholder summary/generate logic with real LLM calls.
