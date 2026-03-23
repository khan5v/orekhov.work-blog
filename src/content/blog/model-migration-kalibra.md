---
title: "Your model migration passed. Here's what the aggregate didn't show."
date: 2026-03-23
description: "75% of AI agents break working behavior over time — including across model upgrades. Dashboards show the aggregate. Statistical comparison shows what moved underneath."
tags: ["AI agents", "evaluation", "model migration", "open source"]
ogImage: "/og-model-migration-kalibra.png"
draft: false
---

[SWE-CI](https://arxiv.org/abs/2603.03823), a benchmark published this month by Alibaba, tested whether AI coding agents maintain correct behavior over time. The result: [75% of them break previously working code](https://awesomeagents.ai/news/alibaba-swe-ci-ai-coding-agents-long-term-maintenance/) — and model upgrades are one of the triggers.

This isn't unique to coding agents. Every team running an LLM-powered agent hits the same problem quarterly: the provider deprecates your model, or a newer version promises better performance, or you're switching providers for cost. You change one string in your config, run the eval, and check the dashboard.

The dashboard looks fine. But underneath, the behavior may have shifted in ways the aggregate doesn't surface.

## The quarterly forced experiment

Model deprecations used to be annual. Now they're quarterly. Claude Opus 3 was retired earlier this year. GPT-4 Turbo was sunset last year. Each deprecation forces every team on that model to migrate — not on their schedule, on the provider's.

And it's not just deprecations. Teams switch models for cost optimization, latency improvements, or capability upgrades. Every switch is a forced experiment where the variables aren't controlled — the new model behaves differently on every task, and the differences are invisible in the aggregate.

## How migrations hide regressions

Different models have different strengths across task types. GPT-4o might be better at structured extraction while Claude excels at multi-step reasoning. A model that's faster might produce shorter responses — which looks like a cost improvement until you realize the shorter responses are *incomplete* responses.

The standard migration test:

- Success rate: 82% → 83%. Ship it.
- Median cost: $0.04 → $0.03. Even better.
- Median latency: 6.2s → 5.8s. Faster too.

What this misses: 8 out of 25 task types regressed. The high-volume, low-complexity tasks got slightly better — inflating the aggregate. The complex business flows that make up 15% of traffic broke silently.

The aggregate improved. Key task types degraded. And nothing in the top-line numbers flagged it.

This is the same failure mode I described in [Aggregate metrics are a blind spot in agent evaluation](/blog/kalibra-regression-detection/) — but model migrations make it worse because they change *everything at once*. A prompt edit affects one step. A model swap affects every LLM call in every trace.

## What catches it

Two things that help when the aggregate looks flat:

**Statistical significance.** Did the cost actually decrease, or is 50 traces not enough to tell? Kalibra computes bootstrap confidence intervals automatically — if the CI on the median delta includes zero, the "improvement" isn't real, it's sampling noise.

**Per-task breakdown.** Which tasks got better? Which got worse? If 8 task types flipped from pass to fail, that's a regression — even if the aggregate went up.

Here's what this looks like with [Kalibra](https://github.com/khan5v/kalibra), an open-source CLI that compares two trace populations statistically. It works on any JSONL traces that include outcomes (from an LLM-as-judge, deterministic eval, or the provider's finish reason). We ran 25 tasks through the same agent, baseline vs current, 50 traces each. The aggregate shows improvement everywhere:

```
▲ Token usage       963 → 337 tokens/trace (median)  -65.0%
                    95% CI [-75.4%, -14.9%]
▲ Duration          7.1s → 3.3s median  -52.9%
                    95% CI [-62.3%, -14.0%]
```

Tokens down 65% with a tight CI. Duration halved. Both statistically significant. If you stopped here, it looks like a clear win.

The per-task breakdown:

```
Trace breakdown
▼ Per trace         25 matched — ✗ 10 regressed

Quality gates
  [ OK ] token_delta_pct <= -10   actual: -65.00
  [FAIL] regressions <= 2         actual: 10.00

FAILED — quality gate violation (exit code 1)
```

10 task types broke. The token gate passed — tokens *did* go down. The regressions gate failed — too many tasks regressed. The tokens went down because complex responses were cut short, not because the agent got more efficient. Instead of generating a 40-line SQL query with explanatory comments, the agent output "You can use a SELECT with GROUP BY and HAVING" — drastically fewer tokens, but missing the actual answer.

The `regressions <= 2` gate surfaced what the aggregate metric missed.

## One file, two populations

The practical question: how do you compare traces from two models without managing separate files, separate runs, separate export pipelines?

Tag each trace with its model version. Put everything in one file. Split at compare time:

```yaml
# kalibra.yml — assuming all traces are in one centralized log
sources:
  baseline:
    path: ./traces.jsonl
    where:
      - model_version == gpt-4o
  current:
    path: ./traces.jsonl
    where:
      - model_version == claude-4.6

fields:
  task_id: task_name   # matches the 'task_name' key in your JSONL trace objects

require:
  - regressions <= 2
  - cost_delta_pct <= 30
  - success_rate_delta >= -5
```

```bash
kalibra compare   # reads config, exits 1 on gate failure
```

`where` filters traces by metadata — Prometheus-style matchers (`==`, `!=`, `=~`, `!~`). Both populations come from the same file, split by a tag you control. No separate export pipelines.

The three gates check three different failure modes:
- `regressions <= 2` — the per-task breakdown. Catches hidden regressions.
- `cost_delta_pct <= 30` — the cost didn't blow up.
- `success_rate_delta >= -5` — the aggregate didn't tank either.

If any gate fails, exit code 1. The deploy pauses until you've looked at which tasks were affected.

---

**[Kalibra](https://github.com/khan5v/kalibra)** — regression detection for AI agents · [GitHub](https://github.com/khan5v/kalibra) · [Docs](https://kalibra.cc) · [PyPI](https://pypi.org/project/kalibra/)
