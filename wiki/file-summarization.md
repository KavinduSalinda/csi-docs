---
title: File Summarization
created: 2026-06-21
updated: 2026-06-21
tags: [file-summarization, documents, async, openrouter, background-jobs]
sources: ["[[raw/study-creation/file-summarization.md]]", "[[raw/study-creation/limitations.md]]"]
---

# File Summarization

When a document is attached to an existing study, the backend generates an AI summary asynchronously. This powers document previews in the UI and supplements assistant/chat context without blocking the attach API response.

## Details

### Trigger

Summarization runs only when a document is attached to an **existing** study:

```http
POST /api/v1/studies/{study_id}/documents/
```

It is **not** triggered during assisted study creation's initial protocol link. Assisted mode creates the `StudyDocument` inside `create_study_from_document_assisted()` without calling summarization.

### Processing Flow

```
POST /documents/
  → Create StudyDocument (immediate)
  → 200 OK returned to client
  → Background daemon thread: summarize_study_documents(study_document_id)
      → summary_status = pending
      → extract_text_from_document
        ✗ failure → summary_status = failed, ai_summary = diagnostic text
        ✓ success → trim to 20,000 chars
            → call_ai() (OpenRouter) to summarize
              ✓ → summary_status = completed, ai_summary = stripped AI content
              ✗ → summary_status = failed, ai_summary = "Summary generation failed: {error}"
```

### `StudyDocument` Summary Fields

| Field | Type | Description |
|-------|------|-------------|
| `ai_summary` | text | Generated summary or error diagnostic |
| `summary_status` | string | `pending` → `completed` \| `failed` |
| `summary_updated_at` | datetime | Set on completion or failure |

### AI Call Parameters

**Provider:** OpenRouter via `call_ai()`. Test fallback: `USE_OPENAI_SUMMARY_FOR_TESTING` routes to `get_chat_response()` (OpenAI direct).

| Parameter | Value |
|-----------|-------|
| `temperature` | 0.2 |
| `max_tokens` | 500 |
| Input cap | **20,000** chars (trimmed from extracted text) |
| Output cap | **1,000** chars (enforced in prompt) |

**Prompt rules:** Summarize only information explicitly in the text — no inference, no external knowledge. Single compact paragraph.

### Error Handling Philosophy

Summarization is **fire-and-forget by design**:
- Runs in a daemon thread — never blocks API response.
- All exceptions are swallowed and logged; failures do not roll back document attachment.
- The caller cannot poll status from the create response — must re-fetch `GET /studies/{id}/documents/`.

### Consumers

| Consumer | Uses summary? |
|----------|---------------|
| Study documents list/detail UI | Yes — displays `ai_summary` when `completed` |
| Hybrid assistant | Uses full extracted text (not summary), up to 50,000 chars |

### Example States

**Completed:**
```json
{
  "ai_summary": "This Phase III study evaluates adjuvant immunotherapy...",
  "summary_status": "completed"
}
```

**Failed (unsupported file type):**
```json
{
  "ai_summary": "[Unsupported file type: data.xlsx]",
  "summary_status": "failed"
}
```

**Pending (immediately after attach):**
```json
{
  "summary_status": "pending",
  "ai_summary": null
}
```

### Code Location

`summarize_study_documents()` → background `_run(doc_id)` in `studies/services.py`.

## Related

- [[Study Creation]]
- [[File Uploads]]
- [[Study Creation AI Modes]]

## Open Questions

- Protocol summarization gap: assisted creation does not trigger summarization for the initial protocol link. Summary remains unset until user attaches another document, or summarization is added to the assisted flow.
