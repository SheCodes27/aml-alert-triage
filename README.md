# aml-alert-triage

# AML Alert Triage: Ranking Transactions by Laundering Risk

**Result:** Reviewing the riskiest 1% of transactions (about 15,000 of 1.5M)
catches 70% of the laundering in the test period. A "biggest amounts first"
rule catches 2%. About 1 in 9 of those alerts is real laundering.

![Gains chart](gains_chart.png)

## The problem
Banks get far more suspicious-transaction alerts than investigators can check,
and most are false alarms. The order of review matters.

## What I did
I asked: **does looking at an account's history and network, not just a single
payment, help rank laundering higher?** I trained on the first 7 days and tested
on the later 11 days the model had never seen.

- **Data:** IBM synthetic AML dataset (HI-Small), 5.1M transactions, 0.1% laundering
- **Model:** LightGBM, trained on all laundering cases plus a sample of
  innocent transactions
- **Metric:** share of laundering caught when reviewing the top X% of alerts
  (not accuracy, since 99.9% of transactions are innocent)

| Version | Clues | Top 1% | Top 5% | Top 10% |
|---|---|---|---|---|
| V1 | Single transaction (amount, hour, payment type) | 41% | 78% | 88% |
| V2 | + account history (past activity, how unusual the amount is) | **70%** | 84% | 90% |
| V3 | + network clues (pass-through speed, money looping back) | 72% | 85% | 90% |

**Findings**
- Account history was the big step: top-1% catch went from 41% to 70%.
- Network clues added almost nothing (V3 had a lower PR-AUC than V2:
  0.249 vs 0.287). My network clues were simple. Following money across
  several accounts is the next thing to try.
- Amount alone is useless here: a "biggest amounts first" rule catches 2%
  in the top 1%.

## Where it struggles
- **Laundering outside the listed schemes.** About 28% of laundering cases could
  not be matched to a scheme in the dataset's pattern file. These are smaller
  (median about 2,700 vs 13,100) and use more payment types (cheque, card,
  cash, Bitcoin). The model catches 40% of them in the top 1%, versus 73-89%
  for the listed schemes. Matching was done on time, accounts and amount, so some
  of the "unmatched" group may be a matching gap.
- **A cheque shortcut.** All 224 cheque alerts in the top 1% were false
  alarms, and cheques filled half of the top 20. I found this after seeing the
  test results, so I did not tune it away.
- **Low precision.** About 89% of top-1% alerts are false alarms. This saves
  investigator time but does not replace investigators.

## Limitations
- Synthetic data. Real bank data is messier, so real results would be lower.
- One run, no repeats. The V2 vs V3 gap is small and could be noise.
- Scores single transactions, not whole cases or accounts.
- Importance scores show what the model relies on, not proof of wrongdoing.

## Next steps
Multi-hop network clues, scoring accounts instead of transactions, fixing the
cheque cluster with a held-out check, repeating runs with different seeds.
