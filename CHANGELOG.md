# Changelog

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
