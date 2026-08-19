# Changelog

## v0.5.0
- Move HR and Finance to Mid tier (70) in End-user profile scoring; Procurement/Supply Chain stays in Mid/low (40).
- Customer industry now sourced primarily from public research (vendor site, case studies, customer logos by industry) rather than G2 reviewer data; `survey_responses.industry_name` is a secondary corroborating signal only.

## v0.4.0
- Add new-product floor to "Total customers & end-users": a product live less than 2 years floors this sub-score at 15 regardless of customer count, and is never scored on the vendor's company-wide customer total.

## v0.2.0
- Fix: regional distribution now scores customer/end-user geography, not the vendor's HQ or office footprint. `vendor_hq_region` is a starting prior only, verified against G2 review geography and firmographic sources.

## v0.1.0
- Initial release: RMS fit scoring skill wrapped for marketplace distribution.
