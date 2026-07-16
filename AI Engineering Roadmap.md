# AI Engineering Roadmap: Beginner → Pro
### LLM · RAG · MCP · Agents

Companion to `ai-engineering-reference.md`. That doc is the *what*; this is the *in what order,
and what to build at each stage*. Five tiers, each with all four pillars side by side, plus a
running project so the tiers compound instead of staying academic.

**Running project**: extend KafkaDotnet with an `AiOpsService` — starts as a single LLM call
answering questions about order/inventory state, ends (Tier 4) as a multi-agent system with
MCP-exposed tools, its own RAG index over your service docs/logs, and a full eval/observability
pipeline. Each tier below tells you what to add to it.

---

## Tier 0 — Foundations (before touching any framework)

You can skip pieces of this if you already have them solid — you clearly have DDD/EDA down; the
gap to check is the ML-adjacent math/vocab, not the systems side.

| Area | Must-know before Tier 1 |
|---|---|
| Vectors & similarity | Dot product, cosine similarity, Euclidean distance — this *is* what "semantic search" means mechanically. If you can't explain why cosine similarity ignores magnitude, stop and fix that first. |
| Probability basics | What a probability distribution over tokens means; why "temperature" is a distribution-sharpening operation, not a magic creativity dial. |
| Transformer architecture (conceptual, not from-scratch math) | Attention = each token looks at every other token and weights relevance; this is why context length is quadratic-cost and why "the model forgets the middle of a long context" is a real, documented phenomenon (lost-in-the-middle), not folklore. |
| API mechanics | Read the Anthropic API docs end to end once: request/response shape, streaming, token counting, tool_use blocks. Don't learn this secondhand from a framework's abstraction. |

**Checkpoint**: you can explain to a colleague, without hand-waving, why an LLM has a context
limit, why RAG doesn't "expand" that limit but works around it, and why increasing `temperature`
is not the fix for hallucination.

**Do**: nothing built yet. This tier is reading + one afternoon experimenting directly with the
raw API (no SDK abstractions) — send a request, inspect the raw JSON response, hit a context-limit
error on purpose so you've seen it once.

---

## Tier 1 — Beginner: single-call competence

Goal: reliably get structured, correct output from one LLM call. No agents, no loops, no MCP yet.

### LLM
- Learn prompt structure: system vs user vs assistant roles, why system prompts persist behavior
  across a conversation, few-shot examples (2–3 examples of desired input→output format beats a
  paragraph of instructions describing the format).
- Learn native tool-calling / structured output (JSON Schema → `tool_use` block). This is the
  single highest-leverage skill in this tier — almost everything downstream (RAG assembly, MCP,
  agents) is built on "get the model to return a shape you can parse."
- Build: a script that takes a KafkaDotnet domain object (e.g., a `ClaimSummary` DTO), sends it to
  the model, and gets back a structured classification or summary — parsed via strict schema
  validation, not regex on free text.

### RAG
- You've done this deep-dive already (chunking, hybrid retrieval, reranking) — Tier 1 for you is
  really just: if you haven't, do the smallest possible version by hand once — embed 20 short
  documents, compute cosine similarity manually against a query embedding, retrieve top-3, paste
  into a prompt. No vector DB, no framework. This "RAG with a `for` loop and a dictionary" exercise
  is what makes every later abstraction (Pinecone, pgvector, Elasticsearch) legible as *just* an
  indexed version of that loop.

### MCP
- Install Claude Desktop (or another MCP host) and connect one existing public MCP server
  (filesystem or a hosted connector). Just *use* it as a consumer first — watch what a tool call
  and its result look like from the host side before you ever write a server.
- Read the MCP spec's core concepts page once, specifically the difference between tools,
  resources, and prompts (Section 3.2 in the reference doc) — most beginners conflate all three
  into "tools."

### Agents
- Build the smallest possible ReAct loop by hand: a `while` loop, one tool (e.g., a calculator or
  a single read-only KafkaDotnet query), the model reasons + calls the tool + you feed the result
  back + repeat until it emits a final answer. No framework (no Semantic Kernel, no LangChain
  equivalent) — you want to feel the loop mechanically once before a framework hides it.

**Tier 1 checkpoint / definition of done**: you have (a) one script that reliably gets structured
JSON out of the model, (b) a from-scratch mini-RAG loop you built without a vector DB, (c) used an
MCP host as a consumer, (d) a hand-rolled agent loop that calls one real tool against your own
data. All four should be embarrassingly simple, single-file, no framework. That's the point.

---

## Tier 2 — Intermediate: production-shaped components

Goal: each pillar becomes a real service-shaped component with the discipline you already apply to
KafkaDotnet — typed contracts, tests, error handling, not scripts.

### LLM
- Structured-output reliability: schema validation with retry-on-malformed-response (bounded
  retries, not infinite), and a fallback path when the model refuses/can't comply.
- Prompt versioning: treat prompts as code — under source control, with a changelog, ideally with
  a lightweight eval (Tier 3 formalizes this) run before merging a prompt change, same instinct as
  not merging a schema migration without tests.
- Cost/latency instrumentation: log token counts (input/output) and latency per call from day one,
  same as you'd instrument any external dependency call — this data is what Tier 3's optimization
  work depends on.

### RAG
- Move the Tier-1 toy loop into pgvector (you already have this pattern from your Python RAG
  pipeline) or Elasticsearch (you already have SearchService) — pick whichever the AiOpsService
  should actually own long-term.
- Add hybrid retrieval (lexical + vector, RRF fusion) — you already know this; the Tier-2 bar is
  applying it to a *new* corpus (e.g., KafkaDotnet's own README/ADR docs) rather than reusing
  SearchService's existing index, so you feel the ingestion/chunking decisions fresh on unfamiliar
  data shape.
- Add a reranking pass and measure Recall@k before/after — quantify that it actually helped on
  this corpus rather than assuming the pattern transfers unchanged.

### MCP
- Build your first MCP server: wrap one existing KafkaDotnet read-only query handler (e.g.,
  "get order status by ID") as an MCP tool, stdio transport, run it locally against Claude Desktop
  or another host.
- Deliberately write a bad tool description first, watch the model call it wrong or pick the wrong
  tool among several, then fix the description and watch the behavior change. This is the fastest
  way to internalize that tool descriptions are prompt engineering (Section 3.5).
- Add a second tool with overlapping purpose on purpose (e.g., "get order by ID" and "search
  orders by customer") to practice writing descriptions specific enough that the model
  disambiguates correctly.

### Agents
- Rebuild the Tier-1 loop with a minimal framework (Semantic Kernel plugin, or
  Microsoft.Extensions.AI `IChatClient` + your own loop) — now that you've felt it raw, see what
  the framework actually buys you (retry handling, structured tool registration) versus what it
  obscures.
- Add a max-iteration budget and structured trajectory logging (every reasoning step + tool call +
  result, persisted) — non-negotiable before this touches anything real, per the reliability
  failure modes in reference doc §4.4.
- Wire the Tier-2 MCP server in as the agent's tool source instead of a direct function call — this
  is the first point where MCP and Agents actually connect for you end to end.

**Tier 2 checkpoint**: AiOpsService now has a typed LLM-calling layer with instrumentation, a real
vector-backed RAG index over KafkaDotnet docs, one working MCP server exposing at least two tools
with deliberately-tuned descriptions, and an agent loop (framework-assisted, budget-capped,
logged) that uses that MCP server to answer multi-step questions.

---

## Tier 3 — Advanced: reliability and evaluation

Goal: stop trusting vibes. Everything gets measured, and failure modes get designed against, not
discovered in prod.

### LLM
- Build a proper eval harness for your prompts: a fixed test set of (input, expected-output-shape
  or expected-facts) pairs, run automatically, with both deterministic checks (schema valid?
  required fields present?) and LLM-judged checks (factually consistent with source? — same
  judge-model pattern as your RAG eval harness). Gate prompt changes on this, same as you gate code
  changes on unit tests.
- Learn prompt-caching behavior for your provider and apply it if your system prompt / tool
  definitions are large and repeated — direct cost/latency win once request volume justifies it.
- Add adversarial test cases: what happens on empty input, wildly out-of-domain input, an attempt
  to override the system prompt via user input (prompt injection from your *own* users, not just
  tool output) — same mindset as fuzzing an API endpoint.

### RAG
- You already have this tier's core skill (two-stage hybrid retrieval + eval harness with
  Recall@k/MRR + LLM-judged end-to-end metrics, as a CI gate). The Tier-3 extension specific to
  AiOpsService: ACL-aware retrieval done correctly (filter *before* the vector search, not after —
  reference doc §2 flags exactly why post-hoc filtering leaks signal) since a service like this
  will eventually need to respect the same auth boundaries as your Shared.Auth work.
- Multi-hop retrieval: queries that need two retrieval passes (e.g., "what changed about the
  ClaimsService saga since the FraudDetectionService contract was last updated" needs to resolve
  *which* contract update, then retrieve *around* that). This is where fixed-pipeline RAG starts
  showing its ceiling — note it, don't necessarily solve it here; Tier 4 gives you agentic
  retrieval as the answer.

### MCP
- Move the MCP server from stdio (local subprocess) to Streamable HTTP (remote, auth'd) — this is
  the shape needed for anything beyond your own laptop, and where you actually have to solve
  authn/authz for tool access (map your Shared.Auth JWT validation + Redis revocation onto "who is
  allowed to invoke this tool" — same access-control discipline, new attack surface).
- Add human-in-the-loop confirmation for any tool with side effects (anything beyond read-only
  queries) — build the confirmation step as a first-class part of the protocol flow, not a UI
  afterthought.
- Explicitly threat-model prompt injection via tool *output* (reference doc §3.6): if any tool
  returns text originating outside your control (a customer's claim notes, an external API
  response), write a test that injects an instruction into that text and confirms the agent
  doesn't follow it.

### Agents
- Trajectory-level evaluation (reference doc §4.5): don't just check final answers — build a
  small eval set where you assert on the *path* (did it check inventory before confirming an
  order, in the right sequence) not just the destination.
- Idempotency audit of every tool the agent can call — for each, ask "if the agent calls this twice
  because it looped or retried, what happens?" Fix the ones that aren't idempotent before this
  agent has write access to anything real, same bar you'd apply to a Kafka consumer.
- Reflection/self-critique pass before any side-effecting action: a second, cheaper model call that
  checks "does this proposed action actually match the user's request" before execution — your
  first real guard against compounding error rate on multi-step chains.

**Tier 3 checkpoint**: you have CI-gated evals for prompts, RAG retrieval, and agent trajectories;
the MCP server is remote, auth'd, and confirms before side effects; you've written and passed an
injection test; every write-capable tool has a documented idempotency story.

---

## Tier 4 — Pro: systems that operate autonomously and safely at scale

Goal: this is no longer "does it work in a demo," it's "does it stay correct, safe, and
cost-bounded when it's making hundreds of decisions a day without you watching each one."

### LLM
- Model routing: cheap/fast model for classification and simple extraction, strongest model only
  for the reasoning-heavy steps — same instinct as your write-path separation in InventoryService
  (EF Core+Outbox for domain rules vs. Channel+COPY for bulk throughput) — pick the right tool per
  workload shape instead of one model for everything.
- Continuous eval in production: sample a percentage of live traffic, run it through your eval
  harness asynchronously, alert on regression — the LLM-app equivalent of your Prometheus/HPA
  observability stack, not a one-time pre-deploy gate.
- Fine-tuning decision point: only now, with real production data and a demonstrated *behavioral*
  gap (not a factual one — that's still RAG's job per reference doc §1.2), evaluate whether
  fine-tuning is justified. Most teams never need this tier; don't reach for it prematurely.

### RAG
- Agentic retrieval (reference doc §5): let the agent decide when/whether to retrieve, reformulate
  queries, and retrieve iteratively for multi-hop questions identified as a real gap in Tier 3.
  Measure it against the fixed-pipeline baseline on the same eval set — justify the added
  complexity and cost with numbers, don't adopt it because it's more sophisticated-sounding.
- Index freshness pipeline: your existing event-driven reindexing pattern (KnowledgeService, Kafka
  + Outbox) is already the right shape here — the Pro-tier bar is making staleness *observable*
  (lag metric: time between a source doc changing and it being reflected in retrieval) not just
  eventually-consistent by assumption.

### MCP
- Multi-server orchestration: the agent now draws on several MCP servers (your AiOpsService tools,
  plus third-party ones) — the Pro-tier skill is capability negotiation and graceful degradation
  when a server is down or a tool call fails, not just the happy path.
- Treat your MCP servers as a product surface with the same rigor as a public API: versioned tool
  schemas, deprecation policy for tool changes (a schema change breaks every agent currently
  relying on the old shape, same blast radius as a breaking gRPC contract change).

### Agents
- Multi-agent orchestration only once a single agent's tool set has genuinely outgrown one coherent
  role (reference doc §4.2) — split AiOpsService's agent into, say, a read-only "investigator"
  agent and a separate, more tightly gated "action" agent that requires the investigator's findings
  plus explicit confirmation before it can call a write tool. Mirrors bounded-context service
  splits — don't do it for its own sake.
- Full production reliability posture: max-cost budgets per session (not just max-iterations),
  circuit breakers on repeated tool failures (same `SafeExecuteAsync`-style resilience wrapper
  instinct as your Shared.Caching Redis exception handling, applied to tool calls instead of cache
  calls), and a kill switch that a human can pull mid-session.
- Postmortem discipline: when an agent does something wrong in production, the trajectory log
  (Tier 2) plus the eval harness (Tier 3) should let you reproduce the failure as a new eval case,
  the same way a production incident becomes a regression test in normal software engineering.

**Tier 4 checkpoint / "Pro" bar**: AiOpsService runs unattended against real KafkaDotnet traffic
for read-heavy investigation tasks, has bounded and observable cost, can't take a destructive
action without a confirmed human-in-the-loop step, every prior production failure exists as a
regression test, and you could hand the trajectory logs to someone else and have them understand
*why* the agent did what it did without asking you.

---

## How to actually pace this

- Don't parallelize all four pillars at Tier 3–4 speed at once — MCP and Agents in particular
  compound in complexity together. A reasonable cadence: finish LLM+RAG Tier 2 (you're mostly
  there already on RAG) before starting MCP Tier 2, then Agents Tier 2 last, since Agents Tier 2
  explicitly depends on having an MCP server to call.
- The running project (AiOpsService) is deliberately scoped to reuse KafkaDotnet patterns
  (MediatR handlers as tool implementations, Shared.Auth for tool authz, Shared.Caching's
  resilience pattern for tool circuit-breaking) — the goal is that by Tier 4 you're not maintaining
  a second, disconnected "AI codebase," you're extending the one you already have.
