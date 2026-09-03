# ML Bench

A workbench for testing machine learning algorithms on your own tables — in the browser, with no server, no account, and no upload.

Load a CSV, pick a target, fit nine algorithms, compare the runs, and export a PDF report that includes the caveats as well as the numbers.

**Live:** https://ml-bench.vercel.app · **About the project:** [`/about`](https://ml-bench.vercel.app/about)

> Replace the two URLs above once Vercel gives you the real domain, and replace `carol-mello` throughout this repo with your GitHub username.

---

## What it does

| Step | What happens |
| --- | --- |
| **01 Data** | Drop in CSV, TSV or JSON. Column types, ranges, missing counts, correlations and distributions are profiled on arrival. Multiple tables can be loaded at once. |
| **02 Model setup** | Choose target, features, task, split strategy, preprocessing and algorithm. Hyperparameters are exposed, with a runtime estimate before you commit. |
| **03 Scenarios** | Every run is kept and ranked, with the train−test gap and optional k-fold cross-validation alongside each score. |
| **04 Analytics** | Predicted-vs-actual, residual plots, confusion matrices, ROC curves, permutation importance, coefficients, tree rules, cluster profiles and PCA projections. |
| **05 Projections** | What-if single cases, one-variable partial-dependence sweeps, and batch scoring of a second table with CSV export. |
| **06 Report** | A multi-page PDF: data profile, every scenario with configuration and metrics, charts, findings, and a limitations appendix. |

### Algorithms

All implemented from scratch in the page — there is no ML library.

- **Regression** — ordinary least squares, ridge (L2), k-nearest neighbours, CART, random forest, gradient boosting
- **Classification** — logistic regression (one-vs-rest), Gaussian naive Bayes, k-nearest neighbours, CART, random forest
- **Clustering** — k-means (k-means++ seeding, silhouette, elbow curve), with PCA for the 2-D view

### Correctness choices worth knowing about

- Imputation, standardisation and encoding statistics are fitted on the **training rows only**, then applied unchanged to the held-out rows.
- The target column is force-removed from the feature set, so it cannot leak in through a stale configuration.
- The chronological split sorts by the table's date column when one exists, rather than trusting row order.
- Permutation importance is model-agnostic and measured on up to 400 held-out rows.
- Automatic warnings fire for overfitting, suspiciously perfect scores, class imbalance, tiny samples, unscaled distance-based models, and random splits on dated data.

---

## Running it locally

There is no build step and no dependency install. Either:

```bash
# just open it
open index.html          # macOS
xdg-open index.html      # Linux
start index.html         # Windows
```

or serve it, which is closer to production:

```bash
npx serve .
# then visit http://localhost:3000
```

An internet connection is needed on first load: Chart.js and jsPDF are pulled from cdnjs. Everything else is inside the file. If those requests are blocked, the app still runs — charts render blank and the report falls back to the browser's own Print → Save as PDF.

---

## Deploying to Vercel

Short version:

1. Push this folder to a GitHub repository.
2. In Vercel, **Add New → Project → Import** that repository.
3. Framework preset **Other**, no build command, output directory `.`.
4. Deploy.

Step-by-step commands, custom domains and redeploys: **[docs/DEPLOY.md](docs/DEPLOY.md)**.

---

## Repository layout

```
.
├── index.html            the application — one self-contained file
├── about.html            about page: the why, the how, and the limits
├── vercel.json           static hosting config: clean URLs, security headers
├── README.md
├── LICENSE
├── .gitignore
└── docs/
    ├── DEPLOY.md         GitHub → Vercel, step by step
    └── ARCHITECTURE.md   code map, data flow, how to add an algorithm
```

`index.html` is deliberately a single file, including its CSS and JavaScript. It can be emailed, dropped on a USB stick, or opened offline, and it is the reason the whole app is auditable in one read. `about.html` carries its own copy of the shared design tokens for the same reason. If the project grows past two pages, that is the point to split out a stylesheet — see ARCHITECTURE.md.

---

## Limitations

Stated here rather than discovered later:

- **Not a forecaster.** Point predictions, not time-series forecasts. No seasonality, no prediction intervals.
- **Not causal.** Feature importance reports what the model used, not what drives the outcome.
- **No deep learning.** Tabular data and classical algorithms only.
- **Single-threaded.** Everything runs on the page's main thread; large forests will block the tab. The setup screen estimates the cost first.
- **No hyperparameter search.** Comparisons are between the configurations you ran, not between algorithms at their best.
- **Noise is real.** On a few hundred rows, differences under roughly 0.02 in R² or accuracy are usually seed noise. Cross-validation is the number to quote.

---

## Contributing

Issues and pull requests are welcome. Useful contributions, roughly in order of value: a bug in the maths, a misleading label, a missing warning, a slow path, an algorithm worth adding.

If you are adding an algorithm, ARCHITECTURE.md has the four places to touch.

## Built by

**Carol Mello**, data engineer. The technical foundation comes from data engineering work; the subject comes from years of building data analytics visualisations. More on the [about page](about.html).

## Licence

MIT — see [LICENSE](LICENSE).

Third-party libraries loaded at runtime: [Chart.js](https://www.chartjs.org/) (MIT) and [jsPDF](https://github.com/parallax/jsPDF) (MIT).
