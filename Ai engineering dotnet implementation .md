# Applying LLM / RAG / MCP / Agents to .NET Core

Concrete package choices and code shapes for KafkaDotnet's Clean Architecture. Verified against
current NuGet listings — package names/versions below were checked today, but re-check before
pinning in `Directory.Packages.props` since this ecosystem moves fast.

---

## 0. Package map

| Concern | Package | Notes |
|---|---|---|
| Claude API client | `Anthropic` (official, currently beta, v10+) | `dotnet add package Anthropic`. Ships `IChatClient` adapter via `Microsoft.Extensions.AI`. |
| Claude API client (mature alternative) | `Anthropic.SDK` (community, tghamm) | Older, battle-tested, also exposes `IChatClient`, integrates with Semantic Kernel/Agent Framework. Fine to use instead of the official one if you want longer production track record. |
| Chat abstraction layer | `Microsoft.Extensions.AI` | `IChatClient` — the abstraction every provider/framework below plugs into. Put this in your `Application` layer's dependency direction, same role as `IRequestHandler`. |
| MCP client + server | `ModelContextProtocol` (main), `ModelContextProtocol.Core` (minimal), `ModelContextProtocol.AspNetCore` (HTTP transport) | Official, Microsoft-maintained. `McpClientTool` implements `AIFunction`, so MCP tools plug directly into `IChatClient` tool-calling. |
| Vector storage on your existing Postgres | `Pgvector`, `Pgvector.EntityFrameworkCore` | Reuses your EF Core + Npgsql stack — no new datastore for RAG if you don't want one. Works alongside your existing Elasticsearch SearchService; use whichever fits the corpus. |
| Orchestration (optional) | `Microsoft.SemanticKernel` | Plugins ≈ tools, planners, memory connectors. Optional — a hand-rolled `IChatClient` loop is often enough; reach for SK when you want its planner/plugin ecosystem specifically. |
| Resilience | `Microsoft.Extensions.Http.Resilience` | `AddStandardResilienceHandler()` on the `HttpClient` backing your chat client — retry/circuit-breaker/timeout, same pattern as any other outbound HTTP dependency in KafkaDotnet. |

---

## 1. LLM layer — where it lives, how it's wired

**Layer placement**: define an application-layer port, implement it in `Infrastructure`, exactly
like you already do for `IEmailSender` or gRPC clients.

```csharp
// Application/Common/Interfaces/IAiClient.cs
public interface IAiClient
{
    Task<TResult> CompleteStructuredAsync<TResult>(
        string systemPrompt,
        string userPrompt,
        CancellationToken ct = default);
}
```

```csharp
// Infrastructure/Ai/AnthropicAiClient.cs
public sealed class AnthropicAiClient(IChatClient chatClient) : IAiClient
{
    public async Task<TResult> CompleteStructuredAsync<TResult>(
        string systemPrompt, string userPrompt, CancellationToken ct = default)
    {
        var messages = new List<ChatMessage>
        {
            new(ChatRole.System, systemPrompt),
            new(ChatRole.User, userPrompt)
        };

        // Native structured output — schema derived from TResult, not prompted JSON.
        var response = await chatClient.GetResponseAsync<TResult>(messages, cancellationToken: ct);
        return response.Result;
    }
}
```

**DI registration** — mirrors how you already register `HttpClient`-backed gRPC/REST clients:

```csharp
// Infrastructure/DependencyInjection.cs
services.AddHttpClient("anthropic")
    .AddStandardResilienceHandler(); // retry, circuit breaker, timeout — same as your other outbound deps

services.AddSingleton<IChatClient>(sp =>
{
    var client = new AnthropicClient(); // reads ANTHROPIC_API_KEY from env — inject via K8s secret, same as your other API keys
    return client.AsIChatClient("claude-sonnet-5")
        .AsBuilder()
        .UseFunctionInvocation() // enables tool-calling loop automatically
        .Build();
});

services.AddScoped<IAiClient, AnthropicAiClient>();
```

**Structured output contract** — same discipline as your `EventEnvelope<T>`: strict typed shape,
validated, never trust-and-deserialize blindly.

```csharp
public sealed record ClaimTriageResult(
    [property: Description("Urgency 1-5, 5 = immediate review")] int UrgencyScore,
    [property: Description("One-sentence rationale")] string Rationale,
    [property: Description("True if fraud indicators present")] bool FlaggedForFraudReview);
```

Call it from a MediatR handler exactly like any other application service:

```csharp
public sealed class TriageClaimHandler(IAiClient ai, IClaimRepository claims)
    : IRequestHandler<TriageClaimCommand, ClaimTriageResult>
{
    public async Task<ClaimTriageResult> Handle(TriageClaimCommand cmd, CancellationToken ct)
    {
        var claim = await claims.GetAsync(cmd.ClaimId, ct);
        return await ai.CompleteStructuredAsync<ClaimTriageResult>(
            systemPrompt: "You triage insurance claims. Be conservative — over-flag rather than miss fraud.",
            userPrompt: JsonSerializer.Serialize(claim),
            ct);
    }
}
```

**Instrumentation** — log token counts/latency per call from day one:

```csharp
// Wrap AnthropicAiClient or add a MediatR pipeline behavior, same shape as your Shared.Auditing behavior
public sealed class AiCallLoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IAiTrackedRequest
{
    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        var result = await next();
        _logger.LogInformation("AI call {RequestType} took {Ms}ms", typeof(TRequest).Name, sw.ElapsedMilliseconds);
        return result;
    }
}
```

---

## 2. RAG layer — pgvector + EF Core

Since SearchService already owns Elasticsearch-backed hybrid retrieval, use pgvector for anything
that should live alongside existing transactional data in the same Postgres instance (e.g., an
AiOpsService retrieving over KafkaDotnet's own docs/ADRs) rather than standing up a second search
infra for a smaller corpus.

**Entity + migration**:

```csharp
public class DocChunk
{
    public Guid Id { get; set; }
    public string Content { get; set; } = "";
    public string SourceDocId { get; set; } = "";
    public Vector? Embedding { get; set; } // 1024-dim if using Voyage, matches your Python RAG pipeline choice
}
```

```csharp
// DbContext registration
optionsBuilder.UseNpgsql(connectionString, npgsql => npgsql.UseVector());

// OnModelCreating
modelBuilder.HasPostgresExtension("vector");
modelBuilder.Entity<DocChunk>()
    .HasIndex(c => c.Embedding)
    .HasMethod("hnsw")
    .HasOperators("vector_cosine_ops")
    .HasStorageParameter("m", 16)
    .HasStorageParameter("ef_construction", 64);
```

```csharp
// Migration — build HNSW index outside the transaction, same caution as any large index build
migrationBuilder.Sql("""
    CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_docchunks_embedding_hnsw
    ON "DocChunks" USING hnsw ("Embedding" vector_cosine_ops) WITH (m = 16, ef_construction = 64)
    """, suppressTransaction: true);
```

**Retrieval query** — cosine distance via the `<=>` operator, EF Core translates it through the
provider:

```csharp
public async Task<List<DocChunk>> RetrieveAsync(Vector queryEmbedding, int topK, CancellationToken ct)
{
    return await _db.DocChunks
        .OrderBy(c => c.Embedding!.CosineDistance(queryEmbedding))
        .Take(topK)
        .ToListAsync(ct);
}
```

**Hybrid retrieval note**: pgvector alone gives you the vector leg. For the lexical leg, either
combine with Postgres full-text search (`tsvector`/`tsquery` on the same table, fuse ranks
yourself — RRF is just arithmetic over two rank lists, nothing framework-specific) or, if this
corpus needs real hybrid quality, it may belong in SearchService/Elasticsearch instead — don't
half-build hybrid search twice.

**Embedding generation** — via `IEmbeddingGenerator<string, Embedding<float>>` (also part of
`Microsoft.Extensions.AI`, same abstraction family as `IChatClient`):

```csharp
services.AddSingleton<IEmbeddingGenerator<string, Embedding<float>>>(sp =>
    new VoyageEmbeddingGenerator(apiKey, "voyage-3")); // or your provider of choice — reuse the Voyage pattern from your Python pipeline
```

**Testing**: same Testcontainers pattern SearchService already uses — `pgvector/pgvector:pg16`
image gives you the extension pre-installed, `HasPostgresExtension("vector")` in
`OnModelCreating` lets EF Core manage it through migrations in CI the same way it does for your
other extensions.

---

## 3. MCP layer — exposing KafkaDotnet handlers as tools

### 3.1 Server — wrap an existing MediatR handler

The key move: an MCP tool is a thin adapter, not a parallel implementation. It calls
`IMediator.Send` exactly like a controller action would.

```csharp
// Infrastructure/Mcp/OrderTools.cs
[McpServerToolType]
public sealed class OrderTools(IMediator mediator)
{
    [McpServerTool, Description("Get the current status of an order by its ID.")]
    public async Task<string> GetOrderStatus(
        [Description("The order ID, e.g. ORD-12345")] string orderId,
        CancellationToken ct)
    {
        var result = await mediator.Send(new GetOrderStatusQuery(orderId), ct);
        return JsonSerializer.Serialize(result); // model-digestible: concise, structured, not a raw EF entity dump
    }
}
```

**Host setup** (stdio, for local/dev — Tier 2 in the roadmap):

```csharp
var builder = Host.CreateApplicationBuilder(args);
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly(); // discovers [McpServerToolType] classes

await builder.Build().RunAsync();
```

**HTTP host** (remote, auth'd — Tier 3, once this leaves your laptop):

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services
    .AddMcpServer()
    .WithHttpTransport()
    .WithToolsFromAssembly();

builder.Services.AddAuthentication(/* reuse your existing Shared.Auth Auth0/JWT setup */);

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapMcp().RequireAuthorization(); // gate tool access the same way you gate any other endpoint
await app.RunAsync();
```

Auth mapping matters here specifically: **who can invoke a tool** is a new axis on top of your
existing JWT scopes — a tool wrapping a write-capable handler needs its own explicit scope check,
not just "caller has a valid token." Same least-privilege discipline as Shared.Auth, applied per
tool rather than per endpoint.

### 3.2 Client — consuming an MCP server from your own code (not just Claude Desktop)

```csharp
McpClient orderToolsServer = await McpClient.CreateAsync(
    new HttpClientTransport(new() { Endpoint = new("https://aiops.internal/mcp") }));

var tools = await orderToolsServer.ListToolsAsync(); // IList<McpClientTool>, each is an AIFunction

var response = await chatClient.GetResponseAsync(
    "What's the status of order ORD-12345?",
    new ChatOptions { Tools = [.. tools] });
```

Because `McpClientTool` inherits `AIFunction`, it drops straight into `ChatOptions.Tools` alongside
any hand-written `AIFunction` — no adapter code needed between "tool defined via MCP" and "tool
defined as a local C# method."

---

## 4. Agent loop — minimal, then framework-assisted

### 4.1 Hand-rolled ReAct loop (Tier 1/2 — do this once before reaching for a framework)

```csharp
public async Task<string> RunAgentAsync(string userGoal, IList<AITool> tools, CancellationToken ct)
{
    var messages = new List<ChatMessage> { new(ChatRole.User, userGoal) };
    var options = new ChatOptions { Tools = tools };
    const int maxSteps = 8; // hard budget — non-negotiable per reliability guidance

    for (var step = 0; step < maxSteps; step++)
    {
        var response = await _chatClient.GetResponseAsync(messages, options, ct);
        messages.AddRange(response.Messages);

        _trajectoryLogger.LogStep(step, response); // structured log: every reasoning step + tool call + result, persisted

        if (response.FinishReason == ChatFinishReason.Stop)
            return response.Text; // model produced a final answer, no more tool calls

        // UseFunctionInvocation() (if configured on the IChatClient) executes tool calls automatically
        // and appends results to `messages` for you — the loop above is what that convenience wraps.
    }

    throw new AgentBudgetExceededException(userGoal, maxSteps);
}
```

With `.UseFunctionInvocation()` configured on the `IChatClient` (as in the DI setup in §1), most of
the tool-execution plumbing above is handled for you — build the raw loop once by hand to feel it,
then lean on the built-in invocation handler for anything beyond a learning exercise.

### 4.2 Idempotency + circuit-breaking on tool calls

Reuse the `SafeExecuteAsync`-style wrapper from Shared.Caching, applied to tool execution instead
of cache calls:

```csharp
[McpServerTool, Description("Reserve inventory for an order. Idempotent by orderId.")]
public async Task<string> ReserveInventory(string orderId, string sku, int qty, CancellationToken ct)
{
    // Idempotency key = orderId+sku, same pattern as your Kafka consumer idempotency —
    // a looping or retrying agent must not double-reserve.
    var existing = await _mediator.Send(new GetReservationQuery(orderId, sku), ct);
    if (existing is not null) return JsonSerializer.Serialize(existing); // no-op on retry

    var result = await _mediator.Send(new ReserveInventoryCommand(orderId, sku, qty), ct);
    return JsonSerializer.Serialize(result);
}
```

### 4.3 Human-in-the-loop confirmation for side-effecting tools

Simplest correct shape: side-effecting tools don't execute directly — they enqueue a pending
action and return a confirmation token; a second, explicit call (or a UI approval) executes it.
Mirrors your Outbox pattern's "propose, then commit" shape.

```csharp
[McpServerTool, Description("Propose a claim payout. Requires ConfirmAction to execute.")]
public async Task<string> ProposePayout(string claimId, decimal amount, CancellationToken ct)
{
    var token = await _mediator.Send(new CreatePendingActionCommand("Payout", claimId, amount), ct);
    return $"Proposed payout of {amount} for {claimId}. Awaiting confirmation: {token}";
}
```

### 4.4 Semantic Kernel, if you want the plugin/planner ecosystem

Same `AiOpsService` tools, exposed as SK plugins instead of (or alongside) MCP:

```csharp
var kernel = Kernel.CreateBuilder()
    .AddAnthropicChatCompletion(modelId: "claude-sonnet-5", apiKey: apiKey) // via Anthropic.SDK's SK integration
    .Build();

kernel.Plugins.AddFromType<OrderTools>(); // same [McpServerTool]-style methods, different attribute set for SK
```

Worth it specifically when you want SK's planner (multi-step plan generation) or its vector-store
connector ecosystem (`Microsoft.SemanticKernel.Connectors.PgVector` wraps the same pgvector setup
from §2 with SK's `VectorStore` abstraction) — not a requirement for a working agent loop.

---

## 5. Project structure — where this lives in KafkaDotnet's Clean Architecture

```
AiOpsService/
├── Domain/                    # unchanged — PendingAction, TriageResult as domain concepts if warranted
├── Application/
│   ├── Common/Interfaces/
│   │   ├── IAiClient.cs        # port, per §1
│   │   └── IVectorRetriever.cs # port, per §2
│   ├── Claims/TriageClaimHandler.cs   # MediatR handler calling IAiClient — same shape as every other handler
│   └── Behaviors/AiCallLoggingBehavior.cs
├── Infrastructure/
│   ├── Ai/AnthropicAiClient.cs
│   ├── Rag/PgVectorRetriever.cs
│   └── Mcp/OrderTools.cs, InventoryTools.cs   # [McpServerToolType] classes, thin wrappers over IMediator
├── McpServer/                 # separate ASP.NET Core host project (or a hosted service in the API project)
│   └── Program.cs             # WithHttpTransport + WithToolsFromAssembly
└── Api/
    └── AgentController.cs      # exposes the ReAct loop as an endpoint, same as any other controller
```

The MCP server can live as its own small ASP.NET Core project (deployed as its own K8s
deployment/service, same pattern as your other services) or as an additional endpoint mapped
inside an existing service's `Program.cs` if the tool surface is small — start with the latter for
AiOpsService's first few tools, split out once the tool count and auth requirements justify a
dedicated deployment, same threshold you'd apply to splitting any other bounded context.
