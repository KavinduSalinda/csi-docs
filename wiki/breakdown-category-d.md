---
title: Breakdown Category D
created: 2026-06-21
updated: 2026-06-21
tags: [breakdown, category-d, regulatory, country, institution-type, etl]
sources: ["[[raw/breakdown/D/README.md]]", "[[raw/breakdown/D/D1.md]]", "[[raw/breakdown/D/D2.md]]"]
---

# Breakdown Category D — Regulatory & Institution

Category D evaluates the external context around the site — country regulatory track record and whether the institution type is suited to the study phase. At **10% of the total match score**.

## Details

### Sub-sections

| Code | Title | Weight (total) | Weight (within D) |
|------|-------|----------------|-------------------|
| D1 | Country Regulatory Environment | 5% | 50% |
| D2 | Institution Type | 5% | 50% |

```
category_d_score = (D1.score × 5 + D2.score × 5) / 10
```

Generation order: D1 → D2 (after Category C completes).

---

## D1 — Country Regulatory Environment (5%)

**Purpose:** Scores the site's country regulatory environment using historical trial completion rates and provides TA-specific MeSH success rates in that country.

**Calculation:**

```
success_rate  = completed / (completed + terminated) × 100
```
(All sites in country, trials with completed or terminated status.)

**Score bands:**

| Success rate | `score_percent` |
|--------------|-----------------|
| ≥ 90% | 100 |
| ≥ 80% | 90 |
| ≥ 70% | 80 |
| ≥ 60% | 70 |
| < 60% | `max(50, success_rate)` |

**MeSH country payload (`mesh_terms`):** Built by `build_ta_mesh_country_payload(study, site)`.
- `study_related` — MeSH terms selected on study (`StudyCondition`), with completion counts and success rates in the site's country.
- `country_other` — other MeSH in study TA not selected on study.
- Only rows with `study_count > 0` are returned.

**Data sources:** `Site.country`, `Country.assessment_text` (ETL pre-written narrative), `StudyHistory` + `StudyHistoryStatus`, `build_ta_mesh_country_payload()`.

**API response includes:** `comparison.location` (city, country), `comparison.country_success_rate` (rate + trial count), `assessment` (ETL narrative text), `mesh_terms` envelope.

**Limitations:** Country-level aggregate — not site-specific regulatory performance. Binary completed vs terminated (no termination reason breakdown). `assessment_text` is static, not study-specific.

---

## D2 — Institution Type (5%)

**Purpose:** Scores suitability of the site's institution type (e.g. Academic Medical Center) for the study's phase using ETL-fed `SiteTypePerformance` benchmarks.

**Calculation:**

1. Query `SiteTypePerformance` matching `institution_type` (case-insensitive).
2. Fallback to `SiteType` **"Other"** if no rows found.
3. Normalize study phase → `phase1`…`phase4` and find exact match → `study_phase_suitability`.

**Score bands:**

| `study_phase_suitability` | `score_percent` |
|---------------------------|-----------------|
| ≥ 90% | 100 |
| ≥ 80% | 90 |
| ≥ 70% | 80 |
| ≥ 60% | 70 |
| < 60% or no data | 60 (default) |

**Phase bar colors:** ≥90 → green | ≥70 → yellow | <70 → red. Study phase row gets `highlighted: true`.

**Data sources:** `Site.institution_type`, `SiteType` (key_strengths, considerations — ETL), `SiteTypePerformance` (per-phase performance % per institution type — ETL).

**API response includes:** `description`, `explanation`, `institution_type`, `key_strengths`, `considerations`, `phase_suitability[]` (each phase with suitability %, color, highlighted flag).

**Limitations:** `institution_type` must match ETL catalog. No match for study phase → default score 60. `explanation` is a template — not site-specific LLM narrative. Does not replace A3 equipment/capability checks.

## Related

- [[Breakdown Scoring]]
- [[Breakdown Category A]]
- [[Breakdown Category C]]
- [[Architecture]]

## Open Questions
