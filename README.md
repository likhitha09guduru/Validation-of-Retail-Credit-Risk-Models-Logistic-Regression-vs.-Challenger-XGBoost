# Retail Credit Risk Scorecard: Logistic Regression vs. XGBoost

**Tech stack:** Python, pandas, NumPy, scikit-learn, XGBoost, statsmodels, scorecardpy

This is my project to build a retail credit scorecard the way it'd actually be done at a bank —
not just training a model, but going through the full lifecycle: cleaning the data, sampling it
properly, building a Logistic Regression scorecard, calibrating it to points, and then validating
it properly, including comparing it against an XGBoost "challenger" model to see if the more
complex model is actually worth it.

I wanted to practice the model validation side of things specifically, since that's usually the
part that gets skipped in typical ML course projects — people build a model and stop, but in
industry someone else has to independently check whether the model is actually good, stable, and
better than the current one before it goes anywhere near production.

## How it's organized

I split the project into 9 notebooks, one per stage of the pipeline, so it's easy to follow what
happens at each step:

| Step | Notebook | What I did |
|---|---|---|
| 1 | `step1 load Data .ipynb` | Load and merge the application, behavioral, and bureau data |
| 2 | `step 2 Vintage Analysis and Sampling .ipynb` | Look at bad rates over time, split into Train / Test / Out-of-Time |
| 3 | `step3 Data Preperation.ipynb` | Data quality checks — missing values, outliers (IQR method), correlation/VIF, and a discussion of sampling risk and reject inference |
| 4 | `step 4 segmentation.ipynb` | KMeans customer segmentation |
| 5 | `step 5 variable reduction.ipynb` | Filter variables by IV, check WOE monotonicity, drop multicollinear variables |
| 6 | `step 6_Model_Creation.ipynb` | WOE binning + Logistic Regression scorecard (using `scorecardpy`) |
| 7 | `step7 Score_Calibration.ipynb` | Convert model output to points using Factor/Offset calibration |
| 8 | `step 8 Model Validation.ipynb` | Validate the champion model — Gini, KS, PSI, CSI across all 26 risk variables, precision/recall/F1 at a matched approval-rate cutoff |
| 9 | `step 9 challenger model xgboost.ipynb` | Compare champion (Logistic Regression) vs. challenger (XGBoost) fairly, using 5-fold out-of-fold cross-validation, plus an overfitting check |

## What I found

I validated the Logistic Regression champion against an XGBoost challenger using 5-fold
out-of-fold cross-validation, so neither model gets scored on data it was trained on — that's
important, otherwise the comparison isn't fair.

| Metric | Champion (Logistic Regression) | Challenger (XGBoost) |
|---|---|---|
| Gini | 0.117 | 0.071 |
| KS | 0.229 | 0.205 |
| Precision (matched approval-rate cutoff) | 0.500 | 0.300 |
| Recall (matched approval-rate cutoff) | 0.333 | 0.200 |
| PSI (score stability) | 0.561 | 0.328 |

The Logistic Regression model actually did better across every metric here. I also checked
stability using CSI across all 26 risk variables in the final dataset (Train vs. Test), on top of
these.

## Checking for overfitting

I didn't want to just say "XGBoost probably overfits on small data" without actually showing it,
so in Step 9 I fit a plain default-parameter XGBoost next to my regularized one, and compared each
model's in-sample AUC (trained and scored on the same data) against its out-of-fold AUC (scored
only on data it never saw during training). The gap between those two numbers is basically a
direct measurement of overfitting.

| Model | In-Sample AUC | Out-of-Fold AUC | Gap |
|---|---|---|---|
| Champion (LR) | *(see Step 9 output)* | *(see Step 9 output)* | smallest gap |
| Challenger (XGB, regularized) | *(see Step 9 output)* | *(see Step 9 output)* | *(see Step 9 output)* |
| XGBoost (default params) | *(see Step 9 output)* | *(see Step 9 output)* | largest gap |

## My conclusion

**I'd keep the Logistic Regression model as the champion.** With only 100 rows in this dataset,
XGBoost doesn't have much of a chance to actually learn anything a well-built logistic scorecard
isn't already capturing, and it overfits more in the process. The full reasoning (and the caveats
around sample size) is written out at the end of `step 9 challenger model xgboost.ipynb`.

## Honest limitations of this project

- **Sample size is small (n=100).** This is a class-project-scale dataset, not a production one,
  so I'm being careful not to over-claim what these results prove — I'm treating this as a
  demonstration of the validation *process*, not a definitive result.
- **No out-of-time split.** I don't have a true time-based population to test on, so the OOF
  cross-validation approach controls for overfitting but not for the model drifting over time,
  which is a different (and important) thing that a real bank would also check.
- **Reject inference isn't possible here.** The dataset only has customers who were already
  approved — there's no data on people who applied and got rejected. I flag this in Step 3 as a
  known gap, since in a real scorecard project this would need to be addressed before trusting the
  model on the full applicant population.

## Running it yourself

Everything uses relative paths, so just clone it and run the notebooks in order from Step 1 to
Step 9 — each one reads the `.xlsx` file the previous step saved.

```
git clone <this-repo>
cd <this-repo>
pip install pandas numpy scikit-learn xgboost statsmodels scorecardpy openpyxl matplotlib
jupyter notebook
```

## What's in the repo

- `customer_scorecard_input.xlsx`, `bureau_data.xlsx` — the raw data I started with
- `EDA.xlsx`, `EDA_final_dataset.xlsx`, `Final_Model_Dataset.xlsx` — the dataset at different stages of cleaning/prep
- `Final_Model_Scorecard.xlsx`, `Final_Model_Scorecard_Clean.xlsx` — the final points-based scorecard
