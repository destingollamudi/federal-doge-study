# Federal Workforce DOGE Study

This project analyzes U.S. federal workforce data to investigate how the Department of Government Efficiency (DOGE) restructuring of 2025–2026 affected federal employees. It examines whether specific agencies and employee profiles were disproportionately impacted, and whether pre-restructuring workforce characteristics can predict which groups experienced the highest separation rates.

Data is ingested from the U.S. Office of Personnel Management (OPM) and stored in a Cloudflare R2 bucket. Notebooks pull directly from the bucket — no local data files required.

---

## Research Question

> Did the 2025 federal workforce restructuring driven by DOGE disproportionately affect specific employee profiles and agencies, and can pre-restructuring workforce characteristics predict which agencies and employee groups experienced the highest separation rates during this period?

### Analysis Plan

**Separations (`separations.ipynb`)**
- Classify separations as voluntary (quits, retirements, transfers) vs involuntary (RIF, terminations)
- Track involuntary separation rate month-by-month to isolate the DOGE signal from seasonal effects (e.g. the September 2025 voluntary surge)
- Identify which agencies drove RIF activity — USAID, FDA, NIH, State Dept, CDC are primary targets
- Profile who was involuntarily separated by age, tenure, pay plan, and appointment type

**Employment (`employment.ipynb`)** *(in progress)*
- Use monthly workforce snapshots as the denominator to compute true separation *rates* per agency
- Establish pre-DOGE baseline workforce characteristics (size, age distribution, tenure mix) by agency
- Track net headcount change over time per agency

**Accessions (`accessions.ipynb`)** *(in progress)*
- Track new hiring trends alongside separations — is DOGE cutting hiring as well?
- Identify which appointment types (career vs excepted vs Schedule C) are being filled

**Combined Analysis (`doge_analysis.ipynb`)** *(planned)*
- Join separations to employment to compute agency-level separation rates
- Regression: do pre-DOGE workforce characteristics (% temporary, % young, % non-tenured) predict involuntary separation rate?
- Identify agencies experiencing net workforce collapse (high separations + low accessions)

### Key Findings So Far

- **September 2025 spike was voluntary, not DOGE** — 123k separations but 94% were quits/retirements; involuntary rate was only 4.9%
- **July 2025 was the DOGE peak** — 36% involuntary rate, 5,392 RIFs in a single month (vs ~1 in Jan–Feb 2025)
- **USAID is the clearest DOGE target** — 3,754 RIFs, while most other agencies show terminations of expired appointments rather than true RIFs
- **Young and non-tenured workers bear the brunt** — 20–29 year olds are 7–9 points overrepresented in involuntary separations; career civil servants (tenure group 1) are 51 points underrepresented
- **Defense Human Resources Activity is accelerating in 2026** — 1,200 involuntary separations in Jan–Feb 2026 vs 362 in all of 2025

---

## Structure

```
federal-doge-study/
├── scripts/
│   ├── data_loader.py       # R2 + local data loaders
│   └── opm_to_r2.py         # Pipeline: OPM website → Cloudflare R2
├── notebooks/
│   ├── info.ipynb           # Data dictionary — columns, types, code lookups
│   ├── separations.ipynb    # Separations analysis (voluntary vs involuntary, agency, profile)
│   ├── employment.ipynb     # Workforce snapshot analysis (in progress)
│   └── accessions.ipynb     # Hiring trend analysis (in progress)
├── requirements.txt
└── .env                     # Cloudflare R2 credentials (not committed)
```

---

## Data Source

Data is obtained from the U.S. Office of Personnel Management (OPM):

[https://data.opm.gov/explore-data/data/data-downloads](https://data.opm.gov/explore-data/data/data-downloads)

Datasets covered:

- **Employment** — full workforce snapshots (~1.5–1.7 GB/month, ~2M rows per file)
- **Accessions** — new hire events per month
- **Separations** — departure events per month, with separation category codes

Coverage: **January 2024 – February 2026** (2025 pipeline complete; 2 months of 2026).

---

## Setup

Create and activate a virtual environment:

**Mac / Linux**
```bash
python3 -m venv .venv
source .venv/bin/activate
```

**Windows**
```bash
python3 -m venv .venv
.venv\Scripts\activate
```

Install dependencies:
```bash
pip install -r requirements.txt
playwright install chromium
```

Create a `.env` file in the project root:
```
account_id        = <cloudflare account id>
access_key_id     = <r2 s3-compat access key>
secret_access_key = <r2 s3-compat secret key>
bucket_name       = opm-data
```

Contact a teammate for credentials.

---

## R2 Bucket Layout

Data lives in a Cloudflare R2 bucket named `opm-data`:

```
opm-data/
  accessions/
    2024/
      accessions_202401.txt
      accessions_202402.txt
      ...
  employment/
    2024/
      employment_202401.txt
      ...
  separations/
    2024/
      separations_202401.txt
      ...
```

Files are pipe-delimited (`.txt`), one file per dataset per month.

---

## Loading Data in a Notebook

```python
from scripts.data_loader import load_r2_data

# All separations (2024-2026)
df = load_r2_data("separations")

# Filter by year or month
df = load_r2_data("separations", year=2025)
df = load_r2_data("employment",  year=2024, month=6)
```

`load_r2_data` streams each matching file from R2 into memory and returns a single concatenated DataFrame. Columns `year`, `month`, and `data_type` are added automatically.

---

## Refreshing / Extending the Data Pipeline

`scripts/opm_to_r2.py` downloads data from the OPM website and uploads it to R2. The pipeline uses Playwright to automate the browser (the OPM site is Blazor Server — no direct download URLs exist).

**Scan what's available on OPM:**
```bash
python scripts/opm_to_r2.py scan --start 2024-01 --end 2026-02
```

**Run the pipeline:**

Mac (use `caffeinate` to prevent sleep):
```bash
caffeinate -i python scripts/opm_to_r2.py run --start 2024-01 --end 2026-02
```

Windows (prevent sleep by running this first, then reset after):
```powershell
powercfg /change standby-timeout-ac 0
python scripts/opm_to_r2.py run --start 2024-01 --end 2026-02
powercfg /change standby-timeout-ac 30
```

**Options:**
```
--types    employment,accessions,separations  (default: all three)
--start    YYYY-MM  start month (default: 2024-01)
--end      YYYY-MM  end month   (default: 2026-02)
--no-skip  re-upload files that already exist in R2
```

The pipeline resumes safely — already-uploaded files are skipped by default. Large employment files (~1.5 GB) upload via multipart with automatic retry on transient network errors.

---

## Dependencies

```
pandas
numpy
statsmodels
scikit-learn
jupyter
boto3
playwright
```

---

## Status

**Data pipeline**
- [x] Automated OPM → R2 pipeline (`opm_to_r2.py`)
- [x] Jan–Dec 2025 uploaded (all three types); 2 employment files failed and need a re-run
- [x] Jan–Feb 2026 uploaded

**Notebooks**
- [x] `info.ipynb` — data dictionary with code→descriptor lookup tables for all three file types
- [x] `separations.ipynb` — voluntary vs involuntary classification, monthly trend, agency breakdown, employee profile analysis
- [ ] `employment.ipynb` — workforce snapshot analysis (not started)
- [ ] `accessions.ipynb` — hiring trend analysis (not started)
- [ ] `doge_analysis.ipynb` — combined regression analysis (not started)
