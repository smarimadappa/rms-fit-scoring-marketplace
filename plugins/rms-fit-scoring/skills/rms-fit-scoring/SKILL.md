---
name: rms-fit-scoring
description: Score a G2 vendor product for Review Managed Services (RMS) fit and produce a go/no-go recommendation. Use whenever someone asks whether RMS should be sold for a product, how good an RMS fit a product is, whether a product is worth an RMS campaign, or wants an RMS fit score, RMS qualification, or an RMS scorecard. Trigger even when someone just names a product and asks "should we do RMS for this?", "is this a good RMS target?", or "score this for review managed services" — run this model, don't answer from general judgment. Also trigger on batch requests ("score these five products for RMS").
---

# RMS Fit Scoring

Scores a G2 vendor product for **Review Managed Services (RMS)** fit: a weighted 0–100 score mapped to a three-band spectrum recommendation (Go / Proceed with caution / No), output as a chat summary plus a downloadable `.xlsx` scorecard. Higher score = better/easier RMS fit.

This is the **pre-sale qualification layer** only — no campaign planning, no review-count targets.

Two caveats to state when presenting results: the score is **directional**, and the weights are **provisional and tunable** (not a validated model). All tunable numbers — weights, bands, confidence thresholds — live in `weights.json`. Sub-score rules and the full formula are in `references/scoring-model.md`; read it before scoring.

## Workflow

### Step 0 — Looker preflight (MANDATORY, first, every run)
Seven of the eleven inputs come from Looker, so verify it before anything else. Do not identify products, gather inputs, or score until this passes.
1. **Connected?** Confirm `looker_run_look` is available; if not, `tool_search` for "looker run look". Nothing found → not connected.
2. **Working?** Run `looker_run_look` with `look_id: 5042`, `limit: 1`. A valid row confirms auth is live.

If either fails, **STOP** and tell the user Looker is not connected/responding, that seven of eleven inputs depend on it, and to connect or reconnect it before retrying. Surface any auth error so they can re-authorize.

### Step 1 — Resolve product identity (MANDATORY — do this once, reuse everywhere)
Resolve the product with a **server-side filtered query**. Never pull the full catalog (160k+ products; scrolling/grepping will fail and is forbidden).

Use `looker_run_query` with:
- `model`: `global_lifecycle`
- `explore`: `categories_products`
- `fields`: `["product_vendor_info.product_id", "product_vendor_info.product_name", "product_vendor_info.vendor_name", "product_vendor_info.main_category_name", "product_vendor_info.vendor_hq_country", "product_vendor_info.vendor_hq_region"]`
- `filters`: `{"product_vendor_info.product_name": "%<name>%"}` — the `%` wildcards do a contains-match; an exact string with no `%` often returns nothing
- `limit`: 25

Returns clean JSON rows (no `text`-wrapper unwrapping, unlike `looker_run_look`).

A `%name%` search returns near-matches (e.g. "Photoshop" also returns Lightroom, Elements, plugins). Pick the **exact** product by name + vendor. If several genuinely distinct products match, or none do, **stop and ask** the user to pick or supply the ID — never score a guessed match.

**Capture and carry these for the rest of the run** — do not look them up again:
- `product_id` — the filter key for every other view below
- `product_name` and `vendor_name` — for output and disambiguation
- `vendor_hq_country` / `vendor_hq_region` — a starting prior only for the regional-distribution input; it must be verified against actual customer geography, not used as the answer

### Step 2 — Gather inputs (filter every view by the captured product_id)
Get a raw value for each of the eleven inputs. For all Looker inputs, **pass the `product_id` from Step 1 as a server-side filter** via `looker_run_query` — one product's rows only, never a full pull. Reuse the ID you already have; do not re-resolve it.

**From Looker** — every query is a confirmed, one-shot `looker_run_query` filtered by `product_id`. **The exact model, explore, fields, and filters are in `references/looker-map.md` — read it and use those parameters verbatim. Do not improvise or guess model/explore names; all six recipes are proven.**
- **5041** — Stacked G2 users → user count
- **5042** — Market Presence Score → most-recent-release `Overall` row's `market_presence` (separate explore; sort by release_date desc, take first)
- **5043** — Account Segment → **`account_contract_info.territory_segment` (Salesforce territory segment) ONLY.** Never the review-derived `survey_responses.company_segment` or any other segment-named field; if it has no row, mark unknown rather than substituting. Ref look 5077.
- **5044** — Approved G2 reviews → `approved_reviews` count
- **5045** — Category popularity → category, competitor count (`products_on_grid`), buyer intent (`category_bi_signals`); take the primary category row, ignore null/"Unknown" fan-out
- **5080/5079/5078** — Review generation likelihood → three-vector velocity read (competitor set, product's own 52-wk weekly series, category 52-wk weekly series). Main category + 2 biggest others. Recipes in `looker-map.md`, math in `scoring-model.md`.

**By web research** (company site, ZoomInfo/Clay-style sources, G2 review geography, job-title signals):
- Total customers & end-users (and whether multiple end-users per customer), plus the product's own launch/GA date — a product live less than 2 years floors this input rather than being scored on the vendor's overall customer count (see `scoring-model.md`)
- Regional distribution (US-heavy / global / APAC-heavy) — measures the customer/end-user base, never the vendor's own HQ or office footprint. Use the captured `vendor_hq_region` only as a starting prior, then verify against actual customer geography (G2 review-country mix, firmographic tools like 6sense/ZoomInfo, customer-logo pages by region). A vendor with offices on six continents can still have a customer base that's 85%+ concentrated in one country — the customer data wins.
- End-user profile (scored by typical persona: public-sector workers floor first regardless of desk status, then non-desk/skilled-labor floors, then a 4-tier role table from Engineering/Sales-type roles down to Legal/Executive — see `scoring-model.md`)
- Internal-integration reviews (sibling products used together with many reviews?)

**Operator overrides (these four inputs only).** If the operator supplies their own value for any web-research input — e.g. firmer numbers from a customer contact — that value **overrides** the researched one for scoring. Still do the research, then replace the value with the override so the scorecard can show both (researched vs. operator-supplied). An overridden input counts as **present** and is tagged as an **authoritative (operator-supplied)** source, so it never counts against confidence. Overrides apply to web-research inputs only; Looker inputs (5041–5045) are system-authoritative and not overridable. If the operator gives an override up front, still run the research pull so both values are captured.

Never guess or fabricate a value. If an input can't be found (or a Look returns nothing), mark it **unknown**; it's dropped and remaining weights renormalize. List every unknown in the output.

### Step 3 — Score
Per `references/scoring-model.md`: compute each 0–100 sub-score, multiply by its renormalized weight, sum for the 0–100 final score.

### Step 4 — Recommend
Map the score to a band (from `weights.json`), a spectrum rather than a binary: **61+** Go · **31–60** Proceed with caution / needs further validation · **0–30** No.

The middle band is deliberate: a product can be a big prize (high market presence, large customer base) yet still be hard to run reviews for (non-desk end users, single-region, small category). Those land in "proceed with caution" — worth pursuing but validate feasibility first, don't treat as an automatic Go.

Then apply the confidence cap. **Low confidence** = 2+ high-weight inputs unknown, OR fewer than 7 of 11 available (thresholds and the high-weight list in `weights.json`). On low confidence, **cap the band at "Proceed with caution"** — never Go — reporting the numeric score unchanged, and name the missing inputs.

### Step 5 — Output
Both:
1. **Chat summary** — final score, recommendation, top 2–3 drivers, any unknowns. Researched inputs as **values only** (sources stay in the scorecard). Note if the band was capped for low confidence.
2. **`.xlsx` scorecard** — generate with `scripts/build_scorecard.py` (JSON format in its header), present with the file tool. For any overridden input, pass both the researched value and the operator value so the scorecard shows both and marks the source authoritative.

Batch requests → one workbook with a summary row per product plus a detail sheet each.

## Guardrails
- Be honest about missing data; a score on few inputs is weak, say so.
- To retune, edit `weights.json` only — never hardcode weights elsewhere.
- No review-count targets or campaign plans — out of scope.
