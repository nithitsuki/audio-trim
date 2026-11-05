# Genre Classifier: Model Card (Audelle)

RandomForest over librosa features. Feeds genre-aware suggestions in `POST /api/ml/understand`.

- **Artifact:** `server/ml/models/genre_classifier.joblib` (~ few KB), trained by `scripts/train_genre_classifier.py --clips-per-genre 10` (GTZAN subset, 100 clips, 10 genres).
- **Features (75):** 20 MFCC mean+std (40), 12 chroma mean+std (24), spectral centroid/bandwidth/rolloff mean+std (6), ZCR mean+std (2), RMS mean+std (2), tempo (1). See `genre_classifier.py:_extract_features`.
- **Model:** `RandomForestClassifier(n_estimators=200, random_state=42, n_jobs=-1)` + `LabelEncoder`. Split 80/20 stratified, with 5-fold CV.

## Training and Evaluation
```bash
python scripts/train_genre_classifier.py --clips-per-genre 10
# optional: --clips-per-genre 30 for stronger baseline
python scripts/train_genre_classifier.py --keep-data
```

Reproducibility: `random_state=42` everywhere, pinned `librosa==0.11.0` and `numpy==2.0.0`. The joblib artifact stores `feature_size` and `classes`, so load fails loudly on mismatch.

```bash
python -m pytest server/tests/test_eval.py -v
curl http://localhost:8000/api/eval/metrics
curl http://localhost:8000/api/eval/prompt-bench
```

Prompt bench: 36 paraphrases across 15 intents. The regex engine scores **accuracy 1.0** (`server/ml/eval/prompt_bench.py`). Genre: the training script prints test accuracy and CV. The production guard `confidence < 0.3 → None` hides weak predictions in the UI.

## Limits and Intended Use
- The GTZAN subset is small, and genre is inherently fuzzy. Treat the label as a suggestion, not ground truth. The instrument and mood fallback is intentionally light. Set `USE_PRETRAINED=1` and `INSTRUMENT_MODEL=laion/clap-htsat-unfused` to enable the CLAP tagger (requires `transformers` + `torch`).
- **Use:** UI suggestions (`genre_tuning_actions`) and paper baseline. Not for standalone genre research without retraining on full GTZAN/FMA. **Versioning:** re-run training to bump the artifact, and commit a new `.joblib` with git LFS if over 50 MB (current is tiny). The metrics snapshot lives in `scripts/report_figs/results.json` for the IEEE report.
