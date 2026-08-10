# Looker query map

How to pull each input with a **server-side filtered** `looker_run_query` — never `looker_run_look` (returns the whole dataset, `text`-wrapped, capped at 5000 rows). Resolve the product ONCE (Step 1), then pass its `product_id` as the filter into every query below. All parameters here are **confirmed working** — do not guess or improvise model/explore names.

## Step 1 — Product identity resolver
- model: `global_lifecycle` · explore: `categories_products`
- fields: `product_vendor_info.product_id`, `.product_name`, `.vendor_name`, `.main_category_name`, `.vendor_hq_country`, `.vendor_hq_region`
- filter: `{"product_vendor_info.product_name": "%<name>%"}` (wildcards = contains-match)
- Capture `product_id` and reuse it everywhere below.

## Scoring inputs — five share ONE explore+model
**model: `global_lifecycle` · explore: `categories_products`** · filter each by the product_id, e.g. `{"products.product_id": "<id>"}`.

| Input (look) | Fields | Notes |
|---|---|---|
| Stacked G2 users (5041) | `stacked_users.product_id`, `stacked_users.count_distinct_products_and_users` | single row = user count |
| Account Segment (5043) | `account_contract_info.product_id`, `account_contract_info.territory_segment` | **`territory_segment` ONLY** — the Salesforce territory segment. See "Segment field" warning below. Ref look: 5077 (RMS Skill Accounts Table) |
| Approved reviews (5044) | `products.product_id`, `products.approved_reviews` | total approved-review count |
| Category popularity (5045) | `product_vendor_info.product_id`, `.main_category_name`, `categories.products_on_grid`, `categories.category_bi_signals` | products_on_grid = competitor count (context only); category_bi_signals = buyer intent (scored). Take the primary category row; ignore null/"Unknown" fan-out rows. **PENDING:** traffic, reviews-total, reviews-rolling fields not yet confirmed — see note below |
| Customer industry (survey_responses) | `survey_responses.industry_name`, `survey_responses.count` | filter `{"survey_responses.product_id": "<id>"}`, sort `count desc`, ignore null row. Take top 3 verticals; compare to `low_activity_industries` in weights.json. Same explore/model (`categories_products` / `global_lifecycle`) |

## Review generation likelihood (looks 5080/5079/5078) — same explore/model, 52-wk window
All three vectors confirmed in `categories_products` / `global_lifecycle`:
- **Categories & competitors (5080):** fields `product_category_mapping.category_id`, `.category`, `.product_in_category_flag`, filter `{"product_category_mapping.product_id": "<id>"}`. Keep flag `Y` rows only (N = stale mapping). For competitor lists/counts: filter `{"product_category_mapping.category_id": "<cat_id>", "product_category_mapping.product_in_category_flag": "Y"}`.
- **Product velocity (5079):** fields `survey_responses.submitted_at_week`, `survey_responses.count`, filters `{"survey_responses.product_id": "<id>", "survey_responses.submitted_at_week": "52 weeks"}`, sort week desc. Works for a competitor too (swap the id). Ignore the null-week row.
- **Category velocity (5078):** fields `survey_responses.submitted_at_week`, `survey_responses.count`, filters `{"product_category_mapping.category_id": "<cat_id>", "product_category_mapping.product_in_category_flag": "Y", "survey_responses.submitted_at_week": "52 weeks"}` — the cross-view filter aggregates the whole category's reviews by week in one call.
- 52-wk **totals** (to rank "biggest" categories or get sums without the series): same queries minus the week dimension → one row per query.

## Market Presence (5042) — separate explore+model
- model: `global_revenue_marketing` · explore: `revenue_marketing` · view: `grid_report_audit`
- fields: `grid_report_audit.product_id`, `.company_segment`, `.market_presence`, `.release_date`, `.category`
- filters: `{"grid_report_audit.product_id": "<id>", "grid_report_audit.company_segment": "Overall"}`
- sorts: `["grid_report_audit.release_date desc"]`, limit small — **take the most recent Overall row that has a non-null `market_presence`.** The latest release can return two Overall rows (one null, one real); skip nulls and use the populated one. Value is already ~0–100; use directly. If no non-null row exists, mark unknown.

---
**Category popularity — SCOREABLE (resolved).** `category_bi_signals` is a category-level value (all products in a category share it). That's correct for scoring *category popularity* — score the category's absolute BI-signal value against fixed bands (see scoring-model.md); do NOT rank a product against category peers on it. Confirmed discrimination: Photo Editing ≈ 42,013 vs. Car Dealer ≈ 7,423. Pull the populated row, ignore null/0 fan-out. `products_on_grid` = competitor count, context only. (Product-level traffic/review fields remain unconfirmed but are no longer needed for this input.)

**⚠️ Segment field — use the Salesforce territory segment, nothing else.**
The Account Segment input MUST come from **`account_contract_info.territory_segment`** (Salesforce territory segment; reference look **5077 — RMS Skill Accounts Table**). Several other segment-named fields exist in these explores and are **wrong for this input** — do not substitute them:
- `survey_responses.company_segment` — review-derived. **Never use for Account Segment.**
- `account_contract_info.account_segment`, `.reporting_segment` — different Salesforce cuts, not the territory segment.
- `product_vendor_info.company_segment`, `users.company_segment`, `vendors.company_segment` — unrelated.
- `grid_report_audit.company_segment` — used ONLY as a filter (`= "Overall"`) when pulling market presence. It is not the Account Segment input; do not confuse the two.

If `territory_segment` returns no row for the product, mark Account Segment **unknown** — do not fall back to another segment field.

**Two known row-shape edge cases** (confirmed harmless for Photoshop, unverified elsewhere):
- *Category fan-out:* 5045 can return multiple rows per product (real + null/"Unknown"). "Primary category row" = the row whose `main_category_name` matches the product's main category and has non-null signals. If a product legitimately sits in multiple categories, use the `main_category_name` row.
- *Segment collapse:* 5043 returned one `territory_segment` for Photoshop. If a product returns multiple, this is unhandled — flag it rather than picking one.

---
Confirmed against Adobe Photoshop (id 1692): MPS 99.28 · Segment Enterprise · Approved reviews 13,395 · Stacked users 52,171 · Category Photo Editing (152 on grid, 42,013 BI signals).
