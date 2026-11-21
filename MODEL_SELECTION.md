# Model selection

Batch `573dc57` (the commit that trained it), data `data/churn_v1.csv` with hash `8685886ae7ce`: 1,200 customers, 29.6% churn.
Test set: 240 customers (20%, stratified), 71 of them churners. Slices are the contract type.

## The question
The model flags customers who are likely to leave, so the retention team can call them or offer a discount first.
A missed churner is a lost customer; a false alarm costs a discount or a call that was not needed. Missing churners hurts more.

## Candidates
| Candidate | Why I tried it | AUC | F1 | F1 per slice (lowest) | p95 latency (ms) | Size (KB) | MLflow run ID |
|---|---|---|---|---|---|---|---|
| `logreg` | The Lab 2 model, kept as the baseline to beat | 0.783 | 0.512 | 0.00 (two-year) | 7.1 | 4 | `ea898d73` |
| `logreg_balanced` | Same linear model, but `class_weight="balanced"`: with 30% churn, a missed churner should count as much as a false alarm | 0.782 | **0.619** | **0.52** (one-year) | 7.7 | 4 | `c349cbd6` |
| `random_forest_balanced` | Bagged trees can learn interactions (fiber plus many support calls); 100 trees, depth 6, kept small for latency | 0.760 | 0.568 | 0.31 (two-year) | 18.4 | 651 | `3a119d86` |
| `hist_gradient_boosting` | Boosted trees, usually the strongest family on tabular data; shallow trees (depth 3), slow learning rate | 0.768 | 0.577 | 0.26 (one-year) | 15.4 | 185 | `b6103686` |

The three new candidates all use balanced class weights, so they differ in model family, not in how they treat the imbalance.

## The champion
**`logreg_balanced`** (MLflow run `c349cbd62c8948629560274d53c6dc4b`, `churn-model` version 1, alias `champion`).

- It has the best F1 overall (0.619) and the best F1 on its weakest slice (0.52). It is also the fastest and the smallest.
- The baseline shows why the slice view matters: its AUC is the same (0.783), but its F1 on two-year customers is **0.00**.
  It caught 0 of the 10 two-year churners in the test set. `logreg_balanced` catches 4 of 10, and 52 of all 71 churners (baseline: 31).
- The tree models were no better on any slice that matters, and they are 2× slower and 46 to 160× larger.

What I accepted losing:
- **More false alarms.** 45 against 19 for the baseline, so precision drops from 0.62 to 0.54. That is 26 more unneeded offers to catch 21 more churners, which I think is a good trade for retention.
- **0.001 AUC.** It ranks customers just as well; it just uses the 0.5 threshold better.
- **No interactions between features**, which the tree models could learn. On this data they did not turn that into better F1.

## My promotion rules
- **Latency budget (p95): 30 ms**, because the score is requested while an agent is looking at the customer, so the whole API should answer within about 100 ms. 30 ms for the model leaves room for HTTP, validation and JSON. The champion measures 7.7 ms on my laptop and 4.5 ms on the GitHub runner. The same model's timing changes from machine to machine and from run to run (the random forest: 18.4 ms on my laptop, 12.7 ms in CI), so I keep about a 4× margin instead of setting the budget just above today's number.
- **Minimum gain to replace the champion (F1 points): 0.02**, because with 71 churners in the test set, one extra churner caught moves F1 by only 0.008. A gain below 0.02 means fewer than about 3 customers changed, which a different random split could undo. Every promotion means a new image and a redeploy, so the champion should only change for a real improvement, not for noise.

## What I would check next
The small slices. Two-year customers are 59 of the 240 test rows, with only 10 churners, so one customer moves that slice's F1 by about 0.1. The 0.52 to 0.53 on one-year and two-year rests on a handful of people.
Before I trust it, I would check it with cross-validation (five splits instead of one) and on the newer data (`churn_v2.csv`), in case the churn pattern of long-contract customers changes.
