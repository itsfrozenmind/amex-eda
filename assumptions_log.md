# Amex Profitability Model — Assumptions & Decisions Log
Last updated: 2026-07-07 23:26 IST (Part 1 follow-up — deeper correlation/behavioral-hypothesis pass)

> **Maintenance rule:** this file is append-only history — never delete or rewrite a past row. When a decision changes, add a new row noting the supersession and update the "Last updated" timestamp above. Every script run, threshold choice, or finding that Part 2 might rely on gets logged here before moving on.

## 1. Data Cleaning Decisions
| Column | Issue | Decision | Rationale | Date/Step |
|---|---|---|---|---|
| f4 (Rewards Points Balance), f21 (Rewards Redeemed) | 51.4% missing, co-missing 100% of the time | Impute 0 + add `has_rewards_program` flag | Perfectly co-missing pair; Total Spend/Active Cards still present for these rows → looks like "no rewards program on this product," not a data gap | 2026-07-07, Section B/H |
| f17 (Total Lend Line), f18 (Consumer Lend Line) | 58.4% / 61.9% missing | Impute 0 + add `has_lend_line` / `has_consumer_lend_line` flags | f18 missing whenever f17 missing, plus 17,190 extra NaNs beyond f17 → likely charge-card-only (pay-in-full) customers with no revolving lend line | 2026-07-07, Section B/H |
| f6–f10 (Airlines/Other/Entertainment/Lodging/Dining Spend) | 23.1% missing, co-missing exactly together (115,698 rows) | Impute 0 + add `spend_breakdown_available` flag | Total Spend (f5) still present for these rows → category breakdown untracked, not zero spend | 2026-07-07, Section B/H |
| f13–f16 (Lounge Access, Airline Credits, Cab Benefits, Entertainment Credit) | 2.7% missing, co-missing exactly together (13,716 rows) | Impute 0, no flag | No structural link found to a product-tier column (checked against f20 Active Charge Cards) → treated as genuine small random gap | 2026-07-07, Section B/H |
| f22 (Emails Opened) | 18.9% missing | Impute 0 + add `has_email_on_file` flag | Likely no email/opt-out on file for this segment | 2026-07-07, Section B/H |
| f23 (Emails Clicked) | 87.8% missing | Impute 0, **no flag** | Confirmed structural zero: always NaN when Opens=0; when Opens>0 and present, values are always ≥1, never 0 → NaN reliably means 0 clicks | 2026-07-07, Section B/H |
| f5, f11, f12 (Total Spend, Risk Score, Login Counts) | ≤5% missing each | Impute with column median | No structural link to any other column found → genuine random gap | 2026-07-07, Section B/H |
| f19, f20 (Supplementary Accounts, Active Charge Cards) | ≤0.02% missing each | Impute with column mode | Negligible volume, random | 2026-07-07, Section B/H |
| f2, f3 (Cancellation Calls, Collection Calls) | Discovered to be binary (0/1) flags, not counts | **Left as-is, unresolved** — see Open Questions #1 | 54,205 rows (10.8%) have Collection=1 with Cancellation=0, breaking the assumed subset relationship; needs a definitional decision before use | 2026-07-07, Section C |
| All 23 feature columns | 97 rows fully duplicate across all features when `id` excluded | **Left in dataset, flagged** — see Open Questions #3 | Could be coincidental given continuous features at 500K rows, or synthetic/padded records; not dropped without confirmation | 2026-07-07, Section A |

## 2. Thresholds & Magic Numbers
| Value | Used for | Reasoning | Sensitivity tested? |
|---|---|---|---|
| tol = 1.0 | Rounding tolerance when testing if f6–f10 sum to f5 (Total Spend) | Standard float-rounding allowance for currency-scale sums | No — moot once corr(sum, f5) = 0.099 showed they aren't components at all |
| \|corr\| > 0.6 | Flagging "redundant" column pairs in the correlation matrix (Section G) | Conventional cutoff for "strong" correlation risking double-counted signal | No — not swept across alternative cutoffs (e.g. 0.5, 0.7) yet |
| >10x the 99th percentile | Flagging single-row extreme outliers (Section D) | Task-list-specified rule of thumb for "likely data error, not just a big spender" | N/A — result was 0 rows for every column, superseded by the capping finding below |
| Row exactly at column max (0% tolerance) / within 0.1% of max | Detecting hard-capping/winsorization (Section D) | Used to confirm p99 = p99.9 = max wasn't a fluke of quantile interpolation | Checked at both 0% and 0.1% bands — counts nearly identical, confirms a real cap, not a coincidence |
| Exact-match duplicate rows (all 23 features, `id` excluded) | Detecting cross-feature duplicate customers (Section A) | Strictest possible duplicate definition | No fuzzy/near-duplicate threshold tested yet |
| \|Spearman − Pearson\| > 0.15 | Flagging pairs where the linear (Pearson) matrix understates a monotonic relationship (2026-07-07 follow-up) | Chosen as a "meaningfully different" cutoff, not derived from a formal test | No — arbitrary band, not swept |
| Engagement score = sum of z-scores(f12, f22, benefit_usage_count), split at median / quartiles | Silent-churn hypothesis test — defining "high" vs "low" engagement among f2=0 customers | Simple unweighted composite so no single proxy dominates | No — not tested against alternative weightings of the three components |
| benefit_usage_count = count of {f13,f14,f15,f16} that are > 0 | Input to the engagement score above | Treats "used the benefit at all" as the signal, ignoring usage magnitude | No |
| Max single-category share ≥ 80% of f6–f10 sum | Proxy for "concentrated"/possible one-off spend (spend-volatility check) | Arbitrary round-number cutoff | No — and see caveat in Section 7 about this proxy's limits |

## 3. Feature Engineering Decisions
| Feature(s) | Transformation | Reasoning | Alternatives considered |
|---|---|---|---|
| f4, f21, f17, f18, f6–f10, f22 | Added 5 binary availability flags (`has_rewards_program`, `has_lend_line`, `has_consumer_lend_line`, `spend_breakdown_available`, `has_email_on_file`) before zero-imputing | Preserve the "not applicable" signal that would otherwise be silently destroyed by imputing to 0 | Considered leaving as NaN for downstream model to handle natively — rejected because Part 2 scoring logic is unlikely to be NaN-aware by default |
| f1, f4, f6, f8, f9, f11, f21 | **Not yet applied** — flagged as log-transform candidates | Skew 1.4–2.8 (strongly right-skewed) in raw distribution | Deferred to Part 2 / feature-engineering phase; no transform code written yet |
| f16 (Entertainment Credit Used) | **Not yet decided** — flagged for special handling | Capped on 35% of rows (vs. ~2.6% for everything else) and floored at 8.88 (never 0) → behaves more like a bounded/near-binary variable than a normal spend amount | None yet — needs its own investigation in Part 2 |
| f6–f10 as "Total Spend breakdown" | **Rejected** as a modeling assumption | corr(sum(f6:f10), f5) = 0.099, and subcategory sums average ~13x larger than Total Spend — these are independent features on a different scale | Original task-list framing assumed they *were* a breakdown; discarded after the correlation check (see Section 8, Rejected Approaches) |
| `engagement_score` (z-score composite of f12 + f22 + benefit_usage_count) | **Experimental, not yet in `dataset_clean.csv`** — used only to test the silent-churn hypothesis | Needed a single "disengagement" proxy since no ground-truth engagement label exists | Considered using f12 (logins) alone — rejected because it ignores email and benefit-usage signal, which showed the effect too |
| `benefit_usage_count` (count of f13–f16 used at all) | Experimental, input to engagement_score | Simpler and more robust to f16's capping/floor quirks than summing raw amounts | Considered summing raw benefit amounts — rejected due to f16's very different scale swamping the others |

## 4. Correlation Findings
| Pair | Correlation | Interpretation | Action taken |
|---|---|---|---|
| f17 (Total Lend Line) / f18 (Consumer Lend Line) | 0.90 | Near-duplicate signal | Flagged as double-counting risk for Part 2 scoring; not dropped yet, pending Open Question #2 |
| f6 (Airlines Spend) / f9 (Lodging Spend) | 0.70 | Moderately redundant within the spend-subcategory cluster | Flagged, no action taken |
| f7 (Other Spend) / f10 (Dining Spend) | 0.66 | Moderately redundant within the spend-subcategory cluster | Flagged, no action taken |
| f8 (Entertainment Spend) / f10 (Dining Spend) | 0.62 | Moderately redundant within the spend-subcategory cluster | Flagged, no action taken |
| f3 (Collection Calls) / f11 (Risk Score) | 0.58 | Just under the 0.6 cutoff but notable — collection calls track with risk score as expected | Not flagged as "redundant" per the 0.6 rule, but worth a look given the f2/f3 inconsistency in Section 1 |
| f2 (Cancellation Calls) / f16 (Entertainment Credit Used) | -0.52 | Just under threshold; moderate negative relationship, unexplained | Not investigated further yet |
| sum(f6:f10) / f5 (Total Spend) | 0.099 | Essentially unrelated — overturned the assumption that f6–f10 are a Total Spend breakdown | Recommendation revised: f6–f10 imputation treated as independent features, not components of a known total |
| f4 (Rewards Balance) / f21 (Rewards Redeemed) | 0.15 | Confirmed to carry distinct signal, not redundant, despite both being "rewards"-named columns | No action needed — kept as separate features |
| **CONFIRMED, high confidence:** f11 (Risk Score) / f3 (Collection Calls) | Pearson 0.58, Spearman 0.45 | f11 behaves per its name — higher risk score, more collection calls | No further action, documentation only |
| **CONFIRMED, high confidence:** f11 (Risk Score) / f17 (Total Lend Line) | Pearson -0.30, Spearman -0.35 | Riskier customers get smaller lend lines, as expected | No further action, documentation only |
| **CONFIRMED, high confidence:** f11 (Risk Score) / f18 (Consumer Lend Line) | Pearson -0.32, Spearman -0.38 | Same direction as f17, consistent | No further action, documentation only |
| f1 (Revolve Balance) / f11 (Risk Score) | **Pearson 0.196 vs Spearman 0.577** (largest Pearson/Spearman divergence in the matrix) | Pearson understates this relationship badly — driven by f11's 33% mass at exactly 0. Binned check (2026-07-07): mean f1 rises monotonically from ₹429 (f11=0) → ₹1,346 → ₹3,937 → ₹5,178 across risk bins | Treat f1 vs f11 as a genuinely strong monotonic relationship for Part 2, not the "weak 0.2" the Pearson number implies |
| f4 (Rewards Balance) / f21 (Rewards Redeemed) | **Pearson 0.15 vs Spearman -0.036** | Sign essentially flips under rank correlation — the weak positive Pearson number looks driven by a handful of extreme-value rows, not a real monotonic relationship | Reinforces Section 4's existing "independent, not redundant" conclusion — even more strongly now |
| f7 (Other Spend) / f8 (Entertainment Spend) | Pearson 0.57, **Spearman 0.61** (newly crosses the 0.6 redundancy threshold) | Spend-subcategory cluster is more tightly coupled under rank correlation than Pearson suggested | Add to the redundancy watch-list alongside the 4 pairs already flagged |
| f9 (Lodging Spend) / f10 (Dining Spend) | Pearson 0.56, **Spearman 0.61** (newly crosses the 0.6 redundancy threshold) | Same cluster effect as above | Add to the redundancy watch-list |
| f2 (Cancellation Calls, voluntary) / f16 (Entertainment Credit Used) | Pearson -0.52, Spearman -0.46 | **Loss-aversion hypothesis supported for f16 specifically** — voluntary cancellation correlates more strongly with this benefit than f3 does (-0.40) | See Section 9, Behavioral Hypotheses |
| f2 / f13, f14, f15 (Lounge, Airline Credits, Cab benefits) | -0.08, 0.00, -0.11 respectively | Weak — loss-aversion effect does **not** extend to these three benefits the way it does to f16 | See Section 9 |
| f3 (Collection Calls, involuntary/risk-driven) / f13, f14, f15, f16 | -0.14, -0.17, -0.22, -0.40 respectively | **All four benefits correlate more strongly with f3 than with f2**, except f16 (where f2 is stronger) — suggests f13/f14/f15's link to "calls" is a general risk-profile artifact, not a voluntary-retention signal | See Section 9 |

## 5. Attribute Classification (Sign & Weight)
| Attribute | Category | Sign | Weight tier | Confidence (High/Med/Low) | Reasoning |
|---|---|---|---|---|---|
| *(not yet started — this table is Part 2 scope; EDA/cleaning in Part 1 did not assign sign/weight to any attribute)* | | | | | |

## 6. Formula Version History
| Version | Formula/logic summary | Public LB score | Notes on what changed vs. previous |
|---|---|---|---|
| *(not yet started — no scoring formula has been written; Part 1 was EDA/cleaning only)* | | | |

## 7. Open Questions / Unresolved Items
| Question | Status | Owner (human/CC) |
|---|---|---|
| f2/f3 binary-flag inconsistency: 54,205 rows (10.8%) have a Collection call (f3=1) with no Cancellation call (f2=0) — data error, or are these genuinely independent signals (e.g. proactive collections outreach unrelated to a customer-initiated cancellation call)? | Open | Human |
| f17 vs f18 subset violation: 36.5% of overlapping rows have Consumer Lend Line (f18) > Total Lend Line (f17). Which column is authoritative, and should f18 be capped at f17? | Open | Human |
| 97 cross-feature duplicate rows (excluding `id`): keep as-is, or investigate as synthetic/padded records? | Open | Human |
| ~2.6% of rows sit exactly at the max value in every continuous column (35% for f16) — should Part 2 treat these as censored/capped (e.g. a `*_at_cap` flag) rather than as ordinary large values? | Open | Human + CC |
| f19 (Supplementary Accounts) > f20 (Active Charge Cards) in 42% of rows — is this expected (multiple supplementary cardholders per active account) or worth flagging? | Open, leaning "expected" | Human |
| Should f1, f4, f6, f8, f9, f11, f21 be log-transformed before standardization, given skew 1.4–2.8? | Open | CC (Part 2) |
| How should f16 (Entertainment Credit Used) be treated given its unusual 35%-at-cap / floor-at-8.88 behavior? | Open | CC (Part 2) |
| ~~Is the f6-f10-vs-f5 gap a scale/window mismatch or an outlier artifact?~~ | **Resolved (2026-07-07):** neither — row-level ratio spans orders of magnitude (median 10.6x, range up to 14.4M x) with only 3.84% of rows roughly 1:1; the gap is pervasive across the whole distribution, confirming f6–f10 are simply unrelated to f5 for every customer, not a fixed-ratio mismatch or a tail-driven artifact | CC (Part 2) |
| Should `engagement_score` (login + email + benefit-usage composite) become a formal Part 2 feature, given it shows an 81.5% risk-score gap between engagement quartiles among non-callers? | Open | CC (Part 2) |
| Category-spend concentration (max_share ≥80% of f6-f10) can't distinguish a one-off "milestone" spend event from a durable category preference without sub-period data — is this proxy worth using at all in Part 2, with that caveat? | Open | Human |
| Should the redundancy watch-list (Section 4) be widened to include f7/f8 and f9/f10, which only cross 0.6 under Spearman, not Pearson? | Open | CC (Part 2) |

## 8. Rejected Approaches
| Approach | Why rejected | Could revisit in Part 2? |
|---|---|---|
| Treating f6–f10 as a literal breakdown of f5 (Total Spend), per the original task-list framing | corr(sum(f6:f10), f5) = 0.099 and subcategory sums average ~13x larger than Total Spend — the numbers flatly contradict the assumption | No — this should be considered settled unless new data changes the picture |
| Flagging outliers via ">10x the 99th percentile" as originally scoped | Returned 0 rows for every continuous column, because every column is already hard-capped (p99 = p99.9 = max) — the rule can't fire on this dataset as constructed | No — superseded by the capping/censoring finding; a `*_at_cap` flag approach is the live alternative (see Open Questions) |
| Assuming missingness in f4/f21/f17/f18/f6–f10/f22 was random and imputable via median/mean | Cross-tabs showed perfect co-missingness within groups and no "zero activity" signal in related columns (e.g. Total Spend still present) → pointed to structural "not applicable," not randomness | No — structural-zero + flag approach is the adopted method |
| **Family Approval Anchor hypothesis, as originally stated** (more supplementary accounts → lower cancellation-call rate, justifying a stronger positive weight) | **Disproven (2026-07-07):** cancellation rate actually *rises* monotonically with f19 — 16.08% (f19=1) → 18.55% (f19=2) → 18.57% (f19=3) → 19.28% (f19=4), a 17.4% relative *increase* from f19=1 to f19≥3, the opposite of the hypothesized direction | Could revisit with a *reversed*-sign or null weight instead of the originally proposed stronger-positive weight — see Section 9 |

## 9. Behavioral Hypotheses (Psychology-Grounded)
| Hypothesis | Grounding principle | Feature(s) involved | Status | Test result |
|---|---|---|---|---|
| Benefit usage creates loss-aversion lock-in, not just cost | Loss Aversion Mechanics | f13–f16 vs f2 | **Partially confirmed** | Holds specifically for f16: f2 vs f16 = -0.52 (Pearson) / -0.46 (Spearman), noticeably stronger than f3 vs f16 (-0.40). Does **not** extend to f13/f14/f15 — their correlations with f2 are weak (-0.08, 0.00, -0.11) and are actually *stronger* with f3 (involuntary/risk-driven calls) than with f2. Treat f16 as the genuine loss-aversion signal; f13-f15's relationship to cancellation looks more like a general risk-profile artifact. See Section 4. |
| Supplementary accounts = family embeddedness, stronger retention than spend-multiplier | Family Approval Anchor | f19 vs f2 | **Rejected** | Cancellation rate *increases* with f19 (16.08% at f19=1 → 19.28% at f19=4, +17.4% relative from f19=1 to f19≥3) — the opposite of the hypothesized direction. See Section 8. |
| Cancellation calls undercapture true churn risk; disengagement is the honest signal | Fear of Admitting Feedback (silent churn) | f2 vs f12/f22/f23 (+ benefit usage) | **Supported** | Among f2=0 (no cancellation call) customers, the bottom-quartile-engagement group (composite of f12 logins + f22 email opens + benefit-usage count) has an **81.5% higher mean Risk Score (f11)** and elevated Revolve Balance (f1) vs. the top-quartile group. Disengagement looks like a real, independent negative signal not captured by f2 alone. `engagement_score` is experimental (not yet in `dataset_clean.csv`) — see Open Questions, Section 7. |
| Lounge/lifestyle credits carry status value beyond dollar cost | Bragging Right Engine, Hyper-Local Envy | f13, lifestyle credits | **Untestable directly (no peer data)** — treat as directional adjustment only | Not tested this round; f13 (Lounge Access) does correlate weakly with f2 (-0.08) and moderately with f3 (-0.14), but neither test speaks to "status value," only to call propensity. No peer/social-comparison data exists in this dataset to test the mechanism itself. |
| Single-category spend spikes may be one-off events, not durable profitability | Milestone Wealth Liquidation | f8, f9, f10 | **Untestable as "spikes" — only a concentration proxy available** | No sub-period (monthly/quarterly) columns exist anywhere in the 23-feature schema, so a temporal spike can't be measured. Proxy computed instead: 32.4% of customers have one f6-f10 category ≥80% of their category mix — but this measures cross-sectional concentration, not a spike vs. their own baseline, so it cannot distinguish a milestone event from a durable preference (e.g., someone who always mostly spends on dining). See Section 2 (thresholds) and Section 7 (open question on whether to use this proxy at all). |
