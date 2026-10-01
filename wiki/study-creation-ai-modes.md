---
title: Study Creation AI Modes
created: 2026-06-21
updated: 2026-06-21
tags: [study-creation, ai, openrouter, extraction, assistant, protocol]
sources: ["[[raw/study-creation/ai-modes.md]]", "[[raw/study-creation/limitations.md]]"]
---

# Study Creation AI Modes

Sites Intelligence supports two AI-driven flows during study creation: **assisted mode** (AI extracts structured fields from an uploaded protocol PDF at create time) and the **post-creation hybrid assistant** (conversational field updates and clarification handling for both manual and assisted studies).

## Details

### Mode Comparison

| Aspect | Manual (`mode=manual`) | Assisted (`mode=assisted`) |
|--------|------------------------|------------------------------|
| `Study.create_mode` | `manual` | `protocol` |
| Input required | `title`, `description` | `file_name`, `file_hash` |
| AI at create time | No | Yes — structured extraction |
| Initial fields | Title + description only | AI-extracted scalars + relations + custom sections |
| Clarification questions | None | `pending_clarification_json` populated |
| Assistant session | Created empty | Seeded with extraction payload |
| Default flexibility | `strict` | `balanced` (model default) |
| Protocol document link | No | `StudyDocument(source_type=protocol)` |

### Assisted Mode: Structured Extraction Pipeline

```
POST /studies/?mode=assisted
  → verify document ownership
  → extract_text_from_document (first 60,000 chars)
  → _run_structured_extraction (OpenRouter via call_ai())
  → parse fields, sections, clarification_questions
  → Create Study (create_mode=protocol)
  → Apply phase/TA/capabilities/MeSH/regions/countries
  → StudyDocument (source_type=protocol)
  → StudyAssistantSession + seed message
```

**AI provider:** OpenRouter via `call_ai()`. Test fallback: `USE_OPENAI_SUMMARY_FOR_TESTING` routes to OpenAI direct.

**Input cap:** First **60,000** characters of extracted text.

### Structured Extraction Output Shape

```json
{
  "fields": {
    "title": null,
    "description": null,
    "phase": null,
    "therapeutic_area": null,
    "currency": null,
    "primary_indication": null,
    "target_enrollment": null,
    "capabilities": [],
    "mesh_terms": [],
    "regions": [],
    "countries": []
  },
  "sections": [],
  "clarification_questions": []
}
```

### Relation Field Resolution

AI returns string names; the backend resolves them to DB records:

| AI field | Resolution |
|----------|------------|
| `phase` | Fuzzy match against `StudyPhase` (`PHASE_MAPPING`) |
| `therapeutic_area` | Match `TherapeuticArea.name` |
| `currency` | Match `Currency.currency_code` |
| `capabilities` | Match `CapabilityEquipment.name` |
| `mesh_terms` | Match `Condition.condition` text |
| `regions` | Match `Region.name` |
| `countries` | Match `Country.name` |

Unmapped relations may generate additional clarification questions (`mapping_clarifications`).

Custom sections stored in `custom_data.sections`; clarifications stored in `pending_clarification_json`.

### AI Extraction Constraints

| Rule | Detail |
|------|--------|
| No hallucination | Prompt forbids inference; null when unknown |
| Date strictness | Only full dates (`YYYY-MM-DD` etc.); `Q3 2025` → null |
| Custom sections | Max 2 sections × 4 fields |
| Field types allowed | `text`, `long_text`, `number`, `date`, `boolean`, `select` |
| Clarification questions | Dropped if fewer than 2 answer options |
| JSON retry | One automatic retry with higher token limit (6000 → 10000) on parse failure |
| Title / description length | Title max 100 chars; description max 1500 chars |

### Post-Creation Assistant

Available to both manual and assisted studies after creation.

**Endpoint:** `POST /api/v1/studies/{study_id}/assistant/`

**Capabilities:**

| AI response `type` | Behavior |
|--------------------|----------|
| `CHAT` | Conversational reply only |
| Field updates | Applies validated changes to `Study` scalars and relations |
| Clarification | Updates `pending_clarification_json`; resolves answered questions |

**Context loaded for AI:**
- Full updatable study scalar fields (`_build_assistant_study_context`)
- Pending clarification questions + saved answers
- Recent session message history
- Protocol document text up to **50,000** chars aggregated from `StudyDocument` records

**Field update safety:**
- Blocked fields: `id`, `project`, `status`, `create_mode`, etc. (`BLOCKED_STUDY_UPDATE_FIELDS`)
- Invalid coercions silently skipped (`INVALID_UPDATE`)
- Session must be active and owned by requesting user

### Text Character Caps by Stage

| Stage | Limit |
|-------|-------|
| Text extraction per doc | 80,000 chars (`MAX_CHARS_PER_DOC`) |
| AI structured extraction input | 60,000 chars |
| Summarization input | 20,000 chars |
| Assistant protocol context | 50,000 chars (aggregated docs) |
| Summarization output | 1,000 chars (prompt rule) |

### Error Conditions (Assisted Mode)

| Exception | HTTP | Message |
|-----------|------|---------|
| Document not found | 404 | Document not found in UserDocument table |
| Wrong owner / not authenticated | 403 | User has no access to document |
| Extraction text failure | 422 | Could not extract text from document |
| AI / seed failure | 422 | AI extraction failed |

## Related

- [[Study Creation]]
- [[File Uploads]]
- [[File Summarization]]
- [[AI Features]]

## Open Questions
