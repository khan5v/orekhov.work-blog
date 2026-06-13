---
title: "Your quality gate has two answers. The question underneath has three."
date: 2026-06-13
description: "A CI gate has to answer merge-or-block, but the evidence underneath has three states: worse, fine, and not-enough-data-to-tell. Fusing the last two is how gates quietly lie — and it's the design shift I'm making in Kalibra."
tags: ["AI agents", "evaluation", "statistics"]
ogImage: "/og-quality-gate-three-answers.png"
---

If you ship an AI agent, sooner or later you change the thing underneath it — a new model version, a different provider, a reworked prompt. The careful way to do that is to run an eval suite before it merges: a few hundred tasks, measuring not just whether each one succeeded but what it cost, how long it took, how many steps it burned. You compare the new run against the old across all of those, and a gate in CI blocks the pull request if something slipped. You make the change, the evals run, the gate comes back green.

A green checkmark there means one of two things, and you can't tell which.

Either the new agent is genuinely no worse than the old one: the change was measured, and it's fine. Or you just didn't run enough evals to see the regression that's actually there. Those are very distinct states of the world. One says *ship it*. The other says *you're flying blind*. Your gate prints the same green check for both, because git asked it a yes/no question and it answered the only way it can: pass, fail.

That missing third answer — *not enough data to tell yet* — is the one I keep wishing every CI gate had. This post is about why it matters, and the verdict layer I'm building into [Kalibra](https://github.com/khan5v/kalibra) to surface it.

## The two answers a gate quietly fuses

Start with what every comparison is actually doing. You ran the old agent and the new one, and you're staring at a difference: success rate down 2 points, cost up 4%. The hard part isn't the subtraction. It's that **a difference between two runs is always two things at once**: a real difference in the systems, plus a difference that would have appeared anyway from run-to-run randomness. Agents are nondeterministic; rerun the same agent on the same tasks and you'll get a different number; [measured run-to-run variation sits above 1.5 points even at temperature zero](https://arxiv.org/abs/2602.07150). Telling signal from that noise is the whole job.

A significance test does part of that job, but it has a famous blind spot. It can say "this is bigger than noise" (a regression) or "I can't distinguish this from noise." And that second answer is *not* the same as "there is no real difference" — *absence of evidence isn't evidence of absence*. So when a gate collapses everything to pass/fail, the green "pass" quietly merges two states that mean completely different things:

- **Equivalent** — we measured the change, and it's too small to care about.
- **Inconclusive** — we couldn't tell, because there wasn't enough data.

Same checkmark — yet one means *ship it* and the other means *you can't tell yet*. Which one you landed on was decided not by your data but by *which test the tool happened to run*, off-screen. The fix is to stop fusing them: report three answers, not two.

## From two answers to three

The way out is to stop reporting a single number and report an *interval* (the range the true change very probably lives in), then compare it against a band you declare up front: the **region of practical equivalence**, or ROPE. The ROPE is just you saying out loud "changes smaller than this, I don't care about" — say, ±2 points of success rate. (Not an arbitrary number: since [identical reruns already drift more than a point on their own](https://arxiv.org/abs/2602.07150), a band narrower than that wobble would just be flagging noise.)

Where the whole interval falls relative to that band gives the verdict — three answers to *did it get worse?*, plus *improved* for the mirror case:

| Where the interval sits | Verdict |
|---|---|
| Entirely **outside** the ROPE, on the bad side | **regressed** |
| Entirely **outside**, on the good side | **improved** |
| Entirely **inside** the ROPE | **equivalent** — proven small |
| **Straddling** a ROPE edge | **inconclusive** — run more evals |

<figure style="margin:2rem 0">
<svg viewBox="0 0 760 322" role="img" aria-label="A number line of the change in a metric — current minus baseline — with no change at zero, and a shaded grey band around zero called the ROPE that marks changes too small to matter. Four example 95% intervals show the four verdicts: an interval entirely left of the band is regressed; one that overlaps a band edge is inconclusive; one entirely inside the band is equivalent; one entirely right of the band is improved. Each comparison produces a single such interval, and its width already includes the run-to-run noise of both trace sets." style="width:100%;height:auto;max-width:720px;display:block;margin:0 auto" xmlns="http://www.w3.org/2000/svg" fill="none">
<rect x="300" y="48" width="80" height="194" fill="currentColor" opacity="0.08"/>
<text x="340" y="36" text-anchor="middle" font-size="13" fill="currentColor" opacity="0.75">ROPE — "too small to matter"</text>
<line x1="340" y1="48" x2="340" y2="250" stroke="currentColor" stroke-width="1" stroke-dasharray="4 4" opacity="0.45"/>
<text x="78" y="270" font-size="12" fill="currentColor" opacity="0.7">← worse</text>
<text x="340" y="270" text-anchor="middle" font-size="12" fill="currentColor" opacity="0.7">0</text>
<text x="600" y="270" text-anchor="end" font-size="12" fill="currentColor" opacity="0.7">better →</text>
<text x="340" y="300" text-anchor="middle" font-size="12.5" fill="currentColor" opacity="0.85">change in the metric  (current − baseline)</text>
<g stroke="#e5484d" fill="#e5484d"><line x1="110" y1="78" x2="250" y2="78" stroke-width="4" stroke-linecap="round"/><line x1="110" y1="69" x2="110" y2="87" stroke-width="2"/><line x1="250" y1="69" x2="250" y2="87" stroke-width="2"/><circle cx="180" cy="78" r="5" stroke="none"/></g>
<text x="630" y="83" font-size="14" font-weight="600" fill="currentColor">regressed</text>
<g stroke="#e0a020" fill="#e0a020"><line x1="208" y1="122" x2="352" y2="122" stroke-width="4" stroke-linecap="round"/><line x1="208" y1="113" x2="208" y2="131" stroke-width="2"/><line x1="352" y1="113" x2="352" y2="131" stroke-width="2"/><circle cx="280" cy="122" r="5" stroke="none"/></g>
<text x="630" y="127" font-size="14" font-weight="600" fill="currentColor">inconclusive</text>
<g stroke="#4f9bd9" fill="#4f9bd9"><line x1="305" y1="166" x2="375" y2="166" stroke-width="4" stroke-linecap="round"/><line x1="305" y1="157" x2="305" y2="175" stroke-width="2"/><line x1="375" y1="157" x2="375" y2="175" stroke-width="2"/><circle cx="340" cy="166" r="5" stroke="none"/></g>
<text x="630" y="171" font-size="14" font-weight="600" fill="currentColor">equivalent</text>
<g stroke="#30a46c" fill="#30a46c"><line x1="430" y1="210" x2="580" y2="210" stroke-width="4" stroke-linecap="round"/><line x1="430" y1="201" x2="430" y2="219" stroke-width="2"/><line x1="580" y1="201" x2="580" y2="219" stroke-width="2"/><circle cx="505" cy="210" r="5" stroke="none"/></g>
<text x="630" y="215" font-size="14" font-weight="600" fill="currentColor">improved</text>
</svg>
<figcaption style="text-align:center;font-size:0.85em;opacity:0.7;margin-top:0.6rem">Each bar is one comparison's 95&#37; interval on the change, and its width already folds in the run-to-run noise of both runs. The verdict turns on one thing: where the <em>whole</em> interval sits relative to the ROPE — and an interval that overlaps the edge is genuinely inconclusive, not a pass.</figcaption>
</figure>

In agent terms: you swap the model and rerun 200 eval tasks against a ±2-point ROPE. If success rate falls 90% → 80%, the interval lands around [−17, −3] — entirely past the ROPE on the bad side: the red bar, a regression you can act on. If it falls 90% → 86%, the interval runs roughly [−10, +2] — it straddles the band: not entirely past it (so not a confirmed regression), not entirely inside it (so not provably fine). That's the amber bar: *inconclusive*. Same 200 runs, same direction — the smaller drop is the one you can't yet call, and the answer there is "run more evals," not "pass." (What decides it is the ROPE edge, not zero — "the interval crosses zero" is the reflex this replaces.)

None of this is new to statistics — equivalence testing has existed for decades, frequentist and Bayesian alike. What's new is putting a three-answer verdict where a CI gate can use it, instead of letting pass/fail flatten it back to two.

## Verdict inside, gate outside

Git still needs a binary answer in the end: the PR merges or it doesn't. So the design is two layers, because two different questions are being asked.

**Inside, the verdict** — the three-answer read above, per metric — is what a human looks at. And it carries more than a label: it carries a *number*, how likely the change is a real regression. "Inconclusive" never means we learned nothing — it means the evidence isn't yet decisive enough to commit to a category. The measurement can be sharp even when the label can't be.

**Outside, the gate** — pass or fail — is produced by a predicate *you* wrote, applied to that evidence and printed right next to it. Two shapes cover most needs:

1. **A bound on the interval** — "block if the worst-case cost increase clears 10%." Harder to game than a point estimate: a noisy result has a wider interval, so it trips the threshold sooner, not later.
2. **A probability threshold** — "block if the chance of a meaningful regression is over 95%." This is the number stakeholders actually ask for — *how likely is it worse?* — which a p-value pointedly does not answer. The amber 90% → 86% case above is about 73% likely to be a real drop: not nothing, not decisive. A probability like that, on the same 0–1 scale for every metric regardless of units, is something you can act on deliberately — block it, or gather more data — instead of having it vanish into a green check.

The principle holding the two layers apart is the one I care about most:

> Statistics makes the evidence. You make the call. A tool's job is to produce sound evidence and apply *your* decision rule transparently — never to make the decision implicitly through its choice of test. A gate that decides for you and won't show its work isn't a quality gate; it's a magic 8-ball with a config file.

## Where Kalibra is, and where it's going

Straight talk about the current state, because the gap is the point of writing this.

Kalibra already ships a tri-state verdict — the `inconclusive` state exists today, and the engine already refuses to let an inconclusive result trip a gate. The *philosophy* is in the tool. Under the hood, the engine currently relies on standard frequentist mechanics ([documented in the current methods spec](https://kalibra.cc/methods/)): a percentile bootstrap on the median delta, a two-proportion z-test for rates, and a noise band on the point estimate. So today's "no change" still bundles "equivalent" together with "can't tell," and the probability users actually want — how likely the new agent is worse — isn't computed at all, because there's no posterior — a full probability distribution over the change — to read it from.

Closing that gap is the milestone I'm building now: a Bayesian verdict layer in place of that mechanism — posteriors that separate *equivalent* from *inconclusive*, a real probability of regression, and a gate that runs on that probability. There's a concrete reason it's Bayesian rather than a tidied-up frequentist core. At the sample sizes agent evals actually run — a few hundred tasks or fewer — the textbook confidence interval [stops covering at its stated rate](https://arxiv.org/abs/2503.01747) (Bowyer, Aitchison & Ivanova, ICML 2025); and in the small-effect regime where most CI runs live, posterior intervals get the *direction* of a change wrong far less often than classical tests ([Gelman & Tuerlinckx, 2000](https://ppw.kuleuven.be/okp/_pdf/Gelman2000TSERF.pdf)). The three-answer verdict itself is framework-agnostic; the calibration at these sample sizes is why I'm reaching for posteriors. (I'm not committing to the exact output format here — what matters is the shape of the answer, not how it prints.)

Statistical regression gating for agents hasn't become standard practice yet. The field is young and moving fast, and the tooling is only starting to catch up. Where a check exists at all, it's usually a dashboard read by eye, or a threshold on a single noisy number. The bar I'm aiming at isn't "a fancier number." It's a gate that can say *I don't know yet* — and that shows you exactly how it reached whatever it decided.

---

**[Kalibra](https://github.com/khan5v/kalibra)** — regression detection for AI agents · [GitHub](https://github.com/khan5v/kalibra) · [Docs](https://kalibra.cc) · [PyPI](https://pypi.org/project/kalibra/)
