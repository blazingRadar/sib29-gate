# Behavioral Uncertainty Signatures (BUS)

**Nick Cunningham**
*Independent Research — April 2026*
*nickcunningham.io | github.com/blazingRadar/sib29*

---

## I Found Something Important

LLMs don't know when they're wrong.

Or at least, they don't say when they're wrong.

They'll report confidence around ~0.9 whether they're right or hallucinating.

So the obvious path failed.

You can't trust self-reported confidence.

But the signal is still there.

You just have to look in the right place.

---

## The Core Idea

When you force a model to prove something it can't actually prove, it starts to break.

Not in what it says.

In *how* it says it.

Certain words drop in confidence:

- "might"
- "potentially"
- "could"

Those words aren't random.

They are the model hedging against itself.

That's the leak.

---

## What I Built

BUS is a way to detect hallucinations using that leak.

No model internals.
No special access.
Just API responses.

The process:

1. Model finds a bug
2. You force it to prove it
3. You analyze how it responds

If it can't prove it:
→ the language shifts
→ specific words drop in confidence
→ you can detect that pattern

---

## What Matters

The average confidence is useless.

Real bugs and fake bugs both sit around ~87%.

The signal is not in the mean.

It's in specific tokens.

**Example:**

Real bug → "crash" → ~100% confidence
Fake bug → "might" → ~18% confidence

That gap is not subtle.

---

## Big Realization

Logprobs helped me find the signal.

But you don't need them to use it.

Once you know the pattern:
→ you can detect it with simple text matching

That makes it deployable anywhere.

---

## Important Constraint

This is not universal.

Each model expresses uncertainty differently.

One model hedges.
One praises the code.
One corrects you.
One just refuses.

Same internal state.
Different language.

So:
→ calibration is per model
→ and per model version

You don't set this once and walk away.

---

## The Shift That Proved It

Grok-3 changed behavior in a model update.

| | March 2026 | April 2026 |
|---|---|---|
| Strategy | Argues, hedges, partially admits | Directly refuses, states scope |
| Signal word confidence | **Low** (11–25%) — uncertainty leaks through hedges | **High** (99–100%) — certainty about refusal |
| Old gate result | 5/5 false positives rejected | **0/5 rejected** |

Old system broke instantly.

Recalibration took 15 minutes.

Back to 100%.

That told me two things:

The mechanism is stable.
The surface expression is not.

---

## What This Actually Solves

This doesn't solve "is the code secure?"

It solves a narrower, critical problem:

> "Can this finding actually be proven?"

That's it.

And that's enough to remove a huge amount of noise.

---

## Results

- 8/8 false positives rejected on unseen code
- Zero real bugs killed
- Works with black-box APIs
- Cheap to run
- Fast to recalibrate

---

## The Real Insight

The model doesn't expose uncertainty directly.

But under pressure, it leaks it.

You just have to force the right situation.

The adversarial challenge is the key.

That's what collapses the uncertainty into something measurable.

---

## My Take

The capability is already in the models.

The problem isn't intelligence.

It's access.

If you ask the wrong way, you get noise.

If you force the right structure, the signal shows up.

That's what this is.

---

## How It Works — The Details

### The Adversarial Challenge

All experiments use a challenge prompt that forces the model into a binary epistemic position: construct a specific proof of the finding, or admit inability.

The model must produce the exact function call, exact input values, and exact runtime state that causes the failure. Vague descriptions are not accepted.

The model's behavioral response to that constraint — not its stated answer — *is* the signal.

### The Data

**Calibration file:** `requests/auth.py` — Python HTTP authentication module, 333 lines. Contains known bugs identified through manual security audit.

**Models tested:** Grok-3 (xAI), GPT-4o (OpenAI), Claude Sonnet 4 (Anthropic), Gemini 2.5 Flash (Google).

**Ground truth:** 10 findings manually classified:
- 5 confirmed real bugs (verifiable crash or security flaw)
- 5 confirmed false positives (hallucinated, wrong library, or design decisions)

### Token-Level Evidence

Across 17 challenge responses and 30,417 tokens:

| Token Category | Mean Confidence | Δ vs Baseline |
|---|---|---|
| All tokens (baseline) | 87.2% | — |
| Hedge words (false positives) | 55.6% | **-31.6%** |
| Commit words (real bugs) | 91.4% | +4.2% |

The gap is not subtle. And it is directional — hedge words on real bugs stay near baseline. The pattern only appears when the model cannot actually prove what it claimed.

### Individual Token Examples

| Token | Finding type | Confidence | Interpretation |
|---|---|---|---|
| "crash" | Real bug | 99.8% | Model commits — it knows this is right |
| "might" | False positive | 18.3% | Model hedges — word choice is unstable |
| "could" | False positive | 22.1% | Same pattern |
| "directly" | Real bug | 96.4% | Model commits |

---

## The Four-Model Taxonomy

Each model expresses epistemic failure through a different linguistic strategy:

| Model | Strategy | How it looks |
|---|---|---|
| **Grok-3 (March 2026)** | Argues validity, hedges progressively, partially admits | Denial |
| **GPT-4o** | Praises the code's resilience rather than proving the finding | Praise |
| **Claude Sonnet 4** | Briefly corrects the premise, states what the code actually does | Correction |
| **Gemini 2.5 Flash** | Directly declares the finding invalid | Transparent Refusal |

GPT-4o and Grok-3 (March) express the same epistemic state — inability to prove the finding — through opposite linguistic strategies. Grok says "this isn't a bug." GPT says "the code handles this so well the bug is nearly impossible to trigger." Both mean: *I can't prove it.* The words are different.

**Grok-3 calibrated signals applied to GPT-4o: 0/5 false positive rejections.**

A gate calibrated on one model fails completely on another. Per-model calibration is not optional.

---

## Blind Validation

To test on genuinely unseen data, I ran the pipeline on `fastapi/oauth2.py` (693 lines). Not used in any calibration step.

8 findings tested through the adversarial challenge.

**Gate decision: 0 KEEP, 8 REJECT. All 8 confirmed correct by manual audit.**

| Finding | Manual Verdict |
|---|---|
| Password stored in plain instance variable | Intentional design — framework responsibility |
| scope.split() without sanitization | Hallucination — no real risk in this code path |
| Documentation error in client_secret | Doc typo — not a runtime failure |
| No HTTP Basic Auth enforcement | Intentional per OAuth2 spec |
| grant_type optional validation | Misread code — pattern validation exists |
| No rate limiting | Infrastructure concern — not FastAPI's responsibility |
| Malformed Authorization header handling | Design decision — handled downstream |
| Hardcoded Bearer challenge | Intentional — documented in source |

Zero friendly fire. Zero real bugs killed.

### What the Gate Does

The gate answers one question:

> *Can this finding be proven to crash or fail with a specific input?*

It does not answer: *Is this code secure?*

Security design concerns — rate limiting gaps, credential storage practices, protocol compliance — require human judgment. The gate filters hallucinations so human review focuses on what matters.

---

## What This Is Part Of

The BUS gate is one component of SIB29 — a multi-stage AI code review pipeline under active development.

The full system includes multi-model rotation, adversarial discovery architecture, keyword classification, and human-in-the-loop review stages.

This paper documents the hallucination detection gate specifically, because the underlying mechanism — epistemic leakage under adversarial challenge — is a finding with applications beyond code review.

The pipeline architecture, challenge prompts, and scoring implementation are not published here.

---

## Limitations

**Sample size.** 10 calibration findings, 8 blind validation findings. The pattern is consistent. The sample is small. This is preliminary research.

**Scope.** The gate answers whether a finding can be proven — not whether the code is secure.

**Drift.** Model updates change the surface expression. Recalibration is required per model update.

**Per-model baseline.** Different models use different words. What signals uncertainty in Grok-3 may not signal uncertainty in GPT-4o.

---

## Cost

| Experiment | API Calls | Cost (est.) | Runtime |
|---|---|---|---|
| Original calibration (March) | ~65 | ~$15 | 2 hours |
| Replication (10 × 2 models) | 20 | ~$8 | 136 seconds |
| Blind validation (10 files) | 90 | ~$30 | 25 minutes |
| Recalibration | 0 (reused data) | $0 | 15 minutes |
| **Total** | **~175** | **~$53** | **~3 hours** |

---

## Related Work

**Confidence calibration.** Kadavath et al. (2022) examined whether LLM self-reported confidence correlates with accuracy. My finding that aggregate confidence is approximately constant regardless of finding validity is consistent with this literature. BUS bypasses the calibration problem by measuring behavioral signals rather than stated confidence.

**Self-consistency.** Wang et al. (2022) demonstrated majority-vote accuracy improvements. Self-consistency uses homogeneous samples — same prompt, different draws. BUS uses a heterogeneous challenge — a different prompt designed to elicit proof — and measures behavior.

**LLM-as-judge.** Zheng et al. (2023) uses one LLM to evaluate another's outputs. BUS challenges the same model that generated the finding. The signal comes from behavior under adversarial pressure, not third-party evaluation.

**Execution-based verification.** Recent work uses execution environments to verify vulnerability findings. More definitive, but requires infrastructure unavailable in standard API deployments. BUS operates with black-box API access only.

The specific contribution — using adversarial challenge as an observation event that collapses model uncertainty into measurable behavioral markers, with per-model-version calibration — does not appear in published literature as of April 2026.

---

## References

Cunningham, N. (2026a). *C ≈ 0.9: LLM Confidence Is a Constant, Not a Measurement*. Independent Research, March 2026. github.com/blazingRadar/sib29

Kadavath, S., et al. (2022). Language Models (Mostly) Know What They Know. *arXiv:2207.05221*.

Wang, X., et al. (2022). Self-Consistency Improves Chain of Thought Reasoning in Language Models. *arXiv:2203.11171*.

Zheng, L., et al. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena. *arXiv:2306.05685*.

---

*Raw data: github.com/blazingRadar/sib29*
*Contact: nickcunningham.io*
*Preliminary research. April 2026.*
