---
title: "When agent trace metrics lie: the span tree double-counting problem"
date: 2026-03-20
description: "When agent traces are trees, naive aggregation of cost, tokens, and step counts produces wrong numbers. Here's the problem, what major platforms do about it, and the concrete approaches that work."
tags: ["AI agents", "observability", "OpenInference", "evaluation"]
draft: false
---

I was building OpenInference support for an agent trace comparison tool when the token counts came back double what they should have been. The code was simple — sum tokens across all spans in a trace. The bug was that "all spans" included orchestration wrappers that carried their children's totals. Nothing crashed. The numbers just looked plausible enough to ship.

This is the span tree double-counting problem. It's not hard to fix once you see it, but it's easy to miss because the wrong numbers look reasonable.

## The tree

In OpenInference and OpenTelemetry, agent traces are trees. A root AGENT or CHAIN span wraps child spans — LLM calls, tool invocations, retrievals. Those children can have children of their own. A planning step spawns sub-queries. A tool call triggers an LLM to parse the result. The depth is arbitrary.

```
root (AGENT)
├── plan (LLM)         ← 500 input, 200 output tokens, $0.02
├── search (TOOL)      ← no tokens, no cost
│   └── parse (LLM)   ← 300 input, 100 output tokens, $0.01
└── respond (LLM)      ← 800 input, 400 output tokens, $0.03
```

Four spans. Three are LLM calls with token counts and costs. One is a tool invocation. The AGENT span at the root is an orchestration wrapper — it didn't make an LLM call itself.

The total cost is $0.06. The total tokens are 2,300. Straightforward — you sum the three LLM spans.

But whether this works depends entirely on your instrumentation. What happens when parent spans *also* carry token and cost attributes?

## Where things go wrong

Some instrumentations record aggregated subtotals on parent spans. A parent AGENT span might carry `total_tokens: 2300` — the sum of its children. If you now sum *all* spans, you get 4,600 tokens. Double the actual value.

This isn't hypothetical. Langfuse has [active](https://github.com/langfuse/langfuse/issues/11244) [bug](https://github.com/langfuse/langfuse/issues/10914) [reports](https://github.com/orgs/langfuse/discussions/11252) about it. When both a parent and child observation are typed as GENERATION, Langfuse sums tokens from both. The [Microsoft Agent Framework](https://github.com/orgs/langfuse/discussions/11252) integration had to specifically mark parent observations as type SPAN (not GENERATION) to avoid the double-count.

What makes this tricky is that no specification forbids putting aggregated values on parent spans. OpenInference defines `llm.token_count.prompt` and `llm.cost.total` as span-level attributes but doesn't say "only attach these to leaf spans." OpenTelemetry's GenAI semantic conventions define `gen_ai.usage.input_tokens` on LLM call spans but don't warn about aggregation — and these conventions are still in [Development status](https://opentelemetry.io/docs/specs/semconv/gen-ai/), so the attributes themselves may change before stabilization. The convention — cost and tokens live only on the actual LLM call — is implicit, not specified.

When conventions are implicit, they get violated. And the violations are silent — your numbers are wrong, but nothing crashes. Not every dataset has this problem, which makes it harder to catch when one does.

## What to do about it

The double-counting issue manifests differently for each metric type. Here's how I handle each one.

### Cost and tokens

The approach I landed on: `sum(s.cost for s in spans if s.cost is not None)`.

This works regardless of span kind taxonomy because it relies on the data, not the labels. Orchestration spans with `None` cost are excluded. LLM spans with `0.0` cost (cached responses, free-tier models) are correctly included. Non-LLM spans that legitimately have cost (paid API tool calls) are also correctly included. In my experience, this is more robust than filtering by span kind, which requires knowing every possible kind value across every instrumentation library.

The `None` vs `0` distinction is critical here and easy to get wrong. `None` means "this span didn't measure cost" — a TOOL span, a CHAIN wrapper. `0.0` means "this span measured cost and it was zero" — a cached LLM response, a free-tier model call. If you collapse `None` to `0` before summing — a common shortcut — you lose the ability to tell "no cost data" from "genuinely free." Your medians shift toward zero, your comparisons break, and you won't see it in the output because zero looks reasonable.

### Step count

This one is less about correctness and more about what you're trying to measure. A 3-step agent (plan, search, respond) wrapped in a CHAIN has 4 total spans. `len(spans)` returns 4, not 3. Whether that's "wrong" depends on the question. If you're asking "how complex is this trace's orchestration," total span count is fine. If you're asking "how many things did the agent actually do," I found leaf spans — spans with no children — to be more useful. The orchestration wrappers are envelopes, not actions. Though it's worth noting that the boundary isn't always clean — a "search" step might be a parent span that delegates to an LLM call for query parsing. In that case, "search" is a logical step but not a leaf. What you're really counting with leaves is execution primitives, not logical operations.

```python
parent_ids = {s.parent_id for s in spans if s.parent_id}
leaves = [s for s in spans if s.span_id not in parent_ids]
```

One caveat: if the tree is incomplete — child spans missing due to instrumentation gaps or partial exports — a parent will look like a leaf and inflate the count. In practice this is rare with well-instrumented code, but worth knowing about.

### Duration

Summing span durations is always wrong for traces — a parent span's duration overlaps its children. What you want is wall-clock time: `max(end_time) - min(start_time)` across all spans. That gives you total elapsed time without double-counting overlapping execution. This works correctly even when branches execute in parallel.

But for per-span analysis — comparing "how long does the search step take across 100 traces" — each span's own duration is exactly right. Even for parent spans, where the duration tells you how long that sub-pipeline consumed end-to-end. Group by span name, compare independently. This is valid at any tree depth because you're comparing the same span across traces, not summing different spans within a trace.

## How major platforms handle it

The major observability platforms all address this, though the reasoning behind their approaches isn't always well-documented. Here's what I've gathered from their docs and public data.

**Phoenix / OpenInference** relies on span kind. The [OpenInference semantic conventions](https://arize-ai.github.io/openinference/spec/semantic_conventions.html) define `llm.token_count.*` and `llm.cost.*` attributes specifically for [LLM spans](https://github.com/Arize-ai/openinference/blob/main/spec/llm_spans.md) — CHAIN, AGENT, and TOOL spans don't typically carry them. Phoenix also [computes cost server-side](https://arize.com/docs/phoenix/tracing/how-to-tracing/cost-tracking) by combining token counts with built-in model pricing tables, rather than relying on pre-computed cost attributes on spans — the two public Phoenix trace datasets I tested (`context-retrieval` and `random`) have no `llm.cost.*` attributes, consistent with this.

**Langfuse** uses [observation types](https://langfuse.com/docs/observability/features/observation-types): generation, span, embedding, and several others. Only generation and embedding carry [cost and token data](https://langfuse.com/docs/observability/features/token-and-cost-tracking). When the Microsoft Agent Framework integration produced double-counts, the [documented fix](https://github.com/orgs/langfuse/discussions/11252) was to override the parent's observation type to `span` so it wouldn't be treated as a generation. The architecture is sound, but it depends on the instrumentation getting the type right.

**LangSmith** records token usage on LLM call runs. Their [cost tracking docs](https://docs.langchain.com/langsmith/cost-tracking) describe the trace tree as showing "the total usage for the entire trace, aggregated values for each parent run, and token and cost breakdowns for each child run." The docs don't specify whether parent aggregation is computed at display time or stored, but the architecture clearly separates individual run data from rolled-up totals.

**Braintrust** takes a different approach — they fix the source rather than filtering at consumption. Their [v3.1.0 changelog](https://www.braintrust.dev/docs/changelog) notes a fix for "token double counting between parent and child spans in Vercel AI SDK integration." Their data model supports DAG-structured spans, and aggregation happens at query time via their BTQL language rather than at export time.

The common thread: every platform puts cost and tokens on the actual LLM call, not on the orchestration wrapper. The convention exists. It's just not documented as a rule that instrumentation authors are expected to follow.

## What I wish the specs said

One thing that would help: an explicit statement in the OpenInference or OpenTelemetry GenAI specifications that `llm.token_count.*` and `llm.cost.*` attributes represent the span's own values, not cumulative subtotals of children. One sentence would turn an implicit convention into a guarantee that instrumentation authors can code against. The OTel GenAI conventions are still experimental — now, while they're still being shaped, is the right time to clarify this.

Until that happens, defensive coding is the practical answer: filter on `None` for aggregation, don't assume every instrumentation follows the convention, and validate against known-good data before trusting the numbers.

---

*I ran into this while building [OpenInference support in Kalibra](https://github.com/khan5v/kalibra), a regression detection tool for AI agent traces. The tree aggregation problem was one of the design decisions that required real thought — not because it's algorithmically hard, but because getting it wrong produces numbers that look right.*
