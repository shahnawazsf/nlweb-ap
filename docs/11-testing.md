# 11 — Testing an LLM-Backed Pipeline

**Prerequisites:** Chapter 04 at minimum; more chapters built = more to test.

## 11.1 What's testable deterministically vs. not

The uncomfortable truth about testing RAG pipelines: **LLM output is non-deterministic**, so you can't assert exact generated text. What you *can* test deterministically:

- Request validation (empty query → 400)
- `IDataBackend` relevance scoring/ranking logic
- `IQueryProcessor` decontextualization *when it doesn't call an LLM* (or with a fake `IChatClient`)
- Routing logic: does mode `List` skip the LLM entirely? Does `Generate` call `GenerateResponseAsync` and not `GenerateSummaryAsync`?
- Tool selection pattern matching (Chapter 08) — given a query string, is the *correct tool* selected? (You're testing routing, not generation quality.)
- Error handling / graceful degradation paths
- Streaming: does the stream terminate with `[DONE]`? Does it yield at least one chunk?

For anything touching actual LLM output quality, you need a different strategy (11.4).

## 11.2 Test project setup

The reference repo uses MSTest (264 tests as of writing) — a solid, boring, well-supported choice; NUnit or xUnit work identically for everything below.

```bash
dotnet add tests/MyNLWeb.Tests package Moq          # for mocking IDataBackend / IChatClient
dotnet add tests/MyNLWeb.Tests package Microsoft.AspNetCore.Mvc.Testing  # for integration tests
```

## 11.3 Unit test example: the orchestrator

```csharp
[TestClass]
public class NLWebServiceTests
{
    [TestMethod]
    public async Task ListMode_DoesNotCallResultGenerator()
    {
        var backend = new Mock<IDataBackend>();
        backend.Setup(b => b.SearchAsync(It.IsAny<string>(), null, It.IsAny<int>(), default))
               .ReturnsAsync(new List<NLWebResult> { new() { Name = "Test", Url = "http://x" } });

        var generator = new Mock<IResultGenerator>();
        var processor = new Mock<IQueryProcessor>();
        processor.Setup(p => p.ProcessQueryAsync(It.IsAny<string>(), null, null, default))
                 .ReturnsAsync("test query");

        var service = new NLWebService(processor.Object, backend.Object, generator.Object,
            NullLogger<NLWebService>.Instance);

        var response = await service.ProcessRequestAsync(new NLWebRequest { Query = "test query", Mode = QueryMode.List });

        Assert.IsNull(response.Summary);
        Assert.AreEqual(1, response.Results.Count);
        generator.Verify(g => g.GenerateSummaryAsync(It.IsAny<string>(), It.IsAny<IEnumerable<NLWebResult>>(), default), Times.Never);
        generator.Verify(g => g.GenerateResponseAsync(It.IsAny<string>(), It.IsAny<IEnumerable<NLWebResult>>(), default), Times.Never);
    }

    [TestMethod]
    public async Task BackendThrows_ReturnsGracefulErrorResponse_NotException()
    {
        var backend = new Mock<IDataBackend>();
        backend.Setup(b => b.SearchAsync(It.IsAny<string>(), It.IsAny<string?>(), It.IsAny<int>(), default))
               .ThrowsAsync(new InvalidOperationException("backend down"));

        var processor = new Mock<IQueryProcessor>();
        processor.Setup(p => p.ProcessQueryAsync(It.IsAny<string>(), null, null, default)).ReturnsAsync("q");

        var service = new NLWebService(processor.Object, backend.Object, new Mock<IResultGenerator>().Object,
            NullLogger<NLWebService>.Instance);

        var response = await service.ProcessRequestAsync(new NLWebRequest { Query = "q" });

        Assert.IsNotNull(response.Summary); // apologetic message, not a thrown exception
    }
}
```

This second test directly exercises the "graceful degradation" design principle from Chapter 04 — if you refactor error handling later and accidentally let the exception propagate, this test catches it.

## 11.4 Testing generation quality (a different discipline)

Unit tests can't tell you "is this a good answer." For that:

- **Golden-set evals**: a fixed set of (query, expected-facts-that-must-appear) pairs, run against real LLM output, checked with substring/fact-presence assertions rather than exact-match — not part of your CI unit-test suite, but a separate script/pipeline you run before shipping prompt changes.
- **LLM-as-judge**: have a second LLM call score the primary response against a rubric (grounded-in-context? on-topic? no hallucinated URLs?). Useful but adds cost/latency to your eval loop — reserve for pre-release checks, not every commit.
- **Snapshot the prompt, not the output**: assert on what you send *to* the LLM (the constructed context/system prompt from `BuildContext` in Chapter 05) — that's deterministic and catches regressions like "we stopped including the URL in context" without needing to judge generated text at all.

## 11.5 Integration testing with `WebApplicationFactory`

Spin up the whole app in-memory, swap the real `IChatClient` for the `NullChatClient` from Chapter 05, and hit the actual HTTP endpoints:

```csharp
[TestClass]
public class AskEndpointIntegrationTests
{
    private WebApplicationFactory<Program> _factory = null!;

    [TestInitialize]
    public void Setup()
    {
        _factory = new WebApplicationFactory<Program>().WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.RemoveAll<IChatClient>();
                services.AddSingleton<IChatClient, NullChatClient>();
            });
        });
    }

    [TestMethod]
    public async Task Ask_EmptyQuery_Returns400()
    {
        var client = _factory.CreateClient();
        var response = await client.PostAsJsonAsync("/ask", new { query = "" });
        Assert.AreEqual(HttpStatusCode.BadRequest, response.StatusCode);
    }

    [TestMethod]
    public async Task Ask_ValidListQuery_ReturnsResults()
    {
        var client = _factory.CreateClient();
        var response = await client.PostAsJsonAsync("/ask", new { query = "getting started", mode = "list" });
        response.EnsureSuccessStatusCode();
        var body = await response.Content.ReadFromJsonAsync<NLWebResponse>();
        Assert.IsTrue(body!.Results.Count > 0);
    }
}
```

This is the highest-value test class in the whole suite: it exercises real routing, real DI wiring, and the real `MockDataBackend`, with only the LLM call swapped out — so it catches wiring mistakes (wrong lifetime, missing registration) that pure unit tests with hand-built mocks never will.

## 11.6 What to test as you build each chapter

| After building... | Add tests for... |
|---|---|
| Ch.04 pipeline | Mode routing, error → graceful response |
| Ch.06 streaming | Stream yields ≥1 chunk, ends with `[DONE]` |
| Ch.07 MCP | `list_tools` schema shape, unknown tool name → error result (not exception) |
| Ch.08 tool selection | Given query X, correct tool ID selected; unmatched query → null (falls back) |
| Ch.09 config | Options bind correctly from a sample `appsettings.json` |
| Ch.10 middleware | Rate limiter returns 429 after N requests; health check reports unhealthy when backend mock throws |

Next: [`12-ui-layer.md`](12-ui-layer.md) — build a demo UI on top of the API.
