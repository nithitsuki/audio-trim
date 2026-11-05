# Labeling guide

One row per instruction. Labels describe **what the user asked for**, not what
the audio pipeline supports today.

## Operations

Pick every operation the instruction asks for, in the order it appears.

| Prompt | ops |
|--------|-----|
| "remove the vocals" | `["remove_stem"]` |
| "keep only the vocals" | `["isolate_stem"]` |
| "boost the bass 4 db" | `["gain"]` |
| "fade out over 3 seconds" | `["fade"]` |
| "remove the section from 0:02 to 0:04" | `["cut_segment"]` |
| "keep from 0:05 to 0:20" | `["trim_segment"]` |
| "remove speaker 2" | `["remove_speaker"]` |
| "clean up the hiss" | `["denoise"]` |
| "make the loudness even" | `["normalize"]` |
| "speed it up" | `["speed"]` |
| "add reverb to the vocals" | `["add_effect"]` |
| "remove the pauses" | `["remove_silence"]` |

Rules of thumb:

- **cut vs trim.** `cut_segment` deletes a region. `trim_segment` keeps a region.
  "cut from 0:05 to 0:20" deletes 5 to 20 s. "keep only from 0:05 to 0:20" keeps it.
- **remove stem vs remove region.** "remove the vocals" removes the stem.
  "remove the guitar solo" removes a region of the guitar.
- **remove_speaker is canonical.** "keep only speaker 1" is labeled
  `remove_speaker` with target `speaker_2`. The target is always the speaker
  that disappears, so the label is directly executable.
- **normalize vs gain.** `normalize` makes loudness even over time. `gain` applies
  a fixed dB change. "make it louder" is `gain`, not `normalize`.
- **denoise vs remove_silence.** `denoise` removes noise (hiss, hum, wind).
  `remove_silence` removes pauses and gaps.
- **add_effect** covers reverb, delay, distortion, filters, and mood words such
  as "darker", "cinematic", "bigger".

## Targets

Use the smallest set of targets the instruction names. If no stem or speaker is
named, use `full_mix`.

| Surface form | Target |
|--------------|--------|
| vocal(s), voice, singer, acapella, background vocals | `vocals` |
| drum(s), kick, snare, hi-hat, cymbal, percussion | `drums` |
| bass, bassline, sub, 808 | `bass` |
| guitar, acoustic, electric guitar | `guitar` |
| piano, keys, keyboard, organ, synth | `keys` |
| FX, pads, strings, brass, "everything else" | `other` |
| the whole song, track, or mix. No target named | `full_mix` |
| first speaker, host, interviewer | `speaker_1` |
| second speaker, guest, second voice | `speaker_2` |

For `remove_speaker`, the target is the speaker to remove (see above).

## Amount and unit

Only one amount per row, and only when the instruction states a number. Amount
regression is restricted to single-operation rows. Multi-operation rows get
`amount: null`.

| Operation | Unit | Notes |
|-----------|------|-------|
| `gain` | `dB` | "louder/down" with a number. The sign is negative for down. |
| `fade` | `seconds` | Fade duration |
| `speed` | `factor` or `bpm` | "twice as fast" = 2.0. "10% faster" = 1.1 |
| `remove_silence` | `seconds` | pause threshold ("pauses over 0.5 s") |
| `cut_segment` / `trim_segment` | `seconds` | Optional. Usually encoded in `span` |

Percent changes and LUFS targets are not regressed in v1. Leave `amount: null`
and explain in `notes`.

## Span

`span: [start, end]` in seconds. Either bound may be `null`.

- "from 0:05 to 0:20" → `[5.0, 20.0]`
- "at 1:23" → `[83.0, null]`
- "the outro" or "the chorus" → `null` (no timestamp)

The guide does not convert structural names (intro, chorus, bridge) to seconds
by hand. A later module does that.

## Multi-operation rows

"make the drums louder and remove the vocals" →
`ops: ["gain", "remove_stem"]`, `targets: ["drums", "vocals"]`.

Do not force a single label. Do not attach an amount to a multi-operation row.
The number has no clear owner.
