# Architecture

`index.html` is one file: markup, styles, and roughly 2,000 lines of JavaScript in five parts. This document is the map.

## Why one file

The app has no build step, no bundler and no framework. That is a deliberate trade, and it buys three things: it opens from a USB stick with no internet, the entire program can be read top to bottom in one sitting, and nothing between the source and the running page can silently change behaviour. What it costs is the ability to use a strict Content Security Policy without `'unsafe-inline'`, and some editor ergonomics on a 150 KB file.

If a third page appears, split `styles.css` out first — `about.html` currently duplicates the design tokens, and two copies is the limit before they drift.

## The five parts

Search for the banner comments (`Part 1:` … `Part 5:`) to jump between them.

| Part | Responsibility | Key functions |
| --- | --- | --- |
| **1 — Data** | State, formatting helpers, seeded RNG, descriptive statistics, delimited-text parsing, type inference, dataset construction, the data view | `parseDelimited`, `finalizeDataset`, `renderProfile`, `histogram`, `S.*` |
| **2 — Algorithms** | Chart plumbing and every model, written from scratch | `solveLinear`, `fitLinear`, `fitLogistic`, `fitKNN`, `fitNB`, `growTree`, `fitForest`, `fitGBM`, `fitKMeans`, `pca2` |
| **3 — Pipeline** | Algorithm catalog, encoding, splitting, scaling, metrics, the run loop, automatic warnings | `ALGOS`, `buildFrame`, `makeSplit`, `materialize`, `trainModel`, `regMetrics`, `clsMetrics`, `permImportance`, `crossValidate`, `runScenario`, `auditRun` |
| **4 — Views** | The setup form, leaderboard, run strip, and the three analytics layouts | `syncModelForm`, `readConfig`, `doRun`, `renderRuns`, `renderAnalytics`, `renderRegression`, `renderClassification`, `renderClustering` |
| **5 — Output** | Projections, findings, the PDF builder, event wiring | `predictRows`, `drawPD`, `renderFindings`, `generatePDF`, `pdfCharts`, `init` |

## Data flow

```
file  ──▶ parseDelimited ──▶ finalizeDataset ──▶ State.datasets[]
                                                      │
                            readConfig() ─────────────┤
                                                      ▼
                                 runScenario(cfg) ──▶ buildFrame     encode features, drop the target,
                                       │                             build the class list
                                       ├──────────▶ makeSplit        random / stratified / chronological
                                       ├──────────▶ materialize      impute + scale, FITTED ON TRAIN ONLY
                                       ├──────────▶ trainModel       dispatch on cfg.algo
                                       ├──────────▶ regMetrics | clsMetrics
                                       ├──────────▶ permImportance
                                       ├──────────▶ crossValidate    optional
                                       └──────────▶ auditRun         the warnings
                                                      │
                                                      ▼
                                              State.runs[] ──▶ views, projections, PDF
```

Nothing is persisted. `State` lives in memory; closing the tab clears it. That is a privacy feature, not an oversight — no `localStorage`, no cookies, no network calls with user data.

## The parts that are easy to get wrong

**Preprocessing is fitted after the split, not before.** `materialize(frame, cfg, trIdx, teIdx)` computes imputation medians and standardisation statistics from `trIdx` only, then applies them to both matrices and returns a `transform` closure. That closure is what the projections and batch scoring reuse, so a what-if case goes through exactly the same pipeline as a training row.

**The target cannot be a feature.** `buildFrame` filters it out of `cfg.features` on every run, regardless of what the form or a restored configuration says.

**Encoded columns are grouped back to source columns.** One categorical feature becomes several one-hot columns; `frame.groups` maps each encoded column index to its original name so permutation importance shuffles the whole group together and reports one number per real column.

**Importance is capped.** It is measured on at most 400 held-out rows with 2–3 repeats, because the naive version costs `features × repeats` full prediction passes and made k-NN unusable.

**Runtime is estimated before it is spent.** `costEstimate(n, p, algo, hp)` drives the warning on the setup screen. The constants are calibrated by measurement, not theory; if you change a training loop's complexity, re-check them.

## Adding an algorithm

Four places, in order:

1. **Part 2** — write `fitYourModel(X, y, K, hp, task, seed)`. It must return an object with `predict(rows) → array`. For classifiers also return `predictProba(rows) → array of arrays`, which unlocks ROC, AUC and log loss for free.
2. **Part 3, `ALGOS`** — add an entry with `label`, `tasks`, a one-sentence `desc` that says when to reach for it and what it costs, and the `hp` specs. The form, the hyperparameter panel and the "run all algorithms" sweep are generated from this object.
3. **Part 3, `trainModel`** — add the `case` that maps the config to your fit function.
4. **Part 3, `costEstimate`** — add a rough cost so the heavy-run warning stays honest.

Nothing else needs touching: metrics, charts, importance, the leaderboard and the PDF are all model-agnostic.

## Testing

There is no test runner in the repository, but the model core is deliberately free of DOM access, so it can be exercised in Node by concatenating parts 1–3 and calling `runScenario` directly against a generated table — `demoDataset()` produces 600 rows with a known structure for exactly that purpose. Sanity checks worth keeping: a leakage case (target passed in as a feature should fall back to chance once the guard strips it), a chronological split, and a run with missing values and a tiny sample.

## Known trade-offs

- **Single-threaded.** Moving training into a Web Worker would keep the UI responsive on large tables; it would also mean serialising the frame across the boundary.
- **Brute-force k-NN.** Quadratic in rows. A k-d tree would fix it for low dimensions.
- **Gradient boosting is regression-only.** Binary logistic boosting is the obvious next addition.
- **Charts and PDF are external.** Chart.js and jsPDF are the only dependencies; both fail gracefully, and both can be vendored (see DEPLOY.md).
