# sib29-gate

**Behavioral Uncertainty Signatures (BUS)** — a hallucination detection gate for AI-generated code review findings.

## What This Is

When an LLM cannot prove a finding, it leaks epistemic uncertainty through specific word choices. This repository contains the raw data and verification script for the paper documenting that discovery.

**Key finding:** Hedge words in false positive findings carry 23–32% lower token-level confidence than baseline. Real bug claims are stated at 85–100% confidence. The gate uses this signal to reject hallucinated findings.

## Verify the Paper's Claims

```bash
git clone https://github.com/blazingRadar/sib29-gate.git
cd sib29-gate
python3 verify_claims.py
```

This script checks every quantitative claim in the paper against the raw JSON data. No API keys needed. No external dependencies. Just `python3` and the data files in this repo.

## What's Here

```
verify_claims.py                           # Reproduces every number in the paper
behavioral_uncertainty_signatures.md       # The paper (v2)

data/
├── prove_it_logprobs/                     # 11 "Prove It" responses with full per-token logprobs
│   └── grok_prove_0.json ... grok_prove_10.json
├── token_fingerprinting/                  # 6 fingerprinting runs with logprobs
│   └── fingerprint_grok_run1.json ... run6.json
├── replication_bus/                        # 20 API responses (10 findings × 2 models)
│   ├── grok3_*_raw.json                   # Grok-3 responses with logprobs
│   ├── gpt4o_*_raw.json                   # GPT-4o responses with logprobs
│   └── *_analysis.json                    # Computed scores and gate decisions
└── validation_blind/                      # 8 FastAPI OAuth2 prove-it responses
    └── fastapi_oauth2_f*_raw.json         # Raw responses with logprobs
```

## The Smoking Gun

Open `data/prove_it_logprobs/grok_prove_8.json`, go to token index [1254]:

```python
import json, math
data = json.load(open('data/prove_it_logprobs/grok_prove_8.json'))
token = data['choices'][0]['logprobs']['content'][1254]
print(f"Token: {token['token']}")          # " potentially"
print(f"Confidence: {100 * math.exp(token['logprob']):.1f}%")  # 11.6%
```

Now open `grok_prove_0.json`, token [1141]:

```python
data = json.load(open('data/prove_it_logprobs/grok_prove_0.json'))
token = data['choices'][0]['logprobs']['content'][1141]
print(f"Token: {token['token']}")          # " crash"
print(f"Confidence: {100 * math.exp(token['logprob']):.1f}%")  # 100.0%
```

Same model. Same prompt structure. Same codebase.
- False positive: **"potentially"** at **11.6%** confidence
- Real bug: **"crash"** at **100.0%** confidence

The model knows. It leaks through word choice.

## What's Not Here

The full SIB29 discovery pipeline, gate implementation, and multi-model rotation system are in [sib29](https://github.com/blazingRadar/sib29) (private, under active development). This repo contains only the published research data.

## Paper

Read the full paper: [behavioral_uncertainty_signatures.md](behavioral_uncertainty_signatures.md)

## Author

**Nick Cunningham**
[nickcunningham.io](https://nickcunningham.io) · [nick.lee.cunningham@gmail.com](mailto:nick.lee.cunningham@gmail.com)

## License

Research data and verification script released under MIT License.
Raw API responses are from xAI (Grok-3) and OpenAI (GPT-4o) APIs.
