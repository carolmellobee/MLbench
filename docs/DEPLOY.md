# Deploying ML Bench

The site is static HTML. There is nothing to build, so deployment is a file copy with a domain attached.

These instructions assume the GitHub account `carolmellobee` and a repository named `ml-bench`. If you name the repository something else, change it in the commands below and in the two repository links in `about.html`.

---

## 1. Put it on GitHub

Create an empty repository at <https://github.com/new>. Name it `ml-bench`. Do **not** tick "add a README", "add .gitignore" or "choose a license" — this folder already has all three, and pre-adding them creates a merge conflict on your first push.

Then, from inside this folder:

```bash
git init
git add .
git commit -m "ML Bench: browser-based ML algorithm workbench"
git branch -M main
git remote add origin https://github.com/carolmellobee/ml-bench.git
git push -u origin main
```

If GitHub asks for a password, it wants a **personal access token**, not your account password: Settings → Developer settings → Personal access tokens → Fine-grained tokens, with `Contents: Read and write` on this repository. Or install the [GitHub CLI](https://cli.github.com/) and let it handle auth:

```bash
gh auth login
gh repo create ml-bench --public --source=. --push
```

---

## 2. Connect it to Vercel

### Through the dashboard (recommended the first time)

1. Sign in at <https://vercel.com> with your GitHub account.
2. **Add New… → Project**.
3. Find `ml-bench` in the repository list and click **Import**. If it is not listed, click **Adjust GitHub App Permissions** and grant Vercel access to the repo.
4. Configure the project:
   - **Framework Preset:** `Other`
   - **Root Directory:** `./`
   - **Build Command:** leave empty (toggle the override off)
   - **Output Directory:** `.`
   - **Install Command:** leave empty
5. **Deploy.**

It finishes in a few seconds and gives you `https://ml-bench-<hash>.vercel.app`. Under **Settings → Domains** you can claim a cleaner name or attach your own domain — this project currently lives at `m-lbench.vercel.app`, which is the URL linked from the portfolio site.

### Through the CLI

```bash
npm i -g vercel
vercel login
vercel          # preview deployment, answers: Other / ./ / no build command
vercel --prod   # promote to production
```

---

## 3. What happens after that

- Every push to `main` redeploys production automatically.
- Every push to any other branch, and every pull request, gets its own preview URL — useful for checking a change to the maths before it goes live.
- Rollback is instant: **Deployments → the one that worked → Promote to Production**.

---

## What `vercel.json` is doing

| Setting | Effect |
| --- | --- |
| `cleanUrls: true` | `/about` serves `about.html`. Links in the pages still use `about.html`, which works both on Vercel and when you open the files directly from disk. |
| `trailingSlash: false` | `/about/` redirects to `/about`, so there is one canonical URL. |
| `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy` | Standard hardening headers. |
| `Content-Security-Policy` | Restricts what the page may load: scripts from this origin and cdnjs only, images from this origin plus `data:` and `blob:` (needed for chart images and the PDF download), and no framing of third-party content. |

The CSP includes `'unsafe-inline'` for scripts and styles. That is a real weakening, and it is there because the entire application is inline in `index.html` by design. If you ever want to drop it, the fix is to move the `<script>` and `<style>` blocks into `app.js` and `styles.css` and remove `'unsafe-inline'` — at the cost of the single-file property that makes the app portable and offline-openable.

---

## Vendoring the two libraries (optional)

If you would rather not depend on cdnjs at runtime — for offline use, for a stricter CSP, or because a corporate network blocks it:

```bash
mkdir -p vendor
curl -o vendor/chart.umd.min.js https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js
curl -o vendor/jspdf.umd.min.js https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js
```

Then in `index.html`, change the two `<script src="https://cdnjs...">` tags to `vendor/chart.umd.min.js` and `vendor/jspdf.umd.min.js`, and remove `https://cdnjs.cloudflare.com` from the `script-src` line in `vercel.json`. Both libraries are MIT licensed; keep their licence headers in the vendored files.

---

## Troubleshooting

**The deployment succeeds but shows a directory listing or a 404.**
Output Directory is not `.`, or the repository has the files inside a subfolder. Vercel serves the root of the output directory; `index.html` must sit directly in it.

**Charts are blank and the PDF button says the library did not load.**
cdnjs was unreachable. Check the browser console for a blocked request, then either allow `cdnjs.cloudflare.com` or vendor the libraries as above.

**`/about` 404s locally but works on Vercel.**
Expected. `cleanUrls` is a Vercel feature. Opening files from disk, use `about.html`, which is exactly what the links do.

**A change is pushed but the live site is unchanged.**
Check the Deployments tab for a failed build, then hard-reload (`Cmd/Ctrl + Shift + R`) — HTML is cached briefly at the edge.
