# Instruction → Action (Audelle ML project)

Turns a natural-language audio-editing instruction into a structured action:

| Field | Meaning | Example |
|-------|---------|---------|
| `ops` | one or more operations | `["gain", "remove_stem"]` |
| `targets` | what the operation acts on | `["drums", "vocals"]` |
| `amount` / `unit` | how much | `6.0` / `"dB"` |
| `span` | start/end seconds when stated | `[5.0, 20.0]` |

> **This folder is intentionally standalone.** It does not import the Audelle app
> (`server/`, `website/`, `mobile/`), so a change here cannot break Audelle.
> Integration is a later, separate step.

## Why this project exists

Audelle already has a rule-based prompt parser, and that parser is the baseline.
The ML project learns two mappings: classification (which operations and targets)
and regression (how much: +6 dB, a 2-second fade, 1.5× speed). Pretrained audio
models (Demucs, Whisper, pyannote) stay the execution layer.

## Current status

| Piece | Status |
|-------|--------|
| Operation + target schema (12 ops, 9 targets) | done |
| Seed prompts + paraphrase expansion + grouped splits | done (`data/seed_prompts.jsonl`, `scripts/build_dataset.py`) |
| Rule-based + TF-IDF baselines | done (`src/instruction_ml/rules.py`, `scripts/train_baseline.py`) |
| Evaluation + error analysis | done (`scripts/evaluate.py`, `reports/`) |
| Small transformer (DistilBERT multi-task) | scaffolded, bert-tiny check only, not usable yet |
| LLM paraphrasing helper (optional) | scaffolded (`scripts/expand_with_llm.py`) |
| Inference demo | done (`scripts/predict.py`) |
| Audelle integration | out of scope for now |

Results from the last local run are in [`RESULTS.md`](RESULTS.md). The headline number comes from the held-out set.

## Quickstart (Windows / PowerShell)

```powershell
cd ml
python -m pip install -r requirements.txt
# 1. Build the dataset (seed + paraphrases, grouped train/val/test splits)
python scripts/build_dataset.py
# 2. Train the TF-IDF + logistic-regression baseline
python scripts/train_baseline.py
# 3. Evaluate baseline + rule-based parser on the unseen test split
python scripts/evaluate.py --split test
# 3b. The honest benchmark: independent, hand-written prompts
python scripts/evaluate.py --split heldout
# 4. Try a prompt
python scripts/predict.py "make the drums louder by 6 db and remove the vocals"
```

Pure Python + scikit-learn. No API key, no GPU, no network.

### Transformer (optional, needs PyTorch + internet once)

```powershell
# small and fast, good for a laptop or Colab CPU
python scripts/train_transformer.py --model-name prajjwal1/bert-tiny --epochs 8
# the real run (Colab / Kaggle / GPU host)
python scripts/train_transformer.py --model-name distilbert-base-uncased --epochs 4
```

Evaluate a checkpoint with `evaluate.py` and `--transformer-preds` pointing at
`models/transformer/test_predictions.json` or `heldout_predictions.json`.

## Dataset

`data/seed_prompts.jsonl` holds the hand-written, hand-labeled rows:

```json
{"id": "seed-0001", "text": "make the drums louder by 6 db",
 "ops": ["gain"], "targets": ["drums"], "amount": 6.0, "unit": "dB",
 "span": null, "notes": ""}
```

Labeling rules are in [`data/labeling_guide.md`](data/labeling_guide.md).
`scripts/build_dataset.py` expands each seed row into paraphrases. It writes
`data/dataset.jsonl` (seed + template variants), `data/dataset_stats.json`, and
the grouped splits `data/splits/{train,val,test}.jsonl` (70%, 15%, and 15% of
groups).

Splits are grouped by seed id, so no paraphrase of a test prompt appears in
training. `data/heldout_prompts.jsonl` is a separate, hand-written set of 80
prompts that never feeds training or threshold tuning. It is the headline check.

## Models

| Model | What it does |
|-------|--------------|
| `rules.py` | deterministic baseline, adapted from the product parser for comparison |
| `model.py` | TF-IDF (word + char n-grams) → one-vs-rest logistic regression for ops and targets, ridge regression for amounts |
| `train_transformer.py` | DistilBERT with four heads (ops BCE, targets BCE, unit CE, amount L1) |

## Metrics

Operation: subset accuracy, micro/macro F1, per-operation F1, and a confusion
matrix on single-operation prompts. Target: subset accuracy, micro/macro F1,
per-target F1. Amount: MAE per operation/unit on single-operation prompts. Whole
action: exact match of ops + targets. Reports go to `reports/` as JSON +
Markdown + an error CSV.

## Known limitations

- **No op→target pairing.** "make the drums louder and remove the vocals" stores
  two operations and two targets, not `gain(drums)` + `remove_stem(vocals)`. The
  flat schema is deliberate for v1. A per-clause version is the natural next step.
- **Amounts are single-operation only.** A number in a multi-operation prompt is
  ambiguous, so it stays null and is excluded from regression.
- **Span is rule-only.** `rules.py` extracts timestamps. `span` is reported but not scored.
- **The template test split is optimistic.** Use `--split heldout` for the honest comparison.

## Layout

```
ml/
  data/                 seed prompts, labeling guide, generated dataset + splits
  src/instruction_ml/   schema, data, augmentation, rules, model, metrics
  scripts/              build / train / evaluate / predict / transformer / LLM helper
  tests/                fast pytest checks for labels, splits, baseline
  reports/              generated metrics and error analysis
  models/               baseline joblib (committed), transformer checkpoints (ignored)
```

No audio processing and no Demucs/Whisper calls. No website or mobile integration.
