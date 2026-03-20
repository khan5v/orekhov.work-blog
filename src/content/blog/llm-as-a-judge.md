---
title: "LLM-as-a-judge: how to evaluate AI without fooling yourself"
date: 2026-03-04
description: "LLM-as-a-judge from first principles — when to use it, how to design rubrics, the three biases that skew scores, and when to use something simpler."
tags: ["AI", "system design", "evaluation"]
draft: false
---

Before we talk about using an LLM as a judge, let's step back. Yes, we're one sentence in and already stepping back — but this context matters. How do you evaluate a model's output at all?

There are three broad approaches, and they sit on a spectrum of cost vs. nuance.

**Deterministic metrics** are by far the simplest — almost a unit test with inference running. You feed the model an input, compare the output against a known correct answer using exact match, BLEU, ROUGE, or pass@k, and get a binary or numeric result. Fast, cheap, reproducible. Use them when you can. They work well for code correctness (does it pass the test suite?), multiple-choice benchmarks, and translation tasks where you have reference answers.

On the other end of the spectrum sits **human evaluation** — the gold standard. A trained annotator reads the output and scores it against a rubric. It captures nuance that no automated metric can. Use it for establishing ground truth, calibrating automated systems, and high-stakes release decisions. Human evaluation can also take the form of A/B experiments — show real users two model variants and measure which they prefer. You might have seen this pop up occasionally when using ChatGPT — "which response do you prefer, A or B?" That's human evaluation at scale, harvested from real users.

But there are caveats: users tend to prefer novelty and verbosity over actual quality, inter-annotator agreement is often lower than you'd expect, and it costs roughly $1 per annotation. It doesn't scale when you need to evaluate tens of thousands of outputs across model versions.

**LLM-as-a-judge** sits in the middle: you send the model's output to a frontier LLM (GPT-4, Claude) with a scoring rubric and ask it to grade. It's cheaper than humans, more nuanced than string matching, and scales to thousands of judgments per hour.

A typical judge call is ~2,000 input tokens and ~500 output tokens — at current API pricing that's roughly $0.005-0.02 per judgment depending on the model. An eval pipeline running 10,000 judgments per day costs $50-200/day in judge calls alone. Add position bias mitigation (2x) and multi-judge consensus (3x) and you're at $300-1,200/day. Still an order of magnitude cheaper than $1/annotation humans, but it adds up — know your budget before designing the pipeline.

The trap is reaching for LLM-as-a-judge by default. If your task has a verifiable correct answer — math problems, code that either passes tests or doesn't, factual questions with known answers — use deterministic evaluation. It's faster, cheaper, and doesn't introduce a second model's biases into your results. LLM-as-a-judge is for the tasks where correctness is subjective or multi-dimensional: summarization quality, instruction following, helpfulness, tone, creative writing. The cases where two humans might reasonably disagree.

## The three judging modes

Not all judging works the same way. There are three modes, and picking the wrong one is a common early mistake.

**Pointwise scoring** — the judge rates a single output on a scale (1-5). This is what most people start with because it's intuitive. The problem: scores are poorly calibrated. One judge's 3 is another judge's 4. Even the same judge drifts over a long batch. It works for rough sorting (clearly good vs. clearly bad) but the middle of the scale is unreliable.

**Pairwise comparison** — the judge sees two outputs side by side and picks the better one (or calls a tie). This is much more reliable than pointwise scoring because relative judgments ("A is better than B") are easier than absolute ones ("A is a 3.7"). Default to this when comparing two models or two prompt variants.

**Reference-based grading** — the judge compares the output against a gold-standard reference answer. More constrained than pointwise, less noisy. Works well when you have high-quality reference answers but the match isn't exact enough for string metrics.

For most teams evaluating model quality across versions, pairwise comparison gives you the cleanest signal.

## Rubric design is everything

The single biggest factor in judge quality isn't the model — it's the prompt. A vague rubric produces vague scores. "Rate this response 1-5" is essentially asking the judge to make up its own criteria, and it will make up different ones for different outputs.

A good rubric defines what each score level means concretely — think of it like the hiring rubrics at Big Tech companies, where each interview score has a specific behavioral definition so that different interviewers calibrate the same way:

```
5 — Fully addresses the question. Factually accurate. Well-structured.
     No significant issues.
4 — Addresses the question with minor gaps. Slight inaccuracies or
     could be better organized. Mostly complete.
3 — Partially addresses the question. Notable gaps or some inaccuracies.
     Gets the basics right.
2 — Misses major aspects. Contains significant errors or is hard to
     follow. Some useful content.
1 — Does not address the question, is largely incorrect, or incoherent.
```

The difference between "3 = average" and "3 = partially addresses the question with notable gaps" is the difference between random noise and a useful signal.

Two more things that materially improve consistency:

**Put reasoning before score.** Ask the judge to explain its analysis first, then give the score. This matters because of how language models generate text — each token is conditioned on everything that came before it. If the model outputs "SCORE: 3" first, the reasoning that follows is conditioned on already having committed to 3, and the model will rationalize that choice. If reasoning comes first, the score is conditioned on the analysis. It becomes a deliberate conclusion, not a gut reaction followed by justification.

**Include calibration examples.** Add 2-3 example (output, reasoning, score) tuples directly in the system prompt. This anchors the judge's scale. Without them, the judge has to infer what "good" means from the rubric alone — and that inference shifts across batches. With examples, you're saying "this specific output is a 2, and here's why." Yes, it costs extra tokens. It's worth it.

## The three biases that will burn you

LLM judges have systematic biases. Not might-have — they're well-documented in the research (start with [Large Language Models are not Fair Evaluators](https://arxiv.org/abs/2305.17926) if you want the details) and they're large enough to flip your conclusions if you don't account for them.

### Position bias

In pairwise comparisons ("which response is better, A or B?"), judges tend to prefer whichever response appears first. The effect is large — 10-15% score shift depending on the model and task.

The fix is straightforward but non-negotiable: run every pairwise comparison twice with the order swapped. If the judge picks the same winner both times, you have a confident result. If it picks a different winner when the order changes, that judgment is unreliable — flag it as a tie or escalate to a human.

This doubles your cost for pairwise comparisons. There's no shortcut here. If you're not doing this, your pairwise results have up to a 15% error rate that correlates with presentation order, not quality.

### Verbosity bias

Judges prefer longer, more detailed responses even when the shorter one is more accurate or better suited to the question. A 500-word response that's 80% padding will outscore a 100-word response that's 100% substance.

This one is harder to fix mechanically. A few approaches that help:

**State it in the rubric.** Explicitly tell the judge that conciseness is valued — "a shorter response that fully addresses the question should score higher than a longer one with unnecessary detail." This doesn't eliminate the bias, but it reduces it.

**Length-normalized scoring.** Compute a score-per-token and use that to adjust. This can overcorrect, so treat it as a secondary signal.

**Detect it statistically.** When you're reviewing judge results and see a pattern of longer outputs winning, check whether length is actually correlated with quality or if the judge is just impressed by volume. You can measure this with the **Pearson correlation** (*r*) between judge score and token count.

The idea: for each response, compute how far its token count and its score each deviate from their respective means, then multiply those deviations together. When both are above mean (long *and* high score) or both below mean (short *and* low score), the product is positive — they move together. When they point opposite directions, the product is negative. Sum the products, normalize by the spread of each variable, and you get *r*: a number between −1 (perfect inverse) and 1 (perfect positive). Zero means no relationship.

| Response | Tokens | Score | Token deviation | Score deviation | Product |
|----------|:------:|:-----:|:--------------:|:--------------:|:-------:|
| A        |   50   |   3   |     −150       |      −1        | **+150**|
| B        |  100   |   3   |     −100       |      −1        | **+100**|
| C        |  150   |   4   |      −50       |       0        | **0**   |
| D        |  200   |   4   |        0       |       0        | **0**   |
| E        |  300   |   5   |     +100       |      +1        | **+100**|
| F        |  400   |   5   |     +200       |      +1        | **+200**|
| **Mean** | **200**| **4** |                |                | **Σ = 550** |

*r* = 550 / √(85,000 × 4) = 550 / 583 ≈ **0.94** — almost perfect. Every product is zero or positive; length and score move in lockstep.

That's a red flag, but not proof — maybe longer responses genuinely *are* better. To confirm it's bias, check whether the correlation holds among responses that humans rated equally. If the judge still scores longer ones higher within the same quality tier, that's verbosity bias. If you're not looking for it, you won't see it.

### Self-preference bias

When GPT-4 judges GPT-4's outputs against Claude's outputs (or vice versa), it tends to prefer its own style. The model isn't "being biased" in a conscious sense — it's that the outputs it generated share its own statistical patterns, and it rates things that match its distribution as more natural and higher quality.

The cleanest mitigation: don't use the same model as both generator and judge. If you're evaluating GPT-4 outputs, use Claude as the judge (and vice versa). If you must use the same model family, use a different model version or size, and measure the self-preference effect on a calibration set where humans have already rated both.

## When LLM-as-a-judge breaks down

There are categories of tasks where LLM judges are unreliable enough that you shouldn't use them as the primary metric:

**Math and logical reasoning.** A judge model can confidently rate an incorrect proof as "clear and well-structured." It's evaluating surface-level coherence, not mathematical correctness. If the answer is verifiable, verify it. Run the computation. Check the proof steps programmatically.

**Factual accuracy.** The judge might not know that a specific claim is false, especially for niche or recent topics. For factual evaluation, use retrieval-augmented checking against a known source, not vibes from another model.

**Code correctness.** Does the code work? Run it. Does it pass the test suite? That's your metric. An LLM judge might rate clean-looking but broken code higher than ugly but correct code.

**Judging above your weight class.** This is the failure mode teams don't anticipate: using a weaker model to judge a stronger one. If your judge is GPT-4o-mini and the model being evaluated is a frontier model, the judge literally cannot assess quality beyond its own capability ceiling. It'll give high scores to outputs it can't distinguish from perfect — and you'll think your model is better than it is.

As models improve, this becomes the binding constraint on your eval pipeline. Always make sure the judge is at least as capable as the model being judged on the dimensions you're measuring.

The common pattern: if there's a deterministic way to check correctness, use it. LLM-as-a-judge is for the dimensions that don't have a deterministic answer — helpfulness, clarity, tone, completeness, appropriateness.

## Closing the loop: calibration with humans

An LLM judge is only as trustworthy as its agreement with human judgment. You need to measure this, and keep measuring it.

The standard metric is **Cohen's kappa** (κ). It measures agreement between two raters, corrected for chance. The "corrected for chance" part is what makes it useful — and what most people skip over. Let's unpack it.

Say you're labeling 100 outputs as "good" or "bad." Here's what the confusion matrix looks like:

|               | Human: good | Human: bad | Total |
|---------------|:-----------:|:----------:|:-----:|
| **Judge: good** |     82      |      8     |  90   |
| **Judge: bad**  |      5      |      5     |  10   |
| **Total**       |     87      |     13     | 100   |

**p_o (observed agreement)** is the fraction of outputs where both raters actually gave the same label. Here, they agreed on 82 + 5 = 87 out of 100. So p_o = 0.87. That looks great — 87% agreement.

But is it? **p_e (expected agreement by chance)** asks: how often would they agree if they were labeling independently at random, just based on their individual rates? The human says "good" 87% of the time. The judge says "good" 90% of the time. If they were flipping biased coins independently:

| Chance event         | Probability             |
|----------------------|-------------------------|
| Both say "good"      | 0.90 × 0.87 = **0.783** |
| Both say "bad"       | 0.10 × 0.13 = **0.013** |
| **p_e (total)**      | **0.796**               |

So by pure chance, they'd agree 79.6% of the time. That 87% observed agreement is less impressive now — most of it is just because both raters say "good" most of the time.

Kappa strips out this baseline:

| Step               | Value                         |
|--------------------|-------------------------------|
| p_o − p_e          | 0.87 − 0.796 = **0.074**     |
| 1 − p_e            | 1 − 0.796 = **0.204**        |
| **κ = ratio**      | 0.074 / 0.204 ≈ **0.36**     |

κ = 0.36 — that's only "fair agreement." The raw 87% was misleading. The judge and human aren't agreeing because they're evaluating the same things well — they're agreeing mostly because they both default to "good."

κ = 1 means perfect agreement. κ = 0 means no better than chance. κ < 0 means the raters agree *less* than chance — they're actively contradicting each other, which usually signals a broken rubric or a misconfigured judge prompt. For a production judge, aim for κ > 0.8 with your human annotators — that's the "almost perfect agreement" threshold on Cohen's scale, and the baseline recent research uses for human-level judge performance.

The workflow: take a representative sample of outputs, have humans score them with the same rubric, compare against the judge's scores. When they disagree, investigate: is the human wrong, is the judge wrong, or is the rubric ambiguous? Each disagreement is a chance to tighten the rubric.

Do this continuously, not once. Judge behavior drifts — model provider updates, quantization changes, even inference infrastructure changes can shift scores subtly. A calibration set that you re-run weekly catches this before it corrupts a month of evaluation data.

## The bottom line

If you're starting an eval pipeline tomorrow: get your rubric right before you worry about anything else. A mediocre model with a precise rubric will give you more reliable scores than a frontier model with a vague one. Then run position-swapped pairwise comparisons, compute κ against a small set of human annotations, and watch the numbers — not once, but every week.

LLM-as-a-judge isn't a replacement for human evaluation. It's a way to scale it. Treat it like any measurement instrument: calibrate it, know its error modes, and never trust a number you haven't sanity-checked.
