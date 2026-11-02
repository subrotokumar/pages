---
hide:
    - toc
    - navigation
---

# Managing Token Cost in LLM Agents with Large Tool Sets

Most agents I see start the same way. Someone wires up an MCP server, then another, then a third, and a month later every request carries 60K tokens of tool schemas that the model mostly ignores. Nobody notices until the bill arrives or the model starts picking the wrong tool.

This post is about fixing that. It covers what the cost actually is, how to measure it, the techniques that move the number, and how real teams (Cursor, Cloudflare, Manus, Vercel, Claude Code) handle it in production. Code examples use the OpenAI Python SDK and the Responses API. I've tried to stick to documented behaviour, and I flag where a number is a vendor's own benchmark rather than something you should expect to reproduce.

## What you are actually paying for

Every request to the model includes the full text of every tool you expose: name, description, and JSON schema. The API renders them into the prompt, ahead of the conversation. OpenAI's function calling guide puts it plainly: functions are injected into the system message in a syntax the model was trained on. On top of that, providers can add a small fixed block of instructions whenever tools are present, so even one tool has a nonzero floor.

Real numbers from Anthropic's write-up on this problem: a five-server MCP setup with 58 tools came to roughly 55K tokens before the conversation started, Jira alone was about 17K, and they saw definitions reach 134K tokens before they optimised. Independent reports match the shape. One user measured seven MCP servers at about 67K tokens of definitions, a third of a 200K window, before typing anything, and the GitHub server alone at around 18K for 27 tools. That isn't a typo-level edge case. It's what happens when you connect a few well-documented servers.

Two things make this worse than it looks.

**The cost repeats.** An agent loop is N model calls, and each call resends everything. 55K of definitions across an 8-call task is 440K input tokens of overhead, regardless of what the task was.

**It hurts accuracy, not just cost.** More tools means more near-duplicates to choose between, and a longer prompt to attend over. Anthropic reported its tool-search feature moving MCP eval accuracy from 49% to 74% on Opus 4 and 79.5% to 88.1% on Opus 4.5, with large tool libraries. Those are vendor benchmarks on their own evals, so don't expect the same deltas on your workload. But the direction matches what most people see: fewer, more relevant tools gets you better selection.

## Measure before you touch anything

Don't guess which tools are expensive. Count them. The simplest method that works on any provider is to send a minimal request and read `usage`. The marginal cost of a tool is the difference between a request with it and a baseline request.

```python
from openai import OpenAI

client = OpenAI()
MODEL = "gpt-5.5"  # whatever you actually run; tool search needs gpt-5.4+

# A trivial tool as the baseline, because adding *any* tool adds a fixed overhead.
NOOP = {"type": "function", "name": "noop", "description": "noop",
        "parameters": {"type": "object", "properties": {}}}

def input_tokens(tools: list[dict]) -> int:
    r = client.responses.create(
        model=MODEL, input="hi", tools=tools,
        max_output_tokens=16, store=False,
    )
    return r.usage.input_tokens

baseline = input_tokens([NOOP])

def cost_per_tool(tools: list[dict]) -> list[tuple[int, str]]:
    rows = [(input_tokens([t]) - baseline, t["name"]) for t in tools]
    return sorted(rows, reverse=True)

# for tokens, name in cost_per_tool(all_tools)[:20]: print(tokens, name)
```

You'll usually find the cost is concentrated: a dozen tools with essay-length descriptions, giant enums, or deeply nested schemas account for a disproportionate share. That's your first target. (The Responses API also has an input-token counting endpoint that avoids spending a real request. Check your SDK version for the exact method.)

In production, also log `usage` from every response: input tokens, output tokens, and `usage.input_tokens_details.cached_tokens`. Aggregate **per task**, not per call. Per-call numbers hide the loop multiplier.

## Stop sending tools the model won't use

This is the biggest lever. There are three ways to do it, and they're not exclusive.

### Native tool search

OpenAI's Responses API supports tool search on gpt-5.4 and later. You add `{"type": "tool_search"}` to `tools` and mark the functions you want deferred with `defer_loading: true`. The model sees deferred tools only after it searches for them. OpenAI recommends grouping functions into **namespaces** (or using MCP servers) instead of deferring loose functions, because its models are primarily trained to search those surfaces and the savings are usually larger there.

```python
crm = {
    "type": "namespace",
    "name": "crm",
    "description": "CRM tools for customer lookup and order management.",
    "tools": [
        {   # keep the few you use constantly callable immediately
            "type": "function",
            "name": "get_customer",
            "description": "Fetch a customer record by ID.",
            "parameters": {
                "type": "object",
                "properties": {"customer_id": {"type": "string"}},
                "required": ["customer_id"],
                "additionalProperties": False,
            },
        },
        {   # long tail: deferred, loaded through tool search
            "type": "function",
            "name": "list_open_orders",
            "description": "List open orders for a customer ID.",
            "defer_loading": True,
            "parameters": {
                "type": "object",
                "properties": {"customer_id": {"type": "string"}},
                "required": ["customer_id"],
                "additionalProperties": False,
            },
        },
    ],
}

resp = client.responses.create(
    model=MODEL,
    input="List open orders for customer CUST-12345.",
    tools=[crm, {"type": "tool_search"}],
    parallel_tool_calls=False,
)
```

Details that people get wrong:

- For a namespace, `defer_loading` applies to the functions inside it, not to the namespace object. A namespace can mix immediately-callable and deferred functions, so keep your few hot tools non-deferred and defer the long tail.
- There are two modes. In **hosted** mode OpenAI searches the deferred tools you declared and returns the loaded subset in the same response. In **client-executed** mode (`execution: "client"` on the `tool_search` tool) the model emits a `tool_search_call`, your application runs the lookup against whatever catalogue you like, and you return a `tool_search_output`. Use the client mode when the catalogue is too big or too dynamic to declare upfront, or when you want your own ranking.
- OpenAI says tool search is designed to preserve the cache: newly discovered tools are injected at the end of the context rather than rewriting what came before. I'd still verify with `cached_tokens` in your own traces.
- It adds a search step, so it adds latency. It pays off when definitions are large and most tools go unused in a given session. For a handful of compact tools, skip it.
- It's a Responses API feature. If you're on Chat Completions, or calling through an OpenAI-compatible gateway, confirm the gateway supports it before relying on it.
- Retrieval quality is the ceiling. Vendor benchmarks look good, but independent testing of Anthropic's equivalent against a catalogue of about 4,000 tools reported noticeably lower retrieval accuracy than the vendor numbers. Test on your own tools with your own queries.

The same idea exists elsewhere. Anthropic's API has a tool search tool with `defer_loading`, and Claude Code switches to it automatically (more on that below).

### Your own tool retrieval

If you're not on a model with tool search, or you want better ranking than the built-in search, build it. Index each tool's name, description and a few example queries. At request time, retrieve the top 5 to 15 using hybrid search (BM25 plus embeddings, fused with RRF), optionally rerank, and send those plus a fixed core set.

A minimal shape with a `find_tools` escape hatch:

```python
FIND_TOOLS = {
    "type": "function",
    "name": "find_tools",
    "description": "Search for additional tools by capability. Call this when no "
                   "available tool fits the task. Returns tool names; they become "
                   "available on the next step.",
    "parameters": {"type": "object",
                   "properties": {"query": {"type": "string"}},
                   "required": ["query"], "additionalProperties": False},
}

def run(task: str, core_tools: list[dict]):
    active = {t["name"]: t for t in core_tools}          # stable core
    items = [{"role": "user", "content": task}]
    while True:
        tools = sorted(active.values(), key=lambda t: t["name"]) + [FIND_TOOLS]
        r = client.responses.create(model=MODEL, input=items, tools=tools)
        items += r.output
        calls = [i for i in r.output if i.type == "function_call"]
        if not calls:
            return r.output_text
        for c in calls:
            args = json.loads(c.arguments)
            if c.name == "find_tools":
                hits = tool_index.search(args["query"], k=8)   # your hybrid retrieval
                active.update({h["name"]: h for h in hits})    # extend, never swap
                out = json.dumps([h["name"] for h in hits])
            else:
                out = to_output(dispatch(c.name, args))
            items.append({"type": "function_call_output",
                          "call_id": c.call_id, "output": out})
```

Practical points:

1. **Retrieve on the task, not the last message.** "Yes, go ahead" retrieves garbage. Use the original request or a running summary of intent.
2. **Give the model an escape hatch.** `find_tools` lets it recover when retrieval missed. Without it, one bad retrieval makes the task impossible.
3. **Mind the cache.** Tools sit at the front of the prompt. If you change the tool list on every turn, you invalidate the cache for everything behind it, which is the whole conversation. The loop above only ever extends the set, so it breaks the cache at most once per discovery. Native deferred loading avoids even that by injecting discovered tools at the end.
4. **Mind history that refers to tools you removed.** Manus found that earlier turns referencing tools no longer defined confuse the model (see the case studies). Extend, don't swap.

Measure **recall@k for tool retrieval** on a labelled set of real requests. If the right tool isn't in the retrieved set, no amount of prompt work will save you.

### Sub-agents by domain

Split tools by domain and put a router or supervisor in front. Each sub-agent sees 5 to 15 tools. The side benefit is context isolation: the supervisor receives a short result, not the sub-agent's whole working history.

The cost is real. You add a routing hop, more orchestration, and harder debugging, since failures can be a bad route, a bad tool choice, or a bad handoff. I'd reach for this when domains are genuinely distinct (billing vs infrastructure vs HR), not just to dodge a long tool list.

## Caching: the rules that actually bite

Prompt caching is what makes a large, stable tool block tolerable. The mechanics matter, because it's easy to destroy.

On OpenAI, caching is automatic for prompts of 1,024 tokens or more, and it's an exact-prefix match. The docs are explicit that this includes tools, which must be identical between requests. Edit one tool description and you've thrown away the cached system prompt and history behind it. Anthropic's version is the same hierarchy (tools, then system, then messages), with explicit breakpoints and a price premium for cache writes.

```python
SYSTEM = open("system_prompt.md").read()                  # long, stable, no timestamps
TOOLS = sorted(load_tools(), key=lambda t: t["name"])     # deterministic order

def call(items):
    r = client.responses.create(
        model=MODEL,
        instructions=SYSTEM,
        input=items,
        tools=TOOLS,
        prompt_cache_key="support-agent-v3",   # routes similar requests together
    )
    u = r.usage
    print(u.input_tokens, u.input_tokens_details.cached_tokens)
    return r
```

What to know:

- Check `cached_tokens` over time. A healthy agent loop shows most of the prefix coming from cache after the first call. Prompts under 1,024 tokens are never cached, silently.
- `prompt_cache_key` influences routing so requests with a shared prefix land where the cache is. It helps at volume with a common prefix. Don't put per-user values in it, or you'll fragment the cache.
- Retention, pricing and controls depend on the model family. Older models use `prompt_cache_retention` (in-memory, or extended up to 24 hours). Newer families (GPT-5.6 and later, per the current docs) add explicit cache breakpoints, a `prompt_cache_options` object, and separate cache-write reporting. Read the caching guide for the model you run, because this has been changing.
- Cache is per model. Moving a request to a smaller sibling model starts cold.

The failure modes I'd check first when the hit rate is bad:

1. Non-deterministic tool ordering (iterating a set or an unsorted dict of tools). Sort by name.
2. A timestamp, request ID or user name near the top of the system prompt. Manus's team calls out second-precision timestamps at the start of the prompt as a classic mistake.
3. Non-deterministic JSON serialisation. Many libraries don't guarantee key order, which can silently break the cache.
4. Tool descriptions generated dynamically, for instance with a "current date" embedded.
5. Tool list changing per turn because of retrieval.
6. Changing `tool_choice` between turns. Developers report it gets rendered into the context, so flipping it can break reuse. Test this on your model.

If you need to stop the model from calling something this turn, don't delete the tool. Restrict it with `allowed_tools`, which narrows what's callable without changing the `tools` array:

```python
r = client.responses.create(
    model=MODEL,
    input=items,
    tools=TOOLS,                       # unchanged, so the prefix stays valid
    tool_choice={
        "type": "allowed_tools",
        "mode": "auto",
        "tools": [
            {"type": "function", "name": "get_customer"},
            {"type": "function", "name": "list_open_orders"},
        ],
    },
)
```

Keep the allowed set itself stable across turns where you can, since it is part of the request. This is the closest thing a hosted API gives you to what Manus does with logit masking on its own inference stack.

## Make each definition smaller (but don't go overboard)

Once you've cut the count, trim what's left. This is lower leverage than the above, but it's cheap and it compounds across every request.

- Descriptions should say what the tool does and when to use it versus its neighbours. One or two sentences. If a tool needs a paragraph, it's probably doing too much.
- Delete parameter descriptions that restate the parameter name.
- Move rarely used optional parameters behind server-side defaults.
- Don't embed enormous enums. Accept a string, validate on the server, and return an error that lists close matches.
- Merge near-duplicates: `get_user_by_id` and `get_user_by_email` can often be `find_user(id?, email?)`.
- Use consistent name prefixes (`browser_`, `shell_`, `crm_`). Manus does this deliberately so groups of tools are easy to constrain and easy for the model to tell apart.

Merging has a limit. Past a point, it produces a single tool with mode-dependent parameters that the model gets wrong more often than the five simple tools it replaced. If you find yourself writing "if `action` is X then `foo` is required, otherwise ignore it" in the description, split it back up.

Examples can be worth their tokens too. A couple of well-chosen examples on a tricky tool can prevent retries that cost far more than the extra prompt text. Anthropic reported accuracy on complex parameter handling going from 72% to 90% with tool use examples. Spend them on tools the model gets wrong, not ones that are already obvious.

## Results are usually the bigger bill

Definitions are a fixed cost per call. Results are cumulative: once a large tool result lands in the history, it rides along on every later turn. In long sessions this tends to dominate, and it's the part teams forget to look at. One person who measured their own sessions in Cursor found tool results averaging about a third of total input tokens.

Design tools so they can't return unbounded output:

```python
import json

MAX_CHARS = 6_000

def search_tickets(query: str, limit: int = 10, cursor: str | None = None,
                   fields: list[str] | None = None) -> dict:
    rows, next_cursor = backend.search(query, limit=min(limit, 50), cursor=cursor)
    keep = fields or ["id", "title", "status", "assignee"]
    return {
        "results": [{k: r[k] for k in keep if k in r} for r in rows],
        "next_cursor": next_cursor,
        "truncated": next_cursor is not None,
    }

def to_output(obj) -> str:
    s = json.dumps(obj, separators=(",", ":"))      # no pretty-printing
    if len(s) > MAX_CHARS:
        return s[:MAX_CHARS] + '..."[truncated: narrow the query or request fewer fields]"'
    return s
```

The cap lives at the boundary where results re-enter the loop, so no individual tool can bypass it:

```python
def run_agent(user_msg: str) -> str:
    items = [{"role": "user", "content": user_msg}]
    while True:
        r = call(items)
        items += r.output                       # keep the model's output items in history
        calls = [i for i in r.output if i.type == "function_call"]
        if not calls:
            return r.output_text
        for c in calls:
            result = dispatch(c.name, json.loads(c.arguments))
            items.append({
                "type": "function_call_output",
                "call_id": c.call_id,
                "output": to_output(result),
            })
```

The pattern is a hard cap on rows, a small default field set, a cursor for more, and an explicit `truncated` flag so the model knows there's more and can ask. A few more habits that pay off:

- Do filtering and aggregation server-side. "How many open P1s?" shouldn't ship 400 tickets.
- Strip HTML, base64 and boilerplate from fetched pages before returning them.
- For genuinely large outputs, write them to a file and return a path plus a short preview. Both Cursor and Manus do this.
- Keep errors short and actionable.

Then manage the history itself. `previous_response_id` lets you chain turns without resending input yourself, but those tokens are still part of the context, so it saves plumbing, not tokens. For long-running agents, use compaction (the Responses API has server-side compaction, and you can summarise yourself): keep decisions, open questions and key facts, and keep durable notes in files the agent can re-read. One caution that applies everywhere: rewriting history changes the prefix and costs you cache hits, so compact in large, infrequent steps rather than trimming a little every turn. Manus adds a useful rule here: only drop something from the context if the agent can get it back, for example drop a page's contents but keep its URL.

## Let code process the data

If a task involves filtering, joining or aggregating large tool outputs, routing all of it through the model is the expensive way to do it.

The pattern: instead of exposing hundreds of tools, expose a tiny surface. A `search` tool finds relevant operations, an `execute` tool runs code in a sandbox that calls the real API, and only the printed result comes back. Intermediate data never enters the context. A simplified version in OpenAI function-tool form:

```python
CODE_MODE_TOOLS = [
    {
        "type": "function",
        "name": "search_api",
        "description": "Find API operations relevant to a task. Returns method, "
                       "path, summary and parameter names for the best matches.",
        "parameters": {"type": "object",
                       "properties": {"query": {"type": "string"}},
                       "required": ["query"], "additionalProperties": False},
    },
    {
        "type": "function",
        "name": "execute",
        "description": "Run Python in a sandbox. An authenticated `api` client is "
                       "pre-loaded. Filter and aggregate in code; only stdout is "
                       "returned, so print only what you need.",
        "parameters": {"type": "object",
                       "properties": {"code": {"type": "string"}},
                       "required": ["code"], "additionalProperties": False},
    },
]

def dispatch(name: str, args: dict):
    if name == "search_api":
        return api_index.search(args["query"], k=8)
    if name == "execute":
        res = sandbox.run(args["code"], timeout_s=30, memory_mb=512, network=False)
        return {"stdout": res.stdout[-4000:], "error": res.error}
```

Two related approaches exist in the wild. **Code execution with MCP** presents servers as a code API (for instance a directory of typed functions) that the agent explores and calls from code. Anthropic's example, copying a meeting transcript from Google Drive into a Salesforce lead, dropped from about 150K tokens to about 2K. Be careful with that number. It's one demonstrated workflow where the intermediate data was large, not a general expectation. **Programmatic tool calling** does something similar inside a provider-hosted sandbox. Anthropic reports about a 37% token reduction on complex research tasks for its version, plus far fewer inference round trips, though it has platform gaps and data-handling constraints worth checking.

Both need real infrastructure: a sandbox with resource limits, timeouts, no ambient network, audit logs, and approval or dry-run for anything that mutates state. If your tool outputs are small, the sandbox costs more than the tokens it saves. Don't adopt it because it's fashionable. Adopt it when you can point at a workflow where results are large and processing is mechanical.

## How real teams handle this

The techniques above aren't theoretical. Here's what several teams actually shipped, what they measured, and what the measurement does and doesn't tell you. They don't all agree, which is the useful part.

### Cursor: defer MCP tools and put long outputs in files

Cursor's "dynamic context discovery" (January 2026) syncs MCP tool descriptions into a folder, one folder per server. The agent gets a short static list of tool names and looks up details when a task needs them. Their A/B test found a **46.9% reduction in total agent tokens** for runs that called an MCP tool, statistically significant but with high variance depending on how many MCP servers were installed. They apply the same idea to other bulk: long tool responses are written to files the agent can `grep` or `tail`, and terminal sessions are synced to files too.

Read the number carefully. It covers runs that actually called an MCP tool, and per independent summaries the test focused on token usage rather than task quality. It's real evidence that deferring works at scale, not proof of free lunch.

### Claude Code: lazy-load once tools get big

Claude Code's MCP Tool Search (v2.1.7, January 2026) automatically switches from preloading to search-based loading when MCP tool descriptions would take more than 10% of the context window. Anthropic's own motivating complaints were setups of 7+ servers eating 67K+ tokens, and reported internal testing put one case at roughly 134K tokens down to about 5K (a secondhand figure, so treat it as indicative). The design choice worth copying is the **threshold**: small tool sets stay eager, because search has a latency cost, and the switch happens only when it pays.

### Cloudflare: collapse 2,500 endpoints into two tools

Cloudflare's API has over 2,500 endpoints. Exposing each as an MCP tool would cost about 1.17 million tokens with full schemas (about 244K even with only required parameters), more than most context windows. Their MCP server instead exposes two tools, `search()` and `execute()`, in roughly **1,000 tokens** total. The model writes code to search the OpenAPI spec and to call the API, and that code runs in an isolated Worker sandbox with no filesystem, no exposed environment variables, and controlled outbound requests. When they add a product, no new tool definitions are needed. Their numbers were measured with tiktoken, and the sandbox is a real part of the cost of this approach, which is exactly what the example above sketches.

### Vercel: replace 18 tools with one

Vercel's internal text-to-SQL agent, d0, started with 18 specialised tools, heavy prompt engineering and careful context management. It was fragile and slow. They replaced most of it with a single bash tool over a file-based semantic layer (YAML, Markdown, JSON files the model could `ls`, `grep` and `cat`). On their benchmark: success rate from 80% to 100%, average time from about 275s to about 77s, tokens from about 102K to about 61K, steps from about 12 to about 7.

The caveat is in the sample: it was five representative queries, so treat the percentages as directional, not statistical. It also worked because the underlying data was well structured as files and the model was strong at shell use. The transferable lesson isn't "use bash". It's that a general tool over well-organised data can beat a pile of narrow tools, and that every special-purpose tool is something the model might choose wrongly.

### Manus: keep the tool set stable, constrain instead of remove

Manus is the contrarian data point. Their team says KV-cache hit rate is the most important metric for a production agent, with an average input-to-output ratio around 100:1, and that cached tokens can cost roughly a tenth of uncached ones. They tried loading tools dynamically with something RAG-like and walked it back, for two reasons: tool definitions sit near the front of the context, so any change invalidates the cache for everything after, and earlier actions that reference tools no longer defined confuse the model, which leads to schema violations or hallucinated actions.

Their approach: keep the full tool set stable, and use a state machine plus **logit masking** during decoding to prevent or enforce certain tool choices. Tool names share prefixes (`browser_`, `shell_`) so whole groups can be constrained cheaply. For bulk data they treat the file system as context, with "restorable" compression: drop a page body but keep the URL, drop a document body but keep the path.

Masking logits needs control of the decoding stack, which you don't have on a hosted API. On OpenAI the nearest equivalent is `allowed_tools`, shown above.

### What the examples add up to

| Team | Move | Reported result | Main caveat |
|---|---|---|---|
| Cursor | Defer MCP tool details, long outputs to files | 46.9% fewer tokens in MCP-calling runs | Tokens measured, not quality; high variance |
| Claude Code | Auto lazy-load past 10% of context | ~134K to ~5K reported in one case | Secondhand figure; adds a search step |
| Cloudflare | Two tools (search, execute) over 2,500 endpoints | ~1.17M tokens to ~1K | Needs a hardened sandbox |
| Vercel | 18 tools to one bash tool | 80% to 100% success, ~37% fewer tokens | Five-query benchmark; data fit well as files |
| Manus | Stable tool set, mask instead of remove | Cache hit rate treated as the key metric | Needs control of decoding |
| Anthropic (Drive to Salesforce) | Code execution with MCP | ~150K to ~2K tokens | One workflow with large intermediate data |

Notice they split into two camps. Cursor, Claude Code and OpenAI's tool search **defer** definitions and rely on the provider or client to keep caching intact. Cloudflare and Vercel **shrink the surface** by putting a general mechanism in front of the data. Manus **keeps everything and constrains choice**, because for them cache stability outweighs the cost of the extra tokens. Which is right depends on your stack: whether you control inference, how big the catalogue is, and how much of each session reuses the same prefix.

## MCP hygiene

MCP makes it a one-line change to add 30 tools to an agent. Treat that as a cost, not a convenience.

- Enable servers per agent or per task, not globally.
- Allow-list the specific tools you need from each server, rather than importing everything.
- Prefer servers that paginate, support field selection, and return compact output. Wrap the ones that don't. Cursor's team makes the point that it isn't realistic to expect every MCP server to optimise for this, so the client has to.
- If a third-party description is long-winded, override it with your own shorter one in your client.
- With OpenAI's remote MCP tool, set `defer_loading: true` on the MCP server definition so its tools load through tool search instead of up front.
- Track tools-per-agent and definition tokens per agent as metrics, and alert on growth.

## What I'd do, in order

If I were handed an agent with a big tool bill, I'd work through it like this:

1. **Instrument it.** Per-task input tokens, cached tokens, and tool-selection accuracy on a fixed eval set. Without the eval set you can't tell whether a saving cost you quality.
2. **Fix the cache.** Deterministic ordering, stable prefix, no volatile text at the top. This is often a one-day change with an immediate cost drop.
3. **Bound tool results:** caps, pagination, field selection, large outputs to files. Look at the five chattiest tools first.
4. **Defer the long tail.** Namespaces plus `tool_search`, or your own retrieval, keeping a small core loaded. Apply it only above a size threshold, as Claude Code does. Re-run the eval.
5. **Trim the top offenders** from the per-tool report.
6. **Only then** consider sub-agents or a code-execution surface, and only for the workflows that justify the complexity.

Judge each step on three numbers: tokens per task, task success rate, and p95 latency. A change that saves 40% of tokens and drops success by 5 points is not a win, and tool search in particular can trade latency for tokens.

## Where this goes wrong

- **Optimising definitions while results balloon.** Check which one is actually larger in your traces.
- **Per-turn tool lists that silently kill caching.** Hit rate drops, nobody notices, cost goes up.
- **Swapping tools out mid-task** so history refers to tools that no longer exist. Extend the set instead.
- **Over-merging tools** into one multi-mode monster.
- **Retrieval with no fallback.** One miss, dead task.
- **Trusting vendor benchmarks** on your own catalogue. They're a reason to try the feature, not evidence it works for you.
- **Copying a case study's number** without its caveat. Vercel's five queries and Cursor's MCP-calling-runs-only are not your workload.
- **Trimming descriptions until tools become ambiguous**, which turns saved tokens into wrong-tool retries.
- **Connecting every MCP server by default.**
- **Assuming an OpenAI-compatible gateway passes through tool search, `cached_tokens` and cache options.** Many don't. Check the raw response.

The fix for a big tool catalogue isn't a trick. It's showing the model only what it needs for the current step, keeping the part that doesn't change cheap through caching, and keeping bulky data out of the conversation. Measure first, and let the numbers tell you which of those matters most for your system.

## Sources

- OpenAI API docs: Tool search, Prompt caching, Function calling (developers.openai.com)
- Cursor, "Dynamic context discovery" (cursor.com/blog/dynamic-context-discovery)
- Cloudflare, "Code Mode: give agents an entire API in 1,000 tokens" (blog.cloudflare.com/code-mode-mcp)
- Manus, "Context Engineering for AI Agents: Lessons from Building Manus" (manus.im/blog)
- Vercel, "We removed 80% of our agent's tools" (vercel.com/blog/we-removed-80-percent-of-our-agents-tools)
- Anthropic Engineering: "Introducing advanced tool use on the Claude Developer Platform" and "Code execution with MCP"
- Claude Code MCP Tool Search coverage (VentureBeat and release notes for v2.1.7)