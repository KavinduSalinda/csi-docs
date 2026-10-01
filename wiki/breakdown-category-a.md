---
title: Breakdown Category A
created: 2026-06-21
updated: 2026-06-21
tags: [breakdown, category-a, therapeutic-area, phase, capabilities, competing-trials]
sources: ["[[raw/breakdown/A/README.md]]", "[[raw/breakdown/A/A1.md]]", "[[raw/breakdown/A/A2.md]]", "[[raw/breakdown/A/A3.md]]", "[[raw/breakdown/A/A4.md]]"]
---

# Breakdown Category A — Site Capabilities & Match

Category A evaluates how well the site fits the study protocol — therapeutic area experience, phase history, required equipment, and current enrollment competition. At **50% of the total match score**, it is the largest contributor.

## Details

### Sub-sections

| Code | Title | Weight (total) | Weight (within A) |
|------|-------|----------------|-------------------|
| A1 | Therapeutic Area Match | 15% | 30% |
| A2 | Phase Experience Match | 10% | 20% |
| A3 | Site Capabilities Match | 10% | 20% |
| A4 | Competing Trials Analysis | 15% | 30% |

Generation order: A1 → A2 → A3 → A4 (sequential).

---

## A1 — Therapeutic Area Match (15%)

**Purpose:** Measures whether the site has conducted trials in the study's therapeutic area (TA) and how recently. Also builds a MeSH hierarchy tree (`ta_mesh_mapping`) for drill-down UI — the tree does **not** affect the score.

**Calculation:** Two metrics averaged 50/50.

**Step 1 — Exact TA match:**
```
has_study_ta_history = EXISTS StudyHistoryCondition at site
                       WHERE condition_id IN TherapeuticAreaCondition(study.TA)
exact_match_percent = 100 if has_study_ta_history else 0
```

**Step 2 — Recency** (days since last completed trial in study TA):

| Days since last trial | `recency_percent` |
|-----------------------|-------------------|
| ≤ 90 | 100 |
| ≤ 180 | 90 |
| ≤ 365 | 80 |
| ≤ 730 | 60 |
| ≤ 1095 | 40 |
| > 1095 | 20 |
| No completed trial | 0 |

**Final score:** `(exact_match_percent + recency_percent) / 2`

**MeSH tree (`ta_mesh_mapping`):** Terminals = study `StudyCondition` MeSH only. Split into `mesh_hierarchy` (trial_count > 0) and `mesh_hierarchy_zero` (zero site trials). Full spec: `dev-docs/A1_BREAKDOWN_MESH_HIERARCHY_README.md`.

**Limitations:** MeSH tree requires ETL; missing rows use synthetic fallback paths. Recency needs `end_date` on completed trials.

---

## A2 — Phase Experience Match (10%)

**Purpose:** Scores site experience in the study's phase using a blended model — absolute trial volume (log scale) plus relative share of the site's phase history.

**Calculation:**

```
absolute_score = min( (log10(study_phase_count) / 2.0) × 100, 100 )
relative_score = (study_phase_count / total_studies) × 100
score_percent  = round(0.6 × absolute_score + 0.4 × relative_score)
```

| Trials in study phase | `absolute_score` |
|-----------------------|------------------|
| 0 | 0 |
| 10 | 50 |
| 100+ | 100 |

**Limitations:** Zero trials → score 0. Phase name matching is exact string compare on `phase.name`. No score when `Study.phase` is null.

---

## A3 — Site Capabilities Match (10%)

**Purpose:** Binary checklist — for each study-required capability/equipment, does the site have it?

**Calculation:**

```
match_rate     = (matched_count / total_required) × 100
score_percent  = match_rate
```

Per-row status: `matched` (check icon) or `warning`. Zero required capabilities → score 0%.

**Limitations:** Exact equipment ID match only — no substitute/equivalent logic. No partial credit for similar equipment.

---

## A4 — Competing Trials Analysis (15%)

**Purpose:** Assesses enrollment competition and site capacity from active/recruiting trials at the site that overlap the study's TA, phase, or MeSH terms. Higher competition → lower score.

**Competition classification (priority order):**

| Type | Priority | Criteria |
|------|----------|----------|
| `DIRECT` | 0 | Same TA **and** same phase; if study has MeSH, history must cover all study MeSH |
| `MESH` | 1 | All study MeSH IDs present on history |
| `AREA` | 2 | Same therapeutic area |
| `PHASE` | 3 | Same phase only |
| `UNRELATED` | 4 | No overlap |

**Per-trial weighted score:**
```
w_score = BASE_SCORE[type] × enrollment_factor(status) × 0.4 × temporal_factor(days_until_completion)
```

BASE_SCORE: DIRECT=1.0, MESH=0.5, AREA=0.6, PHASE=0.4, UNRELATED=0.05

**Aggregate risk and A4 score:**
```
raw_risk     = 0.5×direct_count + 0.3×area_count + 0.2×phase_count + 0.3×weighted_avg
capped_risk  = min(raw_risk, 1.0)
score_percent = round((1.0 − capped_risk) × 100)
```

**Risk level bands:**

| Condition | `risk_level` |
|-----------|--------------|
| direct ≥ 3 | CRITICAL |
| direct ≥ 2 | HIGH |
| direct ≥ 1 OR area ≥ 3 | MODERATE |
| area ≥ 1 OR phase ≥ 2 OR total_active ≥ 10 | LOW |
| Otherwise | MINIMAL |

**Capacity level:** Based on `total_active_trials` — OVERLOADED (≥20), HIGH_LOAD (≥15), MODERATE_LOAD (≥10), MANAGEABLE (≥5), LOW_LOAD (<5).

**API response includes:** `risk_level`, `competing_trials` (top 10 non-UNRELATED), `score_composition` (penalty breakdown), `capacity`, `enrollment_status_breakdown`.

**Limitations:** Does not model patient pool size or geography. MeSH match requires all study MeSH on history (strict superset).

## Related

- [[Breakdown Scoring]]
- [[Breakdown Category B]]
- [[Breakdown Category C]]
- [[Breakdown Category D]]
- [[Study Creation]]

## Open Questions
