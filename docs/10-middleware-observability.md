# 10 — Middleware & Observability: Rate Limiting, Metrics, Health, Tracing

**Prerequisites:** Chapter 09 (config/DI wrapper in place).

An LLM-backed API has failure modes a normal CRUD API doesn't: expensive/slow upstream calls, cost-per-request, and the need to trace one logical query across retrieval + generation + (optionally) tool routing. This chapter covers the operational layer.

## 10.1 Correlation IDs — tracing one query end-to-end

Every request should carry one ID through all its logs. You already have this for free: `NLWebRequest.QueryId` / the auto-generated fallback from Chapter 04. Thread it through every `ILogger` call as a structured property (`{QueryId}`) as shown in every code sample so far — that's not incidental, it's how you'll actually debug production issues: filter your log aggregator by one `QueryId` and see the whole request lifecycle (decontextualize → search → tool routing → generation) in order.

For requests that *don't* go through `NLWebRequest` (health checks, non-protocol routes), add a small middleware that assigns a correlation ID to `HttpContext.Items` and pushes it into the logging scope:

```csharp
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault() ?? Guid.NewGuid().ToString();
    context.Items["CorrelationId"] = correlationId;
    using (context.RequestServices.GetRequiredService<ILoggerFactory>()
        .CreateLogger("Correlation").BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
    {
        await next();
    }
});
```

## 10.2 Rate limiting

LLM calls cost money per token — an unthrottled `/ask` endpoint is a direct line to your cloud bill. ASP.NET Core has built-in rate limiting middleware (`Microsoft.AspNetCore.RateLimiting`) since .NET 7:

```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("nlweb", opt =>
    {
        opt.PermitLimit = 60;             // 60 requests
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 0;               // reject immediately over limit, don't queue
    });
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await context.HttpContext.Response.WriteAsJsonAsync(new { error = "Rate limit exceeded" }, ct);
    };
});

// ...
app.UseRateLimiter();
app.MapAskEndpoints().RequireRateLimiting("nlweb");
```

For per-client limits (not just a global bucket), key the limiter partition by API key or client IP instead of a fixed window — see `AddPolicy` with a partitioned limiter. The reference repo's roadmap explicitly calls out "rate limiting per user/application" as an open question (`doc/design-decisions.md` §1) alongside authentication — the two are usually designed together, since rate limiting without knowing *who* the caller is can only throttle by IP.

## 10.3 Metrics

Track at minimum: request count by mode, request latency, LLM token usage (cost proxy), backend search latency. Use `System.Diagnostics.Metrics`:

```csharp
using System.Diagnostics.Metrics;

public class NLWebMetrics
{
    private readonly Counter<long> _requestCount;
    private readonly Histogram<double> _requestDuration;

    public NLWebMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyNLWeb");
        _requestCount = meter.CreateCounter<long>("nlweb.requests", description: "Total /ask requests");
        _requestDuration = meter.CreateHistogram<double>("nlweb.request.duration", unit: "ms");
    }

    public void RecordRequest(QueryMode mode, double durationMs)
    {
        _requestCount.Add(1, new KeyValuePair<string, object?>("mode", mode.ToString()));
        _requestDuration.Record(durationMs, new KeyValuePair<string, object?>("mode", mode.ToString()));
    }
}
```

Wrap the pipeline call in `NLWebService.ProcessRequestAsync` with a `Stopwatch` and call `RecordRequest` in a `finally` block so metrics are recorded even on error paths.

## 10.4 Health checks

Two things are worth checking independently, because they fail independently: your data backend, and your AI provider.

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DataBackendHealthCheck>("data_backend")
    .AddCheck<AIServiceHealthCheck>("ai_service");

app.MapHealthChecks("/health");
```

```csharp
public class DataBackendHealthCheck : IHealthCheck
{
    private readonly IDataBackend _backend;
    public DataBackendHealthCheck(IDataBackend backend) => _backend = backend;

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct = default)
    {
        try
        {
            await _backend.SearchAsync("health-check-probe", maxResults: 1, cancellationToken: ct);
            return HealthCheckResult.Healthy();
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Data backend unreachable", ex);
        }
    }
}
```

`AIServiceHealthCheck` follows the same shape against `IChatClient` — a cheap 1-token completion call, or a provider-specific ping endpoint if one exists, so you don't burn real cost on every health probe.

## 10.5 Distributed tracing with OpenTelemetry

Once you have more than one moving part (API → backend → LLM provider), you want traces, not just logs:

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("MyNLWeb")
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddMeter("MyNLWeb")
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter());
```

Wrap the retrieval and generation steps in `NLWebService` with an `ActivitySource` span each (`"retrieval"`, `"generation"`), so a trace view shows exactly where time went for a slow query — this is far more useful than aggregate latency once Generate-mode calls start taking multiple seconds.

## 10.6 What to build when

For a personal project or early prototype: skip straight to Chapter 11. For anything with real users: correlation IDs and health checks are cheap and worth doing immediately; rate limiting is worth doing before your first public demo link; metrics/tracing are worth doing before you have more than one backend or provider to debug across.

Next: [`11-testing.md`](11-testing.md) — unit and integration testing this pipeline.
