# The Geometry of Honest Machines: Why the BUS Gate Works

**Nick Cunningham**
*Independent Research — April 2026*
*nickcunningham.io | github.com/blazingRadar/sib29-gate*

---

## Abstract

The Behavioral Uncertainty Signatures (BUS) gate, documented in *Behavioral Uncertainty Signatures: How Frontier LLMs Leak Epistemic State Under Adversarial Challenge* (Cunningham, 2026a), detects AI hallucinations in code review by measuring token-level confidence patterns in adversarial challenge responses. The original paper established that the mechanism exists, quantified it empirically, and demonstrated 72.2% false positive rejection with 98.7% real bug pass-through. What it did not explain was *why* it works at a mechanistic level.

This paper provides that explanation, grounded in Anthropic's interpretability research on how frontier language models represent uncertainty geometrically in their residual streams. Four specific findings from Gurnee et al. (2025) connect the BUS gate's external behavioral observations to the internal geometric structure of the model's computation. The connection produces three actionable updates to the BUS architecture: a theoretical account of why proof-zone entropy is more durable than the word list under model updates, a falsification of the hypothesis that the gate can be moved upstream into Phase 1a discovery, and a mechanistic explanation for why false positive responses are confident rather than random.

All experimental results referenced here are from the original BUS paper and companion experiments documented in the SIB29 research repository. No new API calls were required to produce this paper.

---

## 1. Background

### 1.1 What the Original Paper Found

The BUS gate works by challenging each AI-generated code review finding with an adversarial prompt: construct a specific proof of this finding. Show the exact function call, the exact input value, the exact runtime state that causes the failure. Then it measures the behavioral response.

When a model cannot substantiate a false finding, specific words in its response — hedge words, pivots, admissions — appear with measurably lower token-level confidence than surrounding context. On real bugs, claim words appear at 85-100% confidence. On false positives, the same word categories appear at 11-26% confidence. The gap is detectable, calibratable, and deployable without model internals.

The clearest single example: on a hashlib false positive, the model produced "might" with 18.3% confidence at a specific token position. The top alternative at that position was "does" at 78.2%. The model believed "does" was more likely. It chose "might" anyway. The uncertainty was involuntary.

Across 191 findings in 7 languages and 10 codebases, proof-zone entropy separates real bugs from false positives at r = +0.326, p = 0.0001.

### 1.2 What the Original Paper Did Not Explain

The original paper had an operational explanation for why hedge words carry lower confidence on false positives: the model is uncertain, and that uncertainty leaks into its word choices. That explanation is correct but incomplete. It describes what happens without explaining where the uncertainty comes from, why it surfaces at specific token positions, or why false positive responses are confident rather than uncertain overall (aggregate confidence is identical at 87.1% for both real bugs and false positives).

The question the original paper left open: what is the internal mechanism that produces this external behavioral signal?

---

## 2. The Mechanistic Foundation

### 2.1 The Manifold Paper

Gurnee et al. (2025), *"When Models Manipulate Manifolds: The Geometry of a Counting Task,"* is a mechanistic interpretability study of how Claude 3.5 Haiku learns to predict line breaks in fixed-width text. The researchers wanted to understand how a model — seeing only a sequence of token integers — learns to count characters, track position, detect approaching line limits, and decide whether to break a line.

What they found was not a threshold rule. The model builds geometric structures in its residual stream: curved one-dimensional manifolds that encode scalar quantities like character count and line width. These manifolds are computational objects, not just representations. Attention heads act on them geometrically — rotating one manifold to align with another, comparing two counts by measuring angular difference rather than subtraction.

Four findings from this paper directly illuminate the BUS gate's mechanism.

---

## 3. Four Connections

### 3.1 Uncertainty as Manifold Position

The paper's central result: when the model tracks a quantity it is uncertain about, it does not commit to a single value. It maintains a position on a curved manifold that spans the plausible range. The uncertainty is structural, encoded in where the activation sits on the manifold — not in any word the model outputs.

**What this means for BUS:**

The model is representing two quantities geometrically during an adversarial challenge: its internal estimate of whether the finding is real, and its confidence in that estimate. On a real bug, the activation has traveled far along the manifold toward a committed position. On a false positive, the activation sits somewhere in the middle of the manifold, near the high-curvature zone where many positions are plausible. When the model generates text from that uncertain manifold position, it reaches for words that match the uncertainty — hedge words. And those words, because the model is genuinely uncertain about which word to use, carry lower logprob confidence.

The BUS gate is not catching post-hoc linguistic style. It is catching the token-level expression of a genuine geometric uncertainty in the model's internal representation. The "might" at 18.3% is not a stylistic choice. It is the surface expression of a manifold position that has not committed.

This also explains why aggregate confidence is identical for real bugs and false positives. The manifold position of overall response confidence is similar for both — the model is committed to generating a response. The manifold position of specific claim tokens differs. The signal is not in the mean. It is in specific word categories at specific positions where the epistemic commitment is tested.

### 3.2 Concordance and Discordance Heads

The paper identifies two classes of attention heads with distinct computational roles:

Concordance heads attend to tokens that agree with the current estimate, reinforcing a committed position on the manifold. Discordance heads attend to tokens that create tension with the current estimate, introducing curvature and uncertainty.

These two head types produce measurably different activation patterns. The concordance/discordance distinction is a structural property of how the model represents agreement and disagreement between its internal state and incoming evidence.

**The Phase 1a experiment and its falsification:**

This framing generated a testable hypothesis. If concordance and discordance heads are active during code audit discovery — Phase 1a, before any adversarial challenge — the same entropy signal that appears in prove-it responses should also be visible in discovery outputs. The geometric uncertainty should exist in Phase 1a whether or not it is surfaced by a challenge prompt.

This hypothesis was tested on auth.py with 6 known bugs: 51 finding segments across 4 Phase 1a calls, 31 real findings and 20 false positives.

The result falsified the hypothesis. Phase 1a pz_entropy showed no meaningful separation between real bugs and false positives (r = -0.012, p = 0.94). The concordance/discordance mechanism is not active during Phase 1a discovery in the same way it is active during adversarial prove-it.

The gradient result added precision. In Phase 1a, false positives show *more negative* gradients — the model commits more confidently while writing hallucinations than while writing real bugs. This is the opposite of the prove-it pattern. The explanation: during discovery, false positives are template completions. The model has seen thousands of security audit reports and generates plausible-sounding findings by pattern matching. Template completion is low-entropy and committed. Real bugs require genuine reasoning about specific code paths, which produces more variance in the entropy trajectory.

The prove-it prompt inverts this entirely. By demanding construction of a specific proof, it transforms the task from template completion — which false positives do fluently — to ground-truth derivation — which only real bugs can support. The concordance/discordance mechanism fires differently under this task structure. The gate is not surfacing a pre-existing signal. It is creating the conditions under which the signal can exist.

**Practical consequence:** The BUS gate cannot be moved upstream into Phase 1a. The prove-it call is not redundant. It is the observation event.

### 3.3 The Visual Illusion — Why False Positives Are Confident

One of the paper's most striking findings was what the researchers called visual illusions. When the string `@@` was inserted into a prompt outside of a git diff context, the character-counting attention heads got distracted. They attended to `@@` instead of the previous newline, miscounted the line position, and produced wrong linebreak predictions. The model's learned prior about what `@@` means in git diffs was applied in the wrong context, corrupting the manifold state.

The researchers' framing: *"mis-application of a learned prior, including the role of cues such as @@ in git diffs, can modulate estimates of properties."*

**What this means for BUS:**

This is exactly the mechanism that produces confident false positives in code review.

The model has learned priors about what bugs look like — patterns from its training data: CVEs, security writeups, audit reports, Stack Overflow threads. When it encounters code that superficially matches one of those patterns, it applies the prior even if the specific code path does not actually produce the vulnerability. The manifold commits early from the pattern match, before the actual code logic is verified.

The visual illusion structure is:

> Correct mechanism (character counting) + wrong context cue (@@ outside git diff) = wrong output

The code review hallucination structure is:

> Correct mechanism (bug pattern recognition) + wrong context cue (code that resembles a known bug) = wrong finding

This explains what the original paper observed but could not account for: false positive responses are *confident*. They are not low-quality responses or random noise. The model is genuinely certain because its pattern-matching mechanism fired cleanly. The manifold committed early and completely. The prove-it gate works precisely because it forces the model to re-derive from the actual code rather than from the activated prior. It breaks the illusion by demanding ground-truth re-entry.

A model challenged to prove a committed-but-wrong manifold position is forced into a different computation than the one that produced the original finding. It cannot simply replay the committed state. It must construct specific evidence. When that evidence does not exist, the manifold cannot commit — and the uncertainty leaks into the tokens.

### 3.4 The Complexity Tax — What the Word List Actually Is

The paper argues that studying discrete features without understanding the underlying geometric structure pays a "complexity tax." You get a true description of the computation, but an unnecessarily complicated one. Both a list of specific features and a manifold description are true. The manifold description reveals structure the feature view obscures.

**What this means for BUS:**

The BUS gate word list — hedge words, scope words, refusal words — is the feature view of the epistemic uncertainty signal. It is a list of discrete surface features that correlate with the model being in an uncertain manifold state. It works. But it pays the complexity tax: it describes which words happen to correlate with uncertainty rather than describing the underlying mechanism.

The manifold view suggests the true signal is simpler: the model's position on the epistemic certainty manifold determines the token confidence at pivotal words, regardless of which specific words appear. The pz_entropy metric is closer to the manifold view. It measures average entropy at the proof-zone tokens, which is a proxy for manifold position. The word list is a noisy discrete approximation of that same quantity.

This reframing explains the March to April 2026 drift documented in the original paper. Grok-3 shifted from Denial to Transparent Refusal behavioral mode between versions. The word list broke completely. The pz_entropy signal remained directionally stable. The underlying manifold behavior — epistemic uncertainty leaking through token confidence at pivotal positions — did not change. Only the vocabulary the model used to express that uncertainty changed.

The word list correlates with manifold position through specific vocabulary. When the vocabulary changes, the word list loses its correlation. The entropy measure correlates with manifold position through the underlying geometry. When the vocabulary changes, the geometry does not necessarily change with it. This is why pz_entropy is a more durable signal and why the recalibration requirement comes primarily from the word list component, not the entropy component.

---

## 4. The Updated Architecture

Combining the four connections above produces an updated theoretical account of the BUS gate:

**What the gate detects:** A model in an uncertain manifold state — an activation that has not committed to a definite position on the epistemic certainty manifold because the underlying evidence does not support commitment.

**Why the prove-it prompt is necessary:** The adversarial challenge is not a measurement tool. It is an observation event — the quantum decoherence framing from the original paper, now grounded in geometric terms. The prove-it structure forces the model to re-derive from ground truth rather than replay a committed prior. This creates the conditions under which manifold uncertainty can surface in token confidence. Phase 1a discovery does not create these conditions. The prove-it call is load-bearing.

**Why false positives are confident before the challenge:** The manifold committed early from a pattern-match prior. The model is genuinely certain. The challenge breaks the commitment by demanding evidence the prior cannot supply.

**Why pz_entropy is more durable than the word list:** The entropy measure tracks manifold position through the underlying geometry. The word list tracks manifold position through specific vocabulary. Geometry is more stable than vocabulary across model updates.

**Updated gate component durability:**

| Component | Signal basis | Durability | Recalibration trigger |
|---|---|---|---|
| pz_entropy | Manifold position via geometry | High — geometry stable across updates | Major architectural change |
| Word list | Manifold position via vocabulary | Medium — vocabulary shifts with updates | Model version change |
| Combined gate | Both signals, near-orthogonal (r = -0.078) | High — complementary failure modes | Per-codebase threshold adjustment |

---

## 5. What Remains Open

**The calibration floor.** Per-codebase calibration is required for both signals. The threshold transfer failure documented in the original paper (42.9% uncalibrated vs. 72.2% calibrated) is a distribution shift problem — different codebases produce different absolute entropy scales. The manifold explanation adds precision: different codebases activate different pattern-match priors with different manifold commitment depths. Per-codebase calibration sets the threshold for a specific distribution of manifold positions, not a universal one.

**The simple proof problem.** The single confirmed friendly fire case in the original paper, `digest_returns_none`, represents a finding whose proof structure does not require navigating genuine technical uncertainty. The model reads a code comment that announces the bug, transcribes the proof mechanically, and produces low-entropy output throughout — indistinguishable from a false positive's manifold-committed template completion. The manifold explanation predicts this failure: when the evidence genuinely supports commitment, even simple bugs produce committed manifold positions. The word list catches this case because the committed proof response contains high-confidence claim vocabulary. The word list's role in the combined gate is precisely to handle cases where pz_entropy cannot discriminate.

**Cross-model geometry.** The original paper showed that competition vector geometry is nearly universal across models at r = 0.9901 cosine similarity. The manifold explanation predicts this: the same underlying need to represent epistemic uncertainty geometrically should produce similar manifold structures across model families trained on similar data. The vocabulary differs. The geometry is conserved. Full cross-model entropy validation has not been run. The prediction is that pz_entropy will transfer across model families at calibrated thresholds. The word list will not.

**Logprob access.** The entropy signal requires logprob access, which is disappearing from newer model APIs. The word list does not. The practical consequence: the combined gate degrades gracefully as logprob access narrows — the word list handles an increasing fraction of the classification load, entropy handles what vocabulary cannot. The architecture is designed for a world where logprob access is a declining resource, not a stable one.

---

## 6. Conclusion

The BUS gate works because:

1. Models represent epistemic uncertainty geometrically in their residual streams as manifold positions.
2. The adversarial prove-it prompt forces the model to re-derive from ground truth rather than replay a committed prior, creating the conditions under which manifold uncertainty surfaces in token confidence.
3. False positives are confident before the challenge because a pattern-match prior committed the manifold early. The challenge breaks this commitment by demanding evidence the prior cannot supply.
4. Proof-zone entropy is a more durable gate signal than the word list because it measures manifold position through geometry, which is more stable than the vocabulary through which the word list tracks the same quantity.

The original paper found the signal empirically and documented it. This paper explains mechanistically why the signal exists, which is what the Phase 1a falsification demanded: a theory precise enough to make testable predictions that could be wrong. The prediction was that concordance/discordance heads would produce a detectable Phase 1a signal. The experiment falsified it. The theory explains the falsification: the observation event — the prove-it prompt — is not revealing a pre-existing signal, it is creating the conditions for the signal to exist. You cannot move that upstream. The prove-it call earns its cost.

---

## References

Cunningham, N. (2026a). *Behavioral Uncertainty Signatures: How Frontier LLMs Leak Epistemic State Under Adversarial Challenge*. Independent Research. github.com/blazingRadar/sib29-gate

Cunningham, N. (2026b). *Single Model, Zero Variance: How Themed Cognitive Splitting Unlocks Hidden Capability in Frontier LLMs*. Independent Research. github.com/blazingRadar/activation-gap

Gurnee, W., Ameisen, E., Kauvar, I., Tarng, J., Pearce, A., Olah, C., and Batson, J. (2025). *When Models Manipulate Manifolds: The Geometry of a Counting Task*. Transformer Circuits Thread. transformer-circuits.pub/2025/linebreaks/index.html

Kamath, H., Ameisen, E., Kauvar, I., Luger, R., Gurnee, W., Pearce, A., Zimmerman, S., Batson, J., Conerly, T., Olah, C., and Lindsey, J. (2025). *Tracing Attention Computation Through Feature Interactions*. Transformer Circuits Thread. transformer-circuits.pub/2025/attention-qk/index.html

---

*Raw data: github.com/blazingRadar/sib29*
*Contact: nickcunningham.io*
*Preliminary research. April 2026.*
