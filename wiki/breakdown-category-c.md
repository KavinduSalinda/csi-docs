---
title: Breakdown Category C
created: 2026-06-21
updated: 2026-06-21
tags: [breakdown, category-c, pi, publications, openalex, openai]
sources: ["[[raw/breakdown/C/README.md]]", "[[raw/breakdown/C/C1.md]]", "[[raw/breakdown/C/C2.md]]"]
---

# Breakdown Category C — PI Experience

Category C evaluates principal investigators at the site — trial experience and publication quality. Both sub-sections share a single PI recommendation pass to avoid duplicate OpenAI calls. At **10% of the total match score**.

## Details

### Sub-sections

| Code | Title | Weight (total) | Weight (within C) |
|------|-------|----------------|-------------------|
| C1 | PI Experience | 5% | 50% |
| C2 | PI Publication Quality | 5% | 50% |

```
category_c_score = (C1.score × 5 + C2.score × 5) / 10
```

### Shared PI Pipeline

`get_study_site_pi_recommendations(study, site)` runs once, shared by both C1 and C2:

1. **Candidate ranking** (`_select_candidates`) — all `SitePI` at site, ranked by:
   ```
   rank = years_experience + total_trials + phase_trial_sum
          + exact_phase_boost × 3 + adjacent_phase_boost × 1.5
   ```
2. **OpenAI call per PI** — each PI receives study phase/TA context and full publication list (abstract truncated to 800 chars). Returns `relevance_score` + relevant publication list.
3. **Primary PI** — top `relevance_score` → `is_primary_recommendation: true`.
4. **Fallback** — if all AI calls fail → `_fallback_pi_recommendations()` (primary facility PI or first).

---

## C1 — PI Experience (5%)

**Purpose:** AI assigns a `relevance_score` per PI; the **primary recommended PI's** discrete experience score becomes `score_percent`.

**Discrete experience score (for primary PI):**

| Condition | PI score |
|-----------|----------|
| `is_experienced` (phase-specific threshold met) OR model says experienced | 100 |
| `total_trials ≥ 5` | 80 |
| `total_trials ≥ 2` | 60 |
| Otherwise | 40 |
| No PIs | 0 |

**Phase-aware `is_experienced` flag:**

| Study phase contains | Experienced if |
|----------------------|----------------|
| Phase II / 2 | `phase_ii ≥ 2` |
| Phase I / 1 | `phase_i ≥ 2` |
| Phase III / 3 | `phase_iii ≥ 2` |
| Phase IV / 4 | `phase_iv ≥ 2` |

**API response includes:** `principal_investigators[]` with `pi_id`, `name`, `specialty`, `years_experience`, `relevance_score`, `is_primary_recommendation`, `stats` (years_active, total_trials, is_experienced), `phase_experience` counts.

**Limitations:** Only primary PI determines `score_percent`. `relevance_score` is display-only; score uses discrete bands. One OpenAI call per PI — latency scales with PI count.

---

## C2 — PI Publication Quality (5%)

**Purpose:** Scores publication quality of the **primary recommended PI** using bibliometric metrics. Reuses C1's PI recommendation list — no extra OpenAI calls.

**Calculation:**

```
base_score        = 0.1
h_index_score:    ≥100 → 0.40 | ≥20 → 0.24 | ≥1 → 0.16 | none → 0.0
pub_count_score:  ≥200 → 0.30 | ≥40 → 0.18 | ≥1 → 0.12 | none → 0.0
retraction_penalty = min(retraction_count × 0.3, 0.9)

raw_score     = max(0, base + h_index_score + pub_count_score − retraction_penalty)
score_percent = round(raw_score × 100)
```

If no h-index **and** no publication count: `raw_score = 0.1` (base only).

**Publication list:** Only articles returned by OpenAI in the C1 pass are hydrated with full metadata. Sorted by `relevance_score` descending.

**Data sources:** `PI.h_index`, `PI.total_citations`, `PIArticle` + `Article` (OpenAlex ETL).

**API response includes:** Per-PI `h_index`, `total_works`, `citations`, `explanation`, `publications[]` (with `title`, `venue`, `citations_count`, `is_retracted`, `author_position`, `relevance_score`).

**Limitations:** Primary PI only. Publication list is AI-filtered subset, not full bibliography. Missing metrics → minimum score ~10%.

## Related

- [[Breakdown Scoring]]
- [[Breakdown Category A]]
- [[Breakdown Category D]]
- [[PI Evaluation]]
- [[AI Features]]

## Open Questions

- PI overrides: deactivation impact on C1/C2 not yet documented.
