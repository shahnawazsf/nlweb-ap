# 07 — The Model Context Protocol: Exposing Your API to AI Agents

**Prerequisites:** Chapter 04 (pipeline working). Chapter 05 helps but isn't required.

## 7.1 What MCP is and why it exists

The **Model Context Protocol** ([modelcontextprotocol.io](https://modelcontextprotocol.io/)) is an open standard for how AI agents (Claude Desktop, other MCP-aware clients) discover and invoke capabilities exposed by a server — "tools" the agent can call, and "prompts" it can request — without you writing bespoke integration code per agent vendor.

The key idea for this tutorial: **you already built the capability (the RAG pipeline in Chapter 04). MCP is just a second, agent-friendly way to expose it**, alongside the human/application-facing `/ask` endpoint. You are not building a second pipeline — you're wrapping the existing `INLWebService` in the protocol shape MCP clients expect.

## 7.2 Core MCP concepts

| Concept | What it means |
|---|---|
| **Tool** | A named, schema-described function the agent can call (e.g. `search`, with parameters `query`, `mode`, `site`) |
| **Prompt** | A reusable prompt template the agent can request and fill in (e.g. a "search_prompt" template) |
| **`list_tools`** | Method a client calls to discover what tools exist and their input schema |
| **`call_tool`** | Method a client calls to actually invoke a tool with arguments |
| **`list_prompts`** / **`get_prompt`** | Equivalent discovery/retrieval methods for prompt templates |

An MCP server, at minimum, needs to answer these four methods.

## 7.3 `IMcpService` contract

`src/MyNLWeb/MCP/IMcpService.cs`:

```csharp
namespace MyNLWeb.MCP;

public interface IMcpService
{
    Task<IEnumerable<McpTool>> ListToolsAsync(CancellationToken ct = default);
    Task<McpToolResult> CallToolAsync(string toolName, Dictionary<string, object?> arguments, CancellationToken ct = default);
    Task<IEnumerable<McpPrompt>> ListPromptsAsync(CancellationToken ct = default);
    Task<McpPromptResult> GetPromptAsync(string promptName, Dictionary<string, object?> arguments, CancellationToken ct = default);
}
```

Supporting model shapes (simplified; see `src/NLWebNet/Models/McpModels.cs` in the reference repo for the full version):

```csharp
namespace MyNLWeb.MCP;

public record McpTool(string Name, string Description, object InputSchema);
public record McpToolResult(bool IsError, List<McpContent> Content);
public record McpContent(string Type, string Text);
public record McpPrompt(string Name, string Description, List<McpPromptArgument> Arguments);
public record McpPromptArgument(string Name, string Description, bool Required);
public record McpPromptResult(string Description, List<McpContent> Messages);
```

## 7.4 Implementing `McpService`

This is the piece that translates MCP tool calls into calls against `INLWebService` — the same orchestrator from Chapter 04:

```csharp
using MyNLWeb.Models;
using MyNLWeb.Services;

namespace MyNLWeb.MCP;

public class McpService : IMcpService
{
    private readonly INLWebService _nlWebService;
    private readonly ILogger<McpService> _logger;

    public McpService(INLWebService nlWebService, ILogger<McpService> logger)
    {
        _nlWebService = nlWebService;
        _logger = logger;
    }

    public Task<IEnumerable<McpTool>> ListToolsAsync(CancellationToken ct = default)
    {
        var tools = new List<McpTool>
        {
            new(
                Name: "search",
                Description: "Search content using natural language. Modes: list (raw results), summarize (AI summary), generate (full grounded answer).",
                InputSchema: new
                {
                    type = "object",
                    properties = new
                    {
                        query = new { type = "string", description = "The natural language query" },
                        mode = new { type = "string", @enum = new[] { "list", "summarize", "generate" }, @default = "list" },
                        site = new { type = "string", description = "Optional site/domain filter" }
                    },
                    required = new[] { "query" }
                }),
            new(
                Name: "search_with_history",
                Description: "Like search, but accepts prior conversation turns to resolve follow-up questions (e.g. 'what about the sequel?').",
                InputSchema: new
                {
                    type = "object",
                    properties = new
                    {
                        query = new { type = "string" },
                        previous_queries = new { type = "string", description = "Comma-separated prior queries" },
                        mode = new { type = "string", @enum = new[] { "list", "summarize", "generate" }, @default = "generate" }
                    },
                    required = new[] { "query" }
                })
        };
        return Task.FromResult<IEnumerable<McpTool>>(tools);
    }

    public async Task<McpToolResult> CallToolAsync(string toolName, Dictionary<string, object?> arguments, CancellationToken ct = default)
    {
        try
        {
            var request = toolName switch
            {
                "search" => new NLWebRequest
                {
                    Query = GetString(arguments, "query"),
                    Mode = ParseMode(GetString(arguments, "mode", "list")),
                    Site = GetString(arguments, "site", null),
                    Streaming = false
                },
                "search_with_history" => new NLWebRequest
                {
                    Query = GetString(arguments, "query"),
                    Prev = GetString(arguments, "previous_queries", null),
                    Mode = ParseMode(GetString(arguments, "mode", "generate")),
                    Streaming = false
                },
                _ => throw new ArgumentException($"Unknown tool: {toolName}")
            };

            var response = await _nlWebService.ProcessRequestAsync(request, ct);
            return new McpToolResult(IsError: false, Content: new() { new("text", FormatForAgent(response)) });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "MCP tool call failed: {Tool}", toolName);
            return new McpToolResult(IsError: true, Content: new() { new("text", $"Error: {ex.Message}") });
        }
    }

    // ListPromptsAsync / GetPromptAsync follow the same pattern — see reference McpService.cs

    private static string FormatForAgent(NLWebResponse response)
    {
        var sb = new System.Text.StringBuilder();
        sb.AppendLine($"Query ID: {response.QueryId}");
        if (!string.IsNullOrEmpty(response.Summary))
            sb.AppendLine($"Summary: {response.Summary}");
        sb.AppendLine($"Results ({response.Results.Count}):");
        foreach (var r in response.Results)
            sb.AppendLine($"- {r.Name} ({r.Url}) [score {r.Score}]: {r.Description}");
        return sb.ToString();
    }

    private static string GetString(Dictionary<string, object?> args, string key, string? @default = null)
        => args.TryGetValue(key, out var v) && v != null ? v.ToString()! : @default ?? throw new ArgumentException($"Missing required argument: {key}");

    private static QueryMode ParseMode(string mode) =>
        Enum.TryParse<QueryMode>(mode, true, out var m) ? m : QueryMode.List;
}
```

**Why format results as text (`FormatForAgent`) instead of returning raw JSON?** MCP tool results are consumed by an LLM inside the agent's own reasoning loop — a clean, readable text block is easier for the model to reason over and cite than a raw JSON blob it has to mentally parse. This is a real, deliberate difference between the `/ask` (JSON, machine-shaped) and `/mcp` (text, LLM-shaped) response formats, even though both wrap the exact same `NLWebResponse`.

## 7.5 The `/mcp` endpoint

```csharp
using MyNLWeb.MCP;

namespace MyNLWeb.Endpoints;

public static class McpEndpoints
{
    public static RouteGroupBuilder MapMcpEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/mcp").WithTags("MCP");

        group.MapPost("/", async (McpRequest request, IMcpService mcp, CancellationToken ct) =>
        {
            return request.Method switch
            {
                "list_tools" => Results.Ok(await mcp.ListToolsAsync(ct)),
                "call_tool" => Results.Ok(await mcp.CallToolAsync(request.ToolName!, request.Arguments ?? new(), ct)),
                "list_prompts" => Results.Ok(await mcp.ListPromptsAsync(ct)),
                "get_prompt" => Results.Ok(await mcp.GetPromptAsync(request.PromptName!, request.Arguments ?? new(), ct)),
                _ => Results.BadRequest(new { error = $"Unknown method: {request.Method}" })
            };
        });

        return group;
    }
}

public record McpRequest(string Method, string? ToolName = null, string? PromptName = null, Dictionary<string, object?>? Arguments = null);
```

Register:

```csharp
builder.Services.AddScoped<IMcpService, McpService>();
// ...
app.MapAskEndpoints();
app.MapMcpEndpoints();
```

## 7.6 Testing it

```bash
curl -X POST localhost:5037/mcp -H "Content-Type: application/json" -d '{"method":"list_tools"}'

curl -X POST localhost:5037/mcp -H "Content-Type: application/json" \
  -d '{"method":"call_tool","toolName":"search","arguments":{"query":"getting started","mode":"generate"}}'
```

To actually connect an MCP client (like Claude Desktop) to this endpoint, you'd typically also implement the MCP transport layer (stdio or HTTP+SSE per the spec) — the simplified request/response shape above illustrates the *logic*; production MCP servers commonly use the official [MCP C# SDK](https://github.com/modelcontextprotocol/csharp-sdk) to handle the wire protocol/transport for you rather than hand-rolling it as shown here.

Next: [`08-tool-selection.md`](08-tool-selection.md) — route different kinds of queries to different specialized tools.
