# RAC_Recovery_Rate_Prediction
Predicts recovery rate on defaulted loans from origination and collections features. Compared baseline, linear, and tree-ensemble models via cross-validation; best model (HistGradientBoostingRegressor) 
# ARC Loan Recovery Prediction

Predicts what fraction of a defaulted loan's outstanding principal will ultimately be recovered, for a
simulated Asset Reconstruction Company (ARC) use case.

## Problem

Given loans that have already defaulted and been handed to collections, predict `recovery_rate` —
`recoveries / outstanding_principal`, clipped to `[0, 1]` — for each loan in a held-out test set. This is a
**regression** problem (a continuous 0–1 target), evaluated on R², MAE, and RMSE rather than accuracy.

## Data

- `arc_loan_recovery_train.csv` — 7,000 loans, includes the target
- `arc_loan_recovery_test.csv` — 2,000 loans, target removed
- `DATA_DICTIONARY.md` — column definitions

Synthetic dataset loosely modeled on real consumer-loan recovery data (LendingClub-style), built with
realistic noise and data-quality quirks.

## Approach

1. **Clean** — normalize inconsistent `home_ownership` casing, convert the `dti == -999` sentinel to `NaN`,
   fill `collateral_value` with `0` where `collateral_flag == 0` (structural, not random, missingness).
2. **Engineer features** — log-transform skewed dollar amounts, add missingness flags, collapse
   `fico_range_low`/`fico_range_high` into one `fico_avg`, derive ratios (`collateral_coverage`,
   `pct_paid_before_default`, `loan_to_income`).
3. **Guard against leakage** — `recoveries` and `collection_recovery_fee` algebraically define the target
   and are dropped; `payment_plan_status` is described in the data dictionary but absent from the test
   file, so it's dropped too.
4. **Validate properly** — 80/20 train/validation split stratified on `recovery_status`, plus 5-fold
   cross-validation for model comparison. The validation split is untouched until one final check.
5. **Compare models** — mean baseline, `RidgeCV`, `RandomForestRegressor`, `HistGradientBoostingRegressor`,
   through one shared preprocessing pipeline.
6. **Evaluate beyond a single number** — R², MAE, RMSE, plus a decile-level calibration check (predicted vs.
   actual mean recovery rate by bucket).
7. **Explain** — permutation importance on the held-out set.

## Results

| Model | CV R² | CV MAE | CV RMSE |
|---|---|---|---|
| mean baseline | −0.00 | 0.162 | 0.193 |
| Random Forest | 0.572 | 0.102 | 0.126 |
| RidgeCV | 0.603 | 0.099 | 0.122 |
| **HistGradientBoosting** | **0.610** | **0.097** | **0.121** |

Held-out validation (never used for model selection): **R² = 0.615, MAE = 0.094, RMSE = 0.115**,
well-calibrated across the full range of recovery rates.

**Top features** (permutation importance): `recovery_status`, `fico_avg`, `days_past_due_at_default`.

**Leakage sanity check:** including `recoveries` as a feature pushes R² to 0.985 — confirming it reflects a
leak, not a better model, and that the honest 0.615 is the right number to trust.

## Repo structure

```
├── README.md
├── ARC_recovery_model.ipynb   # full pipeline: EDA → cleaning → features → models → evaluation → predictions
├── submission.csv             # predicted recovery_rate for the 2,000 test loans
├── WRITEUP.md                 # model rationale, metrics, feature importance, production considerations
├── arc_loan_recovery_train.csv
├── arc_loan_recovery_test.csv
└── DATA_DICTIONARY.md
```

## Running it

Open `ARC_recovery_model.ipynb` in [Google Colab](https://colab.research.google.com), then
**Runtime → Run all**. When prompted, upload `arc_loan_recovery_train.csv` and `arc_loan_recovery_test.csv`.

**Requirements:** Python 3, `pandas`, `numpy`, `scikit-learn`, `matplotlib` — all preinstalled on Colab.

## Before trusting this further

See `WRITEUP.md` for what would be needed before using this in a real underwriting/recovery decision —
more data, calibration monitoring, human-in-the-loop review, and fairness checks.
