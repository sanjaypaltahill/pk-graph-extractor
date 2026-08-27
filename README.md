# PK Graph Extracter

Recover the raw numbers behind a published **concentration-time plot**.

Pharmacokinetic papers almost always show plasma concentration curves as a figure and almost
never publish the underlying table. This repo is a small Jupyter notebook that lets you click
the observed data points off such a figure and writes them out as a tidy CSV — so the data can
be re-plotted, digitised into a PK model, compared across studies, or pooled for meta-analysis.

Extraction is **manual by design**: you tell it where the axis ticks are and where each marker
sits. Nothing is inferred from pixel colours, so it works on noisy scans, overlapping curves,
error bars and hand-drawn axes alike — and the accuracy is exactly as good as your clicks.

## Install

```bash
pip install numpy pandas matplotlib pillow ipympl jupyterlab
```

`ipympl` is what makes the matplotlib figure clickable inside Jupyter. If clicks don't
register, replace `%matplotlib widget` at the top of the notebook with `%matplotlib qt` or
`%matplotlib tk` to get a separate plot window, then restart the kernel.

## Use it

```bash
jupyter lab pk_figure_to_csv.ipynb
```

Then work down the notebook:

**1. Configure.** Drop your figure into `images/` and set the config cell:

```python
IMAGE_PATH = "images/paclitaxel_example.webp"
X_LOG = False     # is the time axis logarithmic?
Y_LOG = True      # is the concentration axis logarithmic?
```

The output path is derived from the image name, so every figure gets exactly one CSV of the
same name in `extracted_data/`.

**2. Calibrate the axes.** Give two known tick values per axis, then click those four ticks on
the figure in order — x-low, x-high, y-low, y-high. The title prompts you for each click and
turns green when calibration is complete. Pick ticks that are far apart; on a log axis both
values must be greater than zero.

```python
extractor.calibrate(x_refs=(0, 50), y_refs=(1, 10_000))
```

**3. Click the data points.** One curve at a time:

```python
extractor.start_trajectory("60 mg/m2")   # left-click each marker, right-click to undo
extractor.finish_trajectory()            # commit the curve
```

Previously captured curves are redrawn as faint grey rings, so you can always see what's left
to do. Name each trajectory the way the figure legend does — that string becomes the
`trajectory` column. Repeat the pair of cells for every curve in the figure.

**4. Save.** `extractor.save_csv(OUTPUT_CSV)` writes one row per observed point:

```csv
trajectory,x,y
60 mg/m2,4.868154158215014,15.459277364194785
60 mg/m2,5.070993914807301,53.66976945540476
```

**5. Verify.** The last cell reloads the CSV from disk and re-plots it. Compare it against the
original figure — if the shape doesn't match, the cause is nearly always a mis-clicked
calibration point or a wrong `X_LOG` / `Y_LOG` flag. Fix and redo from step 2.

## Layout

```
pk_figure_to_csv.ipynb              the extractor + the 5-step workflow
images/                             source figures
  paclitaxel_example.webp
  cisplatin_example.png
extracted_data/                     one CSV per image, same base name
  paclitaxel_example.csv
  cisplatin_example.csv
```

## Example figures

Two worked examples ship with the repo, one for each axis-scale combination, each with its
extracted CSV alongside:

| Image | x-axis | y-axis | Curves | CSV |
|---|---|---|---|---|
| `images/paclitaxel_example.webp` | linear, 0–50 hr | **log**, 1–10,000 µg/L | one per dose level | `extracted_data/paclitaxel_example.csv` |
| `images/cisplatin_example.png` | **log**, 0.25–32 h | linear, 0.0–1.0 µg/mL | SC, HITHOC, HIPEC | `extracted_data/cisplatin_example.csv` |

`paclitaxel_example.csv` is a partial extraction — the 60, 180 and 360 mg/m² arms only, since
the figure has seven overlapping dose levels. `cisplatin_example.csv` covers all three arms at
all nine timepoints (0.5–24 h).

One caveat worth reading before you trust `cisplatin_example.csv`: in the original figure the
three arms converge so tightly after 10 h that their markers overlap into a single black block.
At 16 h and 24 h they cannot be told apart at the source image's resolution, so all three
series carry the same value there. This is a property of the figure, not a bug — it is the kind
of judgement call manual extraction forces you to make and document.

## Notes and limits

- **Accuracy** is bounded by click precision and image resolution. Zoom in on the figure before
  clicking; a couple of pixels on a log axis can be a meaningful fraction of a decade.
- **Only marker centres are captured.** Error bars, shaded CIs and fitted lines are ignored —
  extract them as separate trajectories (e.g. `"SC upper"`, `"SC lower"`) if you need them.
- **Points are sorted by x** when a trajectory is committed, so click order doesn't matter.
- **Calibration is per-figure.** Re-running the config cell creates a fresh extractor and
  discards previously captured trajectories, so save the CSV before switching images.
