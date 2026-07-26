# 05 — LLM Integration via `Microsoft.Extensions.AI`

**Prerequisites:** Chapter 04 (pipeline working with placeholder generation).

## 5.1 Why an abstraction layer over "just call OpenAI"

If you call the Azure OpenAI SDK or OpenAI SDK directly from your services, you've coupled your business logic to one vendor's client shape. `Microsoft.Extensions.AI` (Microsoft's cross-provider abstraction — [learn.microsoft.com/dotnet/ai](https://learn.microsoft.com/en-us/dotnet/ai/)) gives you one interface, `IChatClient`, that Azure OpenAI, OpenAI, GitHub Models, Ollama, and others all implement. This is exactly the decision NLWebNet's own design docs record (`doc/design-decisions.md`, decision #2): *"Full Microsoft.Extensions.AI integration... provides standardized AI client abstraction supporting multiple providers."*

Swap providers by changing DI registration, not by rewriting `ResultGenerator`.

## 5.2 Install packages

```bash
cd src/MyNLWeb
dotnet add package Microsoft.Extensions.AI

cd ../../samples/Demo
dotnet add package Microsoft.Extensions.AI.OpenAI  # brings in the base OpenAI package + .AsIChatClient()
dotnet add package OpenAI                          # OpenAIClient itself, for the OpenAI path
# or, for Azure OpenAI instead:
dotnet add package Azure.AI.OpenAI                 # AzureOpenAIClient
dotnet add package Azure.Identity                  # DefaultAzureCredential
```

## 5.3 The `IChatClient` interface (conceptually)

```csharp
public interface IChatClient
{
    Task<ChatResponse> GetResponseAsync(IList<ChatMessage> messages, ChatOptions? options = null, CancellationToken ct = default);
    IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(IList<ChatMessage> messages, ChatOptions? options = null, CancellationToken ct = default);
}
```

You send a list of `ChatMessage` (role + content, exactly like the OpenAI chat format), get back a response or a stream of incremental updates. Every provider package plugs into this same shape.

## 5.4 Register a real chat client

In `samples/Demo/Program.cs`, for OpenAI:

```csharp
using Microsoft.Extensions.AI;
using OpenAI;

var apiKey = builder.Configuration["OpenAI:ApiKey"]
    ?? throw new InvalidOperationException("Set OpenAI:ApiKey via user-secrets");

builder.Services.AddChatClient(
    new OpenAIClient(apiKey).GetChatClient("gpt-4o-mini").AsIChatClient());
```

For Azure OpenAI:

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;

var endpoint = new Uri(builder.Configuration["AzureOpenAI:Endpoint"]!);
var deployment = builder.Configuration["AzureOpenAI:DeploymentName"]!;

builder.Services.AddChatClient(
    new AzureOpenAIClient(endpoint, new DefaultAzureCredential())
        .GetChatClient(deployment).AsIChatClient());
```

Store secrets with **user secrets** in development (never in `appsettings.json`):

```bash
cd samples/Demo
dotnet user-secrets init
dotnet user-secrets set "OpenAI:ApiKey" "sk-..."
```

## 5.5 Implement `ResultGenerator` for real

Replace the Chapter 04 stub in `src/MyNLWeb/Services/ResultGenerator.cs`:

```csharp
using Microsoft.Extensions.AI;
using MyNLWeb.Models;
using System.Text;

namespace MyNLWeb.Services;

public class ResultGenerator : IResultGenerator
{
    private readonly IChatClient _chatClient;

    public ResultGenerator(IChatClient chatClient) => _chatClient = chatClient;

    public async Task<string> GenerateSummaryAsync(string query, IEnumerable<NLWebResult> results, CancellationToken ct = default)
    {
        var context = BuildContext(results, maxItems: 10);
        var messages = new List<ChatMessage>
        {
            new(ChatRole.System, "Summarize the following search results in 2-3 sentences, in relation to the user's query. Do not add information not present in the results."),
            new(ChatRole.User, $"Query: {query}\n\nResults:\n{context}")
        };
        var response = await _chatClient.GetResponseAsync(messages, cancellationToken: ct);
        return response.Text;
    }

    public async Task<string> GenerateResponseAsync(string query, IEnumerable<NLWebResult> results, CancellationToken ct = default)
    {
        var context = BuildContext(results, maxItems: 5);
        var messages = new List<ChatMessage>
        {
            new(ChatRole.System,
                "Answer the user's question using ONLY the context below. " +
                "If the context doesn't contain the answer, say so — do not make anything up. " +
                "Cite sources by URL where relevant."),
            new(ChatRole.User, $"Question: {query}\n\nContext:\n{context}")
        };
        var response = await _chatClient.GetResponseAsync(messages, cancellationToken: ct);
        return response.Text;
    }

    private static string BuildContext(IEnumerable<NLWebResult> results, int maxItems)
    {
        var sb = new StringBuilder();
        foreach (var r in results.Take(maxItems))
        {
            sb.AppendLine($"- [{r.Name}]({r.Url}): {r.Description}");
        }
        return sb.ToString();
    }
}
```

**This is the entire RAG "augmentation" step**: `BuildContext` formats retrieved documents as text, and the system prompt instructs the model to stay grounded in that text. There's no framework magic beyond string formatting + a system prompt.

Register it:

```csharp
builder.Services.AddScoped<IResultGenerator, ResultGenerator>();
```

## 5.6 Supporting multiple providers at once (like the reference demo)

NLWebNet's demo app supports switching between Azure OpenAI, OpenAI, and **GitHub Models** at runtime (`samples/Demo/Services/DynamicChatClientFactory.cs`, `AIConfigurationService.cs`, `GitHubModelsChatClient.cs`). The pattern: define a factory interface that inspects configuration/user selection and returns the appropriate `IChatClient` implementation, rather than hard-registering one provider at startup. Build this once you actually need runtime provider switching (e.g. a demo UI with a provider dropdown) — don't build it speculatively.

## 5.7 A no-op client for offline development

The reference repo ships a `NullChatClient` (`samples/Demo/Services/NullChatClient.cs`) that returns canned responses when no API key is configured, so the app is runnable and demoable without any credentials. Do the same:

```csharp
public class NullChatClient : IChatClient
{
    public Task<ChatResponse> GetResponseAsync(IList<ChatMessage> messages, ChatOptions? options = null, CancellationToken ct = default)
        => Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant,
            "[No AI provider configured — set an API key via user-secrets to enable real responses.]")));

    public async IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IList<ChatMessage> messages, ChatOptions? options = null, [EnumeratorCancellation] CancellationToken ct = default)
    {
        yield return new ChatResponseUpdate(ChatRole.Assistant, "[No AI provider configured]");
        await Task.CompletedTask;
    }

    public void Dispose() { }
    public object? GetService(Type serviceType, object? key = null) => null;
}
```

Fall back to this when no key is present so `dotnet run` always works for a new contributor.

Next: [`06-streaming.md`](06-streaming.md) — stream tokens to the client as they're generated.
