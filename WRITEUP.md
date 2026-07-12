# ARC Loan Recovery: Strategy & Insights

## The Approach: How We Framed the Problem

Since **recovery_rate** is a continuous value between 0 and 1, we are dealing with a regression problem, not a classification task.

Before throwing algorithms at the data, we had to clean up the real-world mess. This meant fixing inconsistent casing in `home_ownership`, handling the `dti = -999` sentinel value, and addressing the structural (not random) missingness of `collateral_value`. From there, we engineered features that actually make financial sense: log-transforms for highly skewed dollar amounts, missingness flags, a collapsed `fico_avg`, and key ratios like `collateral_coverage` and `pct_paid_before_default`.

We threw four models into a 5-fold cross-validation arena: a basic mean-prediction baseline, `RidgeCV`, `RandomForestRegressor`, and `HistGradientBoostingRegressor`.

### Catching Data Leakage Early

We explicitly excluded `recoveries` and `collection_recovery_fee` because they algebraically define the recovery rate. Keeping them would be blatant data leakage. We also tossed out `payment_plan_status`it was in the data dictionary but completely missing from the test file. Catching this by manually diffing the train and test columns (rather than blindly trusting the documentation) is exactly the kind of practical sanity check that matters more than the actual model choice.

**The Winner:** `HistGradientBoostingRegressor` (built with shallow trees, depth 3) took the crown. However, it only marginally beat out `RidgeCV`. This tells us the underlying data-generating process is quite noisy but fairly additive, meaning an overly complex, highly non-linear model isn't necessary here. This finding held up beautifully on a 20% held-out validation split that the model selection process never touched.

---

## Evaluating Performance: Moving Beyond a Single Number

Chasing a single "accuracy" metric is a trap—especially for continuous targets where "accuracy" isn't even well-defined. Instead, we looked at the problem through a few different lenses:

* **$R^2 \approx 0.61$** – The global summary. The model explains about **61%** of the variance in the recovery rate.
* **MAE $\approx 0.09$** – The metric decision-makers actually care about. On average, the model's predictions are off by about **9 percentage points**, and it remains robust against hard-to-predict outliers.
* **RMSE $\approx 0.12$** – Because RMSE penalizes larger misses more heavily, the small gap between it and the MAE tells us our errors are fairly evenly sized, rather than being driven by a few catastrophic blowups.
* **Calibration (By Decile):** This is the most crucial piece for downstream business decisions. A model can have a fantastic $R^2$ but still quietly ruin your pricing by being over-confident on high-recovery loans and under-confident on low-recovery ones. By binning our predictions into deciles, we verified that the predicted and actual means track each other incredibly closely (e.g., **0.16 vs. 0.17** at the low end, and **0.66 vs. 0.67** at the high end). There is no directional bias at either extreme.

---

## The Real Drivers: What Actually Matters

To figure out what drives the model, we used **permutation importance** on the held-out validation set. By shuffling each feature and measuring the drop in $R^2$, we get a true sense of importance that isn't just an artifact of a specific algorithm's internal scoring.

1. **`recovery_status` (Active vs. Closed):** By far the heaviest hitter. Active accounts average a **~20%** recovery rate, while Closed accounts average **~45%**. This makes total intuitive sense: an active case is just a snapshot mid-process, not a finished outcome.
2. **`fico_avg`:** A borrower's credit quality at origination remains a strong predictor of how collectible they are, even after they have gone into default.
3. **`days_past_due_at_default`:** The longer a borrower was delinquent before officially defaulting, the harder it is to recover the funds.

These three features hold the vast majority of the predictive power; everything else provides minor incremental value.

---

## The Reality Check: What We Need Before Production

While the model performs well on paper, I wouldn't bet real capital on it without addressing a few critical points:

* **More (and fresher) data:** 7,000 synthetic loans are fine for benchmarking models, but they won't cut it in production. The model hasn't seen different loan vintages or shifting macroeconomic cycles.
* **Monitoring for calibration drift:** Recovery dynamics change alongside the economy and collection strategies. A model well-calibrated today can quietly decay tomorrow. We need to track live calibration by decile in real-time.
* **A better approach to "Active" loans:** Treating an active, ongoing collection process as a fixed point-in-time regression adds unnecessary noise. Transitioning to a survival analysis or time-to-recovery framework would be a much more honest way to handle these accounts.
* **Human-in-the-loop guardrails:** Automated models are great, but we should always have human review for high-value loans or predictions sitting right on critical decision thresholds.
* **Fairness and Compliance audits:** Collections are highly regulated. Features like income and homeownership can easily act as proxies for protected characteristics. Before this model influences any real borrower-facing decisions, a thorough disparate-impact analysis is mandatory.