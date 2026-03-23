---
title: "When agent trace metrics lie: the span tree double-counting problem"
date: 2026-03-20
description: "When agent traces are trees, naive aggregation of cost, tokens, and step counts produces wrong numbers. Here's the problem, what major platforms do about it, and the concrete approaches that work."
tags: ["AI agents", "observability", "OpenTelemetry", "OpenInference", "evaluation"]
ogImage: "/og-span-tree-aggregation.png"
draft: false
---

I was building OpenInference support for an agent trace comparison tool when the token counts came back double what they should have been. The code was simple — sum tokens across all spans in a trace. The bug was that "all spans" included orchestration wrappers that carried their children's totals. Nothing crashed. The numbers just looked plausible enough to ship.

This is the span tree double-counting problem. It's not hard to fix once you see it, but it's easy to miss because the wrong numbers look reasonable.

## The tree

Agent traces are trees. This isn't a new data structure — [OpenTelemetry](https://opentelemetry.io/docs/concepts/signals/traces/) has used tree-structured traces for distributed systems since long before LLMs were mainstream. [OpenInference](https://github.com/Arize-ai/openinference), the AI-specific semantic convention layer built on top of OpenTelemetry, inherits this model and adds span kinds tailored to AI workloads: LLM, TOOL, CHAIN, AGENT, RETRIEVER, and others. Every OpenInference trace is a valid [OTLP](https://opentelemetry.io/docs/specs/otel/protocol/) trace — the conventions give attribute names their AI-specific meaning.

A root AGENT or CHAIN span wraps child spans — LLM calls, tool invocations, retrievals. Those children can have children of their own. A planning step spawns sub-queries. A tool call triggers an LLM to parse the result. The depth is arbitrary.

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

## An old problem in new clothes

If you've worked with distributed tracing, you might recognize this. Traditional Application Performance Monitoring (APM) has dealt with a version of it for years under the name **self-time** (or **exclusive time**) — the duration a span spends doing its own work, excluding time enclosed by children. Elastic APM computes [`span.self_time`](https://www.elastic.co/guide/en/observability/current/apm-data-model-metrics.html) metrics specifically for this: they subtract child durations from the parent's total to produce a breakdown visualization that doesn't double-count.

The AI-specific twist is that the double-counting isn't about **duration** — which is inherently hierarchical, since parent spans enclose children by definition. It's about **metric values on spans**: tokens and costs. These are point measurements that should live on the specific span that generated them. They are not hierarchical quantities. When a parent span carries `total_tokens: 2300` as a subtotal of its children, and you sum across all spans, you get 4,600 tokens. Double the actual value.

Duration double-counting is a display and analysis problem — the data itself is correct, you just need to compute self-time. Token and cost double-counting is a data problem — the same value exists in two places, and the spec doesn't tell you which one is the source of truth.

## Where things go wrong

Some instrumentations record aggregated subtotals on parent spans. A parent AGENT span might carry `total_tokens: 2300` — the sum of its children. If you now sum *all* spans, you get 4,600 tokens.

This isn't hypothetical. Langfuse has seen [related](https://github.com/langfuse/langfuse/issues/10914) [reports](https://github.com/orgs/langfuse/discussions/11252) surface in different forms. The [Microsoft Agent Framework](https://github.com/orgs/langfuse/discussions/11252) integration ran into it directly: the framework's `invoke_agent` spans carried a `gen_ai.request.model` attribute, which caused Langfuse to classify them as generations and infer token counts — even though the framework explicitly set `capture_usage=False`. The result: both the orchestration span and the nested LLM calls got counted, doubling the totals. The presence of a `model` attribute on a non-LLM span was enough to trigger it.

What makes this tricky is that no specification forbids putting aggregated values on parent spans. OpenInference defines `llm.token_count.prompt` and `llm.cost.total` as span-level attributes but doesn't say "only attach these to leaf spans." OpenTelemetry's [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/) define `gen_ai.usage.input_tokens` on inference spans but don't warn about aggregation. These conventions are still in [Development status](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — the earliest maturity level in OTel's lifecycle — and they define no cost attributes at all. The convention — cost and tokens live only on the actual LLM call — is implicit, not specified.

When conventions are implicit, they get violated. And the violations are silent — your numbers are wrong, but nothing crashes. Not every dataset has this problem, which makes it harder to catch when one does.

And here's why you can't just detect it after the fact: imagine a parent span with `cost: $0.05` and two children costing `$0.02` and `$0.03`. Is the parent's cost an aggregated subtotal of its children — meaning you should ignore it — or did the parent make its own LLM call that happened to cost `$0.05`? That's not a contrived scenario: an orchestration step that reasons about which tool to call *and then* delegates to children is both an LLM caller and a parent. You can't distinguish "aggregated subtotal" from "coincidentally equal own cost" by looking at the numbers alone.

And this compounds: in a tree of arbitrary height, you're not double-counting — you're potentially N-counting, with the ambiguity multiplying at every level.

## What to do about it

The double-counting issue manifests differently for each metric type. Here's how I handle each one.

### Cost and tokens

The approach I landed on: `sum(s.cost for s in spans if s.cost is not None)`.

This works regardless of span kind taxonomy because it relies on the data, not the labels. Orchestration spans with `None` cost are excluded. LLM spans with `0.0` cost (cached responses, free-tier models) are correctly included. Non-LLM spans that legitimately have cost (paid API tool calls) are also correctly included. In my experience, this is more robust than filtering by span kind, which requires knowing every possible kind value across every instrumentation library.

The `None` vs `0` distinction is critical here and easy to get wrong. `None` means "this span didn't measure cost" — a TOOL span, a CHAIN wrapper. `0.0` means "this span measured cost and it was zero" — a cached LLM response, a free-tier model call. If you collapse `None` to `0` before summing — a common shortcut — you lose the ability to tell "no cost data" from "genuinely free." Your medians shift toward zero, your comparisons break, and you won't see it in the output because zero looks reasonable.

This approach works because the convention places cost and token data exclusively on the spans that generated them — orchestration spans have `None`, not a subtotal. It's a pragmatic shortcut, not a general solution: if a parent span carried an aggregated subtotal as a real value, None-filtering would silently include it. You'd need true self-time-style subtraction to handle that case. But in practice, the convention holds often enough that filtering on `None` is the more robust default.

### Step count

This one is less about correctness and more about what you're trying to measure. A 3-step agent (plan, search, respond) wrapped in a CHAIN has 4 total spans. `len(spans)` returns 4, not 3. Whether that's "wrong" depends on the question. If you're asking "how complex is this trace's orchestration," total span count is fine. If you're asking "how many things did the agent actually do," I found leaf spans — spans with no children — to be more useful. The orchestration wrappers are envelopes, not actions. Though it's worth noting that the boundary isn't always clean — a "search" step might be a parent span that delegates to an LLM call for query parsing. In that case, "search" is a logical step but not a leaf. What you're really counting with leaves is execution primitives, not logical operations.

```python
parent_ids = {s.parent_id for s in spans if s.parent_id}
leaves = [s for s in spans if s.span_id not in parent_ids]
```

One caveat: if the tree is incomplete — child spans missing due to instrumentation gaps or partial exports — a parent will look like a leaf and inflate the count. In practice this is rare with well-instrumented code, but worth knowing about.

### Duration

Summing span durations is always wrong for traces — a parent span's duration overlaps its children. This is the classic self-time problem that APM tools have solved at the visualization layer. What you want for trace-level duration is wall-clock time: `max(end_time) - min(start_time)` across all spans. That gives you total elapsed time without double-counting overlapping execution. This works correctly even when branches execute in parallel.

But for per-span analysis — comparing "how long does the search step take across 100 traces" — each span's own duration is exactly right. Even for parent spans, where the duration tells you how long that sub-pipeline consumed end-to-end. Group by span name, compare independently. This is valid at any tree depth because you're comparing the same span across traces, not summing different spans within a trace.

## How major platforms handle it

The major observability platforms all address this, though the reasoning behind their approaches isn't always well-documented. Here's what I've gathered from their docs and public data.

**Phoenix / OpenInference** relies on span kind. The [OpenInference semantic conventions](https://arize-ai.github.io/openinference/spec/semantic_conventions.html) define `llm.token_count.*` and `llm.cost.*` attributes specifically for [LLM spans](https://github.com/Arize-ai/openinference/blob/main/spec/llm_spans.md) — CHAIN, AGENT, and TOOL spans don't typically carry them. Phoenix also [computes cost server-side](https://arize.com/docs/phoenix/tracing/how-to-tracing/cost-tracking) by combining token counts with built-in model pricing tables, rather than relying on pre-computed cost attributes on spans — the two public Phoenix trace datasets I tested (`context-retrieval` and `random`) have no `llm.cost.*` attributes, consistent with this.

**Langfuse** uses [observation types](https://langfuse.com/docs/observability/features/observation-types): generation, span, embedding, and several others. Only generation and embedding carry [cost and token data](https://langfuse.com/docs/observability/features/token-and-cost-tracking). When the Microsoft Agent Framework integration produced double-counts, the root cause was that any span with a `model` attribute was auto-classified as a generation. The [discussion](https://github.com/orgs/langfuse/discussions/11252) shows this is still being worked through — the architecture is sound, but it depends on the instrumentation not accidentally triggering the heuristic.

**LangSmith** records token usage on LLM call runs. Their [cost tracking docs](https://docs.langchain.com/langsmith/cost-tracking) describe the trace tree as displaying total usage for the entire trace, aggregated values for each parent run, and token and cost breakdowns for each child run. The docs don't specify whether parent aggregation is stored or computed at display time, but the architecture clearly separates individual run data from rolled-up totals.

**Braintrust** fixes the source rather than filtering at consumption. Their [v3.1.0 changelog](https://www.braintrust.dev/docs/changelog) notes a fix for "token double counting between parent and child spans in Vercel AI SDK integration." Their data model supports DAG-structured spans, and aggregation happens at query time via their BTQL language rather than at export time.

The common thread: every platform puts cost and tokens on the actual LLM call, not on the orchestration wrapper. The convention exists. It's just not documented as a rule that instrumentation authors are expected to follow.

## The spec gap

Traditional APM solved duration double-counting by establishing self-time as a first-class concept — Elastic APM has [`span.self_time`](https://www.elastic.co/guide/en/observability/current/apm-data-model-metrics.html) metrics, and most APM UIs distinguish between a span's total time and its exclusive time. The solution was baked into the tooling because the problem was well-understood.

AI trace metrics don't have an equivalent. Neither OpenInference nor the OpenTelemetry [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) specify whether `llm.token_count.*` or `gen_ai.usage.*` attributes represent the span's own values or cumulative subtotals of children. The conventions — still at the earliest maturity level, with work begun in [early 2024](https://opentelemetry.io/blog/2024/otel-generative-ai/) — don't define cost attributes at all. The [agent spans spec](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/) defines span types but says nothing about token or cost rollup. OpenInference, which does define `llm.cost.total`, is [ahead here](https://arize-ai.github.io/openinference/spec/semantic_conventions.html) but still doesn't clarify the aggregation semantics.

One sentence in either spec would fix this: *"Token count and cost attributes on a span represent that span's own values, not cumulative subtotals of descendant spans."* That turns an implicit convention into a guarantee that instrumentation authors can code against. Now, while the conventions are still being shaped, is the time to say it.

Until that happens, defensive coding is the practical answer: filter on `None` for aggregation, don't assume every instrumentation follows the convention, and validate against known-good data before trusting the numbers.

---

*I ran into this while building [OpenInference support in Kalibra](https://github.com/khan5v/kalibra), a regression detection tool for AI agent traces. The tree aggregation problem was one of the design decisions that required real thought — not because it's algorithmically hard, but because getting it wrong produces numbers that look right.*
