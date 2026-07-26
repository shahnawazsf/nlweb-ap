# 12 — A Demo UI with Blazor

**Prerequisites:** Chapters 04–06 (pipeline + streaming working).

## 12.1 Why Blazor (and why this is optional)

Your API doesn't need a UI to be useful — the `/ask` and `/mcp` endpoints are consumable from any HTTP client. A demo UI exists to let humans exercise and showcase the API without writing curl commands. The reference repo uses **Blazor Web App with interactive server rendering**: C# runs on the server, UI updates stream to the browser over a persistent connection — no separate JavaScript frontend framework/build pipeline needed, since you're already in a .NET codebase. If your team is more comfortable with React/Vue, there's nothing protocol-specific about Blazor — swap it for any SPA that calls `/ask` and `/ask/stream` over HTTP; everything below is about the API contract, not the rendering technology.

## 12.2 Project setup

```bash
dotnet new blazor -n MyNLWeb.Demo -o samples/Demo --interactivity Server
dotnet add samples/Demo reference src/MyNLWeb
```

(If you already scaffolded `samples/Demo` as a plain `web` template in Chapter 01, add Blazor to it: `Components/`, `_Imports.razor`, register `AddRazorComponents().AddInteractiveServerComponents()` in `Program.cs`, map `MapRazorComponents<App>().AddInteractiveServerRenderMode()`.)

## 12.3 A query input component

`Components/QueryInput.razor`:

```razor
<div class="mb-3">
    <input @bind="Query" @bind:event="oninput" class="form-control" placeholder="Ask a question..." />
    <select @bind="Mode" class="form-select mt-2">
        <option value="List">List</option>
        <option value="Summarize">Summarize</option>
        <option value="Generate">Generate</option>
    </select>
    <button class="btn btn-primary mt-2" @onclick="Submit" disabled="@IsLoading">
        @(IsLoading ? "Searching..." : "Search")
    </button>
</div>

@code {
    [Parameter] public EventCallback<NLWebRequest> OnSubmit { get; set; }
    private string Query { get; set; } = "";
    private string Mode { get; set; } = "List";
    private bool IsLoading { get; set; }

    private async Task Submit()
    {
        IsLoading = true;
        await OnSubmit.InvokeAsync(new NLWebRequest
        {
            Query = Query,
            Mode = Enum.Parse<QueryMode>(Mode)
        });
        IsLoading = false;
    }
}
```

## 12.4 Calling your own API from the Blazor server

Since Blazor Server runs C# on the server, the cleanest approach is to **call `INLWebService` directly** (in-process, no HTTP round-trip) rather than making the server call its own HTTP endpoint:

```razor
@inject INLWebService NLWebService

<QueryInput OnSubmit="HandleSubmit" />
<ResultsDisplay Response="_response" />

@code {
    private NLWebResponse? _response;

    private async Task HandleSubmit(NLWebRequest request)
    {
        _response = await NLWebService.ProcessRequestAsync(request);
    }
}
```

If your UI is a *separate* app (not sharing a process with the API — e.g. the Aspire frontend in Chapter 13), call the HTTP endpoint instead via a typed `HttpClient`, the same way any external consumer would.

## 12.5 Streaming in the UI

For Generate mode, stream tokens into the UI as they arrive rather than waiting for the full response — this is where Blazor Server's persistent connection pays off, since you can call `StateHasChanged()` after each chunk to re-render incrementally:

```razor
@code {
    private string _streamedText = "";

    private async Task HandleStreamingSubmit(NLWebRequest request)
    {
        _streamedText = "";
        await foreach (var chunk in NLWebService.ProcessRequestStreamAsync(request))
        {
            _streamedText += chunk;
            StateHasChanged();
        }
    }
}
```

## 12.6 Visualizing which data source answered

A nice touch from the reference demo worth copying: color-coded badges per result showing which backend/source it came from (useful once you have more than one `IDataBackend`, or a multi-backend setup from Chapter 03.5). Use the existing `Site` field on `NLWebResult` (Chapter 02) and render a Bootstrap badge per result:

```razor
@foreach (var result in Response.Results)
{
    <div class="card mb-2">
        <div class="card-body">
            <span class="badge bg-info">@result.Site</span>
            <h6>@result.Name</h6>
            <p>@result.Description</p>
        </div>
    </div>
}
```

## 12.7 An MCP demo page

Worth building once Chapter 07 is done: a page that calls `/mcp` with `list_tools` and renders the returned tool schemas, plus a form to invoke `call_tool` manually. This is genuinely useful during development — it's how you'll notice a tool's `InputSchema` doesn't match what you're actually parsing in `CallToolAsync` before an external agent finds the mismatch for you.

Next: [`13-aspire-orchestration.md`](13-aspire-orchestration.md) — running multiple services (API, vector DB, frontend) together locally.
