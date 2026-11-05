# Audelle paper

`audelle_ieee.tex` is an IEEE conference paper (IEEEtran) with 9 sections and 16
references.

## Building

Build with [Tectonic](https://tectonic-typesetting.github.io/). It fetches the
TeX packages it needs on first run:

```powershell
tectonic audelle_ieee.tex
```

Tectonic reruns TeX and `xdvipdfmx` itself, so one command is enough.

With TeX Live instead:

```powershell
# IEEEtran ships with most TeX Live installs
pdflatex -interaction=nonstopmode audelle_ieee.tex
bibtex audelle_ieee          # not used: thebibliography is hand-written
pdflatex -interaction=nonstopmode audelle_ieee.tex
pdflatex -interaction=nonstopmode audelle_ieee.tex
```

Use two passes minimum and three recommended, so that `\ref` resolves. Rebuild
`audelle_ieee.pdf` after any edit to the `.tex` file. Do not edit the PDF.

## Checking it without LaTeX

```powershell
python paper\check_paper.py
```

This script validates the structure without a PDF toolchain:

- braces balance, and every environment opens and closes exactly once.
- every `\cite{key}` resolves to a `\bibitem{key}`, and the text cites every
  `\bibitem`.
- **the bibliography is in first-citation order**, which IEEEtran uses for
  numbering. An out-of-order bibliography renders numbers that contradict the
  text.
- no stray hand-typed `[n]` remains, because LaTeX treats it as plain text and
  not as a citation.
- every `\ref` resolves, and the text references every `\label`.
- `Section~<roman>` cross-references point at sections that exist.

## Files

| File | Purpose |
|---|---|
| `audelle_ieee.tex` | The paper |
| `audelle_ieee.pdf` | Built PDF, 6 pages |
| `REFERENCES.md` | Every reference, what verified it, and what was rejected |
| `check_paper.py` | Structural validation |
| `fix_citations.py` | One-shot conversion of hand-typed `[n]` markers to `\cite{}` |

## Reproducing the numbers in the paper

```powershell
python scripts\evaluate_genre_classifier.py --clips-per-genre 10
python -m pytest server\tests -q
python server\ml\eval\prompt_bench.py
```

The genre evaluation writes `server/ml/models/genre_eval.json`.

## What the paper deliberately does not claim

- **Inpainting quality.** Duration preservation and absence of silence are
  measured. Recovery of the original *notes* is not claimed, because on
  through-composed material that audio no longer exists in the file.
- **Seam smoothness as an improvement.** A 50 ms crossfade can post a smaller
  single-sample step than the reconstructed splice. The tests assert a bound, not
  superiority.
- **Genre accuracy against the literature.** 100 clips is not a benchmark, so no
  comparison is made.
- **Any perceptual quality claim.** No listening test was conducted.
