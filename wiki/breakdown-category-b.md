---
title: Breakdown Category B
created: 2026-06-21
updated: 2026-06-21
tags: [breakdown, category-b, enrollment, historical-performance]
sources: ["[[raw/breakdown/B/README.md]]", "[[raw/breakdown/B/B1.md]]", "[[raw/breakdown/B/B2.md]]", "[[raw/breakdown/B/B3.md]]"]
---

# Breakdown Category B — Historical Performance

Category B evaluates the site's operational track record — how reliably the site has performed on past trials relative to study requirements. At **30% of the total match score**, it is the second-largest contributor.

## Details

### Sub-sections

| Code | Title | Weight (total) | Status |
|------|-------|----------------|--------|
| B1 | Enrollment Performance | 30% | Implemented |
| B2 | — (planned) | — | Not implemented |
| B3 | — (planned) | — | Not implemented |

Currently only B1 exists, so **category B score = B1 score**. B2/B3 are reserved and will require weight rebalancing when implemented. Background job messages already reference "Moving to B2 section" after B1.

---

## B1 — Enrollment Performance (30%)

**Purpose:** Compares the site's historical average monthly enrollment against the study's required enrollment rate.

**Calculation:**

```
study_average_rate    = total_enrollment_target / duration_months
percentage_of_target  = round((site_average_rate / study_average_rate) × 100)
```

**Performance indicator:**

| Condition | Label |
|-----------|-------|
| `site_rate ≥ study_rate` | Above target |
| `site_rate ≥ study_rate × 0.8` | On target |
| Otherwise | Below target |

**Score bands (non-linear):**

| % of target | `score_percent` |
|-------------|-----------------|
| ≥ 100 | 100 |
| ≥ 90 | 95 |
| ≥ 80 | 85 |
| ≥ 70 | 75 |
| ≥ 60 | 65 |
| < 60 | `max(20, percentage_of_target)` |

**Data sources:** `Study.total_enrollment_target`, `Study.duration`, `Site.avg_monthly_enrollment` (from facility ETL).

**API response includes:** `comparison.study_target` (required rate), `comparison.site_average` (actual rate + `performance_indicator`), `chart_data` (metadata for a line chart — site vs study average; no historical time series in the payload).

**Limitations:** Single site-level average — not phase/TA-specific. Zero study duration → 0% of target and score floor of 20.

---

## B2 — Planned

Reserved sub-section for an additional historical-performance dimension. Candidate metrics (not confirmed):
- Protocol deviation rate (`Site.protocol_deviation`)
- Data quality score (`Site.data_quality`)
- Retention rate (`Site.retention_rate`)
- On-time activation history

Not included in `generate_full_breakdown()` today.

---

## B3 — Planned

Reserved third sub-section for category B. Will require weight rebalancing across B1/B2/B3 when implemented.

## Related

- [[Breakdown Scoring]]
- [[Breakdown Category A]]
- [[Breakdown Category C]]

## Open Questions

- What metrics will B2 and B3 cover? Candidate fields exist on `Site` model but generators are not implemented.
