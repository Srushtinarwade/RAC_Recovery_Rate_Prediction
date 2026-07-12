# Data Dictionary — ARC Loan Recovery Dataset

Each row represents a loan that has already **defaulted / been charged off** and handed to recovery.
Your task is to predict `recovery_rate` — the fraction of the outstanding principal that will
ultimately be recovered.

| Column | Description |
|---|---|
| loan_id | Unique loan identifier |
| loan_amount | Original loan amount disbursed ($) |
| term_months | Loan term: 36 or 60 months |
| grade | Lender's internal risk grade at origination (A = lowest risk, G = highest) |
| sub_grade | Finer-grained risk grade (e.g. B3) |
| int_rate | Interest rate at origination (%) |
| emp_length_years | Borrower's employment length in years (10 = 10+ years). May be missing if undisclosed. |
| home_ownership | RENT / MORTGAGE / OWN / ANY. Casing is not always consistent in this export. |
| annual_income | Borrower's self-reported annual income ($). May be missing. |
| verification_status | Whether income was verified at origination |
| purpose | Stated purpose of the loan |
| dti | Debt-to-income ratio (%) at origination. May be missing. A small number of records use `-999` where the bureau pull failed to return a value. |
| fico_range_low / fico_range_high | Borrower's FICO score band at origination |
| delinq_2yrs | Number of 30+ day delinquencies in the 2 years before origination |
| pub_rec | Number of derogatory public records (bankruptcies, judgments, etc.) |
| open_acc | Number of currently open credit lines |
| total_acc | Total number of credit lines ever opened |
| revol_bal | Total revolving credit balance ($) |
| revol_util | Revolving credit utilization (%). May be missing. |
| collections_12_mths_ex_med | Collections in the last 12 months, excluding medical |
| collateral_flag | 1 if the loan is secured by collateral, 0 if unsecured |
| collateral_value | Estimated value of collateral ($). Not applicable (blank) when `collateral_flag` = 0. |
| total_pymnt_before_default | Total amount repaid by the borrower before default ($) |
| outstanding_principal | Principal still owed at the point of default — the amount ARC is trying to recover ($) |
| days_past_due_at_default | Days delinquent at the point the loan was charged off / transferred to recovery |
| months_in_recovery | Months the account has been in the collections/recovery process so far |
| recovery_status | `Closed` if the recovery process has concluded, `Active` if the account is still being worked |
| payment_plan_status | Current status of any repayment plan set up with the borrower during collections: `Completed`, `Active`, `Broken`, or `No_Plan` |

## Target (train file only)

| Column | Description |
|---|---|
| recoveries | Dollar amount recovered so far ($) |
| collection_recovery_fee | Fee charged by the collection agency on amounts recovered ($) |
| **recovery_rate** | **`recoveries / outstanding_principal`, clipped to [0, 1] — the value to predict.** For loans where `recovery_status` = `Active`, this reflects recovery *to date*, since the process hasn't concluded yet. |

## Files

- `arc_loan_recovery_train.csv` — 7,000 loans, includes the target columns above
- `arc_loan_recovery_test.csv` — 2,000 loans, **target columns removed** — predict `recovery_rate` for each `loan_id`
- Submit predictions as a CSV with exactly two columns: `loan_id`, `predicted_recovery_rate`

## Notes on the data

- This is a **synthetic dataset** built to resemble real-world unsecured/secured consumer loan
  recovery data (loosely modeled on public datasets like LendingClub), so there's no confidentiality
  concern in using it or discussing it externally.
- It contains realistic-style noise, missing values, and data-quality quirks, and does not perfectly
  follow any single formula — treat it the way you'd treat a real export from a lending/collections
  system, including deciding for yourself what needs cleaning, what's usable as a feature, and what
  isn't.
