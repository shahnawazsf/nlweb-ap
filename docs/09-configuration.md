# 09 — Configuration: Options Pattern, Multi-Format Config, Secrets

**Prerequisites:** Chapter 02 (`NLWebOptions` defined).

## 9.1 The three layers of configuration in ASP.NET Core

1. **`appsettings.json` / `appsettings.{Environment}.json`** — non-secret defaults, checked into source control
2. **User Secrets** (development) / **environment variables** or **Azure Key Vault** (production) — anything sensitive (API keys, connection strings)
3. **The Options pattern** (`IOptions<T>`) — strongly-typed C# access to configuration, bound once at startup

Never put API keys in `appsettings.json`, even in a private repo — treat that file as if it will eventually leak.

## 9.2 Binding `NLWebOptions`

```csharp
builder.Services.Configure<NLWebOptions>(builder.Configuration.GetSection("MyNLWeb"));
```

`appsettings.json`:

```json
{
  "MyNLWeb": {
    "DefaultMode": "List",
    "EnableStreaming": true,
    "DefaultTimeoutSeconds": 30,
    "MaxResultsPerQuery": 50
  }
}
```

Consume it anywhere via DI:

```csharp
public class SomeService
{
    public SomeService(IOptions<NLWebOptions> options)
    {
        var maxResults = options.Value.MaxResultsPerQuery;
    }
}
```

## 9.3 Secrets: user secrets in development

```bash
cd samples/Demo
dotnet user-secrets init
dotnet user-secrets set "AzureOpenAI:ApiKey" "your-key"
dotnet user-secrets set "AzureOpenAI:Endpoint" "https://your-resource.openai.azure.com/"
dotnet user-secrets set "AzureOpenAI:DeploymentName" "gpt-4o-mini"
```

User secrets are stored outside the repo (`%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` on Windows) and merged into `IConfiguration` automatically in Development — nothing sensitive ever touches git.

## 9.4 Production secrets

For production, don't ship user secrets — use one of:

- **Environment variables**, using ASP.NET Core's double-underscore convention for nested keys: `AzureOpenAI__ApiKey=...`
- **Azure Key Vault** via `builder.Configuration.AddAzureKeyVault(...)`
- Your platform's native secret manager (AWS Secrets Manager, Kubernetes Secrets mounted as env vars, etc.)

## 9.5 Supporting YAML alongside JSON

Some ops teams strongly prefer YAML for backend/retrieval configuration (multi-backend setups especially — see 9.7). Add YAML support via a config provider such as `NetEscapades.Configuration.Yaml`:

```bash
dotnet add package NetEscapades.Configuration.Yaml
```

```csharp
builder.Configuration.AddYamlFile("config_retrieval.yaml", optional: true, reloadOnChange: true);
```

Example `config_retrieval.yaml`:

```yaml
write_endpoint: primary_backend
endpoints:
  primary_backend:
    enabled: true
    db_type: azure_ai_search
    priority: 1

MyNLWeb:
  DefaultMode: List
  EnableStreaming: true
  DefaultTimeoutSeconds: 30
```

**Important**: because both JSON and YAML ultimately populate the same `IConfiguration`/`IOptions<T>` binding target, existing JSON configuration keeps working unchanged when you add YAML support — this is a strict superset, not a breaking migration. Whichever provider is added *last* in `Program.cs` wins on key conflicts, so add YAML after JSON if you want it to override, or before if JSON should win.

## 9.6 Declarative tool definitions (XML)

From Chapter 08 — tool routing rules loaded from `tool_definitions.xml` rather than hard-coded. Load it with plain `System.Xml.Linq`:

```csharp
var doc = XDocument.Load("tool_definitions.xml");
var tools = doc.Descendants("Tool").Select(t => new ToolDefinition
{
    Id = t.Attribute("id")!.Value,
    Name = t.Attribute("name")!.Value,
    Enabled = bool.Parse(t.Attribute("enabled")?.Value ?? "true"),
    TriggerPatterns = t.Descendants("Pattern").Select(p => p.Value).ToList()
}).ToList();
```

## 9.7 Multi-backend configuration

If you built the multi-backend `IBackendManager` mentioned at the end of Chapter 03, its configuration needs to describe *multiple* named endpoints, each with its own type/priority/credentials — this is exactly what the YAML shape in 9.5 models (`endpoints: { primary_backend: {...}, secondary_backend: {...} }`). Bind it to a strongly-typed `MultiBackendOptions` class the same way as `NLWebOptions`, and have `AddMyNLWebMultiBackend(...)` (an alternate DI registration entry point, alongside the single-backend `AddMyNLWeb(...)`) branch on whether multi-backend mode is enabled — mirroring the real `ServiceCollectionExtensions.cs`, which ships exactly this pair of registration methods rather than trying to make one method handle both shapes.

## 9.8 Registering it all as one DI extension method

Consumers of your library shouldn't have to know about every individual service registration. Wrap it:

```csharp
namespace MyNLWeb.Extensions;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMyNLWeb(
        this IServiceCollection services, Action<NLWebOptions>? configure = null)
    {
        if (configure != null) services.Configure(configure);

        services.AddSingleton<IDataBackend, MockDataBackend>();
        services.AddScoped<IQueryProcessor, QueryProcessor>();
        services.AddScoped<IResultGenerator, ResultGenerator>();
        services.AddScoped<INLWebService, NLWebService>();
        services.AddScoped<IMcpService, MCP.McpService>();

        return services;
    }
}

public static class ApplicationBuilderExtensions
{
    public static IEndpointRouteBuilder MapMyNLWeb(this IEndpointRouteBuilder app)
    {
        app.MapAskEndpoints();
        app.MapMcpEndpoints();
        return app;
    }
}
```

Now any host app does exactly what the reference README documents as the public API:

```csharp
builder.Services.AddMyNLWeb(options =>
{
    options.DefaultMode = QueryMode.List;
    options.EnableStreaming = true;
});

app.MapMyNLWeb();
```

This one-line integration surface is the entire point of building the library/host split back in Chapter 01.

Next: [`10-middleware-observability.md`](10-middleware-observability.md) — rate limiting, metrics, health checks.
