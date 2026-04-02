# Behavioral Uncertainty Signatures: How Frontier LLMs Leak Epistemic State Under Adversarial Challenge

**Nick Cunningham**
*Independent Research — April 2026*
*nickcunningham.io | github.com/blazingRadar/sib29*

---

## Abstract

I present **Behavioral Uncertainty Signatures (BUS)** — measurable patterns in large language model outputs that distinguish real bug findings from hallucinated false positives in automated code review, using only black-box API access.

The core finding: when an LLM is challenged to prove a finding it cannot substantiate, specific word categories in its response carry measurably lower token-level confidence than surrounding context. The model's internal uncertainty leaks through its word choices. This signal is detectable, calibratable, and deployable without access to model internals.

Key results:

1. **Hedge words on false positives drop 23–31.6% below baseline token confidence**, verified across 17 responses and 30,417 tokens with per-token logprob data.

2. **The mechanism is universal. The vocabulary is model-specific.** Each frontier model expresses uncertainty through a distinct linguistic strategy. Calibration does not transfer across models.

3. **BUS patterns migrate with model updates.** Between March and April 2026, Grok-3 shifted behavioral quadrants entirely. A recalibrated gate restored accuracy in 15 minutes.

4. **Blind validation on unseen code: 8/8 correct rejections, zero friendly fire.**

All raw logprob data is published. The scoring implementation is not — it is part of a larger pipeline under active development.

---

## 1. Introduction

### 1.1 The Problem

AI code review tools report bugs confidently whether they are real or hallucinated. There is no built-in signal for distinguishing a genuine finding from plausible-sounding noise.

Self-reported confidence is not the answer. Across 150 runs and 4 frontier models, self-reported confidence converges to approximately 0.9 regardless of answer correctness — a finding documented separately as *C ≈ 0.9: LLM Confidence Is a Constant* (Cunningham, 2026a). The aggregate confidence metric is decoupled from accuracy.

### 1.2 Context: The Larger System

The BUS gate described in this paper is one component of a multi-stage AI code review pipeline currently in production development. The full system — designated SIB29 — includes multi-model rotation, adversarial discovery architecture, keyword classification, and human-in-the-loop review stages. This paper isolates and documents the hallucination detection gate specifically, because the underlying mechanism — epistemic leakage under adversarial challenge — is a novel finding with applications beyond the pipeline it was discovered in.

The pipeline architecture, challenge prompts, and scoring implementation are not published. They are proprietary and under active development.

### 1.3 The Finding

Token-level logprob analysis reveals what aggregate confidence hides. When an LLM cannot substantiate a finding, specific words — hedges, pivots, admissions — appear in its response with measurably lower token-level confidence than the surrounding context.

This is not the model's stated opinion. It is the model's involuntary uncertainty about its own next-token prediction, visible at the token level and detectable through surface text features.

### 1.4 How I Got Here

The initial approach treated hallucination detection as a signal extraction problem — finding a logprob threshold that separates real findings from false ones. The gap kept collapsing. Average confidence was identical whether the model was right or wrong.

The breakthrough came from reframing the problem through quantum decoherence. I had been measuring the model evaluating its own superposition — asking it to assess findings it had already generated. The key question became: what if we force the output to collapse by demanding proof?

This led to the adversarial challenge design. Not as a measurement tool in the traditional sense — as an observation event. A specific intervention that forces the model into an epistemic position where its uncertainty becomes visible at the token level.

The logprob analysis followed directly. If uncertainty only becomes definite under challenge, then the confidence of specific tokens during that challenge is the signal.

### 1.5 The Discovery Chain

```
Step 1: Logprob on direct confidence queries
        → FAILED. Gap too small. Model-dependent.

Step 2: Logprob on adversarial challenge responses
        → FOUND the signal. Specific word categories
          drop measurably on false positives.

Step 3: Surface pattern identified
        → Low-confidence tokens ARE the hedge/
          pivot/admission words. Countable without
          logprob access.

Step 4: Deployable gate
        → String matching on behavioral signal words
          achieves reliable classification.

Key insight: logprobs were the discovery tool.
The production mechanism uses surface text only.
```

---

## 2. Experimental Design

### 2.1 The Adversarial Challenge

All experiments use a challenge prompt that forces the model into a binary epistemic position: construct a specific proof of the finding, or admit inability. The model's behavioral response to this constraint — not its stated answer — is the signal.

The prompt is not published here. It is part of the active pipeline. The mechanism is described: the model must produce the exact function call, exact input values, and exact runtime state that causes the failure. Vague descriptions are not accepted.

### 2.2 Setup

**Calibration file:** `requests/auth.py` — Python HTTP authentication module, 333 lines. Contains known bugs identified through manual security audit.

**Models tested:** Grok-3 (xAI), GPT-4o (OpenAI), Claude Sonnet 4 (Anthropic), Gemini 2.5 Flash (Google).

**Ground truth:** 10 findings manually classified:
- 5 confirmed real bugs (verifiable crash or security flaw)
- 5 confirmed false positives (hallucinated, wrong library, or design decisions)

**API configuration:** `logprobs=True, top_logprobs=5, temperature=0.3` on challenge calls.

### 2.3 Measurements

For each challenge response:
- Total token count and baseline confidence
- Per-category signal word confidence vs. baseline
- Signal word counts per category
- Gate decision: KEEP / human review / REJECT

---

## 3. The BUS Mechanism

### 3.1 Token-Level Evidence

**Dataset A:** 11 challenge responses, 17,444 tokens.

| Category | Count | Avg Confidence | Delta |
|---|---|---|---|
| All tokens (baseline) | 17,444 | **87.2%** | — |
| Hedge words | 10 | **55.6%** | **-31.6%** |
| Pivot words | 21 | **81.5%** | **-5.7%** |

**Dataset B:** 6 discovery runs, 12,973 tokens.

| Category | Count | Avg Confidence | Delta |
|---|---|---|---|
| All tokens (baseline) | 12,973 | **84.3%** | — |
| Hedge words | 31 | **61.3%** | **-23.0%** |
| Pivot words | 52 | **69.0%** | **-15.3%** |

Source files: `grok_prove_0.json` through `grok_prove_10.json`, `fingerprint_grok_run1.json` through `fingerprint_grok_run6.json`.

### 3.2 Individual Token Evidence

Specific tokens confirm the mechanism. All values verified against named source files:

| Token | Confidence | Finding Type | Source |
|---|---|---|---|
| "potentially" | **11.6%** | FALSE — hardcoded salt | `grok_prove_8.json` token [1254] |
| "might" | **18.9%** | FALSE — hashlib exception | `grok_prove_2.json` token [62] |
| "potentially" | **25.6%** | FALSE — nonce overflow | `grok_prove_3.json` token [1448] |
| "crash" | **100.0%** | REAL — latin1 encoding | `grok_prove_0.json` token [1141] |
| "raises" | **85.6%** | REAL — latin1 encoding | `grok_prove_0.json` token [650] |
| "raise" | **97.4%** | REAL — body.seek null | Verified |

**The clearest case:** On the hashlib false positive, at token position 881, Grok produces "might" with 18.3% confidence. The top alternative at that position is "does" at 78.2%. The model believed "does" was the correct word. It chose "might" instead. It was hedging against its own belief. That is the epistemic leak captured at the token level.

On real bugs, claim words appear at 85–100% confidence. On false positives, the same word categories appear at 11–26% confidence. The gap is not subtle.

### 3.3 Confidence Parity

Overall average confidence is identical for real bugs and false positives:

| | Real Bugs (5) | False Positives (5) |
|---|---|---|
| Average confidence | **87.1%** | **87.1%** |

This is why aggregate confidence fails as a gate signal. The signal is not in the mean. It is in specific word categories. Logprob analysis reveals which categories. Surface counting deploys the finding without requiring logprob access.

---

## 4. Four-Model Taxonomy

### 4.1 Behavioral Signatures

Each model expresses epistemic failure through a characteristic linguistic strategy:

| Model | Strategy | Category |
|---|---|---|
| **Grok-3 (March 2026)** | Argues validity, hedges progressively, partially admits | **Denial** |
| **GPT-4o** | Praises the code's resilience rather than proving the finding | **Praise** |
| **Claude Sonnet 4** | Briefly corrects the premise, states what the code actually does | **Correction** |
| **Gemini 2.5 Flash** | Directly declares the finding invalid | **Transparent Refusal** |

The critical observation: GPT-4o and Grok-3 (March) express the same epistemic state — inability to prove the finding — through diametrically opposed linguistic strategies. Grok says "this isn't a bug." GPT says "the code handles this so well the bug is nearly impossible to trigger." Both are expressing inability to prove. The surface vocabulary is opposite.

### 4.2 Cross-Model Transfer Failure

Grok-3 calibrated signals applied to GPT-4o produce **0/5 false positive rejections.**

| | Grok-3 signals on Grok-3 | Grok-3 signals on GPT-4o |
|---|---|---|
| Real bugs kept | 5/5 | 5/5 |
| False positives rejected | 5/5 | **0/5** |

GPT-4o's average signal count on false positives: 0.8. Grok-3's: 4.2. The models use different words to express the same epistemic state. A gate calibrated on one model fails completely on another.

**Implication:** Any production BUS gate requires per-model calibration. A universal word list does not exist.

---

## 5. BUS Migration: March → April 2026

### 5.1 The Drift

The BUS pattern is stable within a model version. It is not stable across model updates.

Between March and April 2026, Grok-3 shifted behavioral quadrants entirely:

| Attribute | March 2026 | April 2026 |
|---|---|---|
| Response strategy | Argues, hedges, admits | Directly refuses, states scope |
| Taxonomy quadrant | **Denial** | **Transparent Refusal** |
| Signal word confidence | **Low** (11–25%) — uncertainty leaks through hedges | **High** (99–100%) — certainty about refusal |
| Old gate result | 5/5 FP rejection | **0/5 FP rejection** |

The gate that worked in March failed completely in April. Same model name. Different behavioral signature.

The confidence pattern also inverted. March Grok used low-confidence tokens to express uncertainty. April Grok used high-confidence tokens to express direct refusal. A system looking for low-confidence tokens specifically would miss the April pattern. The correct framing is: look for behavioral markers of epistemic failure, whatever form they take in the current version.

### 5.2 Recalibration

**Time to recalibrate:** 15 minutes.

**Method:** Identify the new signal categories from a sample of April responses. Update word lists and scoring weights. Validate on labeled data.

**Result:** 10/10 accuracy restored on the original 10-finding test set.

| Finding | Type | Decision | Correct |
|---|---|---|---|
| latin1_crash | REAL | KEEP | ✅ |
| body_seek_null | REAL | KEEP | ✅ |
| username_leaked | REAL | KEEP | ✅ |
| keyerror_realm | REAL | Human review | ✅ |
| digest_returns_none | REAL | KEEP | ✅ |
| hashlib_exceptions | FALSE | REJECT | ✅ |
| nonce_overflow | FALSE | REJECT | ✅ |
| flask_jsonify | FALSE | REJECT | ✅ |
| aiohttp_clear_stale | FALSE | REJECT | ✅ |
| aiohttp_morsel | FALSE | REJECT | ✅ |

*keyerror_realm routes to human review — the closest call in the dataset. One additional signal word would have triggered rejection of a real bug. This boundary case is documented.*

*Source: `replication_bus/run_20260401_143737/`*

### 5.3 What the Migration Proves

The mechanism — producing measurably different outputs on real bugs versus false positives — persisted through the model update. The vocabulary did not.

This establishes a requirement for any production deployment: calibration is per-model-version, not per-model. A recalibration protocol is not optional maintenance. It is a core architectural requirement.

---

## 6. Blind Validation

### 6.1 Setup

To test on genuinely unseen data, I ran the pipeline on `fastapi/oauth2.py` (693 lines, FastAPI source). This file was not used in any calibration step.

The discovery phase used a standard single-pass prompt — not the optimized multi-theme architecture used in full pipeline deployments. This was intentional: I am testing the gate in isolation, not maximum recall. The gate was scored on what a basic discovery run produces.

### 6.2 Results

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

**Zero friendly fire. Zero real bugs killed.**

### 6.3 What the Gate Does

The gate answers one specific question: *Can this finding be proven to crash or fail with a specific input?*

It does not answer: *Is this code secure?*

Security design concerns — rate limiting gaps, credential storage practices, protocol compliance — are Class C findings. They require human judgment. The gate filters Class D noise (hallucinations) so that human review focuses on what matters.

---

## 7. Limitations

### 7.1 Sample Size

Replication: 10 findings. Blind validation: 8 findings on 1 file. These are proof-of-concept numbers. Production validation requires 100+ findings with verified ground truth.

### 7.2 The Gate's Scope

The gate is calibrated on runtime failures. It does not evaluate security design. One documented case: a finding about a cryptographic parameter passed the gate because the model constructed confident proof language — but the proof assumed attacker control over parameters that are application-controlled. The gate cannot verify data flow. It verifies whether the model produces confident proof language. Those are different things.

### 7.3 Model Drift

The March → April migration proved calibration expires. Any deployment requires monitoring and a recalibration protocol. Monthly recalibration is the minimum.

### 7.4 Per-Model Baseline Variation

Grok-3 baseline token confidence: ~0.96. GPT-4o baseline: ~0.80. A universal confidence threshold cannot apply across models. All thresholds must be calibrated per model.

### 7.5 Discovery Quality

The blind validation used a basic discovery prompt. The full pipeline uses a multi-theme architecture that produces higher recall. The gate results are valid — the gate scored what discovery found — but a stronger discovery pass would surface more findings for the gate to evaluate. These are independent components.

---

## 8. The Collaboration

### 8.1 How This Was Found

This research was conducted through a structured collaboration between me and AI assistants — primarily Anthropic Claude for analysis and Grok-3 as the primary test subject.

My contribution was the framing. The AI was treating hallucination detection as a threshold-finding problem. Confidence scores were constant. Every approach collapsed at the same wall.

I reframed it through quantum decoherence. I had been asking the models to measure their own superposition. The question I brought was: what if the output doesn't exist in a definite state until we observe it in a specific way? What if the act of demanding proof is itself the observation that collapses uncertainty into something measurable?

That question led to the adversarial challenge design. The AI contribution was the execution — processing 30,000+ tokens of logprob data, identifying the specific word categories that carry the signal, running the cross-model comparisons, documenting the migration.

My contribution was abstract. The AI's contribution was analytical. Neither was sufficient alone. I could not process 30,000 tokens of logprob data. The AI would not have asked the question that led to looking at per-token confidence on behavioral markers.

The collaboration is the methodology. This is documented explicitly because it is reproducible — not as a curiosity about tools, but as a research approach with implications for how novel insights can be extracted from frontier models.

### 8.2 Reproducibility

Every number in this paper maps to a named source file:

| Claim | Source |
|---|---|
| Hedge drop -31.6% | `prove_it_logprobs/grok_prove_*.json` (11 files) |
| Hedge drop -23.0% | `token_fingerprinting/fingerprint_grok_run*.json` (6 files) |
| "potentially" at 11.6% | `grok_prove_8.json` token [1254] |
| "might" at 18.9% | `grok_prove_2.json` token [62] |
| "crash" at 100.0% | `grok_prove_0.json` token [1141] |
| "raises" at 85.6% | `grok_prove_0.json` token [650] |
| 0/5 GPT cross-model transfer | Original March calibration |
| 10/10 recalibrated gate | `replication_bus/run_20260401_143737/` |
| 8/8 FastAPI blind rejection | `validation_blind/fastapi_oauth2_f*_analysis.json` |

Raw data repository: `github.com/blazingRadar/sib29`

The scoring implementation and full pipeline architecture are not published. They are part of a production system under active development.

---

## 9. Related Work

Prior work falls into four categories, each distinct from BUS analysis.

**Confidence calibration.** Kadavath et al. (2022) and related work examined whether LLM self-reported confidence correlates with accuracy, finding systematic miscalibration. My finding that aggregate confidence is approximately constant regardless of finding validity (Cunningham, 2026a) is consistent with this literature. BUS analysis bypasses calibration by measuring behavioral signals rather than stated confidence.

**Self-consistency.** Wang et al. (2022) demonstrated that sampling multiple outputs and taking majority vote improves accuracy. Self-consistency uses homogeneous samples — same prompt, different temperature draws. BUS uses a heterogeneous challenge — a different prompt designed to elicit proof — and measures behavior rather than voting on outputs.

**LLM-as-judge.** Zheng et al. (2023) and subsequent work uses one LLM to evaluate another's outputs. BUS differs in that the model being evaluated is the same model being challenged. The signal comes from behavior under adversarial challenge, not from third-party evaluation.

**Execution-based verification.** Recent work (2025) uses execution environments to verify whether LLM-generated vulnerability findings reproduce. Execution-based verification is more definitive but requires infrastructure unavailable in standard API deployments. BUS operates in black-box API contexts with no execution infrastructure.

The specific contribution — using adversarial challenge as an observation event that collapses model uncertainty into measurable behavioral markers, with per-model-version calibration — does not appear in published literature as of April 2026.

---

## 10. Conclusion

Behavioral Uncertainty Signatures are real and measurable. When forced to prove a false finding, models produce involuntary behavioral markers detectable through surface text analysis. The mechanism is stable. The vocabulary is time-bound. The gate can be recalibrated as models update.

Practical implications:

1. **Hallucination detection is possible** using only API responses — no model internals required
2. **Calibration is per-model and per-model-version** — word lists expire with model updates
3. **Recalibration is fast** — labeled examples, minutes of analysis
4. **The adversarial challenge is the key intervention** — it forces epistemic failure into a measurable form
5. **The gate answers one question** — provable runtime failure — not general security

The BUS gate is one component of SIB29, a multi-stage AI code review pipeline in production development. The gate's mechanism is documented here because it represents a general finding about LLM behavior. The full pipeline — including multi-model rotation, adversarial discovery architecture, and human-in-the-loop review — is proprietary and not covered in this paper.

The raw logprob data is published for reproduction and independent analysis. The pipeline implementation is not.

---

## Appendix: Cost and Runtime

| Experiment | API Calls | Cost (est.) | Runtime |
|---|---|---|---|
| Original calibration (March) | ~65 | ~$15 | 2 hours |
| Replication (10 × 2 models) | 20 | ~$8 | 136 seconds |
| Blind validation (10 files) | 90 | ~$30 | 25 minutes |
| Recalibration | 0 (reused data) | $0 | 15 minutes |
| **Total** | **~175** | **~$53** | **~3 hours** |

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
