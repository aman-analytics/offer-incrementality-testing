# Offer Incrementality & A/B Testing Framework

**Tools:** Python · SQL · Excel · SciPy  
**Domain:** CRM / Campaign Analytics  
**Status:** Complete

---

Most campaign reports I inherited were measuring response rate — which sounds right until you realise it includes everyone who would have bought anyway. A 30% response rate on a cashback offer looks great. But if 22% of those people were already going to spend that month regardless, your actual incremental lift is a lot smaller than the headline number suggests.

This framework was built to answer the only question that actually matters: *did the offer cause the behaviour, or did it just show up while the behaviour was already happening?*

---

## Framework Design

### Test / Holdout Split

The key is making sure test and holdout groups look the same before the campaign starts. Random splits on unbalanced populations give you noisy results.

```python
import pandas as pd
import numpy as np

def create_test_holdout(df, test_size=0.80, seed=42):
    np.random.seed(seed)
    
    # Stratify by baseline spend tier — ensures groups are comparable
    df['spend_tier'] = pd.qcut(df['baseline_spend'], q=4, labels=['L','M','H','VH'])
    
    test    = df.groupby('spend_tier', group_keys=False).apply(
                  lambda x: x.sample(frac=test_size))
    holdout = df[~df.index.isin(test.index)]
    
    return test, holdout
```

### Incrementality Calculation

```python
from scipy import stats

def measure_lift(test_df, holdout_df, metric='revenue'):
    test_rate    = test_df[metric].mean()
    holdout_rate = holdout_df[metric].mean()
    
    lift   = (test_rate - holdout_rate) / holdout_rate * 100
    t_stat, p_value = stats.ttest_ind(test_df[metric], holdout_df[metric])
    
    return {
        'incremental_lift_pct': round(lift, 2),
        'incremental_units':    round((test_rate - holdout_rate) * len(test_df)),
        'p_value':              round(p_value, 4),
        'significant':          p_value < 0.05
    }
```

### Sample Output

```
Offer:               15% cashback — dining category
Test Group:          48,200 customers
Holdout Group:       12,050 customers

Test Response Rate:      34.2%
Holdout Response Rate:   19.7%
─────────────────────────────
Incremental Lift:        +73.6%
Incremental Responders:  6,986
Cost Per Incremental:    $4.82
P-Value:                 0.0012  ✅

Verdict: Scale this offer. Strong signal.
         Refine targeting toward mid-spend, low-redemption segment.
```

---

## Results Across Campaigns

| Campaign | Apparent ROI | True Incremental ROI | What Changed |
|---|---|---|---|
| Dining Cashback | 22% | 38% | Scaled budget 2x |
| Seasonal Promo (Reebok) | 18% | 33% | Narrowed targeting |
| Loyalty Points Bonus | 31% | 41% | Kept as-is |
| New Arrival Push (Reebok) | 14% | 21% | Restructured offer |

The Reebok seasonal promo was the clearest win — the apparent ROI looked mediocre, which was about to get the offer killed. Incrementality testing showed it was actually driving real behaviour change, just in a segment that wasn't obvious from the surface numbers. Saved the offer and improved targeting to push ROI from 18% to 33%.

---

## Files

```
├── data/
│   └── sample_campaign_data.csv    # Synthetic test/holdout dataset
├── notebooks/
│   └── incrementality_testing.ipynb
├── sql/
│   └── extract_campaign_cohorts.sql
├── templates/
│   └── post_campaign_report.xlsx
└── README.md
```

---

*Aman Patel · [Portfolio](https://aman-analytics.github.io) · [LinkedIn](https://linkedin.com/in/amanpatel028)*
