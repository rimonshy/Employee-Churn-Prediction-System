# Executive Summary
## Employee Churn Prediction System

### What Was Built

An end-to-end machine learning system that predicts which employees are at risk of leaving the company within the next 90 days. The system was built on the IBM HR Analytics dataset (1,470 employees, 35 features) and covers the full pipeline: exploratory analysis, feature engineering, model training and selection, hyperparameter tuning, feature importance analysis, and a production deployment plan.

Four model types were trained and compared — Logistic Regression, Random Forest, XGBoost, and SVM — each wrapped in a sklearn `Pipeline` that handles preprocessing internally to prevent data leakage. Class imbalance (~84% stay / 16% leave) was addressed via `class_weight` and `scale_pos_weight`. The winning model was selected through 5-fold stratified cross-validation scored on **F2** (β=2), which weights Recall twice as heavily as Precision to reflect the asymmetric cost of missing a leaver versus triggering a false alarm.

Six domain-informed engineered features were added (Income-to-Experience ratio, Tenure Ratio, Satisfaction Score, Promotion Lag, Career Growth Rate, Manager Stability), which measurably improved model performance.

---

### Model Results

**Winner: Logistic Regression (tuned)** — selected via `RandomizedSearchCV` over penalty type, regularisation strength, and class weighting.

| Metric | Validation Set | 5-Fold CV |
|---|---|---|
| ROC-AUC | 0.854 | 0.814 |
| PR-AUC | 0.677 | 0.605 |
| Recall | 0.750 | 0.716 |
| Precision | 0.551 | 0.373 |
| F1 | 0.635 | 0.489 |
| F2 | — | 0.622 |

*Metrics are reported on the held-out 15% validation set and confirmed via 5-fold CV on train+val. The 15% test set remains untouched for final deployment evaluation.*

**Best parameters found:** `C=0.157`, `penalty=elasticnet`, `class_weight={0:1, 1:3}` — penalising missed leavers 3× more heavily than false alarms was the key driver of improvement over the default `balanced` weighting.

On the validation set (36 actual leavers), the tuned model catches **27 out of 36 leavers (75%)** while generating 22 false alarms. The **F2-optimal decision threshold** is used at inference time to maximise recall within an acceptable false-alarm budget.

---

### Business Value

Employee attrition costs an estimated **50–200% of annual salary** per departed employee in recruitment, onboarding, and lost productivity. For a 2,000-employee company with a 16% annual attrition rate, this represents roughly **320 departures per year**.

This system enables HR to:

1. **Prioritise proactively.** The model flags highest-risk employees before they resign, giving HR a 90-day window to intervene with targeted actions (manager conversation, salary review, flexible working).
2. **Focus limited resources.** HR can concentrate effort on the ~75% of leavers the model identifies — roughly **240 out of 320 annual departures** — maximising the return on retention spend.
3. **Understand root causes.** SHAP analysis confirms the top drivers of attrition are **OverTime, MonthlyIncome relative to experience, and low satisfaction scores** — all directly actionable by HR leadership without waiting for exit interviews.
4. **Operate with transparency.** Logistic Regression's interpretable coefficients allow HR managers to understand *why* a specific employee was flagged, building trust and enabling better intervention design.

If the model prevents even 20% of predicted departures through targeted retention — a conservative estimate based on HR intervention literature — the projected saving on a 2,000-person workforce exceeds **several hundred thousand dollars annually**, far outweighing the cost of building and running the system.

A companion A/B test plan (`Section 9` in the notebook) is included to causally validate these projections before scaling the intervention programme.

---

*For the full technical details see `notebook.ipynb`. Production architecture diagram: `production_pipeline.md`.*
