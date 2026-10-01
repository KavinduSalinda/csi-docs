---
title: File Uploads
created: 2026-06-21
updated: 2026-06-21
tags: [file-uploads, documents, storage, extraction, pymupdf]
sources: ["[[raw/study-creation/file-uploads.md]]", "[[raw/study-creation/limitations.md]]"]
---

# File Uploads

Study creation and document management use a **two-step pattern**: binary files are uploaded to media storage first; the metadata references (`file_name`, `file_hash`) are then passed to study APIs.

## Details

### Two-Step Architecture

```
Step 1 — Binary upload:
  POST /api/v1/users/documents/   (multipart)
  → MD5 hash computed, stored as MEDIA_ROOT/{hash}{ext}
  → returns file_name + file_hash to client

Step 2 — Metadata reference:
  POST /api/v1/studies/?mode=assisted   (JSON with file_name + file_hash)
  or
  POST /api/v1/studies/{id}/documents/  (JSON with file_name + file_hash + source_type)
  → creates users.Document DB row
  → links via StudyDocument
```

No `Document` DB row is created at Step 1 — only the physical file is stored.

### File Requirements for Assisted Study Creation

| Requirement | Detail |
|-------------|--------|
| Allowed format | **PDF only** (`.pdf`) |
| Max file size | **5 MB** |
| Content type | **Text-based PDF** — pages must contain embedded/selectable text |

**Not supported for assisted create:** `.docx`, `.txt`, any non-PDF, PDFs over 5 MB, scanned/image-only PDFs.

### Post-Creation Document Attach (Different Rules)

`POST /api/v1/studies/{id}/documents/` can accept `.docx` and `.txt` in addition to PDF. These rules do not apply to the initial assisted study creation upload.

### Step 1 — Binary Upload Processing

**Endpoint:** `POST /api/v1/users/documents/` (multipart, auth required)

1. Read uploaded file content.
2. Compute MD5 hash, take first 32 hex characters.
3. Preserve original extension: `hash_filename = {hash}{ext}`.
4. Save to `MEDIA_ROOT/{hash_filename}`.
5. Return original filename + storage key.

Identical file content → same storage filename (deduplication), but each API call still creates a **new** `Document` row with the same hash.

### Text Extraction

When content is needed (assisted create, summarization, assistant context), the server reads `MEDIA_ROOT/{document.file_hash}` via `extract_text_from_document()` in `studies/document_content.py`.

| Extension | Library | Method |
|-----------|---------|--------|
| `.pdf` | PyMuPDF (`fitz`) | `page.get_text()` — embedded text only, **no OCR** |
| `.docx` | `python-docx` | Non-empty paragraph text |
| `.txt` | Built-in | UTF-8 decode (`errors="ignore"`) |

**Size cap:** 80,000 chars per document (`MAX_CHARS_PER_DOC`). Oversized text is truncated with `[... truncated: document too large ...]`.

**Extraction error return strings (not exceptions):**

| Condition | Returned string |
|-----------|-----------------|
| File missing on disk | `[File not found: {file_hash}]` |
| Unsupported type | `[Unsupported file type: filename.xyz]` |
| Parse exception | `[Could not extract text from {file_name}: {error}]` |

Assisted creation treats any result starting with `[` or an empty string as failure → **422**.

### Scanned / Image-Only PDFs

PyMuPDF extracts text from the text layer only. Scanned PDFs store pages as images — `get_text()` returns empty or whitespace-only content. There is **no OCR**. The upload succeeds but assisted creation fails with **422 — Could not extract text from document**.

**Workaround:** Re-export from source application as a proper PDF, or run OCR externally and upload a searchable PDF.

### `users.Document` Model

| Field | Description |
|-------|-------------|
| `file_name` | Original display name |
| `file_hash` | Storage filename (MD5 + extension) |
| `uploaded_by` | FK to authenticated user |
| `uploaded_date` | Auto timestamp |

### Document Attach (Step 2B)

```http
POST /api/v1/studies/{study_id}/documents/
```

**Body:**

```json
{
  "file_name": "amendment_1.pdf",
  "file_hash": "deadbeef0123456789abcdef01234567.pdf",
  "source_type": "document"
}
```

`source_type` can be `"protocol"` or `"document"` (default `"document"`). Triggers background summarization after attach.

### List and Delete

- **List:** `GET /api/v1/studies/{study_id}/documents/?source_type=protocol`
- **Delete:** `DELETE /api/v1/studies/{study_id}/documents/{study_document_id}/` — removes DB records but **not** the physical file in `MEDIA_ROOT`.

### Security Model

| Rule | Enforcement |
|------|-------------|
| Upload requires auth | `DocumentUploadAPIView.permission_classes` |
| Assisted create ownership | `doc.uploaded_by_id == user.id` |
| Assistant session ownership | `session.created_by_id == user.id` |

No virus scanning or content-type verification beyond file extension at extraction time.

## Related

- [[Study Creation]]
- [[Study Creation AI Modes]]
- [[File Summarization]]

## Open Questions
