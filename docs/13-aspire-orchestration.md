# 13 — Multi-Service Orchestration with .NET Aspire

**Prerequisites:** Chapters 03 (backend), 05 (AI integration), 12 (UI) — this chapter composes them.

## 13.1 The problem this solves

Once your project grows past "one API + mock data," you'll have several moving parts: the API itself, a vector database (for real semantic search), an ingestion service (pulling in RSS/documents to index), and a frontend. Running each with a separate `dotnet run` in a separate terminal, wiring connection strings by hand, doesn't scale past a couple of services. **.NET Aspire** is an orchestration layer for exactly this: one `AppHost` project declares the topology, starts everything together, and injects service-discovery configuration automatically.

This mirrors the reference repo's second, more advanced sample: `samples/AspireDemo/`, distinct from the simple `samples/Demo/` covered in Chapters 01–12.

## 13.2 When to bother with this

Not every project needs it. Reach for Aspire when you have **more than one runnable process** that need to talk to each other locally (API + vector DB + frontend + ingestion worker). If you're still on a single API process with a mock backend, skip this chapter until you've built a real backend (Chapter 03.4) that itself needs a service (like Qdrant) running alongside your API.

## 13.3 AppHost project

```bash
dotnet new aspire-apphost -n MyNLWeb.AspireHost -o samples/AspireDemo/AspireHost
dotnet new aspire-servicedefaults -n MyNLWeb.ServiceDefaults -o samples/AspireDemo/ServiceDefaults
dotnet new webapi -n MyNLWeb.AspireApp -o samples/AspireDemo/AspireApp
dotnet new blazor -n MyNLWeb.Frontend -o samples/AspireDemo/Frontend

dotnet add samples/AspireDemo/AspireHost reference samples/AspireDemo/AspireApp
dotnet add samples/AspireDemo/AspireHost reference samples/AspireDemo/Frontend
dotnet add samples/AspireDemo/AspireApp reference samples/AspireDemo/ServiceDefaults
dotnet add samples/AspireDemo/AspireApp reference src/MyNLWeb
dotnet add samples/AspireDemo/Frontend reference samples/AspireDemo/ServiceDefaults

dotnet add samples/AspireDemo/AspireHost package Aspire.Hosting.Qdrant   # for builder.AddQdrant(...) below
dotnet add samples/AspireDemo/AspireApp package Qdrant.Client            # for QdrantClient, used in 13.5
```

`MyNLWeb.AspireApp` is a fresh copy of the `/ask` API from Chapters 01–12 (same `IDataBackend`/`INLWebService` wiring), and `MyNLWeb.Frontend` is the Blazor UI from Chapter 12 — both live under `samples/AspireDemo/` rather than `samples/Demo/`, since this is the "several moving parts" sample. Referencing each project *from the AppHost* is what makes them available as generated `Projects.*` types: Aspire emits one static class per referenced project at build time, turning dots and hyphens into underscores (`MyNLWeb.AspireApp` → `Projects.MyNLWeb_AspireApp`, `MyNLWeb.Frontend` → `Projects.MyNLWeb_Frontend`) — that's why `AspireHost/Program.cs` below can reference them directly.

`AspireHost/Program.cs` declares the topology:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var vectorDb = builder.AddQdrant("qdrant")
    .WithDataVolume(); // persists between runs

var api = builder.AddProject<Projects.MyNLWeb_AspireApp>("api")
    .WithReference(vectorDb)
    .WithEnvironment("VectorDb__Endpoint", vectorDb.GetEndpoint("http"));

builder.AddProject<Projects.MyNLWeb_Frontend>("frontend")
    .WithReference(api)
    .WaitFor(api);

builder.Build().Run();
```

`WithReference` + service discovery means the frontend project can call the API using a *logical* name (`http://api`) instead of a hardcoded port — Aspire resolves it at runtime and injects the actual address as configuration.

## 13.4 `ServiceDefaults` — shared cross-cutting config

Every project in the Aspire topology should call one shared extension method that wires up OpenTelemetry, health checks, and service discovery consistently (this is a real pattern from the reference `ServiceDefaults/Extensions.cs`, not something specific to this tutorial):

```csharp
// In ServiceDefaults/Extensions.cs
public static IHostApplicationBuilder AddServiceDefaults(this IHostApplicationBuilder builder)
{
    builder.ConfigureOpenTelemetry();
    builder.AddDefaultHealthChecks();
    builder.Services.AddServiceDiscovery();
    builder.Services.ConfigureHttpClientDefaults(http => http.AddStandardResilienceHandler());
    return builder;
}
```

Every project (`api`, `frontend`, an ingestion worker) calls `builder.AddServiceDefaults()` once at the top of its `Program.cs` — this is what Chapter 10's observability work looks like once applied consistently across N services instead of one.

## 13.5 Vector search backend (a concrete `IDataBackend` implementation)

This is where Chapter 03.4's "build a real backend" advice becomes concrete. A Qdrant-backed `IDataBackend`:

```csharp
public class QdrantDataBackend : IDataBackend
{
    private readonly QdrantClient _client;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embedder; // from Microsoft.Extensions.AI

    public async Task<IEnumerable<NLWebResult>> SearchAsync(
        string query, string? site = null, int maxResults = 10, CancellationToken ct = default)
    {
        var queryVector = await _embedder.GenerateEmbeddingVectorAsync(query, cancellationToken: ct);
        var hits = await _client.SearchAsync("documents", queryVector.ToArray(), limit: (ulong)maxResults);
        return hits.Select(h => new NLWebResult
        {
            Url = h.Payload["url"].StringValue,
            Name = h.Payload["title"].StringValue,
            Score = h.Score,
            Description = h.Payload["snippet"].StringValue
        });
    }
    // ...GetAvailableSitesAsync, GetItemByUrlAsync, GetCapabilities (SupportsSemanticSearch = true)
}
```

`IEmbeddingGenerator<string, Embedding<float>>` is `Microsoft.Extensions.AI`'s abstraction for turning text into vectors — the same cross-provider philosophy as `IChatClient` from Chapter 05, just for embeddings instead of chat completions. An **ingestion service** (a background/hosted service) is what populates this index in the first place — e.g. pulling an RSS feed on a timer, embedding each item, upserting into Qdrant.

## 13.6 Running it

```bash
cd samples/AspireDemo/AspireHost
dotnet run
```

Aspire opens a dashboard (default `http://localhost:15888` or similar) showing every service's logs, traces, and health in one place — genuinely useful once you're debugging "why is the frontend timing out talking to the API" across process boundaries.

Next: [`14-deployment.md`](14-deployment.md) — take this from `dotnet run` to a deployed service.
