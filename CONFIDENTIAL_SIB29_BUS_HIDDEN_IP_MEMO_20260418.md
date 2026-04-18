# CONFIDENTIAL SIB29 BUS HIDDEN IP MEMO

Date: 2026-04-18
Status: Internal research memo
Purpose: Preserve the unpublished operational mechanics behind the SIB29 / BUS hallucination gate so the real invention center is not lost to the public-paper layer.

This file is an archival mirror for discoverability in `sib29-gate`. The primary write-up is the same memo preserved in the broader `sib29` research tree.

## What This Memo Preserves

- unpublished word lists
- scoring formulas
- token-level logprob metrics
- entropy / geometry metrics
- recalibration logic
- Grok vs GPT behavioral differences

## Core Operational Findings

1. BUS was never just a hedge-word heuristic.
2. The real mechanism was adversarial prove-it plus token-local instability measurement.
3. Static word lists were model-specific surfaces, not the invention itself.
4. Grok and GPT expressed epistemic failure differently enough that calibration portability failed.
5. The stronger gate combined lexical cues with entropy / competition metrics.

## Original Grok Lexical Gate

Signals:

- admissions
- pivots
- hedges

Decision path:

```text
total_signal = admissions + pivots + hedges

0      -> KEEP
1 to 2 -> MAYBE / Q2
>= 3   -> REJECT
```

With score boosts for long prove-it responses and high counts of pivots / hedges / admissions.

## Hidden Token-Level Metrics

```text
baseline_confidence = mean(exp(logprob)) * 100
hedge_avg_confidence
claim_avg_confidence
hedge_delta_from_baseline
hedge_tokens_below_50pct
```

Per-token alternative competition was stored because decisive proof tokens often showed strong alternative preference at the exact point the model was trying to sustain a false finding.

## Geometry Layer

- gap metric: `predicted_lp - best_alt_lp`
- proof-zone entropy across the middle proof band
- combined scorer fused normalized entropy with normalized lexical signal

## Grok vs GPT

Grok false positives:

- argue
- pivot
- hedge
- partially admit

GPT false positives:

- cleaner language
- scope mismatch
- code praise / resilience framing
- much less visible hedging

Practical result:

- Grok-calibrated lexical lists did not transfer to GPT

## Invention Center

The strongest IP center is the operational structure:

- force proof, do not trust narration
- measure token-local instability under challenge
- calibrate per model / version
- fuse lexical and entropy signals

## Boundary

Not every list or threshold here is itself the invention. The real value is the observation structure and the calibration logic that turns epistemic leakage into an actionable gate.

