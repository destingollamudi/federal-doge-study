# Federal Workforce DOGE Study

This project analyzes U.S. federal workforce data to investigate how the Department of Government Efficiency (DOGE) restructuring of 2025–2026 affected federal employees. It examines whether specific agencies and employee profiles were disproportionately impacted, and whether pre-restructuring workforce characteristics can predict which groups experienced the highest separation rates.

Data is ingested from the U.S. Office of Personnel Management (OPM) and stored in a Cloudflare R2 bucket. Notebooks pull directly from the bucket — no local data files required.

---

## Research Question

> Did the 2025 federal workforce restructuring driven by DOGE disproportionately affect specific employee profiles and agencies, and can pre-restructuring workforce characteristics predict which agencies and employee groups experienced the highest separation rates during this period?

### Analysis Plan

**Separations (`separations.ipynb`)**
- Classify separations as voluntary (quits, retirements, transfers) vs involuntary (RIF, terminations)
- Track involuntary separation rate month-by-month to isolate the DOGE signal from seasonal effects
- Identify which agencies drove RIF activity — USAID, FDA, NIH, State Dept, CDC are primary targets
- Profile who was involuntarily separated by age, tenure, pay plan, and appointment type

**Employment (`employment.ipynb`)**
- Use monthly workforce snapshots as the denominator to compute true separation rates per agency
- Establish pre-DOGE baseline workforce characteristics (size, age distribution, tenure mix) by agency
- Track net headcount change over time per agency

**Combined Analysis (`doge_analysis.ipynb`)**
- Join separations to employment to compute agency-level separation rates
- Regression: do pre-DOGE workforce characteristics (% temporary, % young, % non-tenured) predict involuntary separation rate?
- Identify agencies experiencing net workforce collapse (high separations + large headcount loss)

### Key Findings So Far

- **Federal workforce has shrunk ~285,000 workers** since Sep 2024 — from ~2.31M to ~2.03M by Feb 2026 (-12.3%)
- **September 2025 spike was voluntary, not DOGE** — 123k separations but 94% were quits/retirements; involuntary rate was only 4.9%
- **July 2025 was the DOGE peak** — 36% involuntary rate, 5,392 RIFs in a single month (vs ~1 in Jan–Feb 2025)
- **USAID was essentially gutted** — 94.7% headcount loss (4,831 → 256 people); 114% peak involuntary separation rate
- **Defense Human Resources Activity accelerating** — 118% peak involuntary rate, 1,200 involuntary seps in Jan–Feb 2026 vs 362 in all of 2025
- **Young and non-tenured workers bear the brunt** — 20–29 year olds dropped from 9% → 7.85% of the workforce; probationary/term workers were cut first
- **Career civil servants are surviving** — tenure group 1 (permanent) rose from 75% → 81.4% of the remaining workforce

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
│   ├── employment.ipynb     # Workforce snapshot analysis
│   └── doge_analysis.ipynb  # Combined regression analysis
├── pyproject.toml
├── requirements.txt
└── .env                     # Cloudflare R2 credentials (not committed)
```

---

## Data Source

Data is obtained from the U.S. Office of Personnel Management (OPM):

[https://data.opm.gov/explore-data/data/data-downloads](https://data.opm.gov/explore-data/data/data-downloads)

Datasets covered:

- **Employment** — full workforce snapshots (~1.5–1.7 GB/month, ~2M rows per file)
- **Separations** — departure events per month, with separation category codes

Coverage: **January 2024 – February 2026**

---

## Setup

### Using uv (recommended)

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create venv and install dependencies
uv sync

# Install Playwright browser
uv run playwright install chromium
```

### Using pip

```bash
python3 -m venv .venv
source .venv/bin/activate      # Mac/Linux
.venv\Scripts\activate         # Windows

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

```
opm-data/
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

# Aggregate on load to reduce memory (recommended for employment)
df = load_r2_data("employment", agg_cols=[
    "agency_subelement_code", "agency_subelement",
    "age_bracket", "tenure_code", "tenure",
    "appointment_type_code", "appointment_type",
    "pay_plan", "work_schedule", "education_level_bracket",
])
```

`load_r2_data` streams each matching file from R2, optionally aggregates by `agg_cols` to keep memory flat, and returns a single concatenated DataFrame. Columns `year`, `month`, and `data_type` are added automatically.

The employment notebook caches the aggregated result to `employment.pkl` — subsequent runs load from cache instantly.

---

## Refreshing / Extending the Data Pipeline

`scripts/opm_to_r2.py` downloads data from the OPM website and uploads it to R2.

**Scan what's available on OPM:**
```bash
python scripts/opm_to_r2.py scan --start 2024-01 --end 2026-02
```

**Run the pipeline (Mac):**
```bash
caffeinate -i python scripts/opm_to_r2.py run --start 2024-01 --end 2026-02
```

**Run the pipeline (Windows):**
```powershell
powercfg /change standby-timeout-ac 0
python scripts/opm_to_r2.py run --start 2024-01 --end 2026-02
powercfg /change standby-timeout-ac 30
```

**Options:**
```
--types    employment,separations  (default: all)
--start    YYYY-MM  start month
--end      YYYY-MM  end month
--no-skip  re-upload files that already exist in R2
```

---

## Dependencies

Managed via `pyproject.toml` (uv) or `requirements.txt` (pip):

- pandas, numpy, statsmodels, scikit-learn
- jupyter, boto3, pyarrow, playwright
