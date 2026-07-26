# 01 — Project Setup: Solution Layout & Minimal APIs From Scratch

**Prerequisites:** none — this is the starting point.

## 1.1 Scaffold the solution

```bash
mkdir MyNLWeb && cd MyNLWeb
dotnet new sln -n MyNLWeb

# Core library — this is the reusable, publishable piece
dotnet new classlib -n MyNLWeb -o src/MyNLWeb
dotnet sln add src/MyNLWeb

# A runnable host app that references the library
dotnet new web -n MyNLWeb.Demo -o samples/Demo
dotnet sln add samples/Demo

# Test project
dotnet new mstest -n MyNLWeb.Tests -o tests/MyNLWeb.Tests
dotnet sln add tests/MyNLWeb.Tests

# Wire references
dotnet add samples/Demo reference src/MyNLWeb
dotnet add tests/MyNLWeb.Tests reference src/MyNLWeb
```

Why split library vs. host app? Because the **protocol implementation** (models, pipeline, endpoints, MCP) is a reusable concern that any ASP.NET Core app should be able to drop in with `builder.Services.AddMyNLWeb()` + `app.MapMyNLWeb()` — exactly how NLWebNet ships as a NuGet package (`src/NLWebNet/NLWebNet.csproj`) consumed by its own demo app (`samples/Demo/NLWebNet.Demo.csproj`). Keeping them separate from day one avoids a painful later split.

## 1.2 Minimal APIs — the concept

ASP.NET Core Minimal APIs let you define HTTP endpoints as functions mapped directly onto routes, without the ceremony of MVC controllers (though NLWebNet actually ships **both** — `Controllers/AskController.cs` for MVC-style consumers and `Endpoints/AskEndpoints.cs` using Minimal APIs — pick one style for your project; this tutorial uses Minimal APIs since that's the lighter-weight, more modern default).

The core shape:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello");
app.MapPost("/ask", (MyRequest req) => new MyResponse { ... });

app.Run();
```

Key Minimal API building blocks you'll use throughout this tutorial:

- **`MapGroup(prefix)`** — groups related endpoints under a shared route prefix and metadata (tags, auth policy). NLWebNet uses this: `app.MapGroup("/ask").WithTags("Ask")`.
- **`TypedResults`** — strongly-typed `IResult` return values (`Ok<T>`, `BadRequest<T>`, etc.) so OpenAPI can generate accurate response schemas.
- **Dependency injection into endpoint delegates** — any parameter of an endpoint handler that isn't bound from the route/query/body is resolved from the DI container automatically. This is how `INLWebService` and `ILoggerFactory` get into your handler without manual wiring.

## 1.3 A minimal `/ask` endpoint (placeholder)

In `src/MyNLWeb/Endpoints/AskEndpoints.cs`:

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;

namespace MyNLWeb.Endpoints;

public static class AskEndpoints
{
    public static RouteGroupBuilder MapAskEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/ask").WithTags("Ask");

        group.MapPost("/", (AskPlaceholderRequest request) =>
            Results.Ok(new { query_id = Guid.NewGuid().ToString(), echo = request.Query }));

        return group;
    }
}

public record AskPlaceholderRequest(string Query);
```

In `samples/Demo/Program.cs`:

```csharp
using MyNLWeb.Endpoints;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapAskEndpoints();

app.Run();
```

Run it:

```bash
cd samples/Demo
dotnet run
```

Test it:

```bash
curl -X POST http://localhost:5037/ask -H "Content-Type: application/json" -d '{"query":"hello"}'
```

This placeholder gets replaced piece by piece as we go: real request/response models in Chapter 02, a real pipeline behind it in Chapter 04.

## 1.4 Why a `.csproj`-per-concern layout matters later

Notice the reference repo also keeps `samples/AspireDemo/` (a *second*, more advanced sample using vector search + service orchestration) alongside the simple `samples/Demo/`. You don't need to build that yet — Chapter 13 covers it — but structuring your solution this way from the start (`src/` = library, `samples/*` = runnable hosts, `tests/` = tests) means adding a second sample later is just another folder, not a restructure.

Next: [`02-data-models.md`](02-data-models.md) — define the real request/response contracts.
