---
title: Breakdown Pipeline
created: 2026-06-21
updated: 2026-06-21
tags: [breakdown, pipeline, async, background-jobs, llm]
sources: ["[[raw/README.md]]"]
---

# Breakdown Pipeline

The breakdown pipeline is the async workflow that generates a weighted feasibility report for a study–site pair, orchestrating data collection across all scoring categories and LLM-assisted narrative generation.

## Details

### Async Flow

```
Frontend → POST /breakdown/
         → start_breakdown_job_in_background
         → A1 → A4 → B1 → C1/C2 → D1/D2
         → PI scoring + narrative insights (OpenAI)
         → persist DetailedBreakdown (score_breakdown + match_score)
Frontend → GET /breakdown/
         → match_score, score_breakdown, overview
```

1. Frontend initiates with `POST /breakdown/`.
2. API starts a background worker via `start_breakdown_job_in_background`.
3. Worker calls `generate_full_breakdown`, which runs category sections sequentially: A1 → A4 → B1 → C1/C2 → D1/D2.
4. LLM (OpenAI) is called for PI scoring and narrative insights (strengths, risks, next steps).
5. Results are persisted as a `DetailedBreakdown` record with a `score_breakdown` and overall `match_score`.
6. Frontend polls `GET /breakdown/` (or the job status endpoint) until the result is ready.

### Job Status Polling

- **Job endpoint**: `GET /api/v1/studies/breakdown-jobs/<job_id>/` — used to track async progress.
- Jobs support progress tracking and versioning, so multiple breakdown runs for the same study–site pair can coexist.

### Implementation

- Code lives under `studies/breakdown/`.
- See `dev-docs/STUDY_SITE_BREAKDOWN_OVERVIEW.md` for overview card fields.
- See `dev-docs/A1_BREAKDOWN_MESH_HIERARCHY_README.md` for A1 MeSH hierarchy details.

## Related

- [[Breakdown Scoring]]
- [[Architecture]]
- [[Sites Intelligence Platform]]
- [[PI Evaluation]]

## Open Questions
