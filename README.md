# Audelle: AI-Powered Audio Editor

Audelle edits audio from natural-language prompts. The AI reads instruments, structure, mood, and speaker intent, then runs the edit. Manual mode is the fallback for direct waveform work.

## Features

### AI audio understanding (core ML)

The model automatically analyzes and catalogs audio when it loads.

- **Source separation**: vocals, drums, bass, guitar, keys, pads, FX.
- **Instrument classification**: for example, "Fender Strat, fingerpicked, with reverb".
- **Song structure**: intro, verse, chorus, bridge, outro with timestamps.
- **Mood/energy curve**: tension, energy, loudness over time.
- **Key, BPM, time signature**.
- **Speaker diarization**: who spoke when, for podcasts and interviews.
- **Transcription**: full word-level transcript with timestamps.

### Natural-language prompting

| Area | Example prompt |
|---|---|
| Trimming and arrangement | "Cut from the second chorus to the outro" |
| Instrument and stem editing | "Remove the kick drum from 2:30 to 3:45" |
| Inpainting and repair | "Remove that cymbal crash at 1:23 and fill smoothly" |
| Mood and style | "Add reverb to the vocals. Make it sound like a cathedral." |
| Podcast and dialogue | "Remove all ums, ahs, and pauses longer than 0.5s" |
| Short-form content | "Make a vertical 30-sec version for TikTok" |
| Film and video | "Separate dialogue, music, and SFX into stems" |

### Manual editor (dedicated page)

Manual mode gives fine control when a prompt is not precise. Open it from the sidebar at `/features/manual`. It loads your last upload automatically.

- Waveform with drag-to-select, draggable trim handles, and click-to-seek.
- Keep the selection (trim) or cut it out.
- Fade in and fade out with per-side durations (0 to 10s).
- Gain knob (-12 to +12 dB), normalize, speed (0.5 to 2x).
- Each edit chains on the last. Download and edit history are available.

### Export and integration

- MP3, WAV, FLAC, AAC. Stems as ZIP.
- Premiere Pro, DaVinci Resolve, and Final Cut XML.
- Share link and download.
- Batch processing.

## Project structure

```
audio-trim/
├── mobile/     # React Native (Expo) app
├── website/    # Next.js web app
├── server/     # FastAPI backend: routes, services, ml, utils, tests
├── ml/         # Standalone instruction-understanding project
├── docs/       # Reports, model cards, and checklists
├── paper/      # IEEE paper sources and PDF
└── scripts/    # Training, evaluation, and report scripts
```

## ML coursework (standalone)

`ml/` is a separate instruction-understanding project. It maps a natural-language edit instruction to an operation, target, and amount. It has its own dataset, training scripts, evaluation, and demo. It does not import or modify the app. Audelle integration comes later on a branch, after the ML submission. Start at `ml/README.md`.

## Work plan

### Phase 1: ML backbone (implemented in `server/ml/`)

- **Source separation**: Demucs / Hybrid Demucs isolates vocals, drums, bass, other.
- **Instrument classification**: feature-based detector with a heuristic `instrument_classifier.py`. You can swap it for a pretrained model (musicnn/YamNet).
- **Genre classification**: random forest trained on a small GTZAN subset. Train with `scripts/train_genre_classifier.py`. Artifact: `server/ml/models/genre_classifier.joblib`. It predicts genre and genre-specific suggested edits in `understand` responses.
- **Song structure**: `song_structure.py` finds intro/verse/chorus/bridge/outro with beat-synced chroma+MFCC recurrence.
- **Mood/energy curve**: `mood_curve.py` reports energy/tension/brightness/loudness over time and a summary mood.
- **Transcription**: Whisper (openai/whisper-tiny) gives a word-level transcript.
- **Speaker diarization**: `diarization.py` uses VAD + MFCC clustering. It is pyannote-ready and needs an HF token.
- **Prompting engine**: `prompt_llm.py` uses the LLM first, with a deterministic regex fallback (`server/ml/prompt_engine.py`).
- **Audio inpainting**: `inpainter.py` gives a crossfade fill for removed or replaced ranges (paint intent).

### Phase 2: Server API and pipeline (implemented in `server/services/`)

- **Async job queue**: `jobs.py` thread pool. `POST /api/jobs/{process,separate,transcribe,understand,diarize,inpaint}` and `GET /api/jobs/{id}`.
- **File upload and storage**: `storage.py` uses local storage by default and optional S3. Uploads return a storage key.
- **Audio backend**: pydub, librosa, ffmpeg (`audio_operations.py`, `converter.py`, `video_extractor.py`).
- **Export pipeline**: `exporter.py` writes stems ZIP, FCPXML (Final Cut), and CMX3600 EDL (Premiere/DaVinci). Download at `GET /api/export/download?path=`.

### Phase 3: Website (Next.js, implemented)

- Upload by drag and drop, file picker, or URL import.
- Prompt input: text box, suggested prompts, and voice input via the Web Speech API.
- Analysis display uses real `/api/ml/understand` data: instruments, song structure, mood.
- Preview player with waveform and playhead (`AudioPreview.tsx`, HTML5 audio + energy-curve canvas).
- Manual mode fallback with play, set start, set end, and trim (`ManualEditor.tsx`).
- Export and share: download the processed result or stems ZIP via `/api/export/zip`.

### Phase 4: Mobile app (planned)

React Native / Expo. The app uses the web flow, adapted for touch. It adds file picker and share sheet import, an offline queue that uploads when connected, local playback, prompt history, and completion push notifications.

### Phase 5: Platform (planned)

The planned platform work covers user accounts, project save/load, prompt favorites, and batch processing. It also covers an API for NLE plugins (Premiere, DaVinci, Final Cut), fine-tuning from user feedback, and a community prompt library.
