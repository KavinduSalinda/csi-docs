---
title: Override Panel
created: 2026-06-21
updated: 2026-06-21
tags: [override-panel, admin, site-override, pi-override, study-history-override]
sources: ["[[raw/override-panel/README.md]]", "[[raw/override-panel/limitations.md]]"]
---

# Override Panel

The Override Panel lets **admin users** correct or enrich master data for **sites**, **principal investigators (PIs)**, and **study history** records without modifying ETL-fed source tables. Overrides are stored in separate Postgres tables and merged at **read time** through serializers.

## Details

### Design Principles

| Principle | Detail |
|-----------|--------|
| **Same primary key** | Override row `id` equals the base entity UUID |
| **Presentation layer** | Active overrides replace selected fields in API responses — base rows are unchanged |
| **Admin-only writes** | Create, update, and deactivate require `IsAuthenticated` + `IsAdminRole` |
| **Soft deactivation** | `PATCH {"is_active": false}` hides the entity from listings; detail returns **404** |
| **Audit trail** | Every override stores `created_by`, `updated_by`, `created_at`, `updated_at` |
| **No source mutation** | ETL rows (`Site`, `PI`, `StudyHistory`) are never updated by override APIs |

Override reads are merged by serializers — no changes reach graph DBs or ETL tables.

### Operations

| Action | Method | Notes |
|--------|--------|-------|
| Create | `POST` | First override for an entity; upserts if override already exists (same PK) |
| Read | `GET` | Returns current override payload, or base entity with `"override": null` if none |
| Update | `PUT` | Full replace — child collections (phases, capabilities, publications, etc.) deleted and recreated |
| Deactivate | `PATCH` | Only `{"is_active": false}` is accepted on PATCH — no partial field updates |

### Permission

| Operation | Requirement |
|-----------|-------------|
| Override create / update / deactivate | Authenticated + **`admin`** role (`UserProfile.role`) |
| Consume overridden values in API | Any user with normal entity access |

Permission class: `csi/permissions.py` → `IsAdminRole`. Missing `UserProfile` → admin check fails → **403**.

---

### Site Override

**Endpoint:** `POST/GET/PUT/PATCH /api/v1/sites/{site_id}/override/{site_id}/`  
**Code:** `sites/views.py` → `SiteOverrideViewSet`

| Overridable field | Type | Notes |
|-------------------|------|-------|
| `name` | string | |
| `description` | text | |
| `location` | string | |
| `institution_type` | UUID | API accepts UUID; stored as resolved **type name** string |
| `capacity` | integer | ≥ 0 |
| `site_phases` | UUID[] | → `SitePhaseOverride` |
| `specializations` | UUID[] | → `SiteTherapeuticAreaOverride` |
| `capabilities` | UUID[] | → `SiteCapabilityOverride`; status copied from base `SiteCapability` when present |
| `principal_investigators` | UUID[] | → `SitePIOverride`; `is_primary_facility` preserved from base |

**Not overridable:** Map coordinates (`map_lang`, `map_lat`) — always copied from base site.

---

### PI Override

**Endpoint:** `POST/GET/PUT/PATCH /api/v1/principal_investigators/{pi_id}/override/{pi_id}/`  
**Code:** `principal_investigators/views.py` → `PIOverrideViewSet`

| Overridable field | Type | Notes |
|-------------------|------|-------|
| `name` | string | |
| `orcid` | string | |
| `openalex_author_id` | string | Falls back to base PI value when omitted |
| `email` | string | |
| `phone` | string | |
| `recent_working` | string | Stored as `location` |
| `years_of_exp` | integer | ≥ 0 |
| `availability` | string | |
| `status` | string | |
| `rating` | float | Rounded to integer on save |
| `publications` | UUID[] | Article IDs → `PIArticleOverride`; invalid IDs skipped silently |
| `specializations` | UUID[] | → `PITherapeuticAreaOverride` |

---

### Study History Override

**Endpoint:** `POST/GET/PUT/PATCH /api/v1/studies/history/{study_history_id}/override/{study_history_id}/`  
**Code:** `studies/views.py` → `StudyHistoryOverrideViewSet`

| Overridable field | Type | Notes |
|-------------------|------|-------|
| `title` | string | |
| `primary_endpoint` | text | optional |
| `mesh_terms` | UUID[] | Conditions → `StudyHistoryConditionOverride`; also accepts `specializations` key |
| `phases` | UUID[] | → `StudyHistoryPhaseOverride` |
| `sites` | UUID[] | → `StudyHistorySiteOverride` |
| `principal_investigators` | UUID[] | → `StudyHistoryPiOverride` |
| `sponsors` | UUID[] | → `StudyHistorySponsorOverride`; optional; empty list clears sponsor overrides |

Invalid FK IDs (phases, sites, PIs, sponsors) are skipped, not errors.

**Not overridable:** Workspace `Study` records (protocol being built) — only study **history** (trial archive) is overridable.

---

### What Overrides Change vs. Don't Change

| Surface | Override applied? |
|---------|-------------------|
| Site/PI/history API responses | ✅ Yes |
| Site AI chat context | ✅ Yes (`SiteOverviewSerializer` uses override fields) |
| `has_override=true` filter on list endpoints | ✅ Yes |
| PI list sidebar filters (experience, publications) | Partial — uses override when PI has active override |
| Breakdown scoring (A1–D2 formulas) | ❌ No — generators read base `Site`/`PI` ORM fields |
| PI AI scoring input (C1) | ❌ No — uses base `PI` and `PIArticle` |
| Graph / Neo4j / Spanner | ❌ No — ETL nodes unchanged |
| Natural search / AI suggested sites | ❌ No — graph matching uses ETL data |
| Capability match after graph search | ❌ No — uses base `SiteCapability` |

Teams validating feasibility should re-run breakdown after source data corrections, or accept that override corrections may not flow into numeric scores.

### Deactivation Behavior

- Entity **hidden** from standard querysets; detail returns **404** to all users including admins.
- Override row and child rows retained in Postgres for audit.
- **Re-activation:** Requires `PUT` with full payload (and implicit `is_active: true`). Cannot re-activate via `PATCH`.
- **Bootstrap on PATCH deactivate:** If no prior override exists, a minimal override is created from base values first, then deactivated.
- **Cross-entity refs:** Deactivated sites/PIs filtered from study history override PI/site lists via `site_is_visible` / `pi_is_visible`.

### Audit Trail

Each override header row stores `created_at`, `created_by`, `updated_at`, `updated_by` (display usernames via `audit_username()`).

List serializers expose a compact summary:
```json
"override": {
  "override_count": 5,
  "created_at": "2025-06-01T10:00:00Z",
  "updated_at": "2025-06-08T14:30:00Z"
}
```
`override_count` includes scalar override + child rows.

**Audit limitations:** No field-level diff, no rollback API, no `StudyLogs` entry. Child row replacement means no history of prior phase/publication sets.

### Error Responses

| Condition | HTTP | Message |
|-----------|------|---------|
| Body `id` ≠ URL entity id | 400 | Body id must match site/PI/history id |
| Entity not found | 404 | Site / PI / study history not found |
| Override not found on PUT | 404 | Override not found |
| Non-admin user | 403 | Only admin role can perform this action |
| PATCH body not exactly `{"is_active": false}` | 400 | Validation error |
| Invalid FK UUID | 400 | Field validation errors (for site/PI; skipped for study history) |

### Operational Recommendations

1. Document *why* an override was applied externally — the API stores only who/when.
2. Prefer ETL fixes when graph matching and breakdown must align with corrected data long-term.
3. Use deactivate rather than workarounds — there is no DELETE endpoint.

## Related

- [[Sites Intelligence Platform]]
- [[Breakdown Scoring]]
- [[Natural Search]]
- [[Architecture]]

## Open Questions
