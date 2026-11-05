# Audelle: Machine Learning Brief (for evaluation)

Audelle is a supervised audio-understanding system (classification, regression, clustering) with a DSP
editing engine. Generative output is procedural (NumPy oscillators), not a trained generative model.

## 1. Which part is ML, and which classifier?

The understanding pipeline runs on every upload (`POST /api/ml/understand`,
`server/routes/ml.py:90`) and reads the file before any edit is offered.

| Head | Model and settings | Classes / target | File |
|------|--------------------|------------------|------|
| Noise-condition | `RandomForestClassifier(n_estimators=200, random_state=42, n_jobs=-1)`, 84-dim features, stratified 80/20 split, `classification_report` + confusion matrix | clean/hiss/hum/crowd/clipped/muffled | `server/ml/audio_understanding/content_classifier.py` |
| Quality | `RandomForestRegressor(n_estimators=150)` or `GradientBoostingRegressor` (chosen by held-out R²), same split, MAE (dB) + R² vs a mean-predictor baseline | effective SNR_dB → 0–100 score | same file |
| Groove/subgenre | `RandomForestClassifier(150, max_depth=12, min_samples_leaf=2)`, 8-dim rhythm features, stratified 75/25 split | 15 EDM feels | `server/ml/rhythm/analyzer.py:140` |
| Genre | `RandomForestClassifier(200)`, 75-dim features, 80/20 split + 5-fold CV | 10 GTZAN genres | `server/ml/audio_understanding/genre_classifier.py:196` |
| Speakers | `KMeans`, k picked by `silhouette_score` | SPK_0..N | `server/ml/diarization/diarization.py:60` |
| Structure | cosine-similarity recurrence | intro/verse/chorus/bridge/outro | `server/ml/audio_understanding/song_structure.py:94` |

RandomForest fits this data: small tabular sets, CPU-only, robust to correlated features,
interpretable (`feature_importances_`), ~1 MB joblibs, fixed seeds.

Pretrained deep nets, used as-is (not trained by us, CPU inference): Demucs `htdemucs` (separation), Whisper
`openai/whisper-tiny` (transcription), optional CLAP `laion/clap-htsat-unfused` (instrument tagging).

**Not ML (say this explicitly):** prompt parsing (`server/ml/prompt_engine.py:581`, regex), drum
and synth rendering (NumPy DSP), mix tips (thresholds). The prompt bench
(`server/ml/eval/prompt_bench.py`) evaluates the regex, it does not train it.

## 2. Datasets and features

| Model | Source | Size | Storage |
|-------|--------|------|---------|
| Content/noise | **Real:** GTZAN subset, HuggingFace `storylinez/gtzan-music-genre-dataset`, `--clips-per-genre` (default 6) × 10 genres, first 10 s each. **Synthetic:** 6 labeled degradations per clip in RAM, clean/hiss/hum/crowd each re-augmented at 5 sampled SNRs per clip (hiss 8–18 dB, hum 50+100+150 Hz at 5–15 dB, crowd/babble 1.5–6 dB, hard-clip, 1 kHz lowpass). Clip-grouped split, no leakage. Classifier **acc 0.76** (12 held-out clips → 264 samples, F1 clipped 1.00 / muffled 1.00 / hiss 0.92 / hum 0.78 / crowd 0.62 / clean 0.64). Regressor (clean/hiss/hum/crowd) **MAE 7.67 dB, R² 0.424** vs a mean-predictor baseline of R² −0.000. See §5. | ~1320 samples × 84 feats | Temp dir only, deleted after training |
| Groove | **Synthetic in RAM:** `render_loops()` 8 s loops, 15 feels × BPM/key/swing/humanize × polished/bare, ~164 clips. Optional real songs `--songs-dir data/songs/<label>/`. | ~164 × 8 feats | No dataset folder |
| Genre | **GTZAN subset** as above, 10 clips × 10 genres. | 100 × 75 feats | Temp dir only, deleted after |

Retrain commands (each prints accuracy or MAE to stdout):
```
python scripts/train_content_classifier.py [--clips-per-genre 6]
python scripts/train_groove_classifier.py [--songs-dir data/songs]
python scripts/train_genre_classifier.py [--clips-per-genre 10]
```

Artifacts: `server/ml/models/{content,groove,genre}_classifier.joblib` (~1 MB each), loaded
**lazily**, with a missing-artifact fallback in every caller, so the product never breaks.

* Content/noise features (84): MFCC20 mean/std (40), chroma mean/std (24),
  centroid/bandwidth/rolloff/ZCR/RMS mean/std (10), tempo (1), flatness mean/std (2), hum-band
  40–130 Hz ratio, hiss-band >6 kHz ratio, clip ratio, crest_dB (4), hum-peak prominence, noise-floor
  ratio, spectral-contrast mean (3). The last three are the targeted noise discriminators (mains
  lines, pause-filling babble, peak/valley structure).
* Groove features (8): tempo, fourfloor, halftime, swing, hats_density, sub_ratio, kick_rate,
  snare_rate. Deliberately **no MFCC**, so synth training transfers to real songs.
* Genre features (75): the same base minus the 6 noise features.

## 3. How ML drives the product

1. Upload calls `/api/ml/understand`, which returns `content {condition, confidence, snr_db,
   quality_score, suggested_action}`, `genre`, and `rhythm {groove}`.
2. The UI shows the audio type (*crowd-noisy, SNR 11 dB, quality 28/100*) and a one-click fix per
   condition (`CONDITION_ACTIONS`: hiss→denoise, clipped→normalize, muffled→brighten,
   crowd→isolate vocals).
3. `add drums` with no named style uses the groove classifier to match the song feel, and a
   named style always overrides it.
4. `GET /api/eval/metrics` exposes all three model cards (classes, feature_size, test_accuracy,
   MAE/R², artifact bytes). `GET /api/eval/content|groove|genre` serves one model each.

## 4. Generative vs discriminative

* **Discriminative, learned:** the 4 classifiers, 1 regressor, and KMeans. They decide what the
  audio is. This is the graded ML.
* **Procedural synthesis:** drums and synths render from NumPy oscillators on the detected beat
  grid. No diffusion or transformer weights are trained, so swapping timbres needs no retraining.
* **Optional LLM:** `USE_LLM=1` routes prompt parsing through an OpenAI-compatible endpoint. The
  offline regex engine is the default. The LLM never touches audio.

## 5. Viva Q&A

* *Overfitting or leakage?* Fixed seeds. **Clip-grouped splits** (`GroupShuffleSplit`, all 6
  degradations of one clip stay on one side, so musical content cannot leak). Confusion matrices in
  training logs. Groove forests capped at `max_depth=12`. Noise features are band-ratios, not
  memorized spectra. Groove synth shares **no code** with playback patterns.
* *Weak spots?* Clean↔crowd is the residual confusion. A second music clip mixed at 12 dB is
  numerically identical to clean (HF-band energy ratio 0.0150 for both, fully overlapping ranges).
  The old 3–12 dB crowd label overlapped "clean" at any accuracy, so clean recall sat at 0.08.
  Restricting crowd to 1.5–6 dB, where the condition is audible, raised clean recall to 0.55.
  Remaining gap: level-aware SNR. The regressor is now MAE 7.67 dB, R² 0.424 vs a mean-predictor
  baseline of R² −0.000, so it beats the constant predictor but explains little variance.
* *Why not deep learning?* Under 500 labeled clips per head. Forests beat CNNs there, run on CPU
  in CI, and stay explainable.
* *More time?* Calibrated probabilities (`CalibratedClassifierCV`), per-class precision/recall
  gates in the UI, and a learned section classifier to replace the cosine heuristic.

## 6. Safe ML extensions

Intent classifier (TF-IDF + LogReg on prompt bench) shadowing the regex, mood regressor
(valence/arousal) replacing mood thresholds, calibrated confidence gates per head. All three
follow the established pattern: script in `scripts/train_*`, joblib in
`server/ml/models/`, lazy loader + `None` fallback, card in `server/ml/eval/metrics.py`, endpoint
in `server/routes/eval.py`, tests in `server/tests/test_*.py`.
