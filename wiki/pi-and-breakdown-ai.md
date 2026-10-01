---
title: PI and Breakdown AI
created: 2026-06-21
updated: 2026-06-21
tags: [ai, breakdown, pi, pi-scoring, llm-insights, section-summaries, openai]
sources: ["[[raw/ai-features/pi-and-breakdown-ai.md]]", "[[raw/ai-features/limitations.md]]"]
---

# PI and Breakdown AI

AI capabilities used during study–site breakdown generation: PI relevance scoring, narrative insights, and executive section summaries. These run as part of the background breakdown job — not as standalone chat endpoints.

## Details

### PI Relevance Scoring

**Trigger:** Breakdown section C1 during `generate_full_breakdown()`.  
**Code:** `studies/breakdown/c_sections.py` → `get_study_site_pi_recommendations()`, `_score_single_pi()`  
**Model:** OpenAI `gpt-5.2`

**One call per PI** — all PIs linked to the site are scored, no count limit.

**Input per PI (JSON to OpenAI):**
```json
{
  "study": { "phase": "Phase 3", "therapeutic_area": "Oncology" },
  "pi": {
    "pi_id": "uuid",
    "name": "Dr. Jane Smith",
    "years_experience": 15,
    "phase_i_trials": 2,
    "phase_ii_trials": 5,
    "phase_iii_trials": 12,
    "phase_iv_trials": 1,
    "total_trials": 20,
    "article_count": 45,
    "publications": [
      {
        "article_id": "uuid",
        "abstract": "... (truncated to 800 chars)",
        "citation_count": 12,
        "publication_date": "2021-03-15",
        "venue": "Journal of Clinical Oncology",
        "is_retracted": false
      }
    ]
  }
}
```

**Output per PI:**
```json
{
  "pi_id": "uuid",
  "relevance_score": 82,
  "publications": [
    { "article_id": "uuid", "relevance_score": 90 },
    { "article_id": "uuid", "relevance_score": 71 }
  ]
}
```

| Score range | Interpretation |
|-------------|----------------|
| 0–39 | Poor fit |
| 40–59 | Marginal fit |
| 60–79 | Good fit |
| 80–100 | Excellent fit |

Retracted publications are penalized in the prompt and excluded from the returned publication list.

**Error handling:**

| Failure | Behavior |
|---------|----------|
| OpenAI call fails for one PI | PI skipped; others continue |
| Invalid JSON response | PI skipped |
| All calls fail | `_fallback_pi_recommendations()` (primary facility PI or first) |
| `OPENAI_API_KEY` missing | All scoring fails; fallback triggered |

Breakdown job continues even if some PI scores fail — partial results returned.

**Cost note:** Sites with hundreds of PIs generate hundreds of API calls. PI scoring is the primary cost and latency driver in full breakdown generation.

---

### LLM Insights

**Trigger:** End of breakdown generation.  
**Code:** `studies/breakdown/full_breakdown.py` → `generate_llm_insights()`  
**Model:** OpenAI `gpt-4o-mini`, temperature **0.7**, JSON response format

**Input:** Study metadata (title, TA, phase, enrollment, duration) + site metadata + text summary of weighted breakdown scores (`extract_breakdown_summary()`).

**Output:**
```json
{
  "strengths": ["Strong phase 3 oncology trial experience at site"],
  "risks": ["Limited MRI capability may affect imaging endpoints"]
}
```

These populate `strength`, `risk`, and `recommended_steps` on the breakdown response. They are **narrative interpretation** of already-computed scores — not a substitute for numeric results.

**Fallback:** Empty arrays when OpenAI unavailable. Numeric breakdown remains valid.

---

### Section Summaries

**Trigger:** During breakdown generation, per sub-section and per main category.  
**Code:** `studies/breakdown/full_breakdown.py` → `generate_sub_section_summary()`, `generate_main_section_summary()`  
**Model:** OpenRouter `openai/gpt-4o-mini`, temperature **0.3**

**Input:** Sub-section code, title, score %, and section JSON data (truncated to ~4000 chars). Category-level uses combined sub-section summary sentences.

**Output:** Single sentence, max **200 characters**, factual executive summary. These appear as `section_summary` on each sub-item in the API response.

**Fallback:** `_get_sub_section_fallback()` — predefined template text when OpenRouter fails. Breakdown always completes with human-readable text even when AI is unavailable.

**Call count:** ~13+ calls per breakdown (one per sub-section + one per main category).

---

### Limitations

| Constraint | Detail |
|------------|--------|
| PI scoring subjectivity | Relevance scores are model judgments, not clinical certification |
| Per-PI cost | Sites with many PIs multiply API calls linearly |
| Partial failure | Failed PI scores omitted; breakdown completes with remaining PIs |
| Insights optional | Empty strengths/risks when OpenAI unavailable — numeric breakdown still valid |
| Summary fallbacks | Template text used when OpenRouter fails — less specific than AI summaries |
| Retracted pubs | Excluded from output but may still influence PI score signal via publication list |
| Temperature variance | Identical PI data may produce slightly different scores across runs (temperature > 0 for insights) |

## Related

- [[AI Features]]
- [[Breakdown Category C]]
- [[Breakdown Scoring]]
- [[Breakdown Pipeline]]

## Open Questions
