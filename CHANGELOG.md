# Changelog

## v0.11.0
- Remove the Go / Proceed with caution / No banding entirely. The review-volume/package/price recommendation (added in v0.10.0) is now the sole headline recommendation — no abstract label is ever shown alongside or instead of it. `weights.json` no longer has `recommendation_bands` or `cap_band`; the low-confidence thresholds now only attach a "LOW CONFIDENCE" flag to the real recommendation rather than capping/downgrading it. Service-disqualified products now surface an explicit "Not a fit for RMS" recommendation instead of a "No" band. `build_scorecard.py`'s summary/detail sheets, SKILL.md's Step 4/4.5, and scoring-model.md updated to match.

## v0.10.1
- Fix Commercial/Enterprise Existing Customer 60-79 review-volume band pricing: Custom-25 was $1,800 and Accelerator-50 was $5,000 (a row-shift artifact in the source spreadsheet — those were actually the 40-59 row's Starter/Custom-25 prices). Corrected to $5,000/$10,000, matching the corrected source spreadsheet and the per-package prices used everywhere else in the table.

## v0.10.0
- Add a review-volume/packaging recommendation (package + price) alongside the fit score, driven by score × segment bucket (SMB & MM vs. Commercial/Enterprise, derived from the Account Segment sub_score) × business type (new business vs. existing customer). New `review_volume_bands.json` holds the tunable band tables; `build_scorecard.py` computes and renders the recommendation; SKILL.md gains a Step 1.6 Salesforce lookup (`Account.Type` keyed by `G2_Vendor_ID__c`) to resolve business type.

## v0.9.0
- Total customers & end-users: rebanded the customer-count table (1,001–1,999 → 60, 500–1,000 → 40) and added a minimum-customer floor — fewer than 500 product-specific customers bottoms this sub-score out at 10, subordinate to the existing new-product and end-user floors.

## v0.8.0
- Add a hard service/provider disqualifier: if the resolved G2 listing's `type` isn't "Software" (e.g. a service/agency listing, `type: "Provider"`), scoring stops before Step 2 runs at all. Final score is forced to 0 and the band to a hard "No — not a fit" regardless of any other input — not averaged, not subject to the low-confidence cap.

## v0.7.0
- Total customers & end-users now floors at 15 whenever End-user profile floored at 10 (Public Sector/Non-desk persona) — a large customer base of an unreachable persona (e.g. Dentally's dental staff) no longer scores on raw count alone.
- Renamed "vertical" to "industry" throughout the Customer industry section for consistency.
- Category popularity is now a blend: 0.6 × category_bi_signals (Looker) + 0.4 × total category review volume (new, summed via G2 MCP `list_products` across every product in the category), with a granular 10-tier band scale. Falls back to bi-signals alone if the G2 MCP pull is unavailable.

## v0.6.0
- Rebalance weight from Account Segment (18→6 raw points) to Total customers & end-users (18→30): a mislabeled/inherited Enterprise or Strategic territory segment (e.g. from an acquisition) can no longer meaningfully offset a small standalone customer count.

## v0.5.0
- Move HR and Finance to Mid tier (70) in End-user profile scoring; Procurement/Supply Chain stays in Mid/low (40).
- Customer industry now sourced primarily from public research (vendor site, case studies, customer logos by industry) rather than G2 reviewer data; `survey_responses.industry_name` is a secondary corroborating signal only.

## v0.4.0
- Add new-product floor to "Total customers & end-users": a product live less than 2 years floors this sub-score at 15 regardless of customer count, and is never scored on the vendor's company-wide customer total.

## v0.2.0
- Fix: regional distribution now scores customer/end-user geography, not the vendor's HQ or office footprint. `vendor_hq_region` is a starting prior only, verified against G2 review geography and firmographic sources.

## v0.1.0
- Initial release: RMS fit scoring skill wrapped for marketplace distribution.
