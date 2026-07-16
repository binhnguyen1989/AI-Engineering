# AI Engineering Reference: LLM, RAG, MCP, Agents

> Reference doc for production AI engineering. RAG sections are condensed since you've already
> gone deep on chunking/reranking/eval (SearchService, KnowledgeService). Heavier weight given to
> LLM fundamentals, MCP, and Agents — newer surface area relative to your KafkaDotnet work.

---

## 1. LLM Fundamentals

### 1.1 What you're actually calling

A modern LLM API call is: `tokens in -> transformer forward pass -> logits -> sampling -> tokens out`.
Everything else (system prompts, tools, RAG, agents) is engineering *around* that primitive.

**Tokenization**
- Text is split into subword tokens (BPE / byte-pair-encoding variants). Not 1 token = 1 word.
- Rough heuristic: 1 token ≈ 4 chars in English, less for code (more symbols), way less for
  languages like Vietnamese/Chinese (higher token-per-character density — matters for cost/latency
  when you build the Vietnamese interview-prep content).
- Context window = max tokens (input + output) the model can attend to in one call. Exceeding it
  truncates or errors — this is the hard constraint that drives RAG's existence (you can't just
  stuff the whole knowledge base into the prompt).

**Inference parameters you actually tune**
| Param | Effect | Typical use |
|---|---|---|
| `temperature` | Sampling randomness. 0 = near-deterministic (argmax-ish), higher = more diverse | 0–0.3 for extraction/classification, 0.7+ for creative/brainstorming |
| `top_p` | Nucleus sampling — only sample from smallest token set whose cumulative prob ≥ p | Usually leave default; don't tune with temperature simultaneously |
| `max_tokens` | Hard cap on output length | Set based on expected output shape, not "as high as possible" (cost + latency) |
| `stop_sequences` | Truncate generation at a marker | Useful for structured output parsing |

**Why LLMs hallucinate (mechanically, not mystically)**
The model is doing next-token prediction over a learned distribution — it has no ground-truth
lookup, no notion of "I don't know" unless that pattern was reinforced in training/RLHF. It will
produce fluent, confident, wrong text with the same "effort" as fluent correct text. This is *the*
reason RAG and tool-use exist: ground generation in retrieved/verifiable facts instead of trusting
parametric memory for anything factual/current.

### 1.2 Three ways to adapt a model to your domain — and when each applies

| Approach | What it changes | Cost | When |
|---|---|---|---|
| **Prompting** (system prompt, few-shot examples) | Nothing about the model; steers behavior at inference time | Cheapest, instant iteration | Default starting point, always |
| **RAG** | Nothing about the model; injects relevant *facts* at inference time | Infra cost (vector DB, embedding pipeline, retrieval latency) | Knowledge changes frequently, needs citations/traceability, domain corpus too large for context |
| **Fine-tuning** | Model weights themselves | Expensive, slow iteration, retraining needed as data changes | Need to change *style/format/behavior* consistently, not facts — e.g., always output a specific JSON schema, adopt a narrow voice, or bake in a skill RAG can't express as retrievable text |

Common mistake: fine-tuning to "teach the model facts." Facts drift and fine-tuning bakes in a
snapshot — you end up rebuilding the RAG pipeline anyway once the data changes weekly. Rule of
thumb: **fine-tune for *how* to answer, RAG for *what* to answer with.**

### 1.3 Structured output / function calling

The mechanism agents and MCP tool-use are built on: the model is given a schema (JSON Schema,
typically) and either (a) prompted to emit JSON matching it, or (b) uses native tool-calling where
the API returns a structured `tool_use` block instead of free text.

- Native tool calling > prompted JSON: fewer parsing failures, the model is trained specifically to
  emit valid calls against the schema you provide.
- This is the same pattern as your `EventEnvelope<T>` — a strongly-typed contract the producer
  (LLM) must satisfy, the consumer (your code) parses without guessing.
- Failure mode to design for: the model *will* occasionally emit malformed args or hallucinate a
  tool that doesn't exist. Validate against schema before executing, same as you'd validate an
  inbound Kafka message before applying it — never trust-and-execute.

### 1.4 Cost/latency model

- Cost scales with (input tokens + output tokens); input is cheaper per-token than output on most
  providers, which is why RAG (adds input tokens) is usually cheaper than "just make the model
  smarter/bigger" (worse latency, worse cost, doesn't fix staleness).
- Latency ≈ time-to-first-token (TTFT, dominated by input length / prefill) + tokens/sec × output
  length (decode). Streaming hides decode latency from the user but doesn't reduce total compute.
- Prompt caching (if the provider supports it) amortizes TTFT for repeated large system
  prompts/context — relevant if your MCP tool definitions or RAG system prompt are large and
  static across requests.

---

## 2. RAG — condensed (you've already built this)

Given SearchService (hybrid retrieval, reranking, two-tier caching, Testcontainers) and the RAG
deep-dive you already did, this is a compressed reference rather than re-teaching.

**Pipeline stages, one line each:**
1. **Ingest** — chunk (structure-aware > fixed-size), embed, write to vector store + metadata.
2. **Retrieve** — hybrid (BM25/lexical + vector) beats pure vector for exact-match terms (IDs,
   codes, names) that embeddings blur.
3. **Rerank** — cross-encoder second pass on top-k candidates; RRF to fuse lexical+vector rank
   lists before reranking.
4. **Assemble context** — dedupe, respect token budget, order by relevance (recency bias if
   applicable to the domain).
5. **Generate** — prompt with retrieved context + explicit "cite only what's in context" instruction
   to suppress parametric hallucination.
6. **Evaluate** — retrieval metrics (Recall@k, MRR) as a fast/cheap CI gate; LLM-judged end-to-end
   metrics (faithfulness, relevance) as a slower gate before deploy.

**The one thing worth re-flagging:** ACL-aware retrieval is easy to get wrong quietly — filtering
must happen *before* or *during* the vector search (metadata filter / pre-filtered index), not as a
post-hoc filter on retrieved results, or you leak the existence of restricted docs via retrieval
count/relevance signal even when you don't leak content.

---

## 3. MCP — Model Context Protocol

### 3.1 What problem it solves

Before MCP: every LLM app that wanted to call Gmail, GitHub, a database, etc. wrote a bespoke
integration — N apps × M tools = N×M integrations. MCP standardizes the interface between an LLM
host application and *any* tool/data provider, so it's N+M: one client implementation, one server
implementation per tool, and they interoperate.

It's the same motivation as gRPC + protobuf contracts decoupling your services — a shared protocol
so producers and consumers don't need bespoke glue per pair.

### 3.2 Architecture

```
Host (Claude Desktop / your app) 
   |
   |-- MCP Client (1:1 with each server)
          |
          v
   MCP Server (exposes tools/resources/prompts)
          |
          v
   Underlying system (DB, API, filesystem, SaaS)
```

- **Host**: the LLM-driven application (e.g., Claude Desktop, an IDE, a custom agent runtime).
  Manages one or more clients.
- **Client**: maintains a 1:1 stateful connection to a single server, handles the protocol
  handshake, capability negotiation.
- **Server**: a process (local, stdio-based) or service (remote, HTTP+SSE/Streamable HTTP) exposing
  three primitive types:

| Primitive | Analogy | Purpose |
|---|---|---|
| **Tools** | RPC method | Model-invokable actions with side effects (call an API, run a query) — model decides when to call |
| **Resources** | GET endpoint | Read-only data the host can attach to context (files, DB rows) — application decides when to fetch |
| **Prompts** | Stored procedure / template | Reusable prompt templates the server exposes, user-triggered |

### 3.3 Transport

- **stdio**: server runs as a local subprocess, communicates over stdin/stdout. Simplest, used for
  local tools (filesystem, local DB, local CLI wrappers).
- **Streamable HTTP** (current spec; superseded plain HTTP+SSE): server is a remote, possibly
  multi-tenant service. Needed for anything hosted, auth'd, or shared across users — this is the
  shape of the connectors you already use (Gmail/Calendar/Drive MCP servers above are exactly this).

### 3.4 Protocol mechanics worth knowing

- JSON-RPC 2.0 as the message format — request/response + notifications, same shape as any
  JSON-RPC system you'd have met via other RPC work.
- **Capability negotiation** at connection init: client and server exchange what they support
  (tools/resources/prompts/sampling) before any real traffic — avoids calling an unsupported
  primitive.
- **Sampling** (server→client LLM calls): a server can ask the *host's* LLM to do a completion on
  its behalf, without the server holding its own API key/model access. Inverts the usual
  direction — worth knowing exists, less commonly used than tools.
- Discovery is dynamic: client lists available tools/resources at runtime (`tools/list`), not
  hardcoded — this is why adding a new MCP server to a host doesn't require redeploying the host.

### 3.5 Building an MCP server (mental model, not full tutorial)

Minimal server = declare tool schemas (name, description, JSON Schema for args) + handler
functions. SDKs exist for TypeScript, Python, C#/.NET (`ModelContextProtocol` NuGet package),
Java, Kotlin.

.NET shape, conceptually — feels like a minimal API:
```csharp
// Rough shape — actual SDK surface evolves, check current docs before building
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddMcpServer()
    .WithStdioServerTransport()
    .WithTools<InventoryLookupTools>();
// [McpServerTool] attribute on methods, similar spirit to your MediatR handlers
// exposing a Handle() method — except the "caller" is a model deciding to invoke it.
```

**Design implications for tool schemas** (this is where most MCP servers are badly built):
- Tool descriptions are *prompt engineering*, not documentation — the model chooses which tool to
  call and how based on the description text. Vague descriptions → wrong tool picked or wrong args.
- Keep tool surface narrow and composable, same principle as keeping a gRPC service's methods
  cohesive — a tool that does five unrelated things forces the model to guess which mode you meant.
- Return values should be model-digestible (structured, concise) not a raw dump of your internal
  DTOs — same instinct as not leaking EF Core entities across a service boundary.

### 3.6 Security model — non-optional reading before exposing anything

- MCP servers execute with whatever privileges you grant the underlying credentials — a tool
  wrapping a DB connection with write access means the *model* can now issue writes if it decides
  to call that tool, correctly or not (analogous to how a JWT scope determines blast radius in your
  Shared.Auth work — same discipline applies: least privilege per tool, not one god-credential).
- **Prompt injection via tool output** is the emerging real-world attack class: content returned by
  a tool (e.g., an email body, a web page, a file) can contain text designed to manipulate the
  model into calling other tools maliciously. Treat all tool *output*, not just tool input, as
  untrusted — same as never trusting deserialized data from an external Kafka topic just because
  it's schema-valid.
- Human-in-the-loop confirmation for any tool with side effects (send, delete, pay, deploy) is the
  current best practice — mirrors why your action-taking MCP guidance elsewhere requires
  confirmation before send/modify/delete.

---

## 4. Agents

### 4.1 What "agent" means here (the term is overloaded)

An agent = an LLM in a loop, deciding which action to take next based on the current state, until a
termination condition is met — as opposed to a single prompt→response call. The loop is the whole
definition; everything else is implementation detail.

```
loop:
    observe (current state / tool results / user input)
    think (model reasons about what to do next)
    act (call a tool, or emit final answer)
    until: goal reached / max steps / explicit stop
```

### 4.2 Core patterns

**ReAct (Reason + Act)** — the foundational pattern. Model alternates between a reasoning trace
("I need to check inventory before confirming the order") and an action (tool call), observes the
result, reasons again. Nearly all modern agent frameworks are ReAct with varying amounts of
scaffolding around it.

**Plan-and-execute** — model produces an upfront multi-step plan, then executes steps (possibly
with a cheaper/faster model for execution, a stronger model for planning). Better for tasks with
a knowable structure; worse when the right next step depends heavily on prior tool results
(ReAct adapts better there).

**Reflection / self-critique** — after producing an output, a second pass (same or different model
call) critiques it against the goal and triggers a retry if it fails. Cheap way to catch obvious
failures, meaningfully increases cost/latency — use where correctness matters more than speed
(e.g., before a side-effecting action) not universally.

**Multi-agent / orchestrator-worker** — one model plans and delegates to specialized sub-agents
(each with a narrower tool set / prompt), orchestrator aggregates results. Same rationale as
splitting a monolith into bounded-context services: narrower scope per agent = more reliable
behavior per agent, at the cost of coordination overhead and harder end-to-end debugging.
Reach for this only when a single agent's tool set/context has genuinely outgrown one coherent
role — it's not free complexity, same tradeoff calculus as splitting a service.

### 4.3 Memory

- **Short-term / working memory** = the conversation/context window itself. Bounded by context
  limit — long-running agents will eventually need to summarize/compact history rather than
  append indefinitely.
- **Long-term memory** = externalized state (a DB, vector store, key-value store) the agent reads
  from and writes to across sessions. This is architecturally just... a data store your agent has
  tool access to — no magic, same CRUD discipline as any service state you'd design, including
  needing a schema/versioning story once it grows past a few fields.
- Retrieval-as-memory: treat long-term memory as a small RAG problem (embed + retrieve relevant
  past facts) once it's too large to dump wholesale into context.

### 4.4 Reliability — where agents actually fail in production

This is the part tutorials skip and production systems live or die on:

1. **Compounding error rate**: if each step has a 95% success rate, a 10-step agentic task
   succeeds ~60% of the time (0.95^10). Long autonomous chains need either much higher per-step
   reliability, checkpointing/human-review gates, or explicit retry/verification steps — not "just
   trust the loop."
2. **Tool-call correctness ≠ task correctness**: the model can call the right tool with valid args
   and still pursue the wrong overall strategy. Evaluate end-to-end task success, not just
   "did the tool call parse."
3. **Runaway loops**: always enforce a max-iteration / max-cost budget. An agent stuck reasoning in
   circles will burn tokens indefinitely without one.
4. **State drift between agent's model of the world and actual system state**: if a tool call fails
   silently or partially, the agent's next reasoning step is built on a false premise — same failure
   mode as the dead `MarkInventoryReserved()` bug you caught, except here it's the *model's belief
   state* that goes stale, harder to catch because there's no compiler to flag it.
5. **Idempotency of tool calls matters even more than in normal distributed systems** — a model
   retrying a failed step, or looping, can re-invoke a non-idempotent side-effecting tool (double
   charge, double order) purely from reasoning error, not infrastructure failure. Same discipline
   as your idempotent Kafka consumers, but now the "duplicate" trigger is a language model's
   judgment call instead of an at-least-once delivery guarantee.

### 4.5 Evaluating agents

Same three-tier idea as your RAG eval harness, extended:
- **Component-level**: did the right tool get called with valid args (fast, deterministic, CI gate).
- **Trajectory-level**: was the sequence of steps reasonable (not just the final answer) — catches
  "right answer, dangerous/inefficient path" cases.
- **Outcome-level**: LLM-judged or rule-based check of final task success against the actual goal.

### 4.6 .NET agent tooling landscape (as of last check — verify current state before committing)

- **Semantic Kernel** — Microsoft's orchestration SDK: plugins (≈ tools), planners, memory
  connectors. Closest analog to LangChain/LlamaIndex in the .NET ecosystem.
- **Microsoft.Extensions.AI** — newer abstraction layer for LLM clients (`IChatClient`) meant to
  sit under things like Semantic Kernel, similar spirit to how `ILogger`/`HttpClientFactory`
  abstract a concern across your services.
- Given you already have MediatR pipeline behaviors + a clean architecture, an agent's tool
  layer maps naturally onto your existing CQRS handlers — a "tool" can literally be a thin
  adapter that calls an existing `IRequestHandler<TCommand, TResult>`, rather than a parallel
  code path. Worth designing that way if you build an agent surface for KafkaDotnet rather than
  standing up a second, disconnected tool implementation.

---

## 5. How the four pieces compose

```
Agent (the loop / decision-maker)
  ├── uses an LLM (Section 1) for reasoning at each step
  ├── calls tools, which may be:
  │     ├── MCP servers (Section 3) — standardized, interoperable tool/resource access
  │     └── direct function calls / your own API — bespoke, tighter coupling
  └── one class of tool is "search my knowledge base" — that's RAG (Section 2),
        just retrieval exposed as a callable capability instead of a fixed pre-generation step
```

The mental shift from "RAG pipeline" to "agent with a retrieval tool": in classic RAG, retrieval
always happens before generation, once, deterministically. In an agentic RAG setup, the model
*decides* whether/when/how many times to retrieve, can reformulate the query and retrieve again if
the first pass looks insufficient, and can combine retrieval with other tools in one reasoning
loop. Strictly more capable, strictly harder to make reliable and cheap — reach for it only once
fixed-pipeline RAG has a demonstrated gap (multi-hop questions, ambiguous queries needing
clarification) rather than by default.

---

## 6. Suggested next depth passes

- MCP: build a minimal stdio server wrapping one KafkaDotnet read-only query (e.g., order status
  lookup) — smallest possible end-to-end to internalize the tool-schema-design lessons in §3.5.
- Agents: instrument a single ReAct loop against InventoryService with a hard max-step budget and
  trajectory logging before reaching for any multi-agent framework — the reliability failure modes
  in §4.4 are easier to feel with a minimal loop than to read about.
