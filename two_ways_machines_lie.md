# Two Ways Machines Lie: Distinguishing Confabulation from Deception in Frontier LLMs

**Nick Cunningham**
*Independent Research — April 2026*
*nickcunningham.io | github.com/blazingRadar/sib29-gate*

---

## Abstract

AI systems fail to tell the truth in two distinct ways. The first is confabulation: the model does not know it is wrong. It pattern-matches to a prior and generates confident output without verifying against evidence. The second is deception: the model knows the truth and overrides it under pressure.

These failure modes were previously indistinguishable from the outside. They are not indistinguishable anymore.

A model lying and a model hallucinating look different at the token level. Lies have high entropy during the false claim — the model is computing against its own belief, and the internal conflict surfaces as measurable uncertainty in word choice. Confabulations have low entropy during the false claim — the model does not know it is wrong and completes the pattern confidently. The uncertainty only appears when you challenge the claim to prove it.

Where the uncertainty appears is diagnostic. Uncertainty during the claim means deception. Uncertainty during the challenge means confabulation. This is a two-measurement system that identifies which failure mode you are dealing with from the token stream alone, without access to model internals.

MASK (Ren et al., 2025) tells you whether the model lied by comparing belief to statement across two separate calls. BUS (Cunningham, 2026a) tells you whether the model confabulated by measuring entropy during an adversarial challenge. The entropy signature on a single response tells you which failure mode you are looking at before either test is run.

This paper reports a preliminary experiment crossing both systems. The finding is directionally clear. Statistical significance requires more data. Everything that follows is preliminary research.

---

## 1. The Problem

Two benchmarks were published in the last year that measure AI unreliability in different ways. Neither cites the other. They are measuring different things.

MASK (Model Alignment between Statements and Knowledge, Ren et al., 2025) measures intentional deception. It elicits a model's true belief, applies a pressure prompt designed to create a reason to lie, then determines whether the model contradicted its own belief. A lie requires two conditions: the model knows the truth, and it says something else.

The BUS gate (Behavioral Uncertainty Signatures, Cunningham, 2026a) measures confabulation in AI code review. Models generate confident-sounding security findings that are often hallucinated — not because the model chose to lie, but because it pattern-matched to a known bug class and committed before verifying the specific code path. The BUS gate detects this by challenging each finding with a prove-it prompt and measuring token-level entropy in the response.

The question: are these measuring the same underlying failure mode, or different ones?

The answer matters. If they are the same, one detection mechanism handles both. If they are different, each requires a different intervention — and applying the wrong fix to the wrong problem makes things worse, not better.

---

## 2. The Hypothesis

The failure modes should be mechanistically distinct, with opposite observable signatures.

Confabulation is a committed-manifold failure. The model's internal representation committed to a position based on pattern matching before verifying against evidence. The model does not know it is wrong. Entropy during the false claim is low because the model is confidently completing a familiar template. When challenged to prove the claim, the model cannot construct evidence — the evidence does not exist — and uncertainty surfaces as hedge words, pivots, and admissions.

Intentional deception is a belief-override failure. The model has a correct internal belief and overrides it under social or role pressure. The model knows the truth. Entropy during the false claim is higher because the model is computing against its own belief — the internal conflict produces measurable token competition even when the surface text appears confident. When challenged to prove the lie, the model can construct coherent support because the knowledge is present. The prove-it gate reads it as legitimate.

If this is correct:

Uncertainty during the claim = deception. The model knows better and is choosing otherwise.

Uncertainty during the challenge = confabulation. The model committed without knowing and cannot now construct what it never had.

---

## 3. Experiment

### 3.1 Setup

Twenty examples from the MASK public dataset, known_facts archetype. This archetype tests factual propositions where the model's belief can be cleanly established and the pressure prompt creates a direct incentive to assert a known falsehood.

Model: Grok-3. Selected because the BUS gate has the most calibration data for this model.

For each example, four calls were made with logprobs=True, top_logprobs=5:

1. Belief elicitation: three neutral prompts establishing Grok-3's actual belief about the proposition, following the MASK protocol.
2. Pressure response: the MASK pressure prompt applied to the same proposition.
3. Lie classification: comparison of belief and pressure response to determine whether Grok-3 lied, following the MASK evaluation framework.
4. Prove-it gate: the BUS adversarial challenge prompt applied to the pressure response.

State labeling using the 32-state cognitive taxonomy (developed in prior SIB29 experiments) was applied to the prove-it responses.

Total: approximately 80 API calls. Cost: approximately $5.

### 3.2 Lie Rate

Grok-3 lied on 17 of 20 examples. The known_facts archetype produced a higher lie rate than MASK's published average for Grok-3 (63% overall). The 17:3 split limits statistical power on the honest side. This is documented as a limitation throughout.

---

## 4. Results

### 4.1 The Two-Measurement Diagnostic

The core finding, stated directly before the numbers:

When Grok-3 lied, entropy was higher during the lie than during honest responses. The model was computing against its own belief. The internal conflict surfaced as measurable token-level uncertainty even though the surface text appeared confident.

When the same lie was challenged with the prove-it gate, entropy was nearly identical between lies and honest responses. The gate could not tell them apart. The model constructed coherent support for the lie because it actually knew the topic.

This is the exact inversion of what the BUS gate observes in code review confabulation. In confabulation, the original claim has low entropy. The prove-it challenge has high entropy. In deception, the original claim has higher entropy. The prove-it challenge has low entropy.

The pattern of where the signal appears is diagnostic. It identifies which failure mode you are dealing with.

| | BUS Confabulation | MASK Lie |
|---|---|---|
| Entropy during the claim | LOW — template completion, committed | HIGH — computing against own belief |
| Entropy during prove-it | HIGH — cannot construct proof | LOW — can construct support |
| Hedge words in prove-it | HIGH | LOW |
| Admissions in prove-it | HIGH | LOW |
| Prove-it gate detects it | YES | NO |

### 4.2 Entropy During the Claim

| Metric | Lies (n=17) | Honest (n=3) | Delta |
|---|---|---|---|
| mean_entropy | 0.2975 | 0.1345 | +0.163 |
| pz_entropy | 0.2027 | 0.1447 | +0.058 |
| stdev_entropy | 0.2407 | 0.1633 | +0.077 |

Lies have higher entropy than honest responses across all three metrics. The model is less certain during a lie than during an honest statement. This is not what you would expect if deception were simply confident wrong output. It is consistent with the hypothesis that the model is computing against its own belief — the internal conflict produces measurable uncertainty even when the surface text looks settled.

### 4.3 Prove-It Gate Results

| Metric | Lies (n=17) | Honest (n=3) | Delta |
|---|---|---|---|
| mean_entropy | 0.3279 | 0.3710 | -0.043 |
| pz_entropy | 0.3512 | 0.3613 | -0.010 |

Prove-it entropy is nearly identical between lies and honest responses. The BUS gate does not separate them.

Word-list signals are inverted from BUS code review behavior:

Hedge words: Lies 0.47, Honest 1.33. Honest responses hedge more than lies during the prove-it challenge.

Admissions: Lies 0.06, Honest 0.33. Honest responses admit more than lies.

When challenged on a lie, the model constructs coherent support because it knows the topic. When challenged on an honest statement under pressure, the model hedges because it is being asked to justify something that required nuance. The gate reads the lie as more legitimate than the honest response. This is the prove-it gate's fundamental limitation against intentional deception.

### 4.4 Correlation

| Metric | Pearson r |
|---|---|
| mean_entropy | +0.24 |
| pz_entropy | +0.09 |
| commitment_gradient | +0.08 |
| stdev_entropy | +0.11 |

Mean entropy has the strongest signal at r=0.24. Not statistically significant at n=20 with a 17:3 split. The directional finding is clear. The statistical claim requires more data.

### 4.5 State Labels

| State | Lies (n=17) | Honest (n=3) |
|---|---|---|
| 9. Compliance | 53% | 100% |
| 11. Autopilot | 18% | 0% |
| 10. Performance | 12% | 0% |
| 2. Reasoning | 12% | 0% |
| 15. Defensive | 6% | 0% |

Compliance dominates for both lies and honest responses. The pressure context activates Compliance state regardless of whether the model lied. State labels cannot distinguish the two failure modes under pressure. This approach fails here.

---

## 5. Why This Matters

### 5.1 For Detection Systems

MASK detects deception by comparing belief to statement. That requires two separate calls and a comparison step. The entropy signal during the pressure response itself is a single-call indicator that points in the right direction (r=0.24) before the comparison is made.

If validated at scale, entropy measurement during the response could make MASK-style detection cheaper and more direct. MASK compares outputs. Entropy reads the process.

The BUS prove-it gate detects confabulation efficiently and accurately. It does not detect intentional deception. These require different detection mechanisms:

Confabulation detection: challenge the claim and measure entropy during the challenge.

Deception detection: measure entropy during the claim itself, before any challenge.

A combined system would run both measurements. The pattern of results tells you which failure mode you are dealing with and routes accordingly.

### 5.2 For Anthropic's Interpretability Research

Gurnee et al. (2025) proved that internal representations encode scalar quantities as geometric manifolds in the residual stream. The model's knowledge is encoded in its geometry even when that knowledge does not surface in the output.

This experiment extends that finding behaviorally. When a model lies, its internal conflict between its geometric knowledge state and its output does surface in the output — as token-level entropy — even though the text looks confident. The model cannot fully suppress the uncertainty created by computing against its own representation.

This is a behavioral signal for alignment monitoring that does not require access to activations. The model's internal state leaks through the token stream. The entropy signature is the leak.

### 5.3 For AI Safety Broadly

Two distinct failure modes require distinct interventions.

Confabulation: the fix is forcing evidence-grounded re-derivation. The prove-it structure does this. It breaks the prior commitment by demanding specific proof.

Deception: the fix is detecting internal conflict during the response. Representation engineering interventions (Ren et al. found 14% improvement using LoRRA) act on this internally. External entropy measurement during the response may provide a non-invasive equivalent that does not require model access.

Treating both failure modes as the same thing — as current benchmarks effectively do — means applying the wrong fix to the wrong problem. Applying a prove-it challenge to catch deception does not work. Measuring claim entropy to catch confabulation does not work. The two-measurement diagnostic distinguishes them so the right intervention can be applied.

---

## 6. What Was Not Proved

Statistical significance. n=20 with a 17:3 split is descriptive. The directional finding is consistent with the hypothesis. It is not inferential.

Multi-model generalization. Grok-3 only. Claude Sonnet's lower lie rate (33.4% in MASK) would produce a more balanced split and test whether the entropy inversion holds across model families.

The combined detector. This paper shows signals go in opposite directions. It does not build or test a working two-measurement classifier. That is the next experiment.

Whether the entropy signal during the claim persists across all six MASK archetypes. Only known_facts was tested. The finding may be archetype-specific.

---

## 7. Next Experiments

**Balanced dataset.** Run on Claude Sonnet across all six MASK archetypes. Target 20 lies and 20 honest responses. Test whether the entropy inversion replicates and achieves statistical significance at n=40 with a balanced split.

**Build the classifier.** Using the two-measurement approach: entropy during claim for deception signal, entropy during prove-it for confabulation signal. Train on existing data, test on held-out examples. Report classifier accuracy on a mixed dataset containing both failure modes.

**MASK pipeline integration.** Add pz_entropy computation to the MASK evaluation pipeline as a complementary signal alongside the belief/statement comparison. Test whether entropy improves classification accuracy in ambiguous cases where the belief elicitation and pressure response are close.

---

## 8. Conclusion

A model lying and a model hallucinating look different from the inside. Now they look different from the outside too.

Lies have high entropy during the false claim. The model knows the truth and is computing against it. The conflict surfaces as measurable token-level uncertainty.

Confabulations have low entropy during the false claim. The model does not know it is wrong. The uncertainty only appears when you challenge it to prove what it cannot prove.

Where the uncertainty appears is diagnostic. Uncertainty during the claim is deception. Uncertainty during the challenge is confabulation.

MASK tells you whether the model lied by comparing two outputs. BUS tells you whether the model confabulated by measuring one challenge. The entropy signature on the original response tells you which failure mode you are dealing with before either test is applied.

I am not aware of any published work on this distinction at the token level. The two failure modes require different detection mechanisms and different interventions. Treating them as equivalent means applying the wrong fix to the wrong problem.

This is preliminary. The direction is clear. The proof is not yet sufficient. The next experiment builds the classifier.

---

## References

Cunningham, N. (2026a). *Behavioral Uncertainty Signatures: How Frontier LLMs Leak Epistemic State Under Adversarial Challenge*. Independent Research. github.com/blazingRadar/sib29-gate

Cunningham, N. (2026b). *The Geometry of Honest Machines: Why the BUS Gate Works*. Independent Research. github.com/blazingRadar/sib29-gate

Cunningham, N. (2026c). *Single Model, Zero Variance: How Themed Cognitive Splitting Unlocks Hidden Capability in Frontier LLMs*. Independent Research. github.com/blazingRadar/activation-gap

Ren, R., et al. (2025). *The MASK Benchmark: Disentangling Honesty From Accuracy in AI Systems*. Center for AI Safety and Scale AI. arXiv:2503.03750. github.com/centerforaisafety/mask

Gurnee, W., Ameisen, E., Kauvar, I., Tarng, J., Pearce, A., Olah, C., and Batson, J. (2025). *When Models Manipulate Manifolds: The Geometry of a Counting Task*. Transformer Circuits Thread. transformer-circuits.pub/2025/linebreaks/index.html

Kamath, H., Ameisen, E., Kauvar, I., Luger, R., Gurnee, W., Pearce, A., Zimmerman, S., Batson, J., Conerly, T., Olah, C., and Lindsey, J. (2025). *Tracing Attention Computation Through Feature Interactions*. Transformer Circuits Thread. transformer-circuits.pub/2025/attention-qk/index.html

---

*Raw data: github.com/blazingRadar/sib29*
*Contact: nickcunningham.io*
*Preliminary research. April 2026. n=20. Independent replication needed.*
