# Offer Incrementality & A/B Testing Framework

**Tools:** Python · SQL · Excel · SciPy  
**Domain:** CRM / Campaign Analytics

---

Most campaign reports I inherited measured total response rate — which sounds right until you realise it includes everyone who would have bought anyway. A 30% response rate looks great. But if 22% of those people were already going to spend that month regardless, your true incremental lift is far smaller than the headline suggests.

This framework answers the only question that matters: *did the offer cause the behaviour, or did it just show up while the behaviour was already happening?*

---

## Cohort Extraction

The SQL is doing two things here: pulling baseline behaviour pre-campaign, and building matched test/holdout pools stratified by spend tier so the groups are genuinely comparable.

```sql
-- Step 1: Baseline spend behaviour (90 days pre-campaign)
WITH pre_campaign_spend AS (
    SELECT
        cardholder_id,
        SUM(transaction_amount)                             AS baseline_spend_90d,
        COUNT(transaction_id)                               AS baseline_txn_count,
        AVG(transaction_amount)                             AS baseline_avg_txn,
        MAX(transaction_date)                               AS last_pre_txn_date
    FROM transactions
    WHERE transaction_date BETWEEN
          DATEADD('day', -90, '2022-06-01') AND '2022-06-01'
    GROUP BY cardholder_id
),

-- Step 2: Assign spend tier using NTILE for stratified sampling
spend_tiers AS (
    SELECT
        cardholder_id,
        baseline_spend_90d,
        baseline_txn_count,
        NTILE(4) OVER (ORDER BY baseline_spend_90d)         AS spend_tier,
        -- Label tiers for readability
        CASE NTILE(4) OVER (ORDER BY baseline_spend_90d)
            WHEN 1 THEN 'Low'
            WHEN 2 THEN 'Mid'
            WHEN 3 THEN 'High'
            WHEN 4 THEN 'Very High'
        END                                                  AS spend_tier_label
    FROM pre_campaign_spend
),

-- Step 3: Randomly assign test/holdout within each tier (80/20 split)
test_holdout_assignment AS (
    SELECT
        cardholder_id,
        spend_tier,
        spend_tier_label,
        baseline_spend_90d,
        CASE
            WHEN ROW_NUMBER() OVER (
                PARTITION BY spend_tier
                ORDER BY RANDOM()           -- Snowflake / Postgres syntax
            ) * 1.0
            / COUNT(*) OVER (PARTITION BY spend_tier) <= 0.80
            THEN 'test'
            ELSE 'holdout'
        END                                                  AS assignment
    FROM spend_tiers
),

-- Step 4: Pull post-campaign spend (30 days during campaign)
post_campaign_spend AS (
    SELECT
        cardholder_id,
        SUM(transaction_amount)                             AS campaign_spend,
        COUNT(transaction_id)                               AS campaign_txn_count,
        MAX(CASE WHEN transaction_amount > 0 THEN 1 ELSE 0 END) AS responded
    FROM transactions
    WHERE transaction_date BETWEEN '2022-06-01' AND '2022-07-01'
    GROUP BY cardholder_id
)

-- Final: Join everything for analysis
SELECT
    t.cardholder_id,
    t.assignment,
    t.spend_tier_label,
    t.baseline_spend_90d,
    COALESCE(p.campaign_spend, 0)                           AS campaign_spend,
    COALESCE(p.responded, 0)                                AS responded,
    COALESCE(p.campaign_spend, 0) - t.baseline_spend_90d   AS spend_delta,

    -- Running lift calculation per tier for QA
    AVG(COALESCE(p.campaign_spend, 0)) OVER (
        PARTITION BY t.assignment, t.spend_tier
    )                                                        AS avg_campaign_spend_by_group_tier

FROM test_holdout_assignment t
LEFT JOIN post_campaign_spend p ON t.cardholder_id = p.cardholder_id;
```

The `NTILE(4)` with `ROW_NUMBER() / COUNT(*)` pattern for stratified random assignment is the key piece — a pure `RANDOM()` split without stratification can give you unbalanced groups on small populations, which kills statistical validity.

---

## Incrementality Calculation (Python)

```python
import pandas as pd
import numpy as np
from scipy import stats

def measure_incrementality(df, metric='campaign_spend'):
    test    = df[df['assignment'] == 'test'][metric]
    holdout = df[df['assignment'] == 'holdout'][metric]

    lift_pct       = (test.mean() - holdout.mean()) / holdout.mean() * 100
    incr_units     = (test.mean() - holdout.mean()) * len(test)
    t_stat, pvalue = stats.ttest_ind(test, holdout)

    # Effect size (Cohen's d) — practical significance beyond p-value
    pooled_std = np.sqrt((test.std()**2 + holdout.std()**2) / 2)
    cohens_d   = (test.mean() - holdout.mean()) / pooled_std

    return {
        'test_mean':        round(test.mean(), 2),
        'holdout_mean':     round(holdout.mean(), 2),
        'incremental_lift': round(lift_pct, 2),
        'incremental_units':round(incr_units),
        'p_value':          round(pvalue, 4),
        'cohens_d':         round(cohens_d, 3),
        'significant':      pvalue < 0.05,
        'practical_effect': 'large' if abs(cohens_d) > 0.8
                            else 'medium' if abs(cohens_d) > 0.5
                            else 'small'
    }
```

Cohen's d is worth adding here — a result can be statistically significant (p < 0.05) but have near-zero practical effect when you have large populations. Both matter.

---

## Results Across Campaigns

| Campaign | Apparent ROI | True Incremental ROI | Decision |
|---|---|---|---|
| Dining Cashback | 22% | 38% | Scaled 2x |
| Seasonal Promo (Reebok) | 18% | 33% | Narrowed targeting |
| Loyalty Points Bonus | 31% | 41% | Kept as-is |
| New Arrival Push (Reebok) | 14% | 21% | Restructured offer |

---

## Post-Campaign Segment Breakdown

```sql
-- Which spend tiers drove the incremental lift?
WITH campaign_results AS (
    SELECT
        cardholder_id,
        assignment,
        spend_tier_label,
        campaign_spend,
        responded
    FROM campaign_analysis_final   -- output of the main query above
),

tier_summary AS (
    SELECT
        spend_tier_label,
        assignment,
        COUNT(*)                                             AS n,
        AVG(campaign_spend)                                  AS avg_spend,
        SUM(responded) * 1.0 / COUNT(*)                     AS response_rate
    FROM campaign_results
    GROUP BY spend_tier_label, assignment
),

-- Pivot to get test vs holdout side by side
pivoted AS (
    SELECT
        spend_tier_label,
        MAX(CASE WHEN assignment = 'test'    THEN avg_spend END) AS test_spend,
        MAX(CASE WHEN assignment = 'holdout' THEN avg_spend END) AS holdout_spend,
        MAX(CASE WHEN assignment = 'test'    THEN response_rate END) AS test_rr,
        MAX(CASE WHEN assignment = 'holdout' THEN response_rate END) AS holdout_rr
    FROM tier_summary
    GROUP BY spend_tier_label
)

SELECT
    spend_tier_label,
    ROUND(test_spend, 2)                                     AS test_avg_spend,
    ROUND(holdout_spend, 2)                                  AS holdout_avg_spend,
    ROUND((test_spend - holdout_spend) / holdout_spend * 100, 1) AS lift_pct,
    ROUND(test_rr * 100, 1)                                  AS test_response_rate_pct,
    ROUND(holdout_rr * 100, 1)                               AS holdout_response_rate_pct
FROM pivoted
ORDER BY lift_pct DESC;
```

---

## Files

```
├── data/
│   └── sample_campaign_data.csv
├── notebooks/
│   └── incrementality_testing.ipynb
├── sql/
│   ├── extract_campaign_cohorts.sql
│   └── post_campaign_tier_breakdown.sql
├── templates/
│   └── post_campaign_report.xlsx
└── README.md
```

---
*Aman Patel · [Portfolio](https://aman-analytics.github.io) · [LinkedIn](https://linkedin.com/in/amanpatel028)*
