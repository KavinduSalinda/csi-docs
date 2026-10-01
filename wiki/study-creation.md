---
title: Study Creation
created: 2026-06-21
updated: 2026-06-21
tags: [study-creation, studies, workflow, protocol, manual, assisted]
sources: ["[[raw/study-creation/README.md]]", "[[raw/study-creation/limitations.md]]"]
---

# Study Creation

Study creation is the entry point for defining a clinical trial in Sites Intelligence. Users can bootstrap a study **manually** (title + description, then complete fields in the workspace) or use **assisted** creation (upload a protocol PDF and let AI extract structured study data).

## Details

### Creation Modes

| Query param | `Study.create_mode` | Description |
|-------------|---------------------|-------------|
| `mode=manual` | `manual` | Minimal bootstrap; user fills all fields |
| `mode=assisted` | `protocol` | Protocol upload + AI structured extraction |

Invalid or missing `mode` returns **400 Invalid mode**. Both modes require `?project=<project_id>`.

### End-to-End Workflow

1. User selects manual or assisted mode.
2. **Manual:** `POST /studies/?mode=manual` with `title` + `description`. Study created, empty assistant session seeded.
3. **Assisted:** Upload PDF first → `POST /studies/?mode=assisted` with `file_name` + `file_hash`. AI extracts fields, clarification questions surfaced.
4. User completes/edits fields via `PUT /studies/{id}/` or through the AI assistant.
5. Additional documents can be attached (`POST /studies/{id}/documents/`) — each triggers background summarization.
6. Study is ready for site search and breakdown.

### API Overview

| Step | Method | Endpoint |
|------|--------|----------|
| Upload file | `POST` | `/api/v1/users/documents/` |
| Create study | `POST` | `/api/v1/studies/?project={id}&mode={manual\|assisted}` |
| Get study | `GET` | `/api/v1/studies/{id}/` |
| Complete / update study | `PUT` | `/api/v1/studies/{id}/` |
| Attach document | `POST` | `/api/v1/studies/{id}/documents/` |
| List documents | `GET` | `/api/v1/studies/{id}/documents/` |
| AI assistant | `POST` | `/api/v1/studies/{id}/assistant/` |
| Assistant sessions | `GET` | `/api/v1/studies/{id}/assistant/sessions/` |
| Session messages | `GET` | `/api/v1/studies/{id}/assistant/sessions/{session_id}/messages/` |

### Full Update Validation

`PUT /api/v1/studies/{id}/` requires these fields on non-partial updates:

| Required field | Notes |
|----------------|-------|
| `title` | Max 100 chars |
| `description` | Max 1500 chars |
| `sponsor` | Also accepts legacy typo key `sponser` |
| `study_phase` | UUID → `phase_id` |
| `therapeutic_area` | UUID |
| `mesh_terms` | UUID list → `StudyCondition` |
| `capabilities` | UUID list → `StudyCapability` |
| `study_region` | Integer ID list → `StudyRegion` |
| `countries` | UUID list → `StudyCountry` |

`PATCH` is **not supported** (405).

### Data Model

| Model | Role |
|-------|------|
| `Study` | Core study record; `create_mode`, `custom_data`, `pending_clarification_json` |
| `users.Document` | Uploaded file metadata + `file_hash` storage key |
| `StudyDocument` | Links study ↔ document; `source_type`, `ai_summary`, `summary_status` |
| `StudyAssistantSession` | Creation assistant conversation scope |
| `StudyAssistantMessage` | Persisted user/assistant turns |

### Code Entry Points

| Area | Location |
|------|----------|
| Study CRUD | `studies/views.py` → `StudyViewSet` |
| Assisted extraction | `studies/services.py` → `create_study_from_document_assisted()` |
| Document text extraction | `studies/document_content.py` |
| AI assistant (post-create) | `studies/services.py` → `process_study_assistant_interaction()` |
| File upload | `users/views.py` → `DocumentUploadAPIView` |

### Notable Edge Cases

- **`GET /studies/{id}/` auto-creates an assistant session** if none exists for the user — a side effect on a read endpoint.
- **Document delete** removes DB records but not the physical file in `MEDIA_ROOT`.
- **Manual mode + unauthenticated:** study created but `assistant_session_id` is null.
- **Assisted mode + unauthenticated:** 403 — ownership check requires authenticated user.
- **StudyStatus seed dependency:** both modes require at least one `StudyStatus` row; missing seed data → 422.
- **`create_mode` vs API `mode`:** `mode=assisted` in the API stores `create_mode=protocol` in the DB.

### Error Code Summary

| HTTP | Typical cause |
|------|---------------|
| 400 | Missing `project`, invalid `mode`, missing file on upload, invalid `source_type` |
| 401 | Unauthenticated upload |
| 403 | Document/session ownership |
| 404 | Document or study not found |
| 405 | PATCH study, unsupported document methods |
| 422 | Extraction/AI failure, validation errors |
| 503 | Assistant unexpected failure |

## Related

- [[Study Creation AI Modes]]
- [[File Uploads]]
- [[File Summarization]]
- [[Breakdown Pipeline]]
- [[Sites Intelligence Platform]]

## Open Questions

- Protocol summarization gap: assisted creation links the protocol via `StudyDocument` but does not call `summarize_study_documents()`. Summary remains unset until user attaches another document.
