# 02 — Data Models: Request/Response Contracts & the Options Pattern

**Prerequisites:** Chapter 01 (solution scaffolded).

## 2.1 Why model the protocol as explicit types

The NLWeb protocol is a *contract*, not just "whatever your API happens to return." Modeling it as explicit C# classes gets you:

- Automatic JSON schema generation for OpenAPI/Swagger
- Compile-time safety when the pipeline is extended
- A single source of truth other developers (and AI agents, via MCP) can read to understand what your API accepts/returns

## 2.2 `QueryMode` enum

`src/MyNLWeb/Models/QueryMode.cs`:

```csharp
namespace MyNLWeb.Models;

/// <summary>The three NLWeb protocol response modes.</summary>
public enum QueryMode
{
    List,
    Summarize,
    Generate
}
```

## 2.3 `NLWebRequest` — the incoming contract

Modeled directly on the real `src/NLWebNet/Models/NLWebRequest.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using System.Text.Json.Serialization;

namespace MyNLWeb.Models;

public class NLWebRequest
{
    /// <summary>The natural-language query. Only required field.</summary>
    [Required]
    [JsonPropertyName("query")]
    public string Query { get; set; } = string.Empty;

    /// <summary>Restricts the search to a subset of the backend's data (e.g. a site/domain).</summary>
    [JsonPropertyName("site")]
    public string? Site { get; set; }

    /// <summary>Comma-separated prior queries, used to decontextualize follow-up questions.</summary>
    [JsonPropertyName("prev")]
    public string? Prev { get; set; }

    /// <summary>Pre-decontextualized query. If set, the server skips its own decontextualization step.</summary>
    [JsonPropertyName("decontextualized_query")]
    public string? DecontextualizedQuery { get; set; }

    /// <summary>Enable streaming responses. Defaults to true.</summary>
    [JsonPropertyName("streaming")]
    public bool Streaming { get; set; } = true;

    /// <summary>Client-supplied correlation ID; auto-generated if omitted.</summary>
    [JsonPropertyName("query_id")]
    public string? QueryId { get; set; }

    [JsonPropertyName("mode")]
    public QueryMode Mode { get; set; } = QueryMode.List;
}
```

**Design notes worth internalizing:**

- `Prev` vs. `DecontextualizedQuery` is a deliberate escape hatch: the protocol lets a *smart client* do its own decontextualization (turning "what about the sequel?" into "what is the sequel to Dune?") and skip server-side work, while a *dumb client* can just pass raw conversation history and let the server do it. Build both paths — don't assume every caller will pre-process.
- `Streaming` defaults to `true`. Design your API so batch (non-streaming) callers have to *opt out*, since that matches modern chat-UI expectations.

## 2.4 `NLWebResult` — a single search hit

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace MyNLWeb.Models;

public class NLWebResult
{
    [JsonPropertyName("url")]
    public string Url { get; set; } = string.Empty;

    [JsonPropertyName("name")]
    public string Name { get; set; } = string.Empty;

    [JsonPropertyName("site")]
    public string? Site { get; set; }

    [JsonPropertyName("score")]
    public double Score { get; set; }

    [JsonPropertyName("description")]
    public string? Description { get; set; }

    /// <summary>Schema.org-style structured data describing this item (JSON-LD shape).</summary>
    [JsonPropertyName("schema_object")]
    public JsonElement? SchemaObject { get; set; }
}
```

Using `schema_object` as a loosely-typed `JsonElement` (rather than a rigid class) is intentional — different backends attach wildly different structured metadata (a `Movie`, a `Recipe`, a `BlogPosting`...), and schema.org already has vocabulary for all of them. Don't invent your own schema; reuse schema.org types in this field.

## 2.5 `NLWebResponse` — the outgoing contract

```csharp
using System.Text.Json.Serialization;

namespace MyNLWeb.Models;

public class NLWebResponse
{
    [JsonPropertyName("query_id")]
    public string QueryId { get; set; } = string.Empty;

    [JsonPropertyName("results")]
    public List<NLWebResult> Results { get; set; } = new();

    /// <summary>Present only for Summarize/Generate modes.</summary>
    [JsonPropertyName("summary")]
    public string? Summary { get; set; }
}
```

## 2.6 The options pattern for configuration

Rather than hard-coding defaults, expose a bindable options class — this is standard ASP.NET Core practice (`IOptions<T>`) and is what lets Chapter 09's JSON/YAML config "just work":

```csharp
namespace MyNLWeb.Models;

public class NLWebOptions
{
    public QueryMode DefaultMode { get; set; } = QueryMode.List;
    public bool EnableStreaming { get; set; } = true;
    public int DefaultTimeoutSeconds { get; set; } = 30;
    public int MaxResultsPerQuery { get; set; } = 50;
}
```

This mirrors `src/NLWebNet/Models/NLWebOptions.cs` and is bound later via:

```csharp
builder.Services.Configure<NLWebOptions>(builder.Configuration.GetSection("MyNLWeb"));
```

## 2.7 Update the placeholder endpoint

Swap the Chapter 01 placeholder to use real models:

```csharp
group.MapPost("/", (NLWebRequest request) =>
{
    var response = new NLWebResponse
    {
        QueryId = request.QueryId ?? Guid.NewGuid().ToString(),
        Results = new List<NLWebResult>() // wired up for real in Chapter 04
    };
    return Results.Ok(response);
});
```

Next: [`03-data-backend.md`](03-data-backend.md) — build the pluggable retrieval layer.
