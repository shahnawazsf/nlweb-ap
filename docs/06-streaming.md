# 06 — Streaming Responses with Server-Sent Events

**Prerequisites:** Chapter 05 (`IChatClient` wired up).

## 6.1 Why streaming matters for a chat-like API

A Generate-mode call can take several seconds for the LLM to produce a full answer. Streaming sends **tokens as they're generated** instead of making the client wait for the whole response — this is the difference between ChatGPT's "typing" effect and a blank screen followed by a wall of text.

## 6.2 Why Server-Sent Events (SSE), not WebSockets

The reference implementation deliberately chose SSE (`doc/design-decisions.md`, decision #3): *"Standard web streaming protocol with good browser support."* SSE is:

- One-directional (server → client), which is all a streaming answer needs
- Built on plain HTTP (works through existing proxies/load balancers without special upgrade handling)
- Trivial to consume from `fetch()` or `EventSource` in a browser, or a simple line reader in any HTTP client

WebSockets would be overkill for a request/response pattern that never needs the client to push data mid-stream.

## 6.3 A known C# gotcha: `yield return` inside `try`/`catch`

C# does not allow `yield return` statements inside a `try` block that has a `catch` clause. Since streaming naturally wants to be `IAsyncEnumerable` + exception handling, you have to structure it carefully — the reference implementation calls this out explicitly (design decision #3: *"Avoids C# yield-return-in-try-catch limitations through proper separation of concerns"*). The fix: separate the code that can throw from the code that yields.

```csharp
// WON'T COMPILE — yield inside try/catch
async IAsyncEnumerable<string> BadStream()
{
    try
    {
        yield return "a"; // CS1626
    }
    catch { }
}
```

```csharp
// WORKS — the loop that yields is outside the try; only the fallible call is inside
async IAsyncEnumerable<string> GoodStream()
{
    IAsyncEnumerable<string> inner;
    try
    {
        inner = GetInnerStream(); // if this can throw synchronously, catch it here
    }
    catch (Exception ex)
    {
        yield return $"error: {ex.Message}";
        yield break;
    }

    await foreach (var item in inner)
        yield return item; // no try/catch wrapping this yield
}
```

In practice, wrap the `await foreach` consumption at the *endpoint* layer (Section 6.5) where you're writing to the HTTP response stream and can catch-and-log without needing to `yield` from inside the catch.

## 6.4 Streaming through the pipeline

Add a streaming counterpart to `INLWebService`:

```csharp
public interface INLWebService
{
    Task<NLWebResponse> ProcessRequestAsync(NLWebRequest request, CancellationToken ct = default);
    IAsyncEnumerable<string> ProcessRequestStreamAsync(NLWebRequest request, CancellationToken ct = default);
}
```

Implementation — retrieval happens once (it's not incremental), then generation streams:

```csharp
public async IAsyncEnumerable<string> ProcessRequestStreamAsync(
    NLWebRequest request, [EnumeratorCancellation] CancellationToken ct = default)
{
    var effectiveQuery = await _queryProcessor.ProcessQueryAsync(
        request.Query, request.Prev, request.DecontextualizedQuery, ct);

    var results = (await _backend.SearchAsync(effectiveQuery, request.Site, cancellationToken: ct)).ToList();

    if (request.Mode == QueryMode.List)
    {
        // Nothing to stream token-by-token — emit the whole result set as one chunk.
        yield return JsonSerializer.Serialize(new { results });
        yield break;
    }

    await foreach (var update in _resultGenerator.GenerateStreamAsync(effectiveQuery, results, request.Mode, ct))
    {
        yield return update; // incremental text chunks from the LLM
    }
}
```

Add the streaming generator method to `IResultGenerator`/`ResultGenerator`, delegating to `IChatClient.GetStreamingResponseAsync`:

```csharp
public async IAsyncEnumerable<string> GenerateStreamAsync(
    string query, List<NLWebResult> results, QueryMode mode,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var messages = BuildMessages(query, results, mode); // same prompt-building as Ch.05

    await foreach (var update in _chatClient.GetStreamingResponseAsync(messages, cancellationToken: ct))
    {
        if (!string.IsNullOrEmpty(update.Text))
            yield return update.Text;
    }
}
```

## 6.5 The SSE endpoint

Modeled on the real `src/NLWebNet/Endpoints/AskEndpoints.cs` `ProcessStreamingQueryAsync`:

```csharp
group.MapGet("/stream", (
    [FromQuery] string query, INLWebService service,
    [FromQuery] string? mode, [FromQuery] string? site, CancellationToken ct) =>
{
    var request = new NLWebRequest
    {
        Query = query,
        Mode = Enum.TryParse<QueryMode>(mode, true, out var m) ? m : QueryMode.List,
        Site = site,
        Streaming = true
    };

    return Results.Stream(async stream =>
    {
        var writer = new StreamWriter(stream);
        await writer.WriteLineAsync("Content-Type: text/event-stream");
        await writer.WriteLineAsync("Cache-Control: no-cache");
        await writer.WriteLineAsync("Connection: keep-alive");
        await writer.WriteLineAsync();
        await writer.FlushAsync();

        try
        {
            await foreach (var chunk in service.ProcessRequestStreamAsync(request, ct))
            {
                await writer.WriteLineAsync($"data: {JsonSerializer.Serialize(chunk)}");
                await writer.WriteLineAsync();
                await writer.FlushAsync();
            }
        }
        catch (Exception ex)
        {
            // Logged here — this is the "outside the yield" catch point from 6.3.
        }

        await writer.WriteLineAsync("data: [DONE]");
        await writer.FlushAsync();
    }, "text/event-stream");
});
```

The `data: [DONE]` sentinel at the end is a widely-used SSE convention (OpenAI's own streaming API uses it) so clients know to stop reading without relying on connection close timing.

## 6.6 Consuming it from a browser

```javascript
const es = new EventSource(`/ask/stream?query=${encodeURIComponent(q)}&mode=generate`);
es.onmessage = (e) => {
  if (e.data === '"[DONE]"') { es.close(); return; }
  appendToUI(JSON.parse(e.data));
};
```

Next: [`07-mcp-protocol.md`](07-mcp-protocol.md) — expose this same pipeline to AI agents via MCP.
