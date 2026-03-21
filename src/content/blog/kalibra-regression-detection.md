---
title: "Aggregate metrics are a blind spot in agent evaluation"
date: 2026-03-19
description: "Why aggregate eval metrics hide AI agent regressions, and how statistical testing catches what aggregates miss."
tags: ["AI agents", "evaluation", "open source", "testing"]
draft: false
---

There's a failure mode in agent evaluation that I keep seeing.

An agent's overall success rate goes up. Cost drops. Latency is flat. The eval looks like an improvement across the board. But underneath, the agent got slightly better at simple, high-volume tasks that make up most of the dataset — while completely losing the ability to handle a critical subset. Five task types that always passed now fail on every run.

And here's the part that breaks intuition: the cost dropped *because* the agent got worse. In traditional software, lower latency and cost are universal wins. In agent architecture, lower cost often means the agent gave up early or failed to trigger its recovery loops. The broken tasks stopped making expensive tool calls. A cost improvement that's actually a symptom of failure.

The aggregate improved. The agent got worse. And any pipeline that only checks the top-line number would ship it.

## Why top-line metrics aren't enough

For traditional software, tracking an aggregate works. Request latency is roughly normal. Error rates are stable per endpoint. You set a threshold, you sleep well.

AI agents break this assumption:

- **They're nondeterministic:** Same input, different execution paths.
- **They're multi-step:** A 12-step trace and a 3-step trace aren't the same workload.
- **They have structured failure modes:** An agent can get better at search and worse at code editing in the same release.

An aggregate over heterogeneous, multi-step, nondeterministic executions hides more than it reveals. What you actually need is population-level comparison — not "what's the number" but "did these two populations actually differ, and where exactly."

## This is the problem Kalibra tries to solve

[Kalibra](https://github.com/khan5v/kalibra) is an open-source CLI that takes two JSONL trace files — from any eval harness, from Langfuse or Braintrust exports, or from your own scripts — and compares them as statistical populations. 10 metrics, one command.

Here's what a comparison looks like. The top-level metrics all point up — success rate, cost, everything looks better:

```
Kalibra Compare
──────────────────────────────────────────────────────────
Baseline       100 traces   (baseline.jsonl)
Current        100 traces   (current.jsonl)
Gates         ✗ 1/2 failed

▲ Success rate      50.0% → 75.0%  +25.0 pp
▲ Cost              $0.036 → $0.021 median  -40.5%
~ Duration          7.6s → 7.5s median  -1.3%
▼ Error rate        0.2% → 4.3%  +4.1 pp
```

But the per-trace breakdown tells a different story:

```
Trace breakdown
~ Per trace         20 matched — ✓ 10 improved, ✗ 5 regressed
                    ▼ draft-email       succeeded: 5/5 → 0/5
                    ▼ extract-receipt    succeeded: 5/5 → 0/5
                    ▼ parse-invoice      succeeded: 5/5 → 0/5
                    ▼ summarize-report   succeeded: 5/5 → 0/5
                    ▼ translate-doc      succeeded: 5/5 → 0/5

Quality gates
  [ OK ] success_rate_delta >= -5
  [FAIL] regressions <= 3             actual: 5

──────────────────────────────────────────────────────────
FAILED — quality gate violation (exit code 1)
```

Five task types went from 100% to 0%. The `regressions <= 3` gate caught it directly — exit code 1, deploy blocked.

## What's under the hood

Each metric compares two populations independently:

- **Proportions** (success rate, error rate): two-proportion z-test. Tests whether the rates actually differ or you're looking at sampling noise.
- **Continuous values** (cost, duration, tokens): bootstrap 95% confidence intervals on the median delta. If the CI includes zero, the change isn't statistically real — even if the point estimate looks large.
- **Per-trace breakdown**: groups traces by task identifier, compares success rates per group. This is where "the aggregate was fine" regressions live — one group improves, another breaks, the number stays flat.
- **Per-span breakdown**: compares each span type across populations on duration, cost, tokens, and errors. Finds the one broken span hiding in trace-level noise.

> **Note on statistical approach:** These tests currently treat trace populations as independent samples. If you're running paired evaluations — the exact same inputs on two agent versions — tighter methods like matched-pair bootstrapping or McNemar's test could isolate the delta further. This is on the roadmap.

Kalibra doesn't flag noise as signal. Each metric has a noise threshold, and changes below it are classified as unchanged. A 0.3% cost shift isn't a regression. A 40% one is — and the confidence interval confirms it.

The `--require` flag turns it into a CI gate — exit code 1 on violation. There's also a [GitHub Action](https://github.com/khan5v/kalibra-action) that posts the comparison as a PR comment and blocks merge on failure.

## What good agent evaluation looks like

The pattern isn't specific to Kalibra. If you're evaluating agents seriously, three things matter:

**Break down by task or scenario.** An aggregate that holds steady can hide equal and opposite movements underneath. Group your traces by task type, difficulty, or input category. Compare group-level outcomes between runs. This is the single highest-value check you can add — and most pipelines skip it.

**Test for significance.** A -3% cost change on 50 traces might be noise. A confidence interval tells you whether it's real. Without one, you're left guessing — flag every small change and drown in false alarms, or ignore them and risk shipping real regressions because the delta "looks small."

**Automate the gate.** If the check requires a human to look at a chart, it won't catch the regression that ships at 11pm on a Friday. Codify the thresholds. Let the pipeline enforce them.

The underlying statistics — z-tests, bootstrap resampling, grouped comparisons — have been well-established for decades. The gap was tooling that packages them for agent traces specifically. [Kalibra](https://github.com/khan5v/kalibra) is my attempt at that. It's open source, two dependencies, and runs anywhere Python does.

## What's still missing

This isn't a solved problem. Two things stand out:

- **Multiple testing correction.** Running 10 metrics at 95% confidence means roughly one false positive every other comparison. Bonferroni or Benjamini-Hochberg correction would tighten this.
- **Format fragmentation.** There is no standard trace format across platforms. Langfuse exports as JSON/JSONL with its own schema. LangSmith uses a tree-of-runs structure exportable as Parquet or JSON. Braintrust has a flat span format. HuggingFace datasets are something else entirely. While OpenTelemetry and the [OpenInference](https://github.com/Arize-ai/openinference) specification are working toward a standard for LLM observability, native exports still differ. Kalibra now supports OpenInference/Phoenix exports natively with auto-detection, mapping trace trees to flat metrics automatically. For other platforms, configurable field mapping bridges the gap.

I'd be curious how others deal with this — especially anyone maintaining integrations across more than one observability platform.

If we want to build reliable agents, we have to stop grading them like traditional APIs. It's time to start treating agent traces as what they are: statistical populations that demand proper comparison, not single numbers on a dashboard.

---

**[Kalibra](https://github.com/khan5v/kalibra)** — regression detection for AI agents · [GitHub](https://github.com/khan5v/kalibra) · [Docs](https://kalibra.cc) · [PyPI](https://pypi.org/project/kalibra/) · [Tutorial notebook](https://colab.research.google.com/github/khan5v/kalibra/blob/main/examples/phoenix_kalibra_tutorial.ipynb)
