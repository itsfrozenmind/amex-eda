# v2 Formula — Bottom-to-Top Rebuild: Assumptions for Review
Date: 2026-07-08. Written before implementation so the reasoning can be checked, not just the result.

## Why a rebuild, not another increment
Six rounds of sign-flips and variable-inclusion tests plateaued in the 0.35-0.365 range, with one regression:

| Version | Score | Change |
|---|---|---|
| v1.1 | 0.175 | initial |
| v1.2 | 0.275 | f1 sign flip |
| v1.3 | 0.289 | exponent shape + f4 |
| v1.4 | 0.352 | f6-f10 added + f19 flip |
| v1.5 | **0.365** | M2 flip + f21 added (best so far) |
| v1.6 | 0.357 | f12 + f20 added (regression) |

The gap to competitive scores (~0.94) is ~0.58 — too large to close with more of the same kind of change. Every v1.x round kept the same skeleton (pctrank-weighted linear sum, risk as a smooth power-curve penalty, ~9 terms of roughly comparable weight) and only adjusted individual signs/inclusions. If the skeleton itself is wrong, no amount of sign-tuning within it will reach 0.9. This document proposes a different skeleton and lists every assumption behind it explicitly, so each one can be checked or challenged before more submissions are spent.

## Assumption 1: This is a charge card, so spend (not interest) is the dominant profit driver
The official deck describes the Premier Card as "Flagship personal ultra-premium card... No preset spending limit (charge card)," targeting "high-income, frequent travelers." Charge cards are traditionally paid in full monthly; their primary issuer revenue is **interchange fees on spend volume**, not interest income. This project spent real effort (v1.2) establishing that revolve balance (f1) has a positive interest-revenue component — that finding isn't wrong, but it may have been **over-weighted relative to spend** in the resulting formula, since interest is a secondary revenue stream for a charge product, not the primary one.
**Implication:** f5 (Total Spend) and f6-f10 (industry-level spend) should carry substantially more combined weight than any other variable group — plausibly 55-65% of the total weight budget, not the ~35-50% they've had in v1.1-v1.6.

## Assumption 2: Risk should be a tail penalty, not a smooth curve across the whole population
This is a **pre-screened, approved population** for a $500-750/year premium product — Amex has already underwritten these customers once. Risk variance among *approved* premium cardholders is plausibly compressed relative to the general population, and the real cost of risk likely only bites hard for a small tail of customers approaching real delinquency/default, not gradually across the whole distribution.

**This isn't speculation — we already have direct empirical evidence for it from this project's own work** (Section 13/V2, decile-level analysis of the real inverse relationship between risk score and lend exposure): the pullback in lend exposure by risk decile is **flat and noisy through decile 8**, then crashes specifically between deciles 8→9→10 (55% of the total pullback happens in decile 9 alone, 100% by decile 10). We found this pattern *and confirmed cubic fits it better than squared*, but never went as far as testing what the data was actually telling us: a **smooth power curve applied to the whole population is the wrong shape entirely** — the real relationship looks like a step/threshold function that's ~zero for the bottom 80% and severe only in the top ~10-20%.
**Implication:** replace `pctrank(f11)^N` (which still applies *some* penalty at every percentile) with an explicit threshold: no risk penalty below roughly the 80th risk percentile, a moderate penalty between the 80th-90th, and a steep penalty above the 90th — matching the actual observed shape instead of approximating it with a smooth curve.

## Assumption 3: Engagement variables (f12, f19, f20) should carry minimal weight, regardless of official category
The official deck categorizes these under "Engagement," which is why v1.4 and v1.6 added/flipped them toward positive. But this project's own **row-level validation (Section 12)** found f19 has a 40.17% individual-customer contradiction rate, and general engagement composites (login+email+benefit) had 39.17% contradiction — both real relationships in aggregate, but far too weak to trust as strong independent weights. **v1.6's regression (f12+f20 added, score went down) is itself evidence this assumption tier was already over-weighted**, not under-weighted. Official category membership tells us *what kind* of variable something is, not *how much* it should move the needle — this project conflated the two starting at v1.4.
**Implication:** keep f12, f19, f20 in the formula (they're not wrong to include) but at much lower weight than v1.4-v1.6 gave them — small tie-breakers, not co-equal terms with spend/risk.

## Assumption 4: Rewards/benefit variables (f4, f21, benefit usage) are second-order relative to raw spend
A rewards points balance or a $250 cab credit redeemed is real money, but it's small relative to the interchange revenue from a high-income frequent traveler's total annual spend on a no-preset-limit charge card. These terms have gone back and forth in sign twice already (f4: excluded → positive; benefit usage: positive → negative) without a large score movement either way (+0.014 and +0.013) — consistent with them being real but minor factors, not major swing variables.
**Implication:** keep at a modest weight tier, meaningfully below the spend group, don't expect these to be where the next big jump comes from.

## Assumption 5: f2/f3 (calls) and f22/f23 (email) remain excluded
No new evidence this round changes their status: f3 is still near-redundant with f11 (Section 4/A1, R²=0.347), f2's row-level signal is weak, and f22 has an unresolved direction anomaly (rises with risk, unlike every other engagement variable). Not part of this rebuild.

## Proposed v2 weight structure (for review before implementation)
| Term | Weight | Tier |
|---|---|---|
| f5 (Total Spend) | 0.30 | Primary revenue |
| f6_f10_sum (industry spend) | 0.25 | Primary revenue |
| lend_exposure (has_lend_line-conditional) | 0.15 | Primary revenue (credit extended) |
| Risk penalty | **threshold, not smooth** — 0 below pctrank 0.80, −0.15 flat between 0.80-0.90, −0.35 flat above 0.90 | Tail-only cost |
| f1 (Revolve) | 0.10 | Secondary revenue (interest) |
| f4 (Rewards Balance) | 0.05 | Secondary (spend proxy) |
| f21 (Rewards Redeemed) | −0.05 | Secondary cost |
| M2 (benefit usage) | −0.03 max | Minor cost, conditional |
| f12 (Logins) | 0.05 | Minor tie-breaker |
| f19 / M3 (Supplementary Accounts) | +0.03, conditional | Minor tie-breaker |
| f20 (Active Cards) | 0.05 | Minor tie-breaker |
| M1 (lend exposure + risk tail override) | −0.20 | Explicit tail safeguard, kept as belt-and-suspenders alongside the new threshold risk penalty |

## What would tell us this is wrong
If v2 scores *below* v1.5's 0.365, the "risk-as-tail-only" and "spend-dominant" assumptions are likely wrong, or at least not the dominant lever — that would point back toward risk mattering more broadly than assumed, or toward a variable/mechanism not considered here at all (e.g., something in the still-fully-excluded f2/f3/f22/f23, or a completely different weighting philosophy). If v2 scores meaningfully better but still well short of 0.9, the direction is right but magnitude/shape calibration needs another pass (e.g., the exact threshold percentiles, or the exact weight split between f5 and f6_f10_sum). If it jumps close to 0.9, the core structural bet (spend-dominant, tail-only risk) was the missing piece all along.

---

# REVISION (same day, before implementation): the pctrank-threshold proposal above is superseded by a stronger hypothesis

## Assumption 0 (the master assumption the v1.x series never questioned): the hidden ground truth is a literal dollar P&L, not a normalized scorecard
Competitors at ~0.94 have nearly perfectly recovered a hidden profitability ranking. For overlap that high to be *achievable*, the ground truth must be a discoverable, simple function of these exact columns — and the problem statement says exactly what it is: *"design a framework or equation to quantify cardmember profitability to issuer **by incorporating revenues and costs**."* The most natural way for organizers to generate ground truth is to compute literal profit-in-dollars per customer from the synthetic columns: `Profit = Σ(revenue terms) − Σ(cost terms)` at raw scale.

Every v1.x formula was a **percentile-rank scorecard** with judgment-set weights (0.35/0.15/0.10...). Percentile ranking destroys magnitude information: a customer with ₹17,968 revolve balance and one with ₹2,441 both land "high" on pctrank(f1), but generate wildly different interest revenue in a dollar P&L. If the truth is dollar-scale, no reweighting of pctranks can fully recover its ranking — which would explain why six rounds of sign/weight adjustment plateaued at 0.35-0.37: **we were tuning inside the wrong function class.**

The deck supports this directly by handing us the actual unit economics:
- Interchange: premium cards earn ~2-3% of spend
- Interest: revolve balances earn ~20%+ APR (the card has "Plan It: installment flexibility" and a lend line, so lending revenue exists despite being a charge card)
- Rewards: "1 CC point ≈ 1-2 cents" — f21 (points redeemed) × ~1-1.5¢ is a literal cash cost
- Lifestyle credits: $15/mo cab credit (f15 = months used), airline credits in dollars (f14), entertainment credit in dollars (f16), lounge visits (f13 × ~$40/visit industry cost)

## Assumption 0a: expected credit loss is multiplicative — risk × exposure — not an additive penalty
Standard credit economics: Expected Loss = PD × EAD (probability of default × exposure at default). In column terms: `f11 × (f1 + portion of lend_exposure)`. This single term **automatically produces the tail-only risk behavior** the earlier decile analysis discovered empirically (flat through decile 8, crash at 9-10): f11 ≈ 0 for ~80% of customers, so their loss term is ~0 regardless of exposure, while a 0.30-risk customer with ₹30K exposure loses ₹9K — dwarfing every revenue term. No threshold gymnastics needed; the economics produce the shape natively. It also creates the *correct interaction* that additive formulas can't express: high revolve is net-positive for a safe customer (0.20×f1 − 0×f1 > 0) and net-negative for a risky one (0.20×f1 − 0.30×f1 < 0).

## Assumption 0b: engagement variables are not in the P&L at all
Logins, supplementary accounts, and card counts don't appear on an issuer's income statement. v1.6's regression when f12/f20 were added (+0.10 weight each) is direct evidence they're noise relative to the truth. All engagement terms and all three modifiers (M1/M2/M3) are dropped — M1's tail-override job is done natively by the multiplicative loss term, M2's benefit-cost job is done by pricing the benefits in dollars, M3 is gone.

## The v2.0 formula (raw values, no percentile ranks, no modifiers)
```
benefit_cost   = f14 + f16 + 15·f15 + 40·f13            [airline credits $ + entertainment credit $ + $15/mo cab + ~$40/lounge visit]
expected_loss  = f11 × (f1 + 0.5·lend_exposure)          [PD × EAD; lend_exposure = mean(f17,f18), 0 for non-holders]

Score = 0.025·f5              [interchange on this card's spend]
      + 0.005·f6_f10_sum      [small weight on industry-level spend — see caveat below]
      + 0.20·f1               [interest revenue on revolve balance]
      − 0.010·f21             [rewards redemption at ~1¢/point]
      − benefit_cost
      − expected_loss
```
Annual fee ($500-750) is constant per customer → no ranking effect → omitted. f4 (points balance) is a stock/liability whose net ranking effect is ambiguous (correlates with earn-side spend too) → omitted for cleanliness.

**Known weak point, stated honestly:** f6_f10_sum ("industry level spends") is not literal Amex revenue — a strict P&L would exclude it. It's kept at a deliberately small coefficient (0.005) because adding it produced v1.4's +0.063, the second-best gain of the series. If v2.0 underperforms, this term and the f21 coefficient are the first two dials to revisit.

**Magnitude check (population means):** interchange ≈ ₹87, industry-spend term ≈ ₹187, interest ≈ ₹493, rewards cost ≈ −₹305, benefits ≈ −₹170, expected loss ≈ −₹200 (but up to −₹10K+ in the risk tail). Interest revenue and rewards cost dominate the mid-ranking; the multiplicative loss term dominates the bottom tail. This matches both the deck's economics and the empirical leaderboard signals (f1's flip was our biggest single gain; f21-negative helped).

## Falsification criteria for v2.0 (3 submissions will remain after it)
- **v2.0 ≥ ~0.6:** function class was the blocker; spend remaining submissions calibrating coefficients (f21 rate, f6-f10 inclusion, loss multiplier).
- **v2.0 in 0.4-0.6:** structure right, magnitudes off — most likely f21's coefficient or the f6_f10_sum term; adjust the largest-variance term first.
- **v2.0 < 0.365 (below current best):** raw-dollar scaling is wrong for these masked columns (they may be arbitrarily rescaled at generation). Fall back to v1.5's skeleton and instead test the ORIGINAL proposal above (pctrank with threshold risk) — the one structural idea from the earlier section not yet tried.

