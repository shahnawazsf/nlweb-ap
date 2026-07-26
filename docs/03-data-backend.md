# 03 — The Data Backend: A Pluggable Retrieval Layer

**Prerequisites:** Chapter 02 (models defined).

This is the **"R" in RAG** — the retrieval step. Everything downstream (summarization, generation, MCP tools) depends on a working retrieval layer, so we build it before touching any LLM code.

## 3.1 Why abstract it behind an interface

Real projects will back this with Azure AI Search, Elasticsearch, a vector DB (Qdrant/Pinecone), or a plain SQL `LIKE` query. You don't want your pipeline (Chapter 04) coupled to any one of them. Define the contract once; swap implementations freely — this is the Strategy pattern applied to search.

`src/MyNLWeb/Services/IDataBackend.cs`, modeled on the real `src/NLWebNet/Services/IDataBackend.cs`:

```csharp
using MyNLWeb.Models;

namespace MyNLWeb.Services;

public interface IDataBackend
{
    /// <summary>Executes a search and returns up to maxResults ranked results.</summary>
    Task<IEnumerable<NLWebResult>> SearchAsync(
        string query, string? site = null, int maxResults = 10,
        CancellationToken cancellationToken = default);

    /// <summary>Lists the distinct site/domain values this backend can filter on.</summary>
    Task<IEnumerable<string>> GetAvailableSitesAsync(CancellationToken cancellationToken = default);

    /// <summary>Fetches a single item by its URL/identifier (used by MCP "details" tools, deep links).</summary>
    Task<NLWebResult?> GetItemByUrlAsync(string url, CancellationToken cancellationToken = default);

    /// <summary>Describes what this backend implementation supports.</summary>
    BackendCapabilities GetCapabilities();
}

public record BackendCapabilities
{
    public bool SupportsSiteFiltering { get; init; }
    public bool SupportsFullTextSearch { get; init; } = true;
    public bool SupportsSemanticSearch { get; init; }
    public int MaxResults { get; init; } = 100;
    public string Description { get; init; } = string.Empty;
}
```

`GetCapabilities()` matters more than it looks: it lets the pipeline (and any UI) *ask* a backend what it can do instead of hard-coding assumptions — e.g. don't offer semantic search in a UI if the active backend reports `SupportsSemanticSearch = false`.

## 3.2 A mock backend to develop against

Before wiring a real search index, build something you can run immediately. This is exactly what NLWebNet's `MockDataBackend` (`src/NLWebNet/Services/MockDataBackend.cs`) is for — a small in-memory dataset with basic term-matching relevance scoring:

```csharp
using Microsoft.Extensions.Logging;
using MyNLWeb.Models;

namespace MyNLWeb.Services;

public class MockDataBackend : IDataBackend
{
    private readonly ILogger<MockDataBackend> _logger;
    private readonly List<NLWebResult> _data;

    public MockDataBackend(ILogger<MockDataBackend> logger)
    {
        _logger = logger;
        _data = SeedData();
    }

    public Task<IEnumerable<NLWebResult>> SearchAsync(
        string query, string? site = null, int maxResults = 10,
        CancellationToken cancellationToken = default)
    {
        if (string.IsNullOrWhiteSpace(query))
            return Task.FromResult(Enumerable.Empty<NLWebResult>());

        var terms = query.ToLowerInvariant()
            .Split(' ', StringSplitOptions.RemoveEmptyEntries)
            .Where(t => t.Length > 2)
            .ToList();

        var ranked = _data
            .Where(item => site == null || item.Site == site)
            .Select(item => (item, score: Score(item, terms)))
            .Where(r => r.score > 0)
            .OrderByDescending(r => r.score)
            .Take(Math.Min(maxResults, 50))
            .Select(r => new NLWebResult
            {
                Url = r.item.Url, Name = r.item.Name, Site = r.item.Site,
                Score = r.score, Description = r.item.Description,
                SchemaObject = r.item.SchemaObject
            });

        return Task.FromResult(ranked.AsEnumerable());
    }

    public Task<IEnumerable<string>> GetAvailableSitesAsync(CancellationToken ct = default) =>
        Task.FromResult(_data.Select(d => d.Site).Where(s => s != null).Distinct()!.AsEnumerable<string>());

    public Task<NLWebResult?> GetItemByUrlAsync(string url, CancellationToken ct = default) =>
        Task.FromResult(_data.FirstOrDefault(d => d.Url == url));

    public BackendCapabilities GetCapabilities() => new()
    {
        SupportsSiteFiltering = true,
        SupportsFullTextSearch = true,
        SupportsSemanticSearch = false,
        MaxResults = 50,
        Description = "In-memory mock backend for local development"
    };

    private static double Score(NLWebResult item, List<string> terms)
    {
        double score = 0;
        var text = $"{item.Name} {item.Description}".ToLowerInvariant();
        foreach (var t in terms)
        {
            if (item.Name.ToLowerInvariant().Contains(t)) score += 10;
            else if (item.Description?.ToLowerInvariant().Contains(t) == true) score += 5;
        }
        var matches = terms.Count(t => text.Contains(t));
        if (matches > 1) score *= 1.0 + 0.2 * (matches - 1);
        return Math.Round(score, 2);
    }

    private static List<NLWebResult> SeedData() => new()
    {
        new() { Url = "https://example.com/a", Name = "Getting Started Guide", Site = "docs",
                Description = "How to install and configure the product." },
        new() { Url = "https://example.com/b", Name = "API Reference", Site = "docs",
                Description = "Full reference for all API endpoints." },
        // add your own seed rows...
    };
}
```

This relevance scoring is deliberately naive (substring matching with weighted boosts) — good enough to develop the pipeline against, useless for production. Chapter 03.4 below covers swapping in something real.

## 3.3 Register it

```csharp
builder.Services.AddSingleton<IDataBackend, MockDataBackend>();
```

Singleton is appropriate here because the mock backend holds static in-memory data; a real backend wrapping an HTTP client to Azure AI Search would typically be registered the same way (the client itself is thread-safe and expensive to construct).

## 3.4 Building a real backend later

When you're ready to replace the mock, implement `IDataBackend` against:

- **Azure AI Search** — call `SearchClient.SearchAsync`, map hits to `NLWebResult`
- **A vector database** (Qdrant, Pinecone, pgvector) — embed the query with an embedding model (see Chapter 05), do a similarity search, map hits
- **A plain database** — full-text or `LIKE` query, ranked by a scoring column

The rest of your pipeline (Chapter 04 onward) never changes — that's the entire point of the interface. NLWebNet's own roadmap (`doc/design-decisions.md`) lists exactly this as an open area: "which real data sources should have first-class support" — Azure Cognitive Search, Elasticsearch/OpenSearch, vector DBs, EF-backed DB search are all called out as natural next backends.

## 3.5 Multiple backends at once

A more advanced pattern (covered fully in Chapter 09) is querying **several backends concurrently** and merging results — e.g. one backend for docs, one for a product catalog, one for a vector store — via an `IBackendManager` that fans a query out and combines/re-ranks the results. Don't build this until you have at least two real backends that justify it.

Next: [`04-rag-modes.md`](04-rag-modes.md) — wire the backend into the List/Summarize/Generate pipeline.
