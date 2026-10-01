---
title: Natural Search
created: 2026-06-21
updated: 2026-06-21
tags: [natural-search, site-search, pi-search, neo4j, spanner, openai, cypher]
sources: ["[[raw/natural-search/README.md]]", "[[raw/natural-search/limitations.md]]"]
---

# Natural Search

Natural language search lets users find **sites** and **principal investigators (PIs)** by typing free-text queries instead of setting every filter manually. Both flows translate user text into graph queries (Neo4j or Spanner), paginate UUID results, and re-fetch full records from PostgreSQL in graph order.

## Details

### Shared Architecture

| Entity | Entry point | Trigger |
|--------|-------------|---------|
| Site | `sites/views.py` → `SiteViewSet.list()` | `?description=<text>` |
| PI | `principal_investigators/views.py` → `PIViewSet.list()` | `?description=<text>` |

Without `?description=`, both endpoints fall back to standard Postgres listing (sidebar filters only, DRF pagination).

**Shared infrastructure:** `csi/graph_providers.py` (Neo4j / Spanner), OpenAI Cypher generation, `PromptCache` (8-hour TTL).

**Graph providers:** Default is Neo4j; Spanner support is available via `GRAPH_DB_PROVIDER` env var. Misconfigured provider → empty listings (fail-soft) or 503 for PI Neo4j driver failure.

---

## Site Natural Search

### When it runs

Graph-backed site search activates when `?description=` is non-empty. Optional structured sidebar params refine the same request.

### Flow

```
GET /sites/?description=oncology in USA
  → merge_site_listing_effective_attrs (resolve postgres_filters + enforcement)
  → get_or_create_site_listing_merge_cypher (OpenAI — cached 8h)
  → Graph DB: COUNT + paginated UUID query
  → Postgres enforcement (geography, TA, phase, mesh)
  → Postgres filters (active studies, capacity)
  → Fetch Site rows in graph order
```

On overlapping dimensions, **sidebar wins** over natural text. Postgres **enforcement** re-applies merged geography / TA / phase / mesh after graph execution so missing Cypher clauses do not leak wrong countries.

### Supported Query Patterns

**Geography:** `USA`, `US`, `U.S.A.` → United States (alias expansion). `UK`, `UAE` expanded. Full country or region names matched by substring against lookup.

**Therapeutic area / phase:** Disease domain (*oncology*, *cardiology*) → `SPECIALISES_IN` → TherapeuticArea. `phase 2`, `Phase II`, `p2` → `OPERATES_IN_PHASE` → Phase. Bare numbers in *"greater than 2"* are **not** treated as phase unless *phase* or `p1`–`p4` appears.

**Active / recruiting study counts:** *"active studies greater than N"* → `active_studies__gt=N` in Postgres. Common typos tolerated (*"count must be than N"* → `> N`).

**Capacity:** *"capacity greater than N"* → Postgres `capacity__gte`. Never filtered in Cypher.

**Combined:** ANDs all resolved dimensions (country + TA + recruiting count, etc.).

### Sidebar Filters (Site)

| Query param | Maps to |
|-------------|---------|
| `specialization` | Therapeutic area ID(s), comma-separated |
| `phase` | Study phase ID |
| `country` | Country ID(s), comma-separated |
| `region` | Region ID |
| `min_capacity` | Minimum site capacity (integer) |

### API

```http
GET /api/v1/sites/?description={text}&page={n}
    &specialization={ta_id}&phase={phase_id}&country={country_id}
    &region={region_id}&min_capacity={n}
Authorization: Bearer <token>
```

Page size fixed at **10**. Response uses custom `_pagination` + `_data` envelope. Cypher or graph failures return **HTTP 200** with empty `_data` (fail-soft).

**Ordering:** `ORDER BY site.uuid` — deterministic, not relevance-ranked.

**Code:** `sites/services.py` — `get_or_create_site_listing_merge_cypher`, `merge_site_listing_effective_attrs`, `paginate_site_listing_from_graph`.

### Fallback Behavior (Site)

| Condition | Result |
|-----------|--------|
| Empty description | Standard Postgres listing (no OpenAI call) |
| Cypher syntax / runtime error | HTTP 200 — empty `_data` |
| Cypher omits geography clause | Postgres enforcement guard re-applies it |

---

## PI Natural Search

### When it runs

Graph-backed PI search activates when `?description=` is present.

### Four-Step Pipeline

```
Step 1 — Lookups: Load TAs, regions, countries, phases from Postgres (passed to OpenAI as canonical UUIDs)
Step 2 — Cypher gen: get_or_create_pi_cypher (PromptCache keyed on normalized description; miss → OpenAI, up to 2 attempts with self-correction)
Step 3 — Sidebar patch: patch_pi_cypher_with_filters (injects/replaces constraints; sidebar NOT part of cache key)
Step 4 — Execute: COUNT + paginated UUID query → normalize UUIDs → re-fetch Postgres rows in order
```

### Supported Query Patterns (PI)

OpenAI interprets free text against the PI graph schema:

| Dimension | Graph pattern |
|-----------|---------------|
| Therapeutic expertise | `(pi)-[:EXPERTISE_IN]->(TherapeuticArea)` |
| Site / geography | `(pi)-[:WORKS_AT]->(Site)-[:LOCATED_IN]->(Country)` |
| Phase experience | `(pi)-[:HAS_PHASE]->(Phase)` |
| Trial participation | `(pi)-[:PARTICIPATED_IN]->(Study)` |
| Publications | `(pi)-[:AUTHORED]->(Article)` with count aggregation |
| MeSH / conditions | `(m)-[:TAGGED_IN]->(Study)` linked to PI trials |

Text matching uses `toLower(x) CONTAINS toLower('value')`. Lookup table UUIDs are inlined — no `$parameter` placeholders.

### Sidebar Filters (PI) — Applied via Cypher Patching

| Query param | Cypher effect |
|-------------|---------------|
| `min_experence` | `pi.years_active >= N` *(note: typo in API param name)* |
| `min_publications` | `count((pi)-[:AUTHORED]->(:Article)) >= N` |
| `max_active_study_count` | `pi.total_trials <= N` |
| `specialization` | `:EXPERTISE_IN` → TherapeuticArea UUIDs (sidebar overrides description TA) |

Patch safety: if patching cannot parse the Cypher tail, the original query is returned unchanged.

### API

```http
GET /api/v1/principal_investigators/?description={text}&page={n}
    &specialization={ta_id}&min_experence={n}&min_publications={n}
    &max_active_study_count={n}
Authorization: Bearer <token>
```

Page size fixed at **10**. Same `_pagination` + `_data` envelope as site search.

| Error condition | HTTP | Behavior |
|-----------------|------|----------|
| Cypher generation failure | 200 | Empty `_data` (fail-soft) |
| Neo4j driver unavailable | 503 | Error in standard envelope |
| Graph execution failure | 200 | Empty `_data` (fail-soft) |

**Code:** `principal_investigators/services.py` — `get_or_create_pi_cypher`, `patch_pi_cypher_with_filters`.

---

## Shared Limitations

| Limitation | Detail |
|------------|--------|
| No semantic ranking | Both flows order by graph UUID — not relevance or match quality |
| Fixed page size | 10 items per page, not configurable |
| OpenAI dependency | Cypher generation requires OpenAI (`gpt-4o-mini`); first request per novel text waits for generation |
| 8-hour prompt cache | Identical normalized descriptions reuse cached Cypher; prompt version bumps invalidate site cache entries |
| Fail-soft listing | Most graph/Cypher failures return HTTP 200 with empty `_data` |
| English-oriented | Geography shortcodes expanded for English; other languages rely on lookup name matching |
| Cache key excludes sidebar (PI) | Same description + different sidebar filters reuses base Cypher; filters are patched per request |
| PI UUID format | Neo4j stores PI UUIDs as 32-char hex without dashes; API normalizes to dashed Postgres format |
| Schema-bound (PI) | OpenAI only uses labels/relationships in `PI_NEO4J_SCHEMA`; schema changes require prompt updates |

## Related

- [[Architecture]]
- [[AI Features]]
- [[Sites Intelligence Platform]]
- [[Overrides and Flexibility]]

## Open Questions
