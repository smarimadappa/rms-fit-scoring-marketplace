# RMS Fit Scoring — Model Reference

This file owns the **sub-score rules** (how each raw input maps to 0–100). Tunable numbers — weights, bands, confidence thresholds — live in `weights.json`; the confidence cap and banding logic are in SKILL.md. Don't duplicate those here.

## Formula
Sub-score each input 0–100 (rules below) → multiply by its weight (`weights_raw_points` in `weights.json`) → sum. Raw points normalize to 100 at runtime, so any one can change without touching the rest. Unknown inputs are dropped and remaining weights renormalize proportionally. **Final score = Σ (sub-score × normalized weight).**

## Sub-score rules (each returns 0–100)

### Market Presence Score (Look 5042, most recent non-null `Overall` row)
Market presence is already ~0–100. **Use it directly** (clamp to 0–100). The latest release may return two Overall rows (one null, one real) — take the most recent non-null one. If none is non-null, treat as unknown.

### Account Segment (Look 5043 / ref look 5077)
**Source: `account_contract_info.territory_segment` — the Salesforce territory segment. Never the review-derived `survey_responses.company_segment` or any other segment-named field** (see the segment warning in `looker-map.md`). If territory_segment has no row, mark unknown rather than substituting.

Higher enterprise focus = better. Manually enforced mapping:
| Segment | Sub-score |
|---|---|
| Strategic / Strat Ent | 100 |
| Enterprise | 75 |
| Commercial | 60 |
| Mid-Market | 50 |
| Small Business / SMB | 20 |

### Total customers & end-users (web research)
Log-scaled on customer count, with a bonus for multiple end-users per customer.
| Customers | Base |
|---|---|
| 10,000+ | 100 |
| 2,000–9,999 | 80 |
| 500–1,999 | 60 |
| 100–499 | 40 |
| < 100 | 20 |
End-user multiplier: if clearly **many end-users per customer**, use base as-is (or +0). If **~1 end-user per customer** (niche), multiply base by 0.7. Cap at 100.

### End-user profile (web research)
Can the typical end-user realistically be reached to leave a review? People who sit at a computer all day are reachable; people who don't are very unlikely to review. **This input mattered enormously in real campaigns — score it strictly.**

Team rule: if the product's end user is a **Skilled Laborer, Brick & Mortar worker, or Public Sector worker**, that is bad for RMS — they are not desk-based and rarely leave reviews.
| Profile | Sub-score |
|---|---|
| Desk / computer-based knowledge workers | 100 |
| Mixed (a real split of desk and non-desk users) | 50 |
| Skilled labor / brick & mortar / public sector / other non-desk | 10 |

Judge by the *typical* end user, not the buyer. E.g. dealership service/parts/floor staff, field technicians, retail/warehouse workers, nurses, government field staff → the non-desk floor (10), even if the product is sold to an enterprise. Only score "mixed" when there's a genuine split, not to soften a mostly-non-desk base.

### Customer industry — vertical G2-activity (Look: survey_responses)
Separate from end-user profile: even desk-based users won't review if their *industry* doesn't engage with G2. Churches, non-profits, and public-sector buyers barely use G2 regardless of job type — this is what sank real campaigns.

Pull the product's reviewer-industry mix from `survey_responses.industry_name` (filtered by `product_id`, sorted by count desc, ignore the null row). Take the **top 3 verticals**. Compare against `low_activity_industries` in `weights.json` (a business-maintained list that grows over time):
| Condition | Sub-score |
|---|---|
| #1 vertical is low-activity | 15 (floor) |
| #1 normal, but 2 of top-3 are low-activity | 50 |
| #1 normal, 1 of top-3 low-activity | 75 |
| all top-3 normal | 100 |

The #1 spot is decisive: if a product's single biggest reviewer vertical is a low-activity one, it floors — that pool won't produce reviews no matter how big the customer is. If `survey_responses` returns no usable industries, mark this input unknown.

**Score penalty (not just the sub-score).** When the #1 reviewer vertical is low-activity, the input floors at 15 *and* the whole final score is multiplied by `low_activity_top1_penalty` (weights.json, default 0.7). Rationale: a dead top vertical is close to disqualifying and must dominate, not be averaged away by strong market-presence/category signals. In the scoring JSON, set `"low_activity_top1": true` on this input to trigger the penalty. (Example: APS — Non-Profit is the #1 reviewer vertical → industry sub-score 15 and final ×0.7, dropping it from ~57 to ~40, matching a real campaign that yielded only 9 reviews.)

### Regional distribution (web research + G2 review geography)
The principle is **breadth of regions**, not which region. Single-region concentration is the limiter — it shrinks the pool of reviewers and makes campaigns hard. More regions = easier.

Score the geography of the customer/reviewer pool, not the vendor's office locations.
| Distribution | Sub-score |
|---|---|
| Global / many regions | 100 |
| Multi-region (2–3 regions) | 70 |
| Single region, non-US (e.g. APAC-only, EMEA-only) | 40 |
| Single region, US-only (nationwide) | 20 |
| Sub-national (operates in only a handful of US states/localities) | 10 |

(US-only is the hardest major-market case; a sub-national footprint — e.g. a vendor serving only ~7 states — is worse still, because the reachable reviewer pool is tiny. Score 10 when research shows the customer base is confined to a few states/regions within one country.)

### Category popularity (Look 5045)
Measures how active/popular the product's **category** is — a genuinely category-level property, so a category-level signal is the right tool. `category_bi_signals` (category buyer-intent, last 6 mo) discriminates well *between* categories (e.g. Photo Editing ≈ 42,000 vs. Car Dealer ≈ 7,400 — ~6x). It does NOT discriminate between products in the same category (all share the category value), so do **not** rank a product against its peers on it — score the category's absolute value directly.

Pull the category's `category_bi_signals` for the product's `main_category_name` (take the populated row; ignore null/0 fan-out rows). Absolute bands:
| Category BI signals (6 mo) | Sub-score |
|---|---|
| 30,000+ | 100 |
| 15,000–29,999 | 75 |
| 7,000–14,999 | 50 |
| 2,000–6,999 | 30 |
| < 2,000 | 15 |

`products_on_grid` = competitor count, context only (not scored). A low-popularity category (like Car Dealer at ~7,400 → 50, low end) is a real headwind and should pull the score down — do not exclude it.

### Review generation likelihood (looks 5078/5079/5080 — three-vector velocity read)
The most direct signal: does this product's competitive neighborhood actually produce reviews, and does the product itself? Query recipes in `looker-map.md`; all in the standard explore; **52-week window**.

**Vector 1 — categories & competitors:** from `product_category_mapping` (filter by product_id, keep `product_in_category_flag = "Y"` rows only), take the **main category + the product's 2 biggest other categories** (biggest = highest 52-wk review volume). Competitor count per category = count of Y-flag products in it.

**Vector 2 — category ease, per category:** 52-wk category reviews ÷ competitor count ÷ 52 = reviews per product per week (RPPW). Band each of the 3 categories, then average:
| Category RPPW | Score |
|---|---|
| ≥ 0.50 | 100 |
| 0.20–0.49 | 75 |
| 0.05–0.19 | 50 |
| 0.02–0.049 | 30 |
| < 0.02 | 15 |

**Vector 3 — product's own velocity:** target's 52-wk reviews ÷ 52, banded:
| Product reviews/week | Score |
|---|---|
| ≥ 1.0 | 100 |
| 0.5–0.99 | 75 |
| 0.2–0.49 | 50 |
| 0.05–0.19 | 30 |
| < 0.05 | 15 |

**Sub-score = 0.5 × (category ease average) + 0.5 × (product velocity score).** Report trend (recent 12 wks vs. prior 12) as context, not scored. If the product shows ~zero 52-wk reviews inside an active neighborhood (category avg ≥ 50), say so explicitly — active category + dormant product is precisely what a campaign can fix, or a red flag if a past campaign already tried.

### Approved G2 reviews (Look 5044)
Log-scaled on the product's own approved review count.
| Approved reviews | Sub-score |
|---|---|
| 1000+ | 100 |
| 250–999 | 75 |
| 50–249 | 50 |
| 10–49 | 30 |
| < 10 | 15 |

### Internal-integration reviews (web research + G2)
Does the vendor have sibling products used together, with many reviews (e.g., a suite like Workday)?
| Signal | Sub-score |
|---|---|
| Strong suite, many cross-product reviews | 100 |
| Some sibling products / moderate | 60 |
| Standalone product, none | 30 |

### Stacked G2 users (Look 5041)
Log-scaled on stacked user count.
| Stacked users | Sub-score |
|---|---|
| 50,000+ | 100 |
| 10,000–49,999 | 75 |
| 2,000–9,999 | 50 |
| 500–1,999 | 30 |
| < 500 | 15 |

## Bands & confidence
Recommendation bands and the low-confidence cap are defined in `weights.json` + SKILL.md — not repeated here. Always report how many of the eleven inputs were found (e.g. "9 of 11 available").
