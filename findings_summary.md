# EDA & Cleaning — Findings Summary

**Dataset:** 500,000 rows × 23 features + id. All columns numeric (no dtype issues). No duplicate ids; no fully-duplicate rows. **97 rows are duplicate across all 23 features when `id` is excluded** — flag for manual review (coincidence vs. synthetic padding).

Outputs: `dataset_clean.csv` (500,000 rows × 29 cols — 23 original + id + 5 new availability flags, zero rows dropped), `correlation_matrix.png`, `dist_histograms_boxplots.png`, `low_cardinality_barcharts.png`.

## 1. Missingness — decisions taken
Five groups of columns are **missing together in perfect lockstep** (same rows every time), which is the strongest evidence their NaNs are structural ("not applicable"), not random data gaps:

| Group | Columns | % missing | Decision |
|---|---|---|---|
| Rewards | f4 (Balance), f21 (Redeemed) | 51.4% | Impute 0 + `has_rewards_program` flag |
| Lend line | f17 (Total), f18 (Consumer) | 58.4% / 61.9% | Impute 0 + `has_lend_line`/`has_consumer_lend_line` flags (f18 has 17,190 extra NaNs beyond f17) |
| Spend subcategories | f6, f7, f8, f9, f10 | 23.1% | Impute 0 + `spend_breakdown_available` flag |
| Benefit usage | f13, f14, f15, f16 | 2.7% | Impute 0, no flag (no link found to a product-tier column, treated as genuine small gap) |
| Email | f22 (Opens), f23 (Clicks) | 18.9% / 87.8% | f22: impute 0 + `has_email_on_file` flag. f23: impute 0, **no flag needed** — confirmed structural zero (always NaN when Opens=0; when present, values are always ≥1, never 0) |

Remaining columns (f5, f11, f12, f19, f20; ≤5% missing each) showed no structural link to any other column — imputed with median (f5, f11, f12) or mode (f19, f20) as ordinary random gaps.

**Net effect:** zero missing values remain in `dataset_clean.csv`, and the 5 new flag columns preserve the "not applicable" signal so it isn't silently destroyed by the 0-imputation.

## 2. Range & sanity issues
- **f2/f3 are binary flags (0/1), not counts.** 54,205 rows (10.8%) have a Collection call (f3=1) with **no** Cancellation call (f2=0) — breaks the expected "collection calls are a subset of cancellation calls" assumption. **Needs a definitional decision before use** — left uncorrected in the clean file.
- f7 (Other Spend) has 22,451 negative values, tightly bounded (−274.65 to ~0) — consistent with refunds, not errors. No other spend column has negatives.
- f11 (Risk Score) ranges 0–0.326 → treat as a 0–1 probability-type score, not 0–100.
- f16 (Entertainment Credit Used) never reaches 0 (floor of 8.88) — likely a minimum allocated credit, not an anomaly.

## 3. Outliers — the data is capped, not organically tailed
Every continuous column has **~2.6% of rows sitting exactly at its max value** (f16 is capped on 35% of rows). Consequently p99 = p99.9 = max for all of them, and **zero rows exceed 10x the 99th percentile** anywhere. This means the dataset appears winsorized/clipped at generation time. Recommendation: treat the at-cap rows as **censored**, not as natural large values — consider a `*_at_cap` indicator per column if these features drive scoring, since a plain log/standardize will understate how extreme the true tail may be.

Skew: f1, f4, f6, f8, f9, f11, f21 are strongly right-skewed (1.4–2.8) → candidates for log-transform. f17/f18 are only mildly skewed (0.5–0.6) → fine as-is.

## 4. Internal consistency — overturns an assumption
- **f6–f10 (spend subcategories) do NOT sum to f5 (Total Spend)** — correlation is only 0.099, and subcategory sums average ~13x larger than Total Spend. These are **not a breakdown of Total Spend**; treat as independent features on their own scale, not components.
- **f17 vs f18 (Lend Line pair): corr = 0.90 (near-duplicate)**, but 36.5% of overlapping rows have Consumer Lend Line (f18) > Total Lend Line (f17), violating the assumed subset relationship — needs a definitional check.
- **f4 vs f21 (Rewards Balance vs Redeemed): corr = 0.15** — confirmed independent signals, not redundant. 37.5% of rows show redeemed > balance, plausibly due to different rolling-window definitions.
- **f19 (Supplementary Accounts) > f20 (Active Charge Cards) in 42% of rows** — not necessarily an error (several supplementary cardholders can sit under one active account), but flagged for confirmation before treating as anomalous.

## 5. Redundant columns (|corr| > 0.6)
| Pair | Corr |
|---|---|
| f17 (Total Lend Line) / f18 (Consumer Lend Line) | 0.90 |
| f6 (Airlines Spend) / f9 (Lodging Spend) | 0.70 |
| f7 (Other Spend) / f10 (Dining Spend) | 0.66 |
| f8 (Entertainment Spend) / f10 (Dining Spend) | 0.62 |

f17/f18 is the clearest double-counting risk for scoring — consider using only one, or a combined feature. The f6–f10 cluster is moderately intercorrelated (0.5–0.7 among themselves) despite not summing to Total Spend, so weighting all five independently may overweight a shared "spend propensity" signal.

## 6. Open items needing a human decision before scoring logic is written
1. f2/f3 binary-flag inconsistency (54,205 rows) — data error or two independent signals?
2. f17 vs f18 subset violation (36.5% of overlap rows) — which one is authoritative?
3. 97 cross-feature duplicate rows (excluding id) — keep, or investigate as synthetic artifacts?
4. Capped/censored columns — should scoring treat the ~2.6% at-cap rows specially?
