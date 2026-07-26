# 08 — Multi-Tool Routing: The Tool Selection Framework

**Prerequisites:** Chapter 04 (pipeline), Chapter 07 (MCP concepts help but aren't required).

## 8.1 The problem this solves

So far, every query goes through the same path: decontextualize → search → summarize/generate. But real query intents differ:

- *"Compare the Falcon and the Enterprise"* → wants a **comparison**, not a plain search
- *"Tell me more about the Nostromo"* → wants **details** on one specific item
- *"Recipe for protein bars"* → wants a **recipe-shaped** answer
- *"Search for space movies"* → wants plain search

A single generic RAG prompt handles all of these poorly. The **tool selection framework** sits in front of the standard pipeline: it looks at the query, decides which specialized handler (if any) fits best, and routes there — falling back to the standard pipeline if nothing matches. This is the same "route to a specialist" pattern function-calling/agentic systems use, just applied to your own query pipeline instead of an LLM's tool-calling loop.

## 8.2 `IToolHandler` — one per specialized capability

```csharp
namespace MyNLWeb.Services;

public interface IToolHandler
{
    string ToolId { get; }
    Task<NLWebResponse> HandleAsync(NLWebRequest request, CancellationToken ct = default);
}
```

The reference repo ships five concrete handlers as a guide for what's worth building: `SearchToolHandler`, `DetailsToolHandler`, `CompareToolHandler`, `EnsembleToolHandler`, `RecipeToolHandler` (`src/NLWebNet/Services/*ToolHandler.cs`). Each is a small class implementing `IToolHandler` with logic/prompting specific to that intent — e.g. `CompareToolHandler` fetches two items via `IDataBackend.GetItemByUrlAsync` and prompts the LLM specifically to produce a side-by-side comparison, rather than a generic RAG answer.

A `BaseToolHandler` abstract class is worth factoring out to hold shared logic (calling the backend, calling the chat client) that all handlers need, so each concrete handler only implements the intent-specific bits.

## 8.3 Declarative tool definitions (XML)

Rather than hard-coding trigger logic in C#, the reference implementation defines tools + their trigger patterns declaratively, so non-developers (or you, later) can tune routing without a recompile:

```xml
<?xml version="1.0" encoding="utf-8"?>
<ToolDefinitions>
  <Tool id="compare-tool" name="Compare Items" type="compare" enabled="true">
    <Description>Compares two or more items side by side</Description>
    <Parameters>
      <MaxItems>3</MaxItems>
    </Parameters>
    <TriggerPatterns>
      <Pattern>compare*</Pattern>
      <Pattern>* vs *</Pattern>
      <Pattern>difference between*</Pattern>
    </TriggerPatterns>
  </Tool>
  <Tool id="search-tool" name="Enhanced Search" type="search" enabled="true">
    <Description>Advanced search with semantic understanding</Description>
    <Parameters>
      <MaxResults>50</MaxResults>
      <TimeoutSeconds>30</TimeoutSeconds>
    </Parameters>
    <TriggerPatterns>
      <Pattern>search for*</Pattern>
      <Pattern>find*</Pattern>
    </TriggerPatterns>
  </Tool>
</ToolDefinitions>
```

`ToolDefinitionLoader` (a service you write, or see `src/NLWebNet/Services/ToolDefinitionLoader.cs`) parses this file at startup into `ToolDefinition` model objects.

## 8.4 `IToolSelector` — matching a query to a tool

```csharp
namespace MyNLWeb.Services;

public interface IToolSelector
{
    Task<ToolDefinition?> SelectToolAsync(string query, CancellationToken ct = default);
}
```

A simple v1 implementation matches the query against each enabled tool's `TriggerPatterns` (glob-style `*` matching); a more advanced version could ask a cheap/fast LLM call to classify intent instead of pattern matching. Start with pattern matching — it's free, fast, and debuggable; only add an LLM classification step if patterns prove too brittle for your real query traffic.

## 8.5 `IToolExecutor` — the routing glue

```csharp
namespace MyNLWeb.Services;

public interface IToolExecutor
{
    Task<NLWebResponse?> TryExecuteAsync(NLWebRequest request, CancellationToken ct = default);
}

public class ToolExecutor : IToolExecutor
{
    private readonly IToolSelector _selector;
    private readonly IEnumerable<IToolHandler> _handlers;
    private readonly ILogger<ToolExecutor> _logger;

    public ToolExecutor(IToolSelector selector, IEnumerable<IToolHandler> handlers, ILogger<ToolExecutor> logger)
    {
        _selector = selector;
        _handlers = handlers;
        _logger = logger;
    }

    public async Task<NLWebResponse?> TryExecuteAsync(NLWebRequest request, CancellationToken ct = default)
    {
        var tool = await _selector.SelectToolAsync(request.Query, ct);
        if (tool == null) return null; // no specialized match — caller falls back to standard pipeline

        var handler = _handlers.FirstOrDefault(h => h.ToolId == tool.Id);
        if (handler == null)
        {
            _logger.LogWarning("Selected tool {ToolId} has no registered handler", tool.Id);
            return null;
        }

        try
        {
            return await handler.HandleAsync(request, ct);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Tool {ToolId} failed, falling back to standard pipeline", tool.Id);
            return null; // graceful degradation — same principle as Ch.04
        }
    }
}
```

## 8.6 Wiring it into `NLWebService`

Chapter 04 left the tool-execution step as a forward reference; here's the actual change. Extend the `NLWebService` constructor from Chapter 04 with a fifth, optional parameter — `IToolExecutor? toolExecutor = null` — so tool routing is opt-in and existing callers that don't register an `IToolExecutor` keep working unchanged:

```csharp
public class NLWebService : INLWebService
{
    private readonly IQueryProcessor _queryProcessor;
    private readonly IDataBackend _backend;
    private readonly IResultGenerator _resultGenerator;
    private readonly IToolExecutor? _toolExecutor;
    private readonly ILogger<NLWebService> _logger;

    public NLWebService(IQueryProcessor queryProcessor, IDataBackend backend,
        IResultGenerator resultGenerator, ILogger<NLWebService> logger,
        IToolExecutor? toolExecutor = null)
    {
        _queryProcessor = queryProcessor;
        _backend = backend;
        _resultGenerator = resultGenerator;
        _logger = logger;
        _toolExecutor = toolExecutor;
    }

    public async Task<NLWebResponse> ProcessRequestAsync(NLWebRequest request, CancellationToken ct = default)
    {
        if (_toolExecutor != null)
        {
            var toolResult = await _toolExecutor.TryExecuteAsync(request, ct);
            if (toolResult != null) return toolResult; // specialized tool handled it
        }

        // ...fall through to the standard pipeline from Chapter 04...
    }
}
```

**Design principle: tool execution never replaces the standard pipeline, it short-circuits it.** If tool selection returns nothing, or the selected handler throws, you fall back to plain List/Summarize/Generate rather than failing the request. This is the same graceful-degradation pattern from Chapter 04's `NLWebService`, applied one layer up.

## 8.7 When to build this

Don't build the tool selection framework on day one. Ship the plain pipeline (Chapters 01–06) first, observe what kinds of queries your real users send, and only add specialized handlers for intents that are common enough and distinct enough from generic RAG to be worth the complexity — the reference repo itself treats this as a "medium term (6–12 months)" enhancement over the core protocol, not a foundational piece.

Next: [`09-configuration.md`](09-configuration.md) — externalize all of this into JSON/YAML config.
