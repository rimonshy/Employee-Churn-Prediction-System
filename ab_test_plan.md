# A/B Test Plan: HR Intervention Effectiveness

**Project:** Employee Churn Prediction System  
**Document type:** Experimental Design  
**Status:** Draft v1.0

---

## 1. Objective

Determine whether using model-predicted attrition risk scores to trigger **proactive HR interventions** (manager conversation, targeted salary review, work-flexibility options) causally reduces 90-day employee attrition — beyond what would occur without any intervention.

---

## 2. Formal Hypotheses

Let **p_c** = 90-day attrition rate in the control group and **p_t** = 90-day attrition rate in the treatment group.

- **H₀ (Null):** p_t ≥ p_c  (intervention does not reduce attrition)
- **H₁ (Alternative):** p_t < p_c  (intervention reduces attrition — one-tailed test)
- **Significance level:** α = 0.05
- **Statistical power:** 1 − β = 0.80

---

## 3. Group Design

### Eligibility Criterion
Only employees whose model-predicted attrition probability exceeds the **F2-optimal threshold (p ≥ 0.57)** are included. This ensures we target genuinely at-risk employees, avoiding noise from low-risk individuals.

### Randomisation Unit
The **department/team** is the unit of randomisation (cluster randomisation), not the individual employee. Randomising at the individual level risks contamination: a manager aware of one team member's intervention may informally help others, and employees may talk and influence each other. Cluster randomisation keeps groups isolated. Clusters are balanced on:
- Department size
- Historical attrition rate
- Average tenure band

### Groups

| Group | Description | N (target) |
|---|---|---|
| **Control** | At-risk employees — no proactive intervention triggered; standard HR operations continue | 136 |
| **Treatment** | At-risk employees — HR receives alert and must initiate ≥ 1 of: manager 1-on-1, salary review, or flex-work offer | 136 |

Randomisation is performed by the HR analytics platform at the time of scoring, using a deterministic hash of `(employee_id, experiment_id)` to ensure reproducibility and prevent drift.

---

## 4. Contamination Prevention

Attrition risk is **individual-level** (unlike pricing experiments where customers can see each other's offers), but HR contamination is still possible:

| Risk | Mitigation |
|---|---|
| Manager discusses intervention with control-group peers | Randomise at manager-team level if intra-team contamination is detected |
| HR informally flags control employees after seeing treatment | Blind HR business partners to group assignment; only experiment coordinator sees labels |
| Salary review for treatment employee changes team compensation culture | Include compensation-band stratification; monitor control group salary changes as a covariate |
| Treatment employees discuss their experience with control employees | Assess via biweekly pulse surveys — flag abnormal satisfaction changes in control group |

---

## 5. Metrics

### Primary Metric
**90-day attrition rate:** proportion of employees in each group who have left the company (voluntary resignation or mutual separation) within 90 days of experiment start.

- **Measurement:** HRIS system exit records, confirmed at Day 90
- **Why 90 days:** long enough to observe intervention effect; short enough to run multiple experiment cycles per year

### Secondary Metrics

| Metric | Source | Measurement Window |
|---|---|---|
| Employee satisfaction score | Monthly pulse survey (eNPS / 5-pt scale) | Days 30, 60, 90 |
| Manager-reported performance (1–5) | Performance management system | Day 90 |
| Absenteeism rate | HRIS time-off records | Days 0–90 |
| Promotion / role-change rate | HRIS | Days 0–90 |
| Cost per retained employee | Finance | Post-experiment |

Secondary metrics serve two purposes: (1) understanding *how* interventions work, and (2) detecting unintended effects (e.g. satisfaction improves but performance drops).

---

## 6. Sample Size Calculation

### Parameters
- Baseline attrition rate (control): **p_c = 0.16** (empirical from IBM HR dataset)
- Minimum Detectable Effect (MDE): **Δ = 0.05** absolute reduction → p_t = 0.11
- Significance level: **α = 0.05** (one-tailed)
- Power: **1 − β = 0.80**

### Formula (two-proportion z-test, one-tailed)
```
n = (z_α + z_β)² × [p_c(1−p_c) + p_t(1−p_t)] / Δ²

z_α = 1.645  (α=0.05, one-tailed)
z_β = 0.842  (power=0.80)

p_c = 0.16,  p_t = 0.11,  Δ = 0.05

Numerator:   (1.645 + 0.842)² × [0.16×0.84 + 0.11×0.89]
           = (2.487)² × [0.1344 + 0.0979]
           = 6.185 × 0.2323
           = 1.437

n = 1.437 / (0.05)² = 1.437 / 0.0025 = 575 per group
```

**Required sample size: 575 employees per group (1,150 total).**

With a 1,470-employee company at 16% at-risk rate ≈ 235 flagged employees per scoring cycle, we need **≈ 4–5 monthly scoring cycles** to reach target N.

---

## 7. Minimum Detectable Effect: Is 5% Realistic?

### Assessment: **Moderately ambitious but achievable**

| Evidence | Detail |
|---|---|
| Literature on manager conversations | Gallup (2019): structured stay interviews reduce attrition by 15–30% in at-risk segments |
| Salary adjustment ROI | SHRM: market-rate correction reduces intention-to-leave by 18–24% for compensation-driven leavers |
| Flex-work impact | Stanford (Bloom et al., 2015): WFH option reduced quit rates by 50% in call-centre study |
| Realistic dilution | Not all treatment employees receive the *right* intervention; compliance is typically 60–80% in HR experiments |

A 5-percentage-point absolute reduction (16% → 11%) corresponds to a **31% relative reduction** in attrition. Given mixed interventions and ~70% HR compliance, a 3–5 pp reduction is realistic. Setting MDE = 5 pp is appropriately conservative.

---

## 8. Minimum Experiment Duration

**Recommended duration: 90 days observation + 30-day ramp-up = 120 days (4 months)**

| Phase | Duration | Purpose |
|---|---|---|
| Ramp-up | 30 days | Enrol employees, train HR BPs, suppress novelty effects |
| Observation | 90 days | Primary metric window (attrition measurement) |
| Analysis | 14 days | Statistical testing, secondary metrics, guardrails check |

**Why not shorter?** Attrition is a slow-moving event. Most voluntary resignations follow a 4–8 week internal deliberation. Observing for < 60 days risks undercounting churners who decided to leave during the window but hadn't resigned yet.

**Why not longer?** Beyond 4 months, external confounders (economic conditions, company-wide layoffs, M&A) become increasingly difficult to control.

---

## 9. Statistical Analysis Plan

```python
from scipy.stats import proportions_ztest
import numpy as np

# Observed attrition counts at Day 90
churned_control   = ...   # number of leavers in control group
n_control         = 575

churned_treatment = ...   # number of leavers in treatment group
n_treatment       = 575

count = np.array([churned_treatment, churned_control])
nobs  = np.array([n_treatment, n_control])

z_stat, p_value = proportions_ztest(count, nobs, alternative='smaller')

print(f"z = {z_stat:.3f},  p = {p_value:.4f}")
if p_value < 0.05:
    print("Reject H0: intervention reduces attrition")
else:
    print("Fail to reject H0")
```

**Additional analysis:**
- Subgroup analysis by intervention type (manager chat / salary / flex-work)
- Survival analysis (Kaplan–Meier) for time-to-attrition curves
- Intent-to-treat (ITT) vs per-protocol analysis
- CUPED variance reduction using pre-experiment attrition-risk score as covariate

---

## 10. Ethics Considerations

| Concern | Mitigation |
|---|---|
| **Differential treatment:** Control employees receive no help even if at risk | Ethical because we do not yet *know* interventions are effective; this experiment establishes causality. Post-experiment, all high-risk employees receive interventions regardless of group. |
| **Privacy:** Employees may not know they are being scored | Disclose in HR policy that risk scoring is used for retention programs. GDPR/CCPA require documentation of automated profiling. |
| **Manager bias:** Knowing an employee is "at-risk" may cause unfair treatment | HR business partners are briefed to use risk scores for *support*, not performance management. Separate access controls. |
| **Coercion:** Employees receiving interventions may feel pressured | Communicate all interventions as voluntary employee-support initiatives, not performance improvement plans. |
| **False positives** impact control group | High-risk employees in the control group who leave are a real cost of running the experiment. This is the ethical price of acquiring causal evidence. Duration should be minimised. |

**IRB / Legal review:** Any HR experiment touching compensation or role changes should be reviewed by Legal, People Operations, and an independent ethics committee before launch.

---

## 11. Limitations of A/B Testing in HR Context

| Limitation | Description |
|---|---|
| **Small population** | With ~1,470 employees, reaching 575 per group requires multiple cycles and assumes stable at-risk rate |
| **SUTVA violation** | Stable Unit Treatment Value Assumption may fail if treatment employee's departure affects team morale (indirect contamination) |
| **Novelty effect** | HR focus on treatment employees might raise retention in the short term for reasons unrelated to the specific intervention |
| **Hawthorne effect** | Both groups may perform differently knowing they are being observed (if transparency is high) |
| **Time-varying confounders** | External labour market changes, team restructuring, or manager changes during the experiment can confound results |
| **Selection bias on risk threshold** | We only experiment on employees above p=0.57, limiting external validity to other employees |

---

## 12. Quasi-Experimental Alternatives

When randomisation is infeasible (e.g. intervention must be company-wide, or ethical concerns prevent a control group):

| Method | When to Use | Key Assumption |
|---|---|---|
| **Difference-in-Differences (DiD)** | Rollout by department/site in waves | Parallel pre-treatment trends |
| **Regression Discontinuity (RDD)** | Sharp eligibility cutoff (e.g. only employees with p ≥ 0.65 get intervention) | No manipulation around the threshold |
| **Interrupted Time Series (ITS)** | Single company-wide rollout with long historical series | No other interventions at the same time |
| **Synthetic Control** | One treated unit (e.g. one country office) vs portfolio of comparable untreated units | Treated unit well-approximated by a weighted average of controls |
| **Propensity Score Matching** | Retrospective analysis of historical intervention programmes | No unmeasured confounders (strong ignorability) |

**Recommended fallback:** **Regression Discontinuity** around the operational threshold (p=0.57). Employees just above the threshold (treatment) are compared to those just below (control), exploiting the quasi-random assignment near the cutoff.
