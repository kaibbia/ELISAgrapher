# ELISAgrapher

A single-file, offline web application for analyzing ELISA (and other dose–response)
data from 96-well plates. Paste raw absorbance values, annotate concentrations and
test articles, subtract background, fit four-parameter logistic (4PL) curves, and
read out EC50, maximum signal, and area under the curve (AUC) — then export
publication-ready figures and reload your work later.

Everything runs locally in your browser. There are no dependencies, no installation,
no server, and no data ever leaves your computer.

---

## Table of contents

- [Features](#features)
- [User guide](#user-guide)
- [Methods (technical detail)](#methods-technical-detail)
- [Interpreting results & limitations](#interpreting-results--limitations)
- [Exports and saving your work](#exports-and-saving-your-work)
- [Methods paragraph for publications](#methods-paragraph-for-publications)
- [For developers](#for-developers)
- [Privacy](#privacy)
- [License](#license)

---

## Features

- **Paste a full 8×12 plate** of raw absorbance values; non-numeric row/column labels are ignored automatically.
- **Flexible annotation**: concentration can run across columns or rows; test articles and replicates are defined by shared labels.
- **Serial-dilution generator** to fill concentrations from a top value, fold factor, and number of steps.
- **Background subtraction** from none, the zero-concentration wells, or hand-picked wells.
- **4PL curve fitting** by nonlinear least squares, with EC50, fitted maximum, Hill slope, R², and AUC reported per curve.
- **Fit the mean of replicates** (with SD error bars) **or every individual replicate point** — toggleable.
- **Multiple plates** in one session, with optional pooling of identically labeled curves across plates.
- **Per-curve color, symbol, and show/hide** controls.
- **Exports**: vector **SVG**, high-resolution **PNG** (2×/3×/4×), and a **JSON session** that restores raw data and all annotations.

---

## User guide

The interface is organized top to bottom as a workflow.

### Plates
Add, rename (double-click the tab name), delete, and switch between plates. A new plate
inherits the current plate's layout (orientation, concentrations, unit, labels, background
setting) as a template, so repeat runs only need a fresh paste. **Load example** and
**Load saved session** also live here.

### 1 · Raw data
Paste an 8×12 grid of absorbance values (rows A–H, columns 1–12). Tabs, spaces, or commas
all work, and any leading text such as row letters is ignored — only numbers are read.
Click **Parse plate**.

### 2 · Concentrations & orientation
Choose whether concentration varies **across columns** (each row is a series) or
**across rows** (each column is a series), set an optional **unit** (e.g. µg/mL), and enter
the concentration for each column/row. Use **0** for blank/background wells. The
serial-dilution helper fills these for you from a top value, fold, and step count.

### 3 · Plate & background
Choose the background source: **none**, the **zero-concentration wells**, or **custom**
wells you click directly on the plate grid. The mean background value is shown and is
subtracted from every well.

### 4 · Test articles & replicates
Label each row (or column, depending on orientation). **Rows/columns sharing the same label
are treated as replicates of one test article.** Leave a label blank to exclude that
row/column from analysis.

### 5 · Fit & analyze
Choose whether to fit the **mean of replicates** (shown with ± SD error bars) or **all
individual points**, optionally **combine identical labels across plates**, then click
**Analyze**. Below the button, each resulting curve gets a **visibility checkbox**, a
**color picker**, and a **symbol selector**; changes apply to the graph immediately.

### 6 · Results
A table lists EC50, fitted max, observed max, bottom, Hill slope, AUC, and R² for every
curve (hidden curves remain in the table, marked). Buttons export the graph as SVG or PNG
and save the session as JSON.

---

## Methods (technical detail)

This section describes exactly what the current build computes.

### Data preprocessing

1. **Parsing.** Each pasted line is scanned for numeric tokens (including decimals and
   scientific notation); the first 12 numbers per line become a row. Up to 8 rows are used.
2. **Background subtraction.** The background value is the arithmetic mean of the selected
   background wells (none → 0; zero-concentration wells; or custom-selected wells). This
   value is subtracted from **every** well **before** replicates are averaged and before
   fitting.
3. **Replicate aggregation.** For each test article and each concentration, the
   background-subtracted replicate values are collected. The per-concentration **mean** and
   **sample standard deviation** (n − 1 denominator) are computed. Error bars in "mean" mode
   are ± 1 SD.
4. **Zero-concentration handling.** Points at concentration = 0 cannot be placed on a
   logarithmic axis. They are used only as a background reference and are **excluded from the
   curve fit, the log x-axis, and the AUC**.

### The 4PL model

Each curve is fit with the four-parameter logistic function:

```
y = D + (A − D) / (1 + (x / C)^B)
```

where `x` is concentration and:

- **A** — response asymptote approached at low concentration (reported as **Bottom**)
- **D** — response asymptote approached at high concentration (reported as **Max (fit top)**)
- **C** — the **EC50**: the concentration at the curve's midpoint, i.e. the inflection
  point halfway (in response) between A and D
- **B** — the **Hill slope**, controlling steepness. Its sign is unconstrained, so the same
  model fits both increasing (agonist) and decreasing (inhibition) curves.

Because EC50 is defined relative to the **fitted** A and D, it is a **relative EC50**, not an
absolute one.

### Fitting algorithm

- **Objective.** Ordinary (unweighted) nonlinear least squares — the sum of squared vertical
  residuals between observed `y` and the model is minimized. No variance weighting (e.g.
  1/Y²) is applied.
- **Points used.** In "mean" mode the fit targets the per-concentration means; in
  "individual" mode it targets every replicate value. Only concentrations > 0 are included.
  A fit requires at least four points.
- **Optimizer.** The Levenberg–Marquardt algorithm with Marquardt diagonal scaling (the
  normal-equations diagonal is multiplied by `1 + λ`). The normal equations are solved by
  Gaussian elimination with partial pivoting.
- **Jacobian.** Computed numerically by forward finite differences (step
  `h = max(1e-9, |pⱼ| × 1e-6)` per parameter), not from analytical derivatives.
- **Initial estimates.** `A` = response at the lowest concentration; `D` = response at the
  highest concentration; `C` = the geometric mean of the fitted concentrations; `B` = 1.
- **Constraints.** `C` (EC50) is kept strictly positive; any step that would make it
  non-positive is rejected. `A`, `D`, and `B` are unconstrained.
- **Iteration & stopping.** Up to 300 outer iterations. Within each, λ is adjusted (up to 14
  attempts) until a step reduces the residual sum of squares; on success λ is decreased and
  the step accepted, on repeated failure λ is increased. Iteration stops when no step reduces
  the residual sum of squares.

Note: there is no multi-start procedure and no analytical covariance estimation, so the
result is the local optimum reachable from the initial estimates, and **no parameter
confidence intervals are reported**.

### Reported quantities

- **EC50** = fitted parameter `C`, in the concentration unit you entered.
- **Max (fit top)** = fitted parameter `D`.
- **Observed max** = the largest background-subtracted replicate **mean** across
  concentrations (independent of the fit).
- **Bottom** = fitted parameter `A`.
- **Hill slope** = fitted parameter `B`.
- **R²** = the coefficient of determination, `1 − SSE/SST`, where SSE is the residual sum of
  squares of the fit and SST is the total sum of squares about the mean of the fitted `y`
  values. It is computed on the same points used for fitting and **can be negative** when the
  model fits worse than a horizontal line through the mean.
- **AUC (log10)** = area under the dose-response curve computed **directly from the
  background-subtracted replicate means** (not from the fitted curve) by the trapezoidal rule
  on the **log10(concentration)** axis, over concentrations > 0. Because it is data-driven,
  AUC remains meaningful even when the 4PL fit is poor. AUC values are only directly
  comparable between curves that share the same concentration range.

---

## Interpreting results & limitations

- **Non-responders produce meaningless EC50s.** A flat curve has no real midpoint; the
  optimizer may return an extreme EC50 and a negative or near-zero R². Treat any fit with a
  low/negative R² as "no reliable fit," and rely on observed max or AUC instead.
- **EC50 depends on a defined top plateau.** If the highest concentration did not reach
  saturation, the fitted `D` — and therefore the EC50 — is an extrapolation and should be
  interpreted with caution.
- **Unweighted fitting.** ELISA signal variance often grows with signal magnitude. Ordinary
  least squares weights high-signal points more heavily; if low-end precision is critical,
  interpret the lower asymptote and EC50 accordingly.
- **No outlier rejection or parameter bounds** (other than EC50 > 0). Review the plotted fit
  against the points rather than relying on R² alone.
- **AUC comparability.** Only compare AUC across curves measured over the same concentration
  range.

### Multiple plates

Each plate is annotated independently. When **Combine identical labels across plates** is
**off**, a label like `B` on two plates yields two independent curves (`B · Plate 1`,
`B · Plate 2`). When **on**, all wells sharing a label are pooled into one curve, matched by
concentration **value**, so plates need not share an identical column layout to pool
correctly.

---

## Exports and saving your work

- **SVG** — a scalable vector figure, ideal for journals and for editing in vector software.
- **PNG** — a raster image rendered from the chart at a selectable 2×/3×/4× scale (3× is
  well above 300 DPI for a typical figure panel). The on-screen pixel dimensions at the
  chosen scale are shown before export. Note: browsers do not write a DPI metadata tag into
  PNGs; the pixel dimensions carry the resolution.
- **JSON session** — saves raw plate data and every annotation, fit setting, color, symbol,
  and visibility state. Reloading restores the full session without re-annotating. Sessions
  are forward/backward compatible across builds; display-only settings (such as on-screen
  font size) are not stored.

Both SVG and PNG reflect exactly what is shown, including which curves are toggled on.

---

## Methods paragraph for publications

> Dose–response data from 96-well ELISA plates were analyzed with the ELISA 4PL Analyzer.
> Raw absorbance values were background-subtracted using the mean of the designated
> blank/zero-concentration wells. For each test article, a four-parameter logistic (4PL)
> model, y = D + (A − D)/(1 + (x/C)^B), was fit to the background-subtracted replicate
> [means / individual values] by unweighted nonlinear least-squares regression
> (Levenberg–Marquardt algorithm), where the half-maximal effective concentration (EC50)
> corresponds to parameter C. Goodness of fit was assessed by the coefficient of
> determination (R²). Area under the curve (AUC) was calculated by the trapezoidal rule on a
> log10-concentration scale from the background-subtracted replicate means. Zero-concentration
> wells were used solely for background subtraction and were excluded from curve fitting and
> AUC calculation.

Edit the bracketed choice to match whether you fit means or individual points, and add
instrument/plate details as needed.

---

## For developers

**Architecture.** One self-contained HTML file: inline CSS for the UI and a single vanilla
JavaScript block for all logic. No build step, no external libraries, no network calls.

**State model.** A top-level `session` object holds an array of `plates` (each with its own
raw text, parsed grid, orientation, concentrations, unit, row/column labels, and background
settings) plus session-wide fit mode, combine flag, per-curve styles, and hidden-curve map.
`lastResults` caches the most recent analysis for re-rendering on style/visibility changes.

**Key functions.**

- `parseActive()` — tokenizes pasted text into the numeric grid.
- `bgValueFor(plate)` — computes the mean background from the selected wells.
- `buildCurves()` — assembles per-curve, per-concentration value arrays across plates,
  applying background subtraction and the combine-labels option.
- `fit4PL(xs, ys)` — the Levenberg–Marquardt 4PL fitter (returns `{a, d, c, b, r2}`).
- `solve(A, b)` — Gaussian elimination with partial pivoting for the normal equations.
- `aucLog10(points)` — trapezoidal AUC on the log10 x-axis.
- `analyze()` — orchestrates building curves, fitting, AUC, and rendering.
- `renderChart()` — draws the SVG; its font size and geometry are set here, independently of
  the page UI CSS.
- `exportSVG()` / `exportPNG()` / `saveJSON()` / `loadJSON()` — output and persistence.

**Common modifications.**

- *Weighted fitting:* change the residual/objective in `fit4PL` to divide residuals by a
  weight (e.g. `1/y`), and apply the same weights in the Jacobian/normal-equations assembly.
- *Chart size or fonts:* adjust the geometry constants and `FS` at the top of `renderChart()`.
- *Confidence intervals:* derive the parameter covariance from the final `JᵀJ` and residual
  variance inside `fit4PL`.

---

## Privacy

All parsing, fitting, rendering, and exporting happen entirely in your browser. No data is
transmitted anywhere, and the app works fully offline.

---

## License

MIT License

Copyright (c) [2026] [Kai Bialucha]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
