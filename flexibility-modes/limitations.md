# Limitations & Edge Cases

Known constraints for flexibility modes and AI suggested site matching.

← [Back to Flexibility Modes & AI Suggestions](./README.md)

---

## Recommendations are graph-driven, not AI-generated

This is the primary product constraint users and integrators should understand:

| Statement | Accurate? |
|-----------|-----------|
| "AI Suggested Sites uses an LLM to pick sites" | **No** |
| "OpenAI generates Cypher from study attrs + templates" | **Yes** — query authoring only |
| "The graph DB determines which sites match" | **Yes** |
| "Flexibility mode changes graph boolean logic" | **Yes** — strict AND / balanced anchor+OR / exploratory UNION |

The *AI* in **AI Suggestions** refers to **automated** graph-based discovery in the product UX, not to an LLM recommendation engine. Breakdown scoring and narrative insights use LLMs separately; the suggestion list does not.

---

## Product limitations

| Limitation | Detail |
|------------|--------|
| **Graph criteria required** | Study must have therapeutic area, MeSH conditions, countries, or regions. Empty criteria → zero suggestions. |
| **Capabilities sparse in graph** | Equipment matching runs in Postgres; excluded from count API to avoid misleading low counts. |
| **No PI / phase / enrollment in graph match** | Site suggestion matching uses TA, mesh, geography only — not full protocol field set. |
| **Fixed page size** | AI suggestions listing returns 10 sites per page. |
| **No semantic ranking** | Cannot sort suggestions by predicted fit; use breakdown for scored evaluation. |
| **OpenAI for Cypher only** | Requires `OPENAI_API_KEY` for Cypher generation on cache miss; graph execution is independent. |
| **8-hour Cypher cache** | Changing prompts requires bumping `PROMPT_VERSION` in `sites/services.py`. |
| **MeSH ID format** | Graph uses `mesh_id` on MeshTerm nodes, not Postgres condition UUIDs directly in Cypher. |

---

## Flexibility mode edge cases

### Invalid or missing mode

- Unknown `match_flexibility` values normalize to **strict**.
- Count API `?flexibility=invalid` is ignored; saved study mode is used.

### Strict with partial study data

Only populated criteria participate. A study with TA + US (no mesh) generates a strict query requiring **both** TA and US — not mesh.

### Balanced with single criterion

If only therapeutic area is populated, balanced degenerates to TA-only match (no secondary OR clauses).

### Exploratory with single criterion

Behaves like a single-branch match — equivalent to matching on that one dimension only.

### Count override vs saved mode

`?flexibility=exploratory` on count API previews exploratory totals without updating the study. The suggestions panel always uses the **saved** mode until `PATCH /flexibility/`.

### Study-only listing without ai_suggestions context

`GET /sites/?study={id}` (no `description`, no sidebar, no `context=ai_suggestions`) forces **strict** mode regardless of saved flexibility — by design, to stay aligned with the default count API path.

---

## Graph and data edge cases



### Strict mesh validation failure

Generated Cypher missing dual-path mesh in strict mode fails validation and triggers regeneration (up to 2 OpenAI attempts). Persistent failure → count API **502**, listing fail-soft empty.

### Neo4j syntax recovery

`match_site_uuids_for_study` retries with `_recover_cypher` on syntax errors. Listing path fail-softs to empty results instead.

### Spanner fallbacks

Balanced/exploratory modes on Spanner may use staged GQL fallbacks when single-query translation fails. Results should remain consistent but latency may increase.

### Region matching by name

Region criteria use **name strings**, not UUIDs, in Cypher. Renamed regions in ETL can break matches until graph is refreshed.

---

## AI suggestions panel edge cases

| Condition | Behavior |
|-----------|----------|
| `context=ai_suggestions` without `study` | **400** — study required |
| Invalid study ID | **404** |
| Sidebar specialization set | Study mesh terms cleared before graph match |
| Graph / Cypher failure | **200** with empty `_data` (fail-soft) |
| Cypher generation failure on count | **502** / **503** / **500** |

### Sidebar + flexibility interaction

Sidebar narrows attrs before Cypher generation; flexibility mode still controls AND/OR template shape on the resulting attr set.

---

## Provider configuration

| Variable | Default | Notes |
|----------|---------|-------|
| `GRAPH_DB_PROVIDER` | `neo4j` | General graph workloads |
| `SITE_STUDY_MATCH_GRAPH_PROVIDER` | falls back to `GRAPH_DB_PROVIDER` | Study match + AI suggestions |



---

## Testing reference

| Test area | Location |
|-----------|----------|
| Flexibility cache signature | `studies/tests.py` → `FlexibilityPromptTests` |
| Capability thresholds per mode | `studies/tests.py` |
| Count API flexibility override | `studies/tests.py` |
| Site listing merge (separate from suggestions) | `sites/tests.py` |

← [Back to Flexibility Modes & AI Suggestions](./README.md)
