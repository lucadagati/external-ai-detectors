# external — third-party AI-text detectors, kept for comparison

This repository collects **external (third-party) AI-text detectors** that we vendor next to our own
detector so that we can run them on the same benchmark and compare them fairly.

At the moment it contains one project:

| Folder | Upstream | License | Upstream commit |
|---|---|---|---|
| [`lynote-ai-text-detector/`](lynote-ai-text-detector/) | <https://github.com/lynote-ai/ai-text-detector> | MIT | `f7db1e4` |

> **Attribution.** All code under `lynote-ai-text-detector/` was written by the upstream authors
> ("Open Source Contributors") and is included **unmodified** under the MIT License
> ([`lynote-ai-text-detector/LICENSE`](lynote-ai-text-detector/LICENSE)). Only the nested `.git` directory and
> Python bytecode caches were dropped. Credit, issues and pull requests for that code belong upstream.
> The top-level `README.md` (this file) is the only original content here.

## Why this repo exists

Our own detector ([`lucadagati/ai-text-detector`](https://github.com/lucadagati/ai-text-detector)) estimates
whether academic text (plain, Markdown or LaTeX) was written by an LLM, paragraph by paragraph and sentence by
sentence. To claim that it is any good we need baselines. Open detectors are easy to *cite* but harder to
*reproduce*: versions drift, weights disappear, defaults change. Pinning the exact upstream snapshot we evaluated
in one place makes the comparison repeatable.

In our benchmark (human arXiv paragraphs from 2016–19 versus Claude and open-weight LLM paragraphs), the
lightweight analyzer in this folder scored **AUROC 0.563 and caught 4% of AI text at 5% false positives**, i.e.
close to random on academic prose. That is not a surprise given what the tool is (see below) and is not a
criticism of it: it is designed for cautious triage, not for stand-alone classification, and its own HC3 report
says the same. The comparison script lives in our detector repo (`bench/eval_lynote.py`).

## What is inside `lynote-ai-text-detector/`

The upstream project has **two tiers**.

### 1. Lightweight local analyzer (`src/aidetect/`, `scripts/`, `tests/`)

An explainable, no-network, rule-based risk estimator, packaged as a CLI, a Python API and an agent skill.

- **Input:** plain text, via file or stdin. Texts under ~80 words are refused with `insufficient_text`.
- **Method:** a set of weighted stylistic signals (no trained model, no weights, no network access).
- **Output:** `score` (0–100), `verdict` (`insufficient_text`, `low_ai_likelihood`, `mixed_or_uncertain`,
  `high_ai_likelihood`), `confidence` (`low`/`medium`), the strongest `signals`, `caveats` and `next_steps`.
- **Philosophy:** conservative wording ("AI-like signals are present", never "this was written by AI").

```bash
cd lynote-ai-text-detector
pip install -e .
ai-detect examples/sample_ai_like.txt --json
cat essay.txt | ai-detect --json
```

```python
from aidetect import analyze_text

result = analyze_text(open("essay.txt", encoding="utf-8").read())
print(result.score, result.confidence, result.verdict)
for signal in result.strongest_signals():
    print(signal.name, signal.note)
```

Dev commands (from `lynote-ai-text-detector/`): `make test`, `make demo`, `make benchmark`, `make eval-hc3`.
Upstream reports its own HC3 evaluation in [`docs/HC3_EVALUATION.md`](lynote-ai-text-detector/docs/HC3_EVALUATION.md)
and a synthetic benchmark in [`docs/BENCHMARK.md`](lynote-ai-text-detector/docs/BENCHMARK.md).

### 2. Production two-level model pipeline (`detection-model-pipeline/`)

The training pipeline behind upstream's hosted service: one joint model that labels a **document**
(`human / ai / mixed`) and every **sentence** (`human / ai / paraphrased`) in a single forward pass, trained with
`L = L_doc + α·L_sent`.

```
detection-log CSV (you supply) → prepare → train → calibrate → eval → serve (FastAPI)
```

| Stage | Script | What it does |
|---|---|---|
| i | `code/i_prepare_hier_dataset.py` | deduplicate, build soft labels, deterministic hash-based split |
| j | `code/j_train_hier_model.py` | joint document + sentence training (XLM-R-large or ModernBERT-base) |
| k | `code/k_calibrate_hier_model.py` | temperature scaling (never changes the argmax) |
| l | `code/l_eval_hier_model.py` | two-level metrics |
| m | `code/m_serve_hier_api.py` | FastAPI service, `{code, msg, data}` envelope |

Important: upstream publishes **code only**. No training data, no model weights and no service logs are shipped,
and no accuracy claims are made. Only synthetic demo data (`data/demos/`, hash-pinned) is included, so this tier
cannot be used out of the box as a detector; you would have to train it on your own labelled data. Details are in
its own `README.md` and `docs/` (`ARCHITECTURE`, `MODEL`, `DATA_CONSTRUCTION`, `API`), also available in Chinese.

## Layout

```text
.
├── README.md                          # this file
└── lynote-ai-text-detector/           # unmodified upstream snapshot (MIT)
    ├── src/aidetect/                  # lightweight analyzer (core.py, cli.py)
    ├── scripts/                       # detect, benchmark, HC3 evaluation, setup
    ├── tests/                         # tests of the lightweight analyzer
    ├── examples/                      # one AI-like and one human-like sample
    ├── docs/, references/, assets/    # reports, API reference, figures, report template
    ├── SKILL.md, AGENTS.md            # agent-skill packaging (Codex, Claude Code, ...)
    └── detection-model-pipeline/      # trained two-level model: code, tests, docs, demo data
```

## Reproducing the comparison

1. Clone this repo and install the lightweight analyzer: `cd lynote-ai-text-detector && pip install -e .`
2. Run it on any plain-text corpus of human and AI paragraphs (keep paragraphs ≥ 80 words, otherwise the
   analyzer abstains), e.g. `ai-detect paragraph.txt --json`.
3. Compute AUROC and TPR at a fixed false-positive rate from the `score` field. Our evaluation script is
   `bench/eval_lynote.py` in [`lucadagati/ai-text-detector`](https://github.com/lucadagati/ai-text-detector).

## Caveats

- AI-text detection is probabilistic. Never use a single score as proof of misconduct or authorship.
- The numbers quoted above describe the lightweight analyzer on our academic-prose benchmark, not the upstream
  production model, which we could not evaluate because no weights are published.
- This is a research snapshot; it is not maintained as a fork and will not track upstream automatically.

## Collaborators and contact

Maintained by [@lucadagati](https://github.com/lucadagati), with
[@fabiooraziomirto](https://github.com/fabiooraziomirto) as collaborator.

## License

Original content (this README): MIT. Vendored code keeps its upstream license, see
[`lynote-ai-text-detector/LICENSE`](lynote-ai-text-detector/LICENSE). Per-folder licenses take precedence.
