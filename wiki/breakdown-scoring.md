---
title: Breakdown Scoring
created: 2026-06-21
updated: 2026-06-21
tags: [breakdown, scoring, match-score, feasibility, categories]
sources: ["[[raw/breakdown/README.md]]"]
---

# Breakdown Scoring

The Breakdown system evaluates how well a clinical research site fits a specific study. For each study–site pair it produces a structured report with per-section scores, an overall **match score** (0–100), narrative strengths/risks, and UI-ready detail payloads.

## Details

### Category Groups and Weights

| Code | Title | Weight |
|------|-------|--------|
| **A** | Site Capabilities & Match | **50%** |
| **B** | Historical Performance | **30%** |
| **C** | PI Experience | **10%** |
| **D** | Regulatory & Institution | **10%** |

### Scoring Math

**Sub-section score:** Each generator returns `item_code`, `item_title`, `weight_percent`, `score_percent`, and `details`. `weight_percent` is expressed as percentage of the **total** match score.

**Category score:**
```
category_score = Σ(sub_item.score_percent × sub_item.weight_percent) / Σ(sub_item.weight_percent)
```

**Match score:**
```
match_score = Σ(category.score_percent × category.weight_percent) / Σ(category.weight_percent)
```

All scores are clamped to **0–100**. LLM narrative (strengths, risks, next steps) does **not** alter `match_score`.

### All Sub-sections at a Glance

| Code | Title | Weight (total) | Status |
|------|-------|----------------|--------|
| A1 | Therapeutic Area Match | 15% | Implemented |
| A2 | Phase Experience Match | 10% | Implemented |
| A3 | Site Capabilities Match | 10% | Implemented |
| A4 | Competing Trials Analysis | 15% | Implemented |
| B1 | Enrollment Performance | 30% | Implemented |
| B2 | — (planned) | — | Not implemented |
| B3 | — (planned) | — | Not implemented |
| C1 | PI Experience | 5% | Implemented |
| C2 | PI Publication Quality | 5% | Implemented |
| D1 | Country Regulatory Environment | 5% | Implemented |
| D2 | Institution Type | 5% | Implemented |

### Generation Order and Progress

| Progress | Milestone |
|----------|-----------|
| 15% | A1 complete |
| 25% | A2 complete |
| 30% | A3 complete |
| 35% | A4 complete |
| 38% | B1 complete |
| — | PI recommendations (shared by C1 + C2) |
| 100% | All categories + LLM insights persisted |

Code entry point: `studies/breakdown/full_breakdown.py` → `generate_full_breakdown()`.

### API Endpoints

| Method | Path | Behavior |
|--------|------|----------|
| `GET` | `/api/v1/studies/<study_pk>/sites/<site_pk>/breakdown/` | Return saved breakdown or in-progress job status |
| `POST` | `/api/v1/studies/<study_pk>/sites/<site_pk>/breakdown/` | Start initial generation (409 if already exists) |
| `POST` | `/api/v1/studies/<study_pk>/sites/<site_pk>/breakdown/regenerate/` | Delete snapshot and regenerate (version increments) |
| `GET` | `/api/v1/studies/breakdown-jobs/<job_id>/` | Poll job progress |

### Top-Level Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `match_score` | float \| null | Overall 0–100 score |
| `score_breakdown` | array \| null | Four category objects (A, B, C, D) with nested `sub_items` |
| `overview` | object | Study/site metadata, report version, timestamps |
| `strength` | string[] | LLM-generated strengths |
| `risk` | string[] | LLM-generated risks |
| `recommended_steps` | string[] | LLM-generated next steps |
| `job` | object | `{ id, status, message, progress }` |

JSON path to a sub-section:
```
data.score_breakdown[] → category "A" → sub_items[] → item_code "A1" → details
```

## Related

- [[Breakdown Pipeline]]
- [[Breakdown Category A]]
- [[Breakdown Category B]]
- [[Breakdown Category C]]
- [[Breakdown Category D]]
- [[PI Evaluation]]
- [[Sites Intelligence Platform]]

## Open Questions
