# Streamlit Cloud deployment

## Quick start

1. Push this repo to GitHub. **Important:** `app.py`, `pipeline/`, and
   `extractors/` must all live at the **same level** at the root of the repo.
2. Sign in at https://share.streamlit.io and click **New app**.
3. Select your repo and branch; main file path = `app.py`.
4. Open **Advanced settings → Secrets** and paste:

   ```toml
   SEC_USER_AGENT = "Your Name your-email@your-domain.com"
   ANTHROPIC_API_KEY = "sk-ant-..."          # optional, enables audit + insights
   USE_PLAYWRIGHT = "1"                      # required for J.D. Power scraping
   ```

5. Deploy.

## Troubleshooting `ModuleNotFoundError: No module named 'pipeline'`

This is the most common deploy issue. The fix is structural: Streamlit
runs `app.py` from a specific working directory, and `pipeline/` and
`extractors/` need to be siblings of `app.py`.

**Check your GitHub repo at the URL `https://github.com/<you>/<repo>`.**
You should see the file tree with `app.py`, `pipeline/`, `extractors/`,
`run.py`, `requirements.txt`, etc. all at the top level — NOT nested
inside another folder.

Wrong (will fail with ModuleNotFoundError):
```
conedproject/
└── utility_pipeline/      ← extra nesting
    ├── app.py
    ├── pipeline/
    └── extractors/
```

Right:
```
conedproject/
├── app.py
├── pipeline/
│   ├── __init__.py
│   ├── orchestrator.py
│   └── ...
└── extractors/
    ├── __init__.py
    └── ...
```

**Two fixes if your repo is the wrong shape:**

A. **Easiest — set the main file path to point inside the subfolder.** In
   Streamlit Cloud, click *Manage app → Settings → Main file path* and
   change it from `app.py` to `utility_pipeline/app.py` (or whatever your
   actual subfolder is). Save and reboot. The `sys.path` bootstrap at the
   top of `app.py` makes imports work from any CWD.

B. **Cleaner — flatten the repo.** On your local machine, move everything
   inside `utility_pipeline/` up one level and commit the move:
   ```
   git mv utility_pipeline/* .
   git mv utility_pipeline/.env.example .  2>/dev/null || true
   git mv utility_pipeline/.gitignore .    2>/dev/null || true
   rmdir utility_pipeline
   git commit -am "Flatten project to repo root for Streamlit"
   git push
   ```

After either fix, Streamlit will auto-redeploy. If you still see the
import error, the updated `app.py` now prints a diagnostic page showing
exactly which directory it ran from and what files it found there, which
makes the rest of the issue obvious.

## What's required vs optional

| Variable             | Required? | Why                                                         |
|----------------------|-----------|-------------------------------------------------------------|
| `SEC_USER_AGENT`     | Required  | SEC blocks generic UAs; expect 403 on the very first call.  |
| `ANTHROPIC_API_KEY`  | Optional  | Enables the AI verification & insights tabs. The data pipeline runs fine without it. |
| `USE_PLAYWRIGHT`     | Optional  | Default "1". Set "0" if Playwright deps cause issues on your platform. J.D. Power then becomes unavailable. |


## Playwright on Streamlit Cloud

`packages.txt` in this repo includes the apt packages Playwright's
headless Chromium needs (libnss3, libcups2, etc.). On first deploy
Streamlit Cloud installs them automatically. After deploy, run
`playwright install chromium` once via the **Manage app → Reboot app**
mechanism, or include this snippet at the top of `app.py` for the first
boot:

```python
import subprocess, sys
try:
    from playwright.sync_api import sync_playwright
    with sync_playwright() as p:
        p.chromium.launch(headless=True).close()
except Exception:
    subprocess.run([sys.executable, "-m", "playwright", "install", "chromium"])
```

(Not included in the default app.py because it can slow cold starts. Add
it if your deployment can't run `playwright install` from a shell.)

## Deploying without Playwright

If your platform doesn't support headless browsers (e.g. some restricted
hosts), set `USE_PLAYWRIGHT = "0"`. The pipeline still runs end-to-end
using only `requests`. The cost: J.D. Power customer satisfaction
extraction will fail and return null for affected rows. Every other
metric still works (SEC, ProPublica, EPA eGRID, CSR PDF download).

## Memory considerations

EPA eGRID's full plant-level workbook is ~10MB. The pipeline caches it
to `.cache/` for 7 days so repeat runs don't re-download. On Streamlit
Cloud the `.cache/` directory persists across reboots.

## Custom company list

To benchmark a utility not in the default registry, add an entry to
`extractors/company_registry.py`. You'll need:

- The SEC CIK (search at https://www.sec.gov/cgi-bin/browse-edgar)
- The corporate foundation EIN (search at
  https://projects.propublica.org/nonprofits)
- For integrated generators: eGRID operator names (lift them from the
  latest eGRID workbook PLNT sheet)
- The CSR / sustainability report URL (add to `CSR_URL_HINTS` in
  `extractors/csr_report.py`)

Pull request these additions back upstream — they're useful to anyone
benchmarking the same set of utilities.
