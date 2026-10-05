# RMTI API

A small, stateless HTTP API for the **Risk Mechanism Theory Index (RMTI)**, covering both
applications in one service:

- **Disaster / urban risk** (`/v1/disaster/...`)
- **Oncology: HCC surveillance triage** (`/v1/oncology/...`)

It calls the **unmodified v2.1.0 reference modules** in `core/`, `disaster_climate/` and
`oncology/`. The API layer in `app/` contains no RMTI arithmetic, only validation, routing and
safety limits. Standard library only: no packages to install.

> **Educational and research use only.** Scores are relative, rubric-based indices, not forecasts,
> engineering assessments or actuarial figures. The oncology endpoints are not a medical device, are
> not clinically validated, and **must not be sent patient-identifiable information.**

## About this API

The **Risk Mechanism Theory Index (RMTI)** is a deterministic, rubric-based way to rank where risk-reduction effort
matters most. It rests on Risk Mechanism Design and Theory (Marolla, 2018, 2025). It does not forecast events. It
turns a handful of bounded inputs into a score, a fixed tier and a plain-language posture, so that two people
feeding in the same inputs always get the same answer. This API puts that calculation behind a simple web address,
so a registry, a spreadsheet, a dashboard or a research script can call it instead of re-implementing it.

### The calculation

Every input is bounded on a 0 to 1 scale (or a fixed 1 to 5 ladder for consequence):

| Symbol | Meaning |
|---|---|
| **L** | Likelihood of the hazard or event |
| **E** | Exposure: how much of the population or asset base is in the path of the hazard |
| **V** | Vulnerability: how easily exposure becomes harm |
| **ρ (rho)** | Strength of the risk-reducing mechanism already in place (`rho_now`), or planned (`rho_after`) |
| **C** | Consequence severity, from a fixed five-step scale (0.10, 0.30, 0.50, 0.75, 1.00) |

- **Inherent risk:** `IR = L × E × V`
- **Residual risk:** `R = IR × (1 − ρ)`, the score after existing or planned mitigation
- **Expected-annual-loss index:** `EAL = L × C × (1 − ρ)`, an ordinal severity index, not a monetary loss
- **Equity rate:** per 1,000,000 people (disaster) or per 1,000,000 at-risk patients (oncology)

Scores fall into five **pre-registered tiers** that are identical for every request: Low (below 0.05), Moderate
(below 0.15), High (below 0.35), Severe (below 0.60) and Critical (0.60 and above). Calling the API with
`rho_now` and `rho_after` shows how far a proposed intervention would move an asset or cohort, as
`risk_reduction_pct`.

### Two applications, one engine

**Disaster and urban risk** (`/v1/disaster/...`). Score a city or asset, rank up to 100 of them, or retrieve the
published eight-city cohort (Bamako, Lagos, Lilongwe, Nairobi, Dar es Salaam, Lusaka, Pacific Palisades,
Altadena). Exposure can be derived from population with the advisory log-normalised helper `E_log`.

**Oncology: hepatocellular carcinoma (HCC) surveillance triage** (`/v1/oncology/...`). Score a cohort or stratum
for surveillance prioritisation. Vulnerability can come from liver reserve (ALBI grade 1 to 3, or a distribution of
grades), consequence from the stage at which a missed tumour would be found (BCLC 0 through C-D), and ρ from the
strength of the surveillance programme. Posture text is worded for surveillance programmes (adherence and recall,
navigation, specialist hepatology follow-up).

Both applications call the same shared core, so a fix or validation update propagates to both.

### What the API does and does not do

- It performs the published arithmetic exactly, with decimal rounding that matches the manuscript tables and the
  Excel calculator.
- It validates every input and rejects unknown fields, so a typo cannot silently change a result.
- It stores nothing. Request bodies and query strings are never logged, and a test fails if one ever is.
- It is **not** a forecast, an engineering assessment, an actuarial figure or a diagnostic tool. The HCC
  application is a decision-support reading and has not been validated against prospective incident HCC.
- **Never send patient-identifiable information.** Send only cohort-level or stratum-level numbers.

### How to cite

Cite this API together with the RMTI reference implementation (concept DOI 10.5281/zenodo.21445892). A
machine-readable citation is in `CITATION.cff`.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/docs` | Interactive documentation (Swagger UI) |
| GET | `/health` | Liveness check |
| GET | `/v1/framework` | Fixed tiers, consequence scale C, anchors |
| GET | `/v1/exposure?N=1100000&domain=disaster` | Advisory log-normalised exposure `E_log` |
| POST | `/v1/disaster/assess` | Score one asset or city |
| POST | `/v1/disaster/rank` | Score and rank up to 100 assets |
| GET | `/v1/disaster/cohort` | Published eight-asset cohort, ranked |
| POST | `/v1/multi` | Multi-hazard / multi-category form (Eq. 3/5), not field-validated |
| POST | `/v1/oncology/assess` | Score one HCC cohort or stratum |
| POST | `/v1/oncology/rank` | Score and rank up to 100 strata |
| GET | `/v1/oncology/cohorts` | Published HCC cohorts and Pearl-protocol aetiologies |

### Example: Lilongwe

```bash
curl -X POST https://marolla-rmti.onrender.com/v1/disaster/assess \
  -H "Content-Type: application/json" \
  -d '{"N":1100000,"L":0.875,"E_sel":0.8,"V":0.8,"rho_now":0.25,"rho_after":0.48,"C":4}'
```
returns `R_now 0.42 (Tier 4 - Severe)`, `R_after 0.291 (Tier 3 - High)`, `risk_reduction_pct 30.7`,
`EAL_now 0.492`, `equity_now_per_1M 0.4474`, matching the paper's Tables 4 and 6.

### Example: oncology

```bash
curl -X POST https://marolla-rmti.onrender.com/v1/oncology/assess \
  -H "Content-Type: application/json" \
  -d '{"N":100000,"L":0.30,"E":0.65,"albi_grade":2,"rho_now":0.20,"rho_after":0.50,"bclc_stage":"BCLC_CD"}'
```
Vulnerability can come from `V`, `albi_grade` (1-3) or `albi_grade_counts` (e.g. `{"1":10,"2":18,"3":1}`);
exactly one is required. Consequence comes from `C` (1-5) or `bclc_stage`.

Errors are always `{"error": {"code": "...", "message": "..."}}`. Unknown fields are rejected on purpose.

## Run locally

```bash
python -m app.server                      # http://localhost:8000/docs
python -m unittest discover -s tests -v   # 74 tests (37 API + 37 framework)
```

## Deployed on Render

The source folders (`app/`, `core/`, `oncology/`, `disaster_climate/`, `docs/`, `tests/`) are bundled in `rmti-code.zip` at the
repo root; Render unpacks it during the build. Unzip it locally to run the tests or read the code.

This repo includes a `render.yaml` Blueprint, so Render builds and starts it with no further configuration
(`pip install -r requirements.txt`, then `python -m app.server`, health check at `/health`). The public base URL is
`https://marolla-rmti.onrender.com` (Render adds a short suffix if that name is already taken); open `/docs` on it
for the interactive documentation.

- Built-in limits (override under Render *Environment*): `RMTI_CLIENT_RPM` (30/min per client),
  `RMTI_GLOBAL_RPM` (60/min total), `RMTI_MAX_CONCURRENCY` (16). Excess traffic gets `429`/`503`.
- Optional API key: add `RMTI_API_KEY` as an environment variable and every `POST` must send
  `X-API-Key: <value>`. `GET` stays open. A key embedded in a public web page is not secret.
- Free Render services sleep after about 15 minutes without traffic; the first request afterwards takes
  roughly a minute to wake the service.

## Privacy

Request bodies and query strings are never stored or logged; the log line is `METHOD PATH STATUS ms`.
Nothing persists between requests. Responses carry a `notice` field with the intended-use limits.
A test (`Privacy`) fails if a body ever reaches the log.

## Data note

`/v1/disaster/cohort` serves Nairobi, Dar es Salaam and Lusaka **baseline-only**, as in the paper's
Tables 4-6 and the Calculator workbook. (The v1.1.0 reference code adds scenario `rho_after` values for
these three that appear in neither source; see `BASELINE_ONLY` in `app/routes.py`.)

## Optional: archive a DOI on Zenodo

Zenodo only archives code; it does not host the running API. To mint a DOI for this repo: enable it at
https://zenodo.org/account/settings/github/, then create a GitHub release (`v1.0.0`). `.zenodo.json` and
`CITATION.cff` are included. Add the DOI to `CITATION.cff` afterwards.

## License

Apache-2.0 (see `LICENSE`) for the code. See `docs/LICENSE-DOCS.md` for how the framework's code license
relates to the separately licensed teaching materials; the API serves only numeric tables and scores, not rubric text.
