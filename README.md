# Kratos

Auditable agentic workflow for **target-mediated drug disposition (TMDD)** modelling: data validation →
raw graphics → versioned TMDD model → bounded adaptive multistart calibration (Python/SciPy) → **nlmixr2 FOCEi**
population estimation (R) → diagnostics (convergence, covariance, RSE, bounds, shrinkage, CWRES, VPC) →
reproducible Quarto report (`moDoc.qmd`) and export bundle. Every stage transition and its evidence is stored
in the run directory.

> Research prototype. Heuristic checks support scientist review; Kratos never declares a model scientifically valid.

## Layout

| Path | Contents |
|---|---|
| `src/kratos/` | `orchestrator.py` (controller), `data_agent.py`, `model_agent.py`, `calibration_agent.py`, `planner.py` (rules / fixed / optional LLM), `llm.py`, `estimation_agent.py`, `diagnostics_agent.py`, `reporting_agent.py`, `r_bridge.py`, `schemas.py` (action/result contracts), `simulate.py` (Python TMDD engine), `crossengine.py`, `crosscheck.py`, `bootstrap.py`, `evaluation.py`, `split.py`, `synthetic.py`, `plots.py`, `runstore.py`, `cli.py` |
| `R/` | `tmdd_rxode2.R` (model translation), `simulate.R`, `nlmixr_estimation.R`, `diagnostics.R` (VPC), `bootstrap.R`, `results.R`, `setup_packages.R` |
| `models/tmdd2c_full/template.yaml` | approved, versioned TMDD template (v1.0.0 IV; v1.1.0 adds SC depot) |
| `config/` | `schema.json` (configuration contract), `synthetic_demo.yaml`, `mab_partA.yaml`, `mab_partB.yaml` |
| `data/` | `synthetic/` (SYNTHETIC example + truth), `mab/` (two-part split of the mAb dataset + manifest) |
| `app/streamlit_app.py` | browser interface |
| `scripts/` | `results.py` (regenerate from saved results), `generate_synthetic.py`, `split_dataset.py`, `build_project_report.py` |
| `reports/moDoc.qmd` | report entry point (rendered per run and for the whole project) |
| `tests/` | pytest suite, `tests/R/test_tmdd.R`, invalid-input fixtures |
| `docs/` | data dictionary, migration note, feature status |
| `runs/<run_id>/` | one directory per run (inputs, normalized data, ID map, config snapshot, model code, hashes, seeds, `audit.jsonl`, calibration history, accepted starts, `fit.rds`, tables, figures + figure data, report, manifest, export zip) |

## Setup

### Windows (PowerShell)
```powershell
cd C:\Users\devamdesai\Downloads\Kratos
python -m pip install -e .                    # core (numpy, scipy, pandas, matplotlib, PyYAML, jsonschema)
python -m pip install streamlit pytest jupyter ipykernel   # UI, tests, Quarto Python engine
# R >= 4.5 with rxode2, nlmixr2 (and Rtools for model compilation). Record/repair versions:
$env:R_LIBS = "$PWD\.Rlib"
& "$env:LOCALAPPDATA\Programs\R\R-4.5.1\bin\Rscript.exe" R\setup_packages.R --install
# Quarto >= 1.4 for the report (https://quarto.org)
```

### Linux (bash)
```bash
cd Kratos
python3 -m pip install -e . streamlit pytest jupyter ipykernel
export R_LIBS="$PWD/.Rlib"
Rscript R/setup_packages.R --install
```

`R_LIBS` is set automatically by Kratos when it calls R; the project library `.Rlib` holds only packages that
had to be upgraded for compatibility (on the development machine: RcppParallel 6.2.1, needed by `stringfish`).
Exact versions: `requirements-lock.txt`, `R/r-packages-lock.json` (`renv.lock` is written when renv is installed).

## Commands

All scientific commands accept a configuration and/or explicit run path, exit non-zero on failure and accept `--seed`.

```bash
kratos validate  --config config/mab_partA.yaml [--out outdir]
kratos simulate  --config config/synthetic_demo.yaml --engine both --out sim.csv   # Python + R agreement check
kratos calibrate --config config/synthetic_demo.yaml --seed 7                      # stops after calibration
kratos run       --config config/synthetic_demo.yaml --run-id demo                 # full workflow
kratos estimate  --run runs/demo        # stage-by-stage on an existing run
kratos diagnose  --run runs/demo
kratos report    --run runs/demo        # regenerates figures/tables/report from saved results (no refit)
kratos ui                               # http://localhost:8501
kratos status    --run runs/demo ; kratos cancel --run runs/demo
kratos split     --src "<...>/mAb_dataset.csv" --outdir data/mab --alias Benralizuman=Benralizumab
kratos crosscheck --run-a runs/mab_partA_v1 --run-b runs/mab_partB_v1 --out reports/crosscheck
kratos bootstrap --run runs/demo --n 20 --seed 1
kratos evaluate  --config config/synthetic_demo.yaml --replicates 5 --out evaluation/hypothesis
python scripts/results.py --run runs/demo           # Python regeneration (no estimation)
Rscript R/results.R --run runs/demo                  # R/ggplot2 regeneration (no estimation)
python -m pytest -q                                  # tests (R tests skip automatically without R)
```

Windows PowerShell uses the same commands (`kratos ...`); `python -m kratos ...` works without the entry point.

## Reproducible demonstration (SYNTHETIC)

```bash
python scripts/generate_synthetic.py            # data/synthetic/synthetic_tmdd.csv (+ truth, invalid fixtures)
kratos run --config config/synthetic_demo.yaml --run-id synthetic_demo_v1
```

## Behavioural contracts (summary)

* **Data**: original CSV preserved; normalized copy in h / nmol / nmol/L with every conversion recorded; molecular
  weights must be declared; EVID 0/1 only; IV bolus, finite infusion (RATE takes precedence, DUR converted, both must
  agree), SC depot (template ≥ 1.1.0), ADDL/II expanded deterministically; SS, modeled rates, unknown routes,
  censored records (no tested M3 likelihood; no LLOQ/2) are **rejected** with row/field-specific errors; outliers and
  suspicious records are **flagged, never deleted**; pre/post-dose convention applied identically in both engines.
* **Calibration**: pooled, bounded, transformed-space multistart (user start + Latin hypercube), adaptive rules planner,
  budgets (attempts, time, evaluations), plateau / reproduced-optimum stopping; all attempts incl. failures recorded.
  Its objective is not comparable with the nlmixr2 OFV.
* **Estimation**: nlmixr2 FOCEi from ranked distinct calibration candidates with bounded retries. Without R the run is
  explicitly *calibration-only* and estimation is `not_run` — no fit, CWRES or VPC is ever synthesized.
* **LLM (optional)**: receives only an approved diagnostic summary (no subject records), must return schema-valid JSON,
  proposals outside bounds or touching fixed parameters are rejected, and the rules planner is the fallback.
