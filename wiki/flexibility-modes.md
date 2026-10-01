---
title: Flexibility Modes and AI Suggested Sites
created: 2026-06-21
updated: 2026-06-21
tags: [flexibility-modes, ai-suggestions, site-matching, graph, cypher, neo4j, spanner]
sources: ["[[raw/flexibility-modes/README.md]]", "[[raw/flexibility-modes/ai-suggestions.md]]", "[[raw/flexibility-modes/limitations.md]]"]
---

# Flexibility Modes and AI Suggested Sites

Flexibility modes control **how strictly** study requirements are applied when the platform suggests candidate sites. **AI Suggested Sites** is the study workspace feature that lists those candidates — matched from the knowledge graph against saved study attributes.

> **Important:** Suggested sites are **graph-driven, not AI-generated.** The product label "AI Suggestions" refers to automated site discovery. Matching is performed by graph database queries (Neo4j or Google Spanner) against structured study criteria. OpenAI is used only to **author Cypher query strings** from fixed templates — it does not score, rank, or recommend sites.

## Details

### What Gets Matched

Study attributes are converted to graph match criteria via `build_site_match_attrs(study)`:

| Study data | Graph criterion |
|------------|-----------------|
| `therapeutic_area` | `(site)-[:SPECIALISES_IN]->(TherapeuticArea)` |
| `StudyCondition` rows (MeSH) | Mesh term via dual-path (studies at site + TA mapping) |
| `StudyCountry` rows | `(site)-[:LOCATED_IN]->(Country)` |
| `StudyRegion` rows | Region name via `(:Country)-[:PART_OF]->(Region)` |
| `StudyCapability` rows | **Not in graph** — Postgres overlay after graph match |
| PI requirements, enrollment, phase | **Not used** for site suggestion matching |

At least one graph criterion (TA, mesh, country, or region) must be present; otherwise matching returns zero results.

### Flexibility Modes

Three modes are stored on the study as `match_flexibility` and drive Cypher template selection:

| Mode | Logic | User-facing intent |
|------|-------|-------------------|
| **strict** | AND — site must satisfy **every** criterion | No compromises on any dimension |
| **balanced** | Anchor + optional — site must satisfy the anchor criterion **and at least one other** | Most requirements met; minor deviations acceptable |
| **exploratory** | OR — site must satisfy **at least one** criterion | Broader net; close matches included |

**Anchor order** (first populated criterion anchors balanced query): TherapeuticArea → Country → Region → MeshTerm.

**Strict mesh rule:** Requires both graph paths OR'd: studies at site tagged with MeSH term (`CONDUCTED_AT` → `Study` ← `TAGGED_IN`) **plus** site TA linked via `MAPS_TO_THERAPEUTIC_AREA`.

**Balanced mesh rule:** Uses only the `TAGGED_IN` / `CONDUCTED_AT` branch — not `MAPS_TO_THERAPEUTIC_AREA`.

**Exploratory:** Each criterion becomes an independent `UNION` branch ending in `RETURN DISTINCT site.uuid AS uuid`.

### Default and Override Rules

| Context | Flexibility used |
|---------|------------------|
| New study (manual create) | `strict` |
| Study model default | `balanced` |
| AI suggestions listing | Saved `study.match_flexibility` |
| Count API (no param) | Saved `study.match_flexibility` |
| Count API `?flexibility=` | Request-only override — does not persist |
| Study-only site list (no `context=ai_suggestions`, no description) | Always **strict** |

Invalid flexibility values fall back to `strict`.

---

### Suggestion Generation Process

```
Study attrs → build_site_match_attrs
           → Apply match_flexibility template (strict / balanced / exploratory)
           → get_or_create_site_match_cypher (PromptCache — 8h TTL)
             Cache miss → OpenAI gpt-4o-mini writes Cypher from template + attrs
           → Execute on Neo4j / Spanner → matched site UUIDs
           → Paginate UUIDs → return Site records from Postgres in graph order
```

OpenAI receives a fixed system prompt, graph schema, mode-specific Cypher templates, and structured study attributes. It outputs a Cypher string only. The graph DB executes it. No LLM evaluates or ranks individual sites.

### Capability Overlay

Equipment matching is **excluded from graph queries**. After graph matching, `filter_by_capabilities` intersects candidates with `SiteCapability` rows:

| Flexibility | Minimum capability overlap |
|-------------|---------------------------|
| strict | All study capability IDs (`cnt >= N`) |
| balanced | At least half, rounded up: `max(1, (N+1)//2)` |
| exploratory | At least one (`cnt >= 1`) |

Only capabilities with `status="ok"` count. The **count API intentionally excludes capabilities** to avoid misleadingly low banner numbers when capability data is sparse.

---

### API Reference

**Get/update flexibility mode:**

```http
GET  /api/v1/studies/{id}/flexibility/
PATCH /api/v1/studies/{id}/flexibility/   Body: { "flexibility": "exploratory" }
```

**AI suggested site count:**

```http
GET /api/v1/studies/{id}/ai-suggested-sites/count/?flexibility=balanced
```
Response: `{ "count": 47 }`. `?flexibility=` overrides for preview without saving.

| Count error | HTTP |
|-------------|------|
| Study not found | 404 |
| Invalid Cypher generation | 502 |
| Graph backend unavailable | 503 |

**AI suggestions listing:**

```http
GET /api/v1/sites/?study={id}&context=ai_suggestions&page=1
    &specialization={ta_id}&country={id}&region={id}&min_capacity={n}&mesh_term={mesh_id}
```

Page size fixed at **10**. Response includes `_meta.context` and `_meta.flexibility`. Graph/Cypher failures return **HTTP 200** with empty `_data` (fail-soft).

### Sidebar Override Rules

| Sidebar param | Effect on study attrs |
|---------------|----------------------|
| `specialization` | Replaces therapeutic area; **clears mesh terms** |
| `country` | Replaces country list |
| `mesh_term` | Replaces mesh term list |
| `region`, `phase` | Applied via Postgres enforcement after graph |
| `min_capacity` | Postgres filter on `Site.capacity` |

Sidebar wins over study baseline on the same dimension.

---

### Result Ordering

Suggested sites are ordered by **`site.uuid`** — deterministic, not AI relevance ranking. No semantic match score is applied at the suggestion layer. Breakdown `match_score` is computed separately and not used to order suggestions.

---

### Graph Schema (Site Matching Subset)

```
(site:Site)          uuid, name, country_id, institution_type
(ta:TherapeuticArea) uuid, name
(c:Country)          uuid, name
(r:Region)           uuid, name
(m:MeshTerm)         mesh_id, name
(st:Study)           uuid, study_status_id

(site)-[:LOCATED_IN]->(c)-[:PART_OF]->(r)
(site)-[:SPECIALISES_IN]->(ta)
(site)-[:CONDUCTED_AT]->(st)
(m)-[:TAGGED_IN]->(st)
(m)-[:MAPS_TO_THERAPEUTIC_AREA]->(ta)
```

MeSH node IDs use `mesh_id` (not UUID). Region matching uses **region name** strings — renamed regions in ETL break matches until graph is refreshed.

**Provider config:**

| Variable | Default | Notes |
|----------|---------|-------|
| `GRAPH_DB_PROVIDER` | `neo4j` | General graph workloads |
| `SITE_STUDY_MATCH_GRAPH_PROVIDER` | falls back to `GRAPH_DB_PROVIDER` | Study match + AI suggestions |

---

### Limitations

| Limitation | Detail |
|------------|--------|
| Graph criteria required | Study must have TA, MeSH, countries, or regions; empty criteria → zero suggestions |
| Capabilities sparse in graph | Equipment matching in Postgres only; excluded from count API |
| No PI / phase / enrollment in graph match | Suggestion matching uses TA, mesh, geography only |
| Fixed page size | 10 sites per page |
| No semantic ranking | Cannot sort by predicted fit; use breakdown for scored evaluation |
| OpenAI for Cypher only | Requires `OPENAI_API_KEY` for Cypher generation on cache miss |
| 8-hour Cypher cache | Changing prompts requires bumping `PROMPT_VERSION` in `sites/services.py` |
| MeSH ID format | Graph uses `mesh_id` on MeshTerm nodes, not Postgres condition UUIDs |

### Edge Cases

| Condition | Behavior |
|-----------|----------|
| `context=ai_suggestions` without `study` | **400** |
| Invalid study ID | **404** |
| Unknown `match_flexibility` value | Normalizes to **strict** |
| Balanced with single criterion | Degenerates to single-dimension match |
| Strict mesh Cypher failure | Triggers regeneration (up to 2 attempts); persistent failure → 502 on count, empty list |
| Neo4j syntax error | `_recover_cypher` retry; listing fail-softs to empty |
| Spanner balanced/exploratory | May use staged GQL fallbacks; results consistent, higher latency |
| Graph/Cypher failure on listing | HTTP 200 with empty `_data` |

### Code Entry Points

| Area | Location |
|------|----------|
| Flexibility API | `studies/views.py` → `StudyFlexibilityAPIView` |
| Suggested-site count | `studies/views.py` → `StudyAISuggestedSitesCountAPIView` |
| Site study selection listing | `sites/views.py` → `SiteViewSet.list()` (`context=ai_suggestions`) |
| Match attrs + Cypher generation | `sites/services.py` |
| Graph providers | `csi/graph_providers.py` |
| Spanner study-match execution | `csi/spanner_service.py` |

## Related

- [[Sites Intelligence Platform]]
- [[Architecture]]
- [[Natural Search]]
- [[Breakdown Scoring]]
- [[Override Panel]]

## Open Questions
